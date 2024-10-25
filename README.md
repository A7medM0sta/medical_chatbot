# What is Llama 3?
Meta launched Llama 3, the latest in its Llama series of open-source AI models. Llama 3 comes in two variants: one with 8 billion parameters and another with 70 billion parameters.
Meta claims that Llama 3 sets a new standard for large language models at these parameter scales. They have improved pretraining and post-training processes, resulting in reduced false refusal rates, better alignment, and more diverse responses from the model. Notably, Llama 3 boasts enhanced capabilities in reasoning, code generation, and instruction following.
<img src="assets/meta_com.webp" height="500" width="1000">
## Technical Specs
LLaMA Architecture :
The key difference between the predecessors models is, the size of the pretraining corpus increased by 650% LLaMA — 2 was trained on 2T tokens where as LLaMA — 3 trained on 15T tokens, doubled the context length of the model from 4K to 8K on both 8B and 70B models, and adopted grouped-query attention for both 8B and 70B variant as compared to the previous generation (GQA) was only used in bigger models 34B and 70B. The most impactful part I felt was new approach to safety with two rewards models for Safety and Helpfulness.
LLaMA 3 imbibes it’s Architecture from its previous generation models
LLaMA 3 :
Model size, Architectures, optimization hyper-parameters

<center> <img src="assets/parameter.webp" width="300"  height="300"> </center>

LLaMA 2 :
<center><img src="assets/paramter_2.webp" width="300" height="300"> </center>
LLaMA 1 :
<center><img src="assets/llama1.webp" width="300" height="300"> </center>



LLaMA Architecture:
<center><img src="assets/archi.webp" width="500" height="1000"> </center>


LLaMA 3 Architecture mostly imbibes the same architecture as to LLaMA 2 with GQA ( Grouped Query Attention) being used in both the 8B and 70B model, RoPE (Rotary Positional Embeddings) used for Q, K as the V is only multiplied before applying the SoftMax Function, RMS (Root mean squared Error) is used for Normalizing applied before Self Attention , Feedforward Block, KV Cache also remains the same used in LLMA
Note: This model architecture is only focused on the Inferencing the model not for training so the Decoder Block with Cross Attention is not being covered her also the KV Cache will not be used for Training Phase of the model.
Now let’s delve deeper into understanding each of these components.


## RoPE (Rotary Positional Encoding)
Before diving into RoPE it’s Important to understand what’s the difference between the absolute positional encodings and relative encoding.
Absolute positional encodings are fixed vectors that are added to the embedding of a token to represent its absolute position in the sentence. So, it deals with one token at a time. You can think of it as the pair (latitude, longitude) on a map: each point on earth will have a unique pair.
Relative positional encodings, on the other hand, deals with two tokens at a time and it is involved when we calculate the attention: since the attention mechanism captures the “intensity” of how much two words are related two each other, relative positional encodings tells the attention mechanism the distance between the to words involved in it. So, given two tokens, we create a vector that represents their distance.
Rotoary Positional Encoding can be considered as an midground between Absolute Positional Embeddings and Relative Positional Embeddings as each token do have a fixed or an absolute embedding value and is multiplied with an inner dot product with its polar form which is relative to the rotation of the vectors on the 2D plane.
The dot product used in the attention mechanism is a type of inner product, which can be through of as a generalization of the dot product.
Can we find an inner product over the two vectors q (query) and k (key) used in the attention mechanism that only depends on the two vectors and the relative distance of the token they represent?

<center><img src="assets/rpe.webp" width="300" height="300"> </center>

* We can define a function g like the following that only depends on the on the two embeddings vector q and k and their relative distance.
<center><img src="assets/1.webp" width="300" height="300"> </center>
Using Euler’s formula, we can write it into its matrix form.
<center><img src="assets/2.webp" width="300" height="300"> </center>
Rotation matrix in a 2d space, hence the name rotary position embeddings
<center><img src="assets/3.webp" width="300" height="300"> </center>


* The rotary position embeddings are only applied to the query and the keys, but not the values.
* The rotary position embeddings are applied after the vector q and k have been multiplied by the W matrix in the attention mechanism, while in the vanilla transformer they’re applied before.

```python
def precomputed_theta_pos_frequencies(head_dim: int, seq_len: int, device: str, theta: float = 10000.0):
    # As written in the paper, the dimentions o the embedding must be even
    assert head_dim % 2 == 0, "The head_dim must be even"
    # Built the theta parameters
    # According to the formula theta_i = 10000 ^ (-2(i-1)/dim) for i = [1,2,3,..dim/2]
    # Shape: (head_dim / 2)
    theta_numerator = torch.arange(0, head_dim, 2).float()
    # Shape : (head_dim / 2)
    theta = 1.0 / (theta ** (theta_numerator / head_dim)).to(device)
    # Construct the positions (the "m" parameter)
    # shape: (seq_len)
    m = torch.arange(seq_len, device=device)
    # multiply each theta by each position using the outer product
    # shape : (seq_len) outer_product * (head_dim / 2) -> (seq_len, head_dim / 2)
    freq = torch.outer(m, theta).float()
    # we can computer complex numbers in the polar form c = R * exp(i * m * theta), where R = 1 as follow
    # shape: (seq_len, head_dim/2) -> (seq-len, head_dim/2)
    freq_complex = torch.polar(torch.ones_like(freq), freq)
    return freq_complex

def apply_rotary_embeddings(x: torch.Tensor, freq_complex: torch.Tensor, device: str):
    # We transform the each subsequent pair of tokens into a pair of complex numbers
    # shape : (B, seq_len, head_dim) -> (B, seq_len, h, head_dim / 2)
    x_complex = torch.view_as_complex(x.float().reshape(*x.shape[:-1], -1, 2))
    # shape : (seq_len, head_dim / 2) -> (1, seq_len, 1, head_dim / 2)
    freq_complex = freq_complex.unsqueeze(0).unsqueeze(2)
    # shape : (B, seq_len, h, head_dim / 2) * (1, seq_len, 1, head_dim / 2) = (B, seq_len, h, head_dim / 2)
    x_rotate = x_complex * freq_complex
    # (B, seq_len, h, head_dim / 2) -> (B, seq_len, h, head_dim/2 ,2)
    x_out = torch.view_as_real(x_rotate)
    # (B, seq_len, h, head_dim/2, 2) -> (B, seq_len, h * head_dim / 2 * 2)
    x_out = x_out.reshape(*x.shape)
    return x_out.type_as(x).to(device)
```

## KV Cache
* At every step of the inference, we are only interested in the last token output by the model, because we already have the previous ones. However, the model needs access to all the previous tokens to decide on which token to output, since they constitute its context (or the “prompt”).
* It’s a way to make the model do less computation on the token it has already seen during inference. The solution is the KV cache!

<img src="assets/kache.webp" width="300" height="300"> 

```python
    def repeat_kv(x: torch.Tensor, n_rep: int)-> torch.Tensor:
        batch_size, seq_len, n_kv_heads, head_dim = x.shape
        if n_rep == 1:
            return x
        else:
            return (
            # (B, seq_len, n_kv_heads, 1, head_dim)
            x[:, :, :, None, :]
            .expand(batch_size, seq_len, n_kv_heads, n_rep, head_dim)
            .reshape(batch_size, seq_len, n_kv_heads * n_rep, head_dim)
            )
```

## Grouped Multi-Query Attention
* Grouped-query attention(GQA) is an interpolation of multi-query and multi-head attention. It achieves a quality similar to multi-head attention while maintaining a comparable speed to multi-query attention. The standard practice for autoregressive decoding is to cache the keys and values of the previous tokens in the sequence to speed up attention computation. However, as the context window or batch size increases, the memory cost associated with the size of the key-value cache(kv cache) in the multi-head attention(MHA) model significantly increases. Multi-Query attention(MQA) is a mechanism that uses only a single key-value head for multiple queries, which can save memory and greatly speed up decoder inference. Llama incorporates the (GQA) to address the memory bandwidth challenges during the autoregressive decoding of Transformer models. The primary issue stems from GPU does all the computations faster than they can move them into memory. The need to load the decoder weights and attention keys at each stage which consumes excessive memory.
<img src="assets/qqa.webp">
<img src="assets/qqq_2.webp">

```python
class SelfAttention(nn.Module): 
    def  __init__(self, args: ModelArgs):
        super().__init__()
        self.n_kv_heads = args.n_heads if args.n_kv_heads is None else args.n_kv_heads
        # Indicates the number of heads for the queries
        self.n_heads_q = args.n_heads
        # Indiates how many times the heads of keys and value should be repeated to match the head of the Query
        self.n_rep = self.n_heads_q // self.n_kv_heads
        # Indicates the dimentiona of each head
        self.head_dim = args.dim // args.n_heads

        self.wq = nn.Linear(args.dim, args.n_heads * self.head_dim, bias=False)
        self.wk = nn.Linear(args.dim, self.n_kv_heads * self.head_dim, bias=False)
        self.wv = nn.Linear(args.dim, self.n_kv_heads * self.head_dim, bias=False)
        self.wo = nn.Linear(args.n_heads * self.head_dim, args.dim, bias=False)

        self.cache_k = torch.zeros((args.max_batch_size, args.max_seq_len, self.n_kv_heads, self.head_dim))
        self.cache_v = torch.zeros((args.max_batch_size, args.max_seq_len, self.n_kv_heads, self.head_dim))

    def forward(self, x: torch.Tensor, start_pos: int, freq_complex: torch.Tensor):
        batch_size, seq_len, _ = x.shape #(B, 1, dim)
        # Apply the wq, wk, wv matrices to query, key and value
        # (B, 1, dim) -> (B, 1, H_q * head_dim)
        xq = self.wq(x)
        # (B, 1, dim) -> (B, 1, H_kv * head_dim)
        xk = self.wk(x)
        xv = self.wv(x)

        # (B, 1, H_q * head_dim) -> (B, 1, H_q, head_dim)
        xq = xq.view(batch_size, seq_len, self.n_heads_q, self.head_dim)
        xk = xk.view(batch_size, seq_len, self.n_kv_heads, self.head_dim)
        # (B, 1, H_kv * head_dim) -> (B, 1, H_kv, head_dim)
        xv = xv.view(batch_size, seq_len, self.n_kv_heads, self.head_dim)

        # Apply the rotary embeddings to the keys and values
        # Does not chnage the shape of the tensor
        # (B, 1, H_kv, head_dim) -> (B, 1, H_kv, head_dim)
        xq = apply_rotary_embeddings(xq, freq_complex, device=x.device)
        xk = apply_rotary_embeddings(xk, freq_complex, device=x.device)

        # Replace the enty in the cache for this token
        self.cache_k[:batch_size, start_pos:start_pos + seq_len] = xk
        self.cache_v[:batch_size, start_pos:start_pos + seq_len] = xv

        # Retrive all the cached keys and values so far
        # (B, seq_len_kv, H_kv, head_dim)
        keys = self.cache_k[:batch_size, 0:start_pos + seq_len]
        values = self.cache_v[:batch_size, 0:start_pos+seq_len] 

        # Repeat the heads of the K and V to reach the number of heads of the queries
        keys = repeat_kv(keys, self.n_rep)
        values = repeat_kv(values, self.n_rep)

        # (B, 1, h_q, head_dim) --> (b, h_q, 1, head_dim)
        xq = xq.transpose(1, 2)
        keys = keys.transpose(1, 2)
        values = values.transpose(1, 2)

        # (B, h_q, 1, head_dim) @ (B, h_kv, seq_len-kv, head_dim) -> (B, h_q, 1, seq_len-kv)
        scores = torch.matmul(xq, keys.transpose(2,3)) / math.sqrt(self.head_dim)
        scores = F.softmax(scores.float(), dim=-1).type_as(xq)

        # (B, h_q, 1, seq_len) @ (B, h_q, seq_len-kv, head_dim) --> (b, h-q, q, head_dim)
        output = torch.matmul(scores, values)

        # (B, h_q, 1, head_dim) -> (B, 1, h_q, head_dim) -> ()
        output = (output.transpose(1,2).contiguous().view(batch_size, seq_len, -1))
        return self.wo(output) # (B, 1, dim) -> (B, 1, dim)

```
## RMS (Root Mean Squared Normalization)
Root Mean Square Normalization (RMSNorm) is a relatively novel normalization technique introduced by Biao Zhang, Rico Sennrich in 2019. Unlike BN and LN, RMSNorm normalizes activations based on the root mean square of the activations themselves, rather than using mini-batch or layer statistics. This approach ensures that the activations are consistently scaled regardless of the mini-batch size or the number of features. Additionally, RMSNorm introduces learnable scale parameters,
offering similar adaptability to Batch Normalization.
<img src="assets/rmsn.webp">
Note: Just like Layer Normalization, we also have a learnable parameter gamma (g in the formula on the left) that is multiplied by the normalized values.

```python
class RMSNorm(nn.Module):
    def __init__(self, dim: int, eps: float = 1e-5):
        super().__init__()
        self.eps = eps
        # The gamma parameter
        self.weight = nn.Parameter(torch.ones(dim))

    def _norm(self, x: torch.Tensor):
        # (B, seq_len, dim) -> (B, seq_len, 1)
        return x * torch.rsqrt(x.pow(2).mean(-1, keepdim=True) + self.eps)

    def forward(self, x: torch.Tensor):
        # dim : (B, seq_len, dim) -> (B, seq_len, dim)
        return self.weight * self._norm(x.float()).type_as(x)
```

## SwiGLU Activation Function
SwiGLU is an activation function used in deep neural networks that is a variant of GLU (Gated Linear Unit). It is used to calculate the output of a neuron in a neural network by taking in the weighted sum of the input and applying a non-linear function to it. SwiGLU is defined using a mathematical expression that involves the Swish function and tensor multiplication. SwiGLU is a variant of GLU, which means that it is based on the same mathematical concept as GLU. However, SwiGLU has a different non-linear function than GLU. Specifically, SwiGLU uses the Swish function, which is a recently proposed activation function that has been shown to outperform other activation functions in some applications.
SwiGLU has several benefits that make it a useful activation function in neural networks. First, it is based on the GLU concept, which has been shown to perform well in many applications. Second, it uses the Swish function, which has been shown to outperform other activation functions in some cases, particularly when combined with residual connections. Third, it allows for efficient computation due to its use of element-wise multiplication.
<img src="assets/swin.webp">
The author compared the performance of a Transformer model by using different activation functions in the Feed-Forward layer of the Transformer architecture.
<img src="assets/comapare_active.webp">
<img src="assets/swin_2.webp">
<img src="assets/swin_3.webp">

```python
def forward(self, x: torch.Tensor):
        swish = F.silu(self.w1(x))  # Apply first transformation
        x_V = self.w3(x) 
        x = swish * x_V        # Apply contraction to original dimension
        x = self.w2(x)  # Apply optional additional transformation
        return x

```

## FeedForward Block
In the Transformer architecture, the feedforward layer plays a crucial role, typically following the attention layer and normalization. The feedforward layer consists of three linear transformations.

```python
class FeedForward(nn.Module):
    def __init__(self, args: ModelArgs):
        super().__init__()
        # Assuming 'hidden_dim' is calculated as per your specifications
        hidden_dim = 4 * args.dim
        hidden_dim = int(2 * hidden_dim / 3)  # Applying your specific transformation
        if args.ffn_dim_multiplier is not None:
            hidden_dim = int(args.ffn_dim_multiplier * hidden_dim)
        #hidden_dim = int(2 * hidden_dim / 3)  # Applying your specific transformation
        hidden_dim = args.multiple_of * ((hidden_dim + args.multiple_of - 1) // args.multiple_of)

        self.w1 = nn.Linear(args.dim, hidden_dim, bias=False)
        self.w2 = nn.Linear(hidden_dim, args.dim, bias=False)  # This layer seems to be missing in your original setup
        self.w3 = nn.Linear(args.dim, hidden_dim, bias=False)  # Corrected to match checkpoint

    def forward(self, x: torch.Tensor):
        swish = F.silu(self.w1(x))  # Apply first transformation
        x_V = self.w3(x) 
        x = swish * x_V        # Apply contraction to original dimension
        x = self.w2(x)  # Apply optional additional transformation
        return x
```

During the forward pass, the input tensor x is subjected to multi layer of linear transformations. The SwiGLU activation function, applied after first transformation, enhances the expressive power of the model. The final transformation maps the tensor back to its original dimensions. This unique combination of SwiGLU activation and multiple FeedForward layer enhances the performance of the model.
## Encoder Block
As we are only focusing on the Inferencing of the model we will only be looking into the Encoder Block
```python
class EncoderBlock(nn.Module):
    def __init__(self, args: ModelArgs):
        super().__init__()
        self.n_heads = args.n_heads
        self.dim = args.dim
        self.head_dim = args.dim // args.n_heads

        self.attention = SelfAttention(args)
        self.feed_forward = FeedForward(args)

        # normalize BEFORE the self attention
        self.attention_norm = RMSNorm(args.dim, eps=args.norm_eps)
        # Normalization BEFORE the feed forward
        self.ffn_norm = RMSNorm(args.dim, eps=args.norm_eps)

    def forward(self, x: torch.Tensor, start_pos: int, freqs_complex: torch.Tensor):
        # (B, seq_len, dim) + (B, seq_len, dim) -> (B, seq_len, dim)
        h = x + self.attention.forward(self.attention_norm(x), start_pos, freqs_complex)
        out = h + self.feed_forward.forward(self.ffn_norm(h))
        return out
```

## Ultimate Transformer Model
Now lets stack all of this individual components into the Transformer Block

```python
class ModelArgs:
    dim: int = 4096
    n_layers: int = 32
    n_heads: int = 32 # Number of heads for the queries
    n_kv_heads: Optional[int] = None # Number of heads for the keys and values. If None, defaults to n_heads
    vocab_size: int = -1 # This will be set when we load the tokenizer
    multiple_of: int = 256 
    ffn_dim_multiplier: Optional[float] = None # If None, defaults to 4.0
    norm_eps: float = 1e-6 # only the eps value has been modified int llama 3
    
    # Needed for KV cache
    max_batch_size: int = 32
    max_seq_len: int = 2048

    device: str = None

class Transformer(nn.Module):
    
    def __init__(self, args: ModelArgs) -> None:
        super().__init__()

        assert args.vocab_size != -1, "Vocab size must be set"

        self.args = args
        self.vocab_size = args.vocab_size
        self.n_layers = args.n_layers
        self.tok_embeddings = nn.Embedding(self.vocab_size, args.dim)

        self.layers = nn.ModuleList()
        for _ in range(args.n_layers):
            self.layers.append(EncoderBlock(args))

        self.norm = RMSNorm(args.dim, eps=args.norm_eps)
        self.output = nn.Linear(args.dim, self.vocab_size, bias=False)

        # To precompute the frequencies of the Rotary Positional Encodings
        self.freqs_complex = precomputed_theta_pos_frequencies(self.args.dim // self.args.n_heads, self.args.max_seq_len * 2, device=self.args.device)

    def forward(self, tokens: torch.Tensor, start_pos: int):
        # (B, seq_len)
        batch_size, seq_len = tokens.shape
        assert seq_len == 1, "Only one token at a time can be processed"
  
        # (B, seq_len) -> (B, seq_len, dim)
        h = self.tok_embeddings(tokens)

        # Retrive the pairs (m, theta) corresponding to the positions [start_pos, start_pos + seq_len]
        freqs_complex = self.freqs_complex[start_pos:start_pos + seq_len]

        # Consecutively apply all the encoder layers
        for layer in self.layers:
            h = layer(h, start_pos, freqs_complex)
        h =  self.norm(h)
        output = self.output(h).float()
        return output
```
For Inferencing the model we need to download the weights of the model from Meta’s Llama 3 repo.
Now let’s write the code for Inferencing the model. There are many tunable parameters to consider when inferencing the model these include top-p, top-k, search mechanism greedy search / beam search for simplicity I have only implemented greedy search for beam search you can refer to Llama 3 repo on GitHub

```python
## Inference
from typing import Optional
import torch
import time
import json 
from pathlib import Path
from sentencepiece import SentencePieceProcessor
from tqdm import tqdm
from model import ModelArgs, Transformer


class LLaMA:

    def __init__(self, model: Transformer, tokenizer: SentencePieceProcessor, model_args: ModelArgs):
        self.model = model
        self.tokenizer = tokenizer
        self.args = model_args

    @staticmethod
    def build(checkpoints_dir: str, tokenizer_path: str, load_model: bool, max_seq_len: int, max_batch_size: int, device: str):
        prev_time = time.time()
        if load_model:
            checkpoints = sorted(Path(checkpoints_dir).glob("*.pth"))
            assert len(checkpoints) > 0, "No checkpoints files found"
            chk_path = checkpoints[0]
            print(f'Loaded checkpoint {chk_path}')
            checkpoint = torch.load(chk_path, map_location="cpu")
            print(f'Loaded checkpoint in {(time.time() - prev_time):.2f} seconds')
            prev_time = time.time()

        with open(Path(checkpoints_dir) / "params.json", "r") as f:
            params = json.loads(f.read())
        model_args: ModelArgs = ModelArgs(
            max_seq_len = max_seq_len,
            max_batch_size = max_batch_size,
            device = device,
            **params
        )
        tokenizer = SentencePieceProcessor()
        tokenizer.load(tokenizer_path)
        model_args.vocab_size = tokenizer.vocab_size()

        if device == "cuda":
            torch.set_default_tensor_type(torch.cuda.HalfTensor)
        else:
            torch.set_default_tensor_type(torch.BFloat16Tensor)

        model = Transformer(model_args).to(device)

        if load_model:
            # Remove repe.freq from checkpoint as we are precomputiong the frequencies
            del checkpoint["rope.freqs"]
            model.load_state_dict(checkpoint, strict=False)
            print(f"Loaded state dict in {(time.time() - prev_time):.2f} seconds")

        return LLaMA(model, tokenizer, model_args)
    
    def text_completion(self, prompts: list[str], temperature: float = 0.6, top_p: float = 0.9, max_gen_len: Optional[int] = None):
        if max_gen_len is None:
            max_gen_len = self.args.max_seq_len - 1
        # Convert each prompt into tokens
        prompt_tokens = [self.tokenizer.encode(prompt, out_type=int, add_bos=True, add_eos=False) for prompt in prompts]
        # Make sure the batch size is not too large
        batch_size = len(prompt_tokens)
        assert batch_size <= self.args.max_batch_size, f"Batch size {batch_size} is too large"
        max_prompt_len = max(len(prompt) for prompt in prompt_tokens)
        # Make sure the prompt length is not larger than the maximum seq length
        assert max_prompt_len < self.args.max_seq_len, f"Prompt length {max_prompt_len} is too large"
        total_len = min(self.args.max_seq_len, max_gen_len + max_prompt_len)

        # Create the list that will contain the generated tokens, along with the initial prompt tokens
        pad_id = self.tokenizer.pad_id()
        tokens = torch.full((batch_size, total_len), pad_id, dtype=torch.long, device=self.args.device)
        for k, t in enumerate(prompt_tokens):
            tokens[k, :len(t)] = torch.tensor(t, dtype=torch.long, device=self.args.device)

        eos_reached = torch.tensor([False] * batch_size, device=self.args.device)
        prompt_tokens_mask = tokens != pad_id  # True if the token is a prompt token, False otherwise
        for cur_pos in tqdm(range(1, total_len), desc='Generating tokens'):
            with torch.no_grad():
                logits = self.model.forward(tokens[:, cur_pos-1:cur_pos], cur_pos)
            if temperature > 0:
                # The temperature is applied BEFORE the softmax
                probs = torch.softmax(logits[:, -1] / temperature, dim=-1)
                next_token = self._sample_top_p(probs, top_p)
            else:
                # Greedily select the token with the maximum probability
                next_token = torch.argmax(logits[:, -1], dim=-1)

            next_token = next_token.reshape(-1)
            # Only replace the token if it is a padding token
            next_token = torch.where(prompt_tokens_mask[:, cur_pos], tokens[:, cur_pos], next_token)
            tokens[:, cur_pos] = next_token
            # EOS is reached only if we found an EOS token for padding position
            eos_reached |= (~prompt_tokens_mask[:, cur_pos]) & (next_token == self.tokenizer.eos_id())
            if all(eos_reached):
                break

        out_tokens = []
        out_text = []
        for prompt_index, current_prompt_tokens in enumerate(tokens.tolist()):
            # Cut to the EOS token, if present
            if self.tokenizer.eos_id() in current_prompt_tokens:
                eos_idx = current_prompt_tokens.index(self.tokenizer.eos_id())
                current_prompt_tokens = current_prompt_tokens[:eos_idx]
            out_tokens.append(current_prompt_tokens)
            out_text.append(self.tokenizer.decode(current_prompt_tokens))

        return (out_tokens, out_text)

            
    def _sample_top_p(self, probs, p):
        probs_sort, probs_idx = torch.sort(probs, dim= -1, descending=True)
        probs_sum = torch.cumsum(probs_sort, dim=-1)
        mask = probs_sum - probs_sort > p
        probs_sort[mask] = 0.0
        probs_sort.div_(probs_sort.sum(dim=-1, keepdim=True))
        next_token = torch.multinomial(probs_sort, num_samples=1)
        next_token = torch.gather(probs_idx, -1, next_token)
        return next_token



if __name__ == '__main__':
    import os
    torch.manual_seed(0)
    prompts = [
        # Few shot prompt
        """Translate English to kananda:
        water : ನೀರು
        land : ಭೂಮಿ
        dusk : ಸಂಜೆ
        dawn : ಬೆಳಗುವಿಕೆ
        milk : ಹಾಲು"""
        # Zero shot prompt
        """Tell me if the following person is actually a real person or a fictional character:
        Name : Vignesh 
        Decision:
        """
    ]
    allow_cuda = True if 'CUDA_VISIBLE_DEVICES' in os.environ else False
    device = 'cuda' if torch.cuda.is_available() and allow_cuda else 'cpu'
    
    model = LLaMA.build(
        checkpoints_dir='Meta-Llama-3-8B/',
        tokenizer_path='Meta-Llama-3-8B/tokenizer.model',
        load_model=True,
        max_seq_len=1024,
        max_batch_size=len(prompts),
        device=device
    )

    print('ALL OK')

    # Inferencing the model
    print("Inferenceing the model")
    out_tokens, out_text = model.text_completion(prompts, max_gen_len=64)
    assert len(out_tokens) == len(prompts)
    for i in range(len(out_text)):
        print(f"{out_text[i]}")
        print('-' * 50)
