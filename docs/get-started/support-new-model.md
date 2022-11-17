# How to support a new model

!!! todo
    Rename titles and focus on supporting new models

!!! todo
    Be consistent when using FX Graph / IR Graph

## How kernel optimize a model

### Overview

To optimize a model, kernl use [Torchdynamo](https://github.com/pytorch/torchdynamo) JIT compiler and provide a custom backend where we replace part of the [Torch FX](https://pytorch.org/docs/1.13/fx.html) graph with optimized kernels.

The custom backend is defined in [src/kernl/model_optimization.py](https://github.com/ELS-RD/kernl/blob/74d26c69cd3b12b0097804735ed0555123926503/src/kernl/model_optimization.py#L31)

``` { .py }
def _compiler(gm: torch.fx.GraphModule, example_inputs: List[torch.Tensor]):
    dynamo_backend_ofi(gm)
    return cuda_graphs_wrapper(gm, example_inputs, pool=_pool)
```

This backend combines two steps:
- First one is to apply the graph replacements
- Second one is to use [CUDA graphs](https://pytorch.org/blog/accelerating-pytorch-with-cuda-graphs/)

We won't elaborate the second step and focus on the first one that actually does the graph replacements, `#!python dynamo_backend_ofi(gm)` defined in [src/kernl/optimizer/dynamo_backend.py](https://github.com/ELS-RD/kernl/blob/1463e39aeaaa49535c4a9144c4ab96c6a7a15eca/src/kernl/optimizer/dynamo_backend.py#L25)

There is two ways to modify the FX graph. One is to [directly modify the FX graph](https://pytorch.org/docs/1.13/fx.html#direct-graph-manipulation). The other is to use a subgraph rewriting function. We'll deep dive on the second one.

### Replace part of the FX graph

To replace part of the FX graph, kernl use the `#!python repace_pattern()` function defined in [src/kernl/utils/extended_matcher.py](https://github.com/ELS-RD/kernl/blob/1463e39aeaaa49535c4a9144c4ab96c6a7a15eca/src/kernl/utils/extended_matcher.py#L337). It's the same function `#!python repace_pattern()` of [Torch FX](https://pytorch.org/docs/1.13/fx.html#subgraph-rewriting-with-replace-pattern) but with some bugfixes (that should be integrated in PyTorch in the future).

``` { .py }
def replace_pattern(gm: GraphModule, pattern: Callable, replacement: Callable) -> List[Match]:
```

The function takes a graph `gm` and two callables `pattern` and `replacement` that can be either a torch module or a function. It'll convert `pattern` and `replacement` to an FX graph and try to replace subgraphs from `gm` matching `pattern` with `replacement`.

 pattern can be defined as follow:

!!! todo 
    Add better examples and explanations

=== "Module"

	``` { .py }
	class Pattern(torch.nn.Module):
		def __init__(self):
			super().__init__()
			self.linear = torch.nn.Linear(1, 1)
			self.activation = torch.nn.Tanh()

		def forward(self, v):
			return self.activation(self.linear(v))

	class Replacement(torch.nn.Module):
		def __init__(self):
			super().__init__()
			self.linear = torch.nn.Linear(1, 1)
			self.activation = torch.nn.ReLU()

		def forward(self, v):
			return self.activation(self.linear(v))

	replace_pattern(gm, Pattern(), Replacement())
	```

=== "Function"

	``` { .py }
	def pattern(w1, w2):
		return torch.cat([w1, w2]).sum()

	def replacement(w1, w2):
		return torch.stack([w1, w2])

	replace_pattern(gm, pattern, replacement)
	```

When using a non-torch function a the subgraph, you'll have to use [Torch wrap function](https://pytorch.org/docs/1.13/fx.html#non-torch-functions) in order to appears in the FX graph but not to be traced

``` { .py }
torch.fx.wrap(fn)
```

### Reading the FX Graph

!!! todo
    introduce `#!python graph_report(gm)` and `#!python gm.code`

https://pytorch.org/docs/1.13/fx.html#torch.fx.Graph.print_tabular


## Example: replacing BERT Attention

In this example, we'll see how to replace the attention part of a BERT model with kernl's optimized attention kernel.

### Understanding Attention

First of all, we need to look how attention actually works, the [original paper](https://arxiv.org/abs/1706.03762) "Attention Is All You Need" is a good starting point. More specifically, we'll focus on the Attention part where the attention function is defined:

!!! quote "Attention Is All You Need"
    An attention function can be described as mapping a query and a set of key-value pairs to an output,
    where the query, keys, values, and output are all vectors. The output is computed as a weighted sum
    of the values, where the weight assigned to each value is computed by a compatibility function of the
    query with the corresponding key.

    (...)

	We call our particular attention "Scaled Dot-Product Attention". The input consists of
	queries and keys of dimension $d_k$, and values of dimension $d_v$ . We compute the dot products of the
	query with all keys, divide each by $\sqrt{d_k}$, and apply a softmax function to obtain the weights on the
	values.
	In practice, we compute the attention function on a set of queries simultaneously, packed together
	into a matrix $Q$. The keys and values are also packed together into matrices $K$ and $V$ . We compute
	the matrix of outputs as:

	$$
	\operatorname{Attention}(Q,K,V)=\operatorname{softmax}(\frac{QK^T}{\sqrt{d_k}})V
	$$

This function with can be represented as a computation graph where the attention mask is added in the process:


<figure markdown>
  ![Scaled Dot-Product Attention](attention_original.png){ width="150"; lazyload=true }
  <figcaption>Scaled Dot-Product Attention</figcaption>
</figure>

This graph representation will be useful as it is this graph we'll try to replace to optimize a BERT model.

### Find the Attention graph pattern

For our example, we'll try to replace the attention part from the "bert-base-uncased" pre-trained model from [Hugging Face Transformers](https://huggingface.co/transformers). If we look at the [BERT implementation](https://github.com/huggingface/transformers/blob/main/src/transformers/models/bert/modeling_bert.py), we find the attention function as a torch module

=== "Code Excerpt"

	```{ .py .annotate } 
	class BertSelfAttention(nn.Module):
		def forward(
			self,
			hidden_states: torch.Tensor,
			attention_mask: Optional[torch.FloatTensor] = None,
			head_mask: Optional[torch.FloatTensor] = None,
			encoder_hidden_states: Optional[torch.FloatTensor] = None,
			encoder_attention_mask: Optional[torch.FloatTensor] = None,
			past_key_value: Optional[Tuple[Tuple[torch.FloatTensor]]] = None,
			output_attentions: Optional[bool] = False,
		) -> Tuple[torch.Tensor]:
			...
			# Take the dot product between "query" and "key" to get the raw attention scores. # (1)
			attention_scores = torch.matmul(query_layer, key_layer.transpose(-1, -2))
			...
			attention_scores = attention_scores / math.sqrt(self.attention_head_size) # (2)
			if attention_mask is not None:
				# Apply the attention mask is (precomputed for all layers in BertModel forward() function)
				attention_scores = attention_scores + attention_mask

			# Normalize the attention scores to probabilities. # (3)
			attention_probs = nn.functional.softmax(attention_scores, dim=-1)
			...
			context_layer = torch.matmul(attention_probs, value_layer) # (4)
			....
	```

	1.  $QK^T$
	2.  $\frac{QK^T}{\sqrt{d_k}}$
	3.  $\operatorname{softmax}(\frac{QK^T}{\sqrt{d_k}})$
	4.  $\operatorname{softmax}(\frac{QK^T}{\sqrt{d_k}})V$

=== "Full Code"

	```{ .py .annotate hl_lines="50 51 75 76 77 78 79 80 81 91" }
	class BertSelfAttention(nn.Module):
		def forward(
			self,
			hidden_states: torch.Tensor,
			attention_mask: Optional[torch.FloatTensor] = None,
			head_mask: Optional[torch.FloatTensor] = None,
			encoder_hidden_states: Optional[torch.FloatTensor] = None,
			encoder_attention_mask: Optional[torch.FloatTensor] = None,
			past_key_value: Optional[Tuple[Tuple[torch.FloatTensor]]] = None,
			output_attentions: Optional[bool] = False,
		) -> Tuple[torch.Tensor]:
			mixed_query_layer = self.query(hidden_states)

			# If this is instantiated as a cross-attention module, the keys
			# and values come from an encoder; the attention mask needs to be
			# such that the encoder's padding tokens are not attended to.
			is_cross_attention = encoder_hidden_states is not None

			if is_cross_attention and past_key_value is not None:
				# reuse k,v, cross_attentions
				key_layer = past_key_value[0]
				value_layer = past_key_value[1]
				attention_mask = encoder_attention_mask
			elif is_cross_attention:
				key_layer = self.transpose_for_scores(self.key(encoder_hidden_states))
				value_layer = self.transpose_for_scores(self.value(encoder_hidden_states))
				attention_mask = encoder_attention_mask
			elif past_key_value is not None:
				key_layer = self.transpose_for_scores(self.key(hidden_states))
				value_layer = self.transpose_for_scores(self.value(hidden_states))
				key_layer = torch.cat([past_key_value[0], key_layer], dim=2)
				value_layer = torch.cat([past_key_value[1], value_layer], dim=2)
			else:
				key_layer = self.transpose_for_scores(self.key(hidden_states))
				value_layer = self.transpose_for_scores(self.value(hidden_states))

			query_layer = self.transpose_for_scores(mixed_query_layer)

			use_cache = past_key_value is not None
			if self.is_decoder:
				# if cross_attention save Tuple(torch.Tensor, torch.Tensor) of all cross attention key/value_states.
				# Further calls to cross_attention layer can then reuse all cross-attention
				# key/value_states (first "if" case)
				# if uni-directional self-attention (decoder) save Tuple(torch.Tensor, torch.Tensor) of
				# all previous decoder key/value_states. Further calls to uni-directional self-attention
				# can concat previous decoder key/value_states to current projected key/value_states (third "elif" case)
				# if encoder bi-directional self-attention `past_key_value` is always `None`
				past_key_value = (key_layer, value_layer)

			# Take the dot product between "query" and "key" to get the raw attention scores. # (2)
			attention_scores = torch.matmul(query_layer, key_layer.transpose(-1, -2))

			if self.position_embedding_type == "relative_key" or self.position_embedding_type == "relative_key_query":
				query_length, key_length = query_layer.shape[2], key_layer.shape[2]
				if use_cache:
					position_ids_l = torch.tensor(key_length - 1, dtype=torch.long, device=hidden_states.device).view(
						-1, 1
					)
				else:
					position_ids_l = torch.arange(query_length, dtype=torch.long, device=hidden_states.device).view(-1, 1)
				position_ids_r = torch.arange(key_length, dtype=torch.long, device=hidden_states.device).view(1, -1)
				distance = position_ids_l - position_ids_r

				positional_embedding = self.distance_embedding(distance + self.max_position_embeddings - 1)
				positional_embedding = positional_embedding.to(dtype=query_layer.dtype)  # fp16 compatibility

				if self.position_embedding_type == "relative_key":
					relative_position_scores = torch.einsum("bhld,lrd->bhlr", query_layer, positional_embedding)
					attention_scores = attention_scores + relative_position_scores
				elif self.position_embedding_type == "relative_key_query":
					relative_position_scores_query = torch.einsum("bhld,lrd->bhlr", query_layer, positional_embedding)
					relative_position_scores_key = torch.einsum("bhrd,lrd->bhlr", key_layer, positional_embedding)
					attention_scores = attention_scores + relative_position_scores_query + relative_position_scores_key

			attention_scores = attention_scores / math.sqrt(self.attention_head_size) # (2)
			if attention_mask is not None:
				# Apply the attention mask is (precomputed for all layers in BertModel forward() function)
				attention_scores = attention_scores + attention_mask

			# Normalize the attention scores to probabilities. # (3)
			attention_probs = nn.functional.softmax(attention_scores, dim=-1)

			# This is actually dropping out entire tokens to attend to, which might
			# seem a bit unusual, but is taken from the original Transformer paper.
			attention_probs = self.dropout(attention_probs)

			# Mask heads if we want to
			if head_mask is not None:
				attention_probs = attention_probs * head_mask

			context_layer = torch.matmul(attention_probs, value_layer) # (4)

			context_layer = context_layer.permute(0, 2, 1, 3).contiguous()
			new_context_layer_shape = context_layer.size()[:-2] + (self.all_head_size,)
			context_layer = context_layer.view(new_context_layer_shape)

			outputs = (context_layer, attention_probs) if output_attentions else (context_layer,)

			if self.is_decoder:
				outputs = outputs + (past_key_value,)
			return outputs
	```

	1.  $QK^T$
	2.  $\frac{QK^T}{\sqrt{d_k}}$
	3.  $\operatorname{softmax}(\frac{QK^T}{\sqrt{d_k}})$
	4.  $\operatorname{softmax}(\frac{QK^T}{\sqrt{d_k}})V$

We see that the Hugging Face implementation is close to the definition in the paper, we can actually want to find the attention pattern in this model.

If we run `#!python graph_report(gm)` and `#!python gm.code` in the `#!python dynamo_backend_ofi(gm)` function, we can print the graph and the python code from the IR of the model computation. For our example, we'll keep the `normalize_operators` and the `remove_dropout` functions as it simplifies a bit the model IR.

``` { .py }
def dynamo_backend_ofi(gm: torch.fx.GraphModule, assume_causal=False):
    normalize_operators(gm)
    remove_dropout(gm)
    print(graph_report(gm))
    print(gm.code)
    return gm
```

Below is the resulting output, we only shows the beggining of the graph until the first attention layer.

???+ example "Intermediate Representation of BERT attention function"

	=== "`#!python graph_report(gm)`"

		```
		   opcode         name                                               target                                                     args                                                                                     kwargs
		-------------  -------------------------------------------------  ---------------------------------------------------------  ---------------------------------------------------------------------------------------  ------------------------
		placeholder    input_ids                                          input_ids                                                  ()                                                                                       {}
		placeholder    attention_mask                                     attention_mask                                             ()                                                                                       {}
		get_attr       self_embeddings_token_type_ids                     self_embeddings_token_type_ids                             ()                                                                                       {}
		call_function  getitem                                            <built-in function getitem>                                (self_embeddings_token_type_ids, (slice(None, None, None), slice(None, 128, None)))      {}
		call_method    expand                                             expand                                                     (getitem, 1, 128)                                                                        {}
		call_function  getitem_1                                          <built-in function getitem>                                (attention_mask, (slice(None, None, None), None, None, slice(None, None, None)))         {}
		call_method    to                                                 to                                                         (getitem_1,)                                                                             {'dtype': torch.float32}
		call_function  sub                                                <built-in function sub>                                    (1.0, to)                                                                                {}
		call_function  mul                                                <built-in function mul>                                    (sub, -3.4028234663852886e+38)                                                           {}
		get_attr       self_embeddings_position_ids                       self_embeddings_position_ids                               ()                                                                                       {}
		call_function  getitem_2                                          <built-in function getitem>                                (self_embeddings_position_ids, (slice(None, None, None), slice(0, 128, None)))           {}
		call_module    self_embeddings_word_embeddings                    self_embeddings_word_embeddings                            (input_ids,)                                                                             {}
		call_module    self_embeddings_token_type_embeddings              self_embeddings_token_type_embeddings                      (expand,)                                                                                {}
		call_function  add                                                <built-in function add>                                    (self_embeddings_word_embeddings, self_embeddings_token_type_embeddings)                 {}
		call_module    self_embeddings_position_embeddings                self_embeddings_position_embeddings                        (getitem_2,)                                                                             {}
		call_function  add_37                                             <built-in method add of type object at 0x7f065046e4e0>     (add, self_embeddings_position_embeddings)                                               {}
		call_module    self_embeddings_layer_norm                         self_embeddings_LayerNorm                                  (add_37,)                                                                                {}
		call_module    self_encoder_layer_0_attention_self_query          self_encoder_layer_0_attention_self_query                  (self_embeddings_layer_norm,)                                                            {}
		call_module    self_encoder_layer_0_attention_self_key            self_encoder_layer_0_attention_self_key                    (self_embeddings_layer_norm,)                                                            {}
		call_method    view                                               view                                                       (self_encoder_layer_0_attention_self_key, (1, 128, 12, 64))                              {}
		call_method    permute                                            permute                                                    (view, 0, 2, 1, 3)                                                                       {}
		call_module    self_encoder_layer_0_attention_self_value          self_encoder_layer_0_attention_self_value                  (self_embeddings_layer_norm,)                                                            {}
		call_method    view_1                                             view                                                       (self_encoder_layer_0_attention_self_value, (1, 128, 12, 64))                            {}
		call_method    permute_1                                          permute                                                    (view_1, 0, 2, 1, 3)                                                                     {}
		call_method    view_2                                             view                                                       (self_encoder_layer_0_attention_self_query, (1, 128, 12, 64))                            {}
		call_method    permute_2                                          permute                                                    (view_2, 0, 2, 1, 3)                                                                     {}
		call_method    transpose                                          transpose                                                  (permute, -1, -2)                                                                        {}
		call_function  matmul                                             <built-in method matmul of type object at 0x7f065046e4e0>  (permute_2, transpose)                                                                   {}
		call_function  truediv                                            <built-in function truediv>                                (matmul, 8.0)                                                                            {}
		call_function  add_1                                              <built-in function add>                                    (truediv, mul)                                                                           {}
		call_function  softmax                                            <function softmax at 0x7f05eca5f790>                       (add_1,)                                                                                 {'dim': -1}
		call_function  matmul_1                                           <built-in method matmul of type object at 0x7f065046e4e0>  (softmax, permute_1)                                                                     {}
		call_method    permute_3                                          permute                                                    (matmul_1, 0, 2, 1, 3)                                                                   {}
		call_method    contiguous                                         contiguous                                                 (permute_3,)                                                                             {}
		call_method    view_3                                             view                                                       (contiguous, (1, 128, 768))                                                              {}
		call_module    self_encoder_layer_0_attention_output_dense        self_encoder_layer_0_attention_output_dense                (view_3,)                                                                                {}
		```

	=== "`#!python gm.code`"

		```{ .py }
		def forward(self, input_ids : torch.Tensor, attention_mask : torch.Tensor):
			self_embeddings_token_type_ids = self.self_embeddings_token_type_ids
			getitem = self_embeddings_token_type_ids[(slice(None, None, None), slice(None, 128, None))];  self_embeddings_token_type_ids = None
			expand = getitem.expand(1, 128);  getitem = None
			getitem_1 = attention_mask[(slice(None, None, None), None, None, slice(None, None, None))];  attention_mask = None
			to = getitem_1.to(dtype = torch.float32);  getitem_1 = None
			sub = 1.0 - to;  to = None
			mul = sub * -3.4028234663852886e+38;  sub = None
			self_embeddings_position_ids = self.self_embeddings_position_ids
			getitem_2 = self_embeddings_position_ids[(slice(None, None, None), slice(0, 128, None))];  self_embeddings_position_ids = None
			self_embeddings_word_embeddings = self.self_embeddings_word_embeddings(input_ids);  input_ids = None
			self_embeddings_token_type_embeddings = self.self_embeddings_token_type_embeddings(expand);  expand = None
			add = self_embeddings_word_embeddings + self_embeddings_token_type_embeddings;  self_embeddings_word_embeddings = self_embeddings_token_type_embeddings = None
			self_embeddings_position_embeddings = self.self_embeddings_position_embeddings(getitem_2);  getitem_2 = None
			add_37 = torch.add(add, self_embeddings_position_embeddings);  add = self_embeddings_position_embeddings = None
			self_embeddings_layer_norm = self.self_embeddings_LayerNorm(add_37);  add_37 = None
			self_encoder_layer_0_attention_self_query = self.self_encoder_layer_0_attention_self_query(self_embeddings_layer_norm)
			self_encoder_layer_0_attention_self_key = self.self_encoder_layer_0_attention_self_key(self_embeddings_layer_norm)
			view = self_encoder_layer_0_attention_self_key.view((1, 128, 12, 64));  self_encoder_layer_0_attention_self_key = None
			permute = view.permute(0, 2, 1, 3);  view = None
			self_encoder_layer_0_attention_self_value = self.self_encoder_layer_0_attention_self_value(self_embeddings_layer_norm)
			view_1 = self_encoder_layer_0_attention_self_value.view((1, 128, 12, 64));  self_encoder_layer_0_attention_self_value = None
			permute_1 = view_1.permute(0, 2, 1, 3);  view_1 = None
			view_2 = self_encoder_layer_0_attention_self_query.view((1, 128, 12, 64));  self_encoder_layer_0_attention_self_query = None
			permute_2 = view_2.permute(0, 2, 1, 3);  view_2 = None
			transpose = permute.transpose(-1, -2);  permute = None
			matmul = torch.matmul(permute_2, transpose);  permute_2 = transpose = None
			truediv = matmul / 8.0;  matmul = None
			add_1 = truediv + mul;  truediv = None
			softmax = torch.nn.functional.softmax(add_1, dim = -1);  add_1 = None
			matmul_1 = torch.matmul(softmax, permute_1);  softmax = permute_1 = None
			permute_3 = matmul_1.permute(0, 2, 1, 3);  matmul_1 = None
			contiguous = permute_3.contiguous();  permute_3 = None
			view_3 = contiguous.view((1, 128, 768));  contiguous = None
			self_encoder_layer_0_attention_output_dense = self.self_encoder_layer_0_attention_output_dense(view_3);  view_3 = None
    	```

If we draw the graph corresponding to this IR representation, we can identify in yellow the attention part:

!!! todo
    Fix this figure as it doesn't appear in mkdocs

<figure markdown>
    ![Attention in BERT IR Graph](attention.drawio.svg){ lazyload=true }
    <figcaption>Attention in BERT IR Graph</figcaption>
</figure>

Now, we'll find in the IR code which lines correspond to these nodes.


``` { .py }
	transpose = permute.transpose(-1, -2);  permute = None
	matmul = torch.matmul(permute_2, transpose);  permute_2 = transpose = None
	truediv = matmul / 8.0;  matmul = None
	add_1 = truediv + mul;  truediv = None
	softmax = torch.nn.functional.softmax(add_1, dim = -1);  add_1 = None
	matmul_1 = torch.matmul(softmax, permute_1);  softmax = permute_1 = None

```

We now have our pattern to catch in the model, to make the pattern easier to read, we rename the following nodes:

- `permute` -> `k`
- `permute_1` -> `v`
- `permute_2` -> `q`
- `mul` -> `attention_mask`

and can write our pattern function:


``` { .py }
def pattern(q, k, attention_mask, v):
	transpose = k.transpose(-1, -2)
	matmul = torch.matmul(q, transpose)
	truediv = matmul / 8.0
	add_1 = truediv + attention_mask
	softmax = torch.nn.functional.softmax(add_1, dim=-1)
	matmul_1 = torch.matmul(softmax, v)
	return matmul_1
```

### Replace the Attention part

We now needs to add our replace function to call the optimized kernel. We can see in [kernl/model_optimization.py](https://github.com/ELS-RD/kernl/blob/9c3ec9b3c03609c20ca9a00eda7a58d4769bf47c/src/kernl/implementations/attention.py#L492) the optimized attention kernel needs in addition to `q`, `k`, `v` and `attention_mask`, the `output` and the `sm_scale` parameter.

``` { .py }
def attention_forward(
    q: torch.Tensor,
    k: torch.Tensor,
    v: torch.Tensor,
    output: torch.Tensor,
    sm_scale: float,
    is_causal: bool = False,
    attention_mask: Optional[torch.Tensor] = None,
):
```

The `output` parameter is simply the resulting tensor. We need to provide the tensor beforehand.

The `sm_scale` parameter corresponds to the scale factor applied to the query-key compatibility function in the attention function. Defined as $\frac{1}{\sqrt{d_k}}$, it correspond to the `true_div` node in the IR graph. In this case `sm_scale` is $\frac{1}{8.0}$.

We can now write our replacement part by calling the optimized kernel:

!!! todo
    Verify replacement works this way

``` { .py }
@torch.fx.wrap(attention_forward)

def replace(q, k, attention_mask, v):
	output = torch.empty_like(q)
	output = attention_forward(q, k, v, output, 1 /8.0, is_causal=False, attention_mask=attention_mask)
	return output
```

!! todo
    Add the part in dynamo_backend to call the new replacement

If we print again the IR graph after the graph replacement, we see that's all the previous nodes from the attention part is now replaced by the call to the optimized kernel.

!!! todo
    Fix attention function name

=== "Attention in BERT IR Graph"

	<figure markdown>
		![Attention in BERT IR Graph](attention.drawio.svg){ lazyload=true }
		<figcaption>Attention in BERT IR Graph</figcaption>
	</figure>


=== "Attention replaced by an optimized kernel in BERT IR Graph"

	<figure markdown>
		![Attention replaced by a kernel in BERT IR Graph](attention_fused.drawio.svg){ lazyload=true }
		<figcaption>Attention replaced by a kernel in BERT IR Graph</figcaption>
	</figure>