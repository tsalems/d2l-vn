<!-- ===================== Bắt đầu dịch Phần 1 ==================== -->
<!-- ========================================= REVISE PHẦN 1 - BẮT ĐẦU =================================== -->

<!--
# Transformer
-->

# *dịch tiêu đề phía trên*
:label:`sec_transformer`

<!--
In previous chapters, we have covered major neural network architectures such as convolution neural networks (CNNs) and  recurrent neural networks (RNNs).
Let us recap their pros and cons:
-->

*dịch đoạn phía trên*

<!--
* **CNNs** are easy to parallelize at a layer but cannot capture the variable-length sequential dependency very well.
-->

*dịch đoạn phía trên*

<!--
* **RNNs** are able to capture the long-range, variable-length sequential information, but suffer from inability to parallelize within a sequence.
-->

*dịch đoạn phía trên*

<!--
To combine the advantages from both CNNs and RNNs, :cite:`Vaswani.Shazeer.Parmar.ea.2017` designed a novel architecture using the attention mechanism.
This architecture, which is called as *Transformer*, achieves parallelization by capturing recurrence sequence with attention and at the same time encodes each item's position in the sequence.
As a result, Transformer leads to a compatible model with significantly shorter training time.
-->

*dịch đoạn phía trên*

<!--
Similar to the seq2seq model in :numref:`sec_seq2seq`, Transformer is also based on the encoder-decoder architecture.
However, Transformer differs to the former by replacing the recurrent layers in seq2seq with *multi-head attention* layers, 
incorporating the position-wise information through *position encoding*, and applying *layer normalization*.
We  compare Transformer and seq2seq  side-by-side in :numref:`fig_transformer`.
-->

*dịch đoạn phía trên*

<!--
Overall, these two models are similar to each other: the source sequence embeddings are fed into $n$ repeated blocks.
The outputs of the last block are then used as attention memory for the decoder.
The target sequence embeddings are similarly fed into $n$ repeated blocks in the decoder, and the final outputs are obtained by applying a dense layer with vocabulary size to the last block's outputs.
-->

*dịch đoạn phía trên*

<!--
![The Transformer architecture.](../img/transformer.svg)
-->

![*dịch chú thích ảnh phía trên*](../img/transformer.svg)
:width:`500px`
:label:`fig_transformer`

<!-- ===================== Kết thúc dịch Phần 1 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 2 ===================== -->


<!--
On the flip side, Transformer differs from the seq2seq with attention model in the following:
-->

*dịch đoạn phía trên*

<!--
1. **Transformer block**: a recurrent layer in seq2seq is replaced by a *Transformer block*. 
This block contains a *multi-head attention* layer and a network with two *position-wise feed-forward network* layers for the encoder. 
For the decoder, another multi-head attention layer is used to take the encoder state.
2. **Add and norm**: the inputs and outputs of both the multi-head attention layer or the position-wise feed-forward network, 
are processed by two "add and norm" layer that contains a residual structure and a *layer normalization* layer.
3. **Position encoding**: since the self-attention layer does not distinguish the item order in a sequence, a positional encoding layer is used to add sequential information into each sequence item.
-->

*dịch đoạn phía trên*


<!--
In the rest of this section, we will equip you with each new component introduced by Transformer, and get you up and running to construct a machine translation model.
-->

*dịch đoạn phía trên*

```{.python .input  n=1}
import d2l
import math
from mxnet import autograd, np, npx
from mxnet.gluon import nn
npx.set_np()
```

<!--
## Multi-Head Attention
-->

## *dịch tiêu đề phía trên*

<!--
Before the discussion of the *multi-head attention* layer, let us quick express the *self-attention* architecture.
The self-attention model is a normal attention model, with its query, its key, and its value being copied exactly the same from each item of the sequential inputs.
As we illustrate in :numref:`fig_self_attention`, self-attention outputs a same-length sequential output for each input item.
Compared with a recurrent layer, output items of a self-attention layer can be computed in parallel and, therefore, it is easy to obtain a highly-efficient implementation.
-->

*dịch đoạn phía trên*

<!--
![Self-attention architecture.](../img/self-attention.svg)
-->

![*dịch chú thích ảnh phía trên*](../img/self-attention.svg)
:label:`fig_self_attention`


<!--
The *multi-head attention* layer consists of $h$ parallel self-attention layers, each one is called a *head*.
For each head, before feeding into the attention layer, we project the queries, keys, and values with three dense layers with hidden sizes $p_q$, $p_k$, and $p_v$, respectively.
The outputs of these $h$ attention heads are concatenated and then processed by a final dense layer.
-->

*dịch đoạn phía trên*


<!--
![Multi-head attention](../img/multi-head-attention.svg)
-->

![*dịch chú thích ảnh phía trên*](../img/multi-head-attention.svg)


<!-- ===================== Kết thúc dịch Phần 2 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 3 ===================== -->

<!--
Assume that the dimension for a query, a key, and a value are $d_q$, $d_k$, and $d_v$, respectively.
Then, for each head $i=1,\ldots, h$, we can train learnable parameters
$\mathbf W_q^{(i)}\in\mathbb R^{p_q\times d_q}$,
$\mathbf W_k^{(i)}\in\mathbb R^{p_k\times d_k}$,
and $\mathbf W_v^{(i)}\in\mathbb R^{p_v\times d_v}$. Therefore, the output for each head is
-->

*dịch đoạn phía trên*


$$\mathbf o^{(i)} = \textrm{attention}(\mathbf W_q^{(i)}\mathbf q, \mathbf W_k^{(i)}\mathbf k,\mathbf W_v^{(i)}\mathbf v),$$


<!--
where $\textrm{attention}$ can be any attention layer, such as the `DotProductAttention` and `MLPAttention` as we introduced in :numref:`sec_attention`.
-->

*dịch đoạn phía trên*


<!--
After that, the output with length $p_v$ from each of the $h$ attention heads are concatenated to be an output of length $h p_v$, which is then passed the final dense layer with $d_o$ hidden units.
The weights of this dense layer can be denoted by $\mathbf W_o\in\mathbb R^{d_o\times h p_v}$.
As a result, the multi-head attention output will be
-->

*dịch đoạn phía trên*


$$\mathbf o = \mathbf W_o \begin{bmatrix}\mathbf o^{(1)}\\\vdots\\\mathbf o^{(h)}\end{bmatrix}.$$


<!--
Now we can implement the multi-head attention.
Assume that the multi-head attention contain the number heads `num_heads` $=h$, the hidden size `num_hiddens` $=p_q=p_k=p_v$ are the same for the query, key, and value dense layers.
In addition, since the multi-head attention keeps the same dimensionality between its input and its output, we have the output feature size $d_o =$ `num_hiddens` as well.
-->

*dịch đoạn phía trên*


```{.python .input  n=2}
# Saved in the d2l package for later use
class MultiHeadAttention(nn.Block):
    def __init__(self, num_hiddens, num_heads, dropout, use_bias=False, **kwargs):
        super(MultiHeadAttention, self).__init__(**kwargs)
        self.num_heads = num_heads
        self.attention = d2l.DotProductAttention(dropout)
        self.W_q = nn.Dense(num_hiddens, use_bias=use_bias, flatten=False)
        self.W_k = nn.Dense(num_hiddens, use_bias=use_bias, flatten=False)
        self.W_v = nn.Dense(num_hiddens, use_bias=use_bias, flatten=False)
        self.W_o = nn.Dense(num_hiddens, use_bias=use_bias, flatten=False)

    def forward(self, query, key, value, valid_len):
        # For self-attention, query, key, and value shape:
        # (batch_size, seq_len, dim), where seq_len is the length of input
        # sequence. valid_len shape is either (batch_size, ) or
        # (batch_size, seq_len).

        # Project and transpose query, key, and value from
        # (batch_size, seq_len, num_hiddens) to
        # (batch_size * num_heads, seq_len, num_hiddens / num_heads).
        query = transpose_qkv(self.W_q(query), self.num_heads)
        key = transpose_qkv(self.W_k(key), self.num_heads)
        value = transpose_qkv(self.W_v(value), self.num_heads)

        if valid_len is not None:
            # Copy valid_len by num_heads times
            if valid_len.ndim == 1:
                valid_len = np.tile(valid_len, self.num_heads)
            else:
                valid_len = np.tile(valid_len, (self.num_heads, 1))

        # For self-attention, output shape:
        # (batch_size * num_heads, seq_len, num_hiddens / num_heads)
        output = self.attention(query, key, value, valid_len)

        # output_concat shape: (batch_size, seq_len, num_hiddens)
        output_concat = transpose_output(output, self.num_heads)
        return self.W_o(output_concat)
```


<!--
Here are the definitions of the transpose functions `transpose_qkv` and `transpose_output`, which are the inverse of each other.
-->

*dịch đoạn phía trên*


```{.python .input  n=3}
# Saved in the d2l package for later use
def transpose_qkv(X, num_heads):
    # Input X shape: (batch_size, seq_len, num_hiddens).
    # Output X shape:
    # (batch_size, seq_len, num_heads, num_hiddens / num_heads)
    X = X.reshape(X.shape[0], X.shape[1], num_heads, -1)

    # X shape: (batch_size, num_heads, seq_len, num_hiddens / num_heads)
    X = X.transpose(0, 2, 1, 3)

    # output shape: (batch_size * num_heads, seq_len, num_hiddens / num_heads)
    output = X.reshape(-1, X.shape[2], X.shape[3])
    return output


# Saved in the d2l package for later use
def transpose_output(X, num_heads):
    # A reversed version of transpose_qkv
    X = X.reshape(-1, num_heads, X.shape[1], X.shape[2])
    X = X.transpose(0, 2, 1, 3)
    return X.reshape(X.shape[0], X.shape[1], -1)
```


<!--
Let us test the `MultiHeadAttention` model in the a toy example. Create a multi-head attention with the hidden size $d_o = 100$, 
the output will share the same batch size and sequence length as the input, but the last dimension will be equal to the `num_hiddens` $= 100$.
-->

*dịch đoạn phía trên*


```{.python .input  n=4}
cell = MultiHeadAttention(90, 9, 0.5)
cell.initialize()
X = np.ones((2, 4, 5))
valid_len = np.array([2, 3])
cell(X, X, X, valid_len).shape
```

<!-- ===================== Kết thúc dịch Phần 3 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 4 ===================== -->

<!-- ========================================= REVISE PHẦN 1 - KẾT THÚC ===================================-->

<!-- ========================================= REVISE PHẦN 2 - BẮT ĐẦU ===================================-->

<!--
## Position-wise Feed-Forward Networks
-->

## *dịch tiêu đề phía trên*

<!--
Another key component in the Transformer block is called *position-wise feed-forward network (FFN)*.
It accepts a $3$-dimensional input with shape (batch size, sequence length, feature size).
The position-wise FFN consists of two dense layers that applies to the last dimension.
Since the same two dense layers are used for each position item in the sequence, we referred to it as *position-wise*.
Indeed, it is equivalent to applying two $1 \times 1$ convolution layers.
-->

*dịch đoạn phía trên*

<!--
Below, the `PositionWiseFFN` shows how to implement a position-wise FFN with two dense layers of hidden size `ffn_num_hiddens` and `pw_num_outputs`, respectively.
-->

*dịch đoạn phía trên*

```{.python .input  n=5}
# Saved in the d2l package for later use
class PositionWiseFFN(nn.Block):
    def __init__(self, ffn_num_hiddens, pw_num_outputs, **kwargs):
        super(PositionWiseFFN, self).__init__(**kwargs)
        self.dense1 = nn.Dense(ffn_num_hiddens, flatten=False,
                               activation='relu')
        self.dense2 = nn.Dense(pw_num_outputs, flatten=False)

    def forward(self, X):
        return self.dense2(self.dense1(X))
```

<!--
Similar to the multi-head attention, the position-wise feed-forward network will only change the last dimension size of the input---the feature dimension.
In addition, if two items in the input sequence are identical, the according outputs will be identical as well.
-->

*dịch đoạn phía trên*

```{.python .input  n=6}
ffn = PositionWiseFFN(4, 8)
ffn.initialize()
ffn(np.ones((2, 3, 4)))[0]
```

<!--
## Add and Norm
-->

## *dịch tiêu đề phía trên*

<!--
Besides the above two components in the Transformer block, the "add and norm" within the block also plays a key role to connect the inputs and outputs of other layers smoothly.
To explain, we add a layer that contains a residual structure and a *layer normalization* after both the multi-head attention layer and the position-wise FFN network.
*Layer normalization* is similar to batch normalization in :numref:`sec_batch_norm`.
One difference is that the mean and variances for the layer normalization are calculated along the last dimension, e.g `X.mean(axis=-1)` instead of the first batch dimension, e.g., `X.mean(axis=0)`.
Layer normalization prevents the range of values in the layers from changing too much, which allows faster training and better generalization ability.
-->

*dịch đoạn phía trên*

<!--
MXNet has both `LayerNorm` and `BatchNorm` implemented within the `nn` block.
Let us call both of them and see the difference in the  example below.
-->

*dịch đoạn phía trên*


```{.python .input  n=7}
layer = nn.LayerNorm()
layer.initialize()
batch = nn.BatchNorm()
batch.initialize()
X = np.array([[1, 2], [2, 3]])
# Compute mean and variance from X in the training mode
with autograd.record():
    print('layer norm:', layer(X), '\nbatch norm:', batch(X))
```

<!--
Now let us implement the connection block `AddNorm` together.
`AddNorm` accepts two inputs $X$ and $Y$. 
We can deem $X$ as the original input in the residual network, and $Y$ as the outputs from either the multi-head attention layer or the position-wise FFN network.
In addition, we apply dropout on $Y$ for regularization.
-->

*dịch đoạn phía trên*


```{.python .input  n=8}
# Saved in the d2l package for later use
class AddNorm(nn.Block):
    def __init__(self, dropout, **kwargs):
        super(AddNorm, self).__init__(**kwargs)
        self.dropout = nn.Dropout(dropout)
        self.ln = nn.LayerNorm()

    def forward(self, X, Y):
        return self.ln(self.dropout(Y) + X)
```

<!--
Due to the residual connection, $X$ and $Y$ should have the same shape.
-->

*dịch đoạn phía trên*

```{.python .input  n=9}
add_norm = AddNorm(0.5)
add_norm.initialize()
add_norm(np.ones((2, 3, 4)), np.ones((2, 3, 4))).shape
```

<!-- ===================== Kết thúc dịch Phần 4 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 5 ===================== -->

<!--
## Positional Encoding
-->

## *dịch tiêu đề phía trên*

<!--
Unlike the recurrent layer, both the multi-head attention layer and the position-wise feed-forward network compute the output of each item in the sequence independently.
This feature enables us to parallelize the computation, but it fails to model the sequential information for a given sequence.
To better capture the sequential information, the Transformer model uses the *positional encoding* to maintain the positional information of the input sequence.
-->

*dịch đoạn phía trên*

<!--
To explain, assume that $X\in\mathbb R^{l\times d}$ is the embedding of an example, where $l$ is the sequence length and $d$ is the embedding size.
This positional encoding layer encodes X's position $P\in\mathbb R^{l\times d}$ and outputs $P+X$.
-->

*dịch đoạn phía trên*

<!--
The position $P$ is a 2-D matrix, where $i$ refers to the order in the sentence, and $j$ refers to the position along the embedding vector dimension.
In this way, each value in the origin sequence is then maintained using the equations below:
-->

*dịch đoạn phía trên*


$$P_{i, 2j} = \sin(i/10000^{2j/d}),$$

$$\quad P_{i, 2j+1} = \cos(i/10000^{2j/d}),$$


<!--
for $i=0,\ldots, l-1$ and $j=0,\ldots,\lfloor(d-1)/2\rfloor$.
-->

*dịch đoạn phía trên*


<!--
:numref:`fig_positional_encoding` illustrates the positional encoding.
-->

*dịch đoạn phía trên*


<!--
![Positional encoding.](../img/positional_encoding.svg)
-->

![*dịch chú thích ảnh phía trên*](../img/positional_encoding.svg)
:label:`fig_positional_encoding`


```{.python .input  n=10}
# Saved in the d2l package for later use
class PositionalEncoding(nn.Block):
    def __init__(self, num_hiddens, dropout, max_len=1000):
        super(PositionalEncoding, self).__init__()
        self.dropout = nn.Dropout(dropout)
        # Create a long enough P
        self.P = np.zeros((1, max_len, num_hiddens))
        X = np.arange(0, max_len).reshape(-1, 1) / np.power(
            10000, np.arange(0, num_hiddens, 2) / num_hiddens)
        self.P[:, :, 0::2] = np.sin(X)
        self.P[:, :, 1::2] = np.cos(X)

    def forward(self, X):
        X = X + self.P[:, :X.shape[1], :].as_in_ctx(X.ctx)
        return self.dropout(X)
```


<!--
Now we test the `PositionalEncoding` class with a toy model for 4 dimensions.
As we can see, the $4^{\mathrm{th}}$ dimension has the same frequency as the $5^{\mathrm{th}}$ but with different offset.
The $5^{\mathrm{th}}$ and $6^{\mathrm{th}}$ dimensions have a lower frequency.
-->

*dịch đoạn phía trên*


```{.python .input  n=11}
pe = PositionalEncoding(20, 0)
pe.initialize()
Y = pe(np.zeros((1, 100, 20)))
d2l.plot(np.arange(100), Y[0, :, 4:8].T, figsize=(6, 2.5),
         legend=["dim %d" % p for p in [4, 5, 6, 7]])
```

<!-- ===================== Kết thúc dịch Phần 5 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 6 ===================== -->

<!-- ========================================= REVISE PHẦN 2 - KẾT THÚC ===================================-->

<!-- ========================================= REVISE PHẦN 3 - BẮT ĐẦU ===================================-->

<!--
## Encoder
-->

## *dịch tiêu đề phía trên*

<!--
Armed with all the essential components of Transformer, let us first build a Transformer encoder block.
This encoder contains a multi-head attention layer, a position-wise feed-forward network, and two "add and norm" connection blocks.
As shown in the code, for both of the attention model and the positional FFN model in the `EncoderBlock`, their outputs' dimension are equal to the `num_hiddens`.
This is due to the nature of the residual block, as we need to add these outputs back to the original value during "add and norm".
-->

*dịch đoạn phía trên*


```{.python .input  n=12}
# Saved in the d2l package for later use
class EncoderBlock(nn.Block):
    def __init__(self, num_hiddens, ffn_num_hiddens, num_heads, dropout,
                 use_bias=False, **kwargs):
        super(EncoderBlock, self).__init__(**kwargs)
        self.attention = MultiHeadAttention(num_hiddens, num_heads, dropout,
                                            use_bias)
        self.addnorm1 = AddNorm(dropout)
        self.ffn = PositionWiseFFN(ffn_num_hiddens, num_hiddens)
        self.addnorm2 = AddNorm(dropout)

    def forward(self, X, valid_len):
        Y = self.addnorm1(X, self.attention(X, X, X, valid_len))
        return self.addnorm2(Y, self.ffn(Y))
```


<!--
Due to the residual connections, this block will not change the input shape.
It means that the `num_hiddens` argument should be equal to the input size of the last dimension.
In our toy example below,  `num_hiddens` $= 24$, `ffn_num_hiddens` $=48$, `num_heads` $= 8$, and `dropout` $= 0.5$.
-->

*dịch đoạn phía trên*


```{.python .input  n=13}
X = np.ones((2, 100, 24))
encoder_blk = EncoderBlock(24, 48, 8, 0.5)
encoder_blk.initialize()
encoder_blk(X, valid_len).shape
```

<!--
Now it comes to the implementation of the entire Transformer encoder.
With the Transformer encoder, $n$ blocks of `EncoderBlock` stack up one after another.
Because of the residual connection, the embedding layer size $d$ is same as the Transformer block output size.
Also note that we multiply the embedding output by $\sqrt{d}$ to prevent its values from being too small.
-->

*dịch đoạn phía trên*


```{.python .input  n=14}
# Saved in the d2l package for later use
class TransformerEncoder(d2l.Encoder):
    def __init__(self, vocab_size, num_hiddens, ffn_num_hiddens,
                 num_heads, num_layers, dropout, use_bias=False, **kwargs):
        super(TransformerEncoder, self).__init__(**kwargs)
        self.num_hiddens = num_hiddens
        self.embedding = nn.Embedding(vocab_size, num_hiddens)
        self.pos_encoding = PositionalEncoding(num_hiddens, dropout)
        self.blks = nn.Sequential()
        for _ in range(num_layers):
            self.blks.add(
                EncoderBlock(num_hiddens, ffn_num_hiddens, num_heads, dropout, use_bias))

    def forward(self, X, valid_len, *args):
        X = self.pos_encoding(self.embedding(X) * math.sqrt(self.num_hiddens))
        for blk in self.blks:
            X = blk(X, valid_len)
        return X
```


<!--
Let us create an encoder with two stacked Transformer encoder blocks, whose hyperparameters are the same as before.
Similar to the previous toy example's parameters, we add two more parameters `vocab_size` to be $200$ and `num_layers` to be $2$ here.
-->

*dịch đoạn phía trên*


```{.python .input  n=15}
encoder = TransformerEncoder(200, 24, 48, 8, 2, 0.5)
encoder.initialize()
encoder(np.ones((2, 100)), valid_len).shape
```

<!-- ===================== Kết thúc dịch Phần 6 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 7 ===================== -->

<!--
## Decoder
-->

## *dịch tiêu đề phía trên*

<!--
The Transformer decoder block looks similar to the Transformer encoder block.
However, besides the two sub-layers (the multi-head attention layer and the positional encoding network), 
the decoder Transformer block contains a third sub-layer, which applies multi-head attention on the output of the encoder stack.
Similar to the Transformer encoder block, the  Transformer decoder block employs "add and norm", 
i.e., the residual connections and the layer normalization to connect each of the sub-layers.
-->

*dịch đoạn phía trên*

<!--
To be specific, at timestep $t$, assume that $\mathbf x_t$ is the current input, i.e., the query.
As illustrated in :numref:`fig_self_attention_predict`, the keys and values of the self-attention layer consist of the current query with all the past queries $\mathbf x_1, \ldots, \mathbf x_{t-1}$.
-->

*dịch đoạn phía trên*

<!--
![Predict at timestep $t$ for a self-attention layer.](../img/self-attention-predict.svg)
-->

![*dịch chú thích ảnh phía trên*](../img/self-attention-predict.svg)
:label:`fig_self_attention_predict`

<!--
During training, the output for the $t$-query could observe all the previous key-value pairs.
It results in an different behavior from prediction.
Thus, during prediction we can eliminate the unnecessary information by specifying the valid length to be $t$ for the $t^\textrm{th}$ query.
-->

*dịch đoạn phía trên*


```{.python .input  n=16}
class DecoderBlock(nn.Block):
    # i means it is the i-th block in the decoder
    def __init__(self, num_hiddens, ffn_num_hiddens, num_heads,
                 dropout, i, **kwargs):
        super(DecoderBlock, self).__init__(**kwargs)
        self.i = i
        self.attention1 = MultiHeadAttention(num_hiddens, num_heads, dropout)
        self.addnorm1 = AddNorm(dropout)
        self.attention2 = MultiHeadAttention(num_hiddens, num_heads, dropout)
        self.addnorm2 = AddNorm(dropout)
        self.ffn = PositionWiseFFN(ffn_num_hiddens, num_hiddens)
        self.addnorm3 = AddNorm(dropout)

    def forward(self, X, state):
        enc_outputs, enc_valid_len = state[0], state[1]
        # state[2][i] contains the past queries for this block
        if state[2][self.i] is None:
            key_values = X
        else:
            key_values = np.concatenate((state[2][self.i], X), axis=1)
        state[2][self.i] = key_values
        if autograd.is_training():
            batch_size, seq_len, _ = X.shape
            # Shape: (batch_size, seq_len), the values in the j-th column
            # are j+1
            valid_len = np.tile(np.arange(1, seq_len+1, ctx=X.ctx),
                                   (batch_size, 1))
        else:
            valid_len = None

        X2 = self.attention1(X, key_values, key_values, valid_len)
        Y = self.addnorm1(X, X2)
        Y2 = self.attention2(Y, enc_outputs, enc_outputs, enc_valid_len)
        Z = self.addnorm2(Y, Y2)
        return self.addnorm3(Z, self.ffn(Z)), state
```

<!--
Similar to the Transformer encoder block, `num_hiddens` should be equal to the last dimension size of $X$.
-->

*dịch đoạn phía trên*

```{.python .input  n=17}
decoder_blk = DecoderBlock(24, 48, 8, 0.5, 0)
decoder_blk.initialize()
X = np.ones((2, 100, 24))
state = [encoder_blk(X, valid_len), valid_len, [None]]
decoder_blk(X, state)[0].shape
```

<!--
The construction of the entire Transformer decoder is identical to the Transformer encoder, except for the additional dense layer to obtain the output confidence scores.
-->

*dịch đoạn phía trên*

<!--
Let us implement the Transformer decoder `TransformerDecoder`.
Besides the regular hyperparameters such as the `vocab_size` and `num_hiddens`, the Transformer decoder also needs the Transformer encoder's outputs `enc_outputs` and `env_valid_len`.
-->

*dịch đoạn phía trên*


```{.python .input  n=18}
class TransformerDecoder(d2l.Decoder):
    def __init__(self, vocab_size, num_hiddens, ffn_num_hiddens,
                 num_heads, num_layers, dropout, **kwargs):
        super(TransformerDecoder, self).__init__(**kwargs)
        self.num_hiddens = num_hiddens
        self.num_layers = num_layers
        self.embedding = nn.Embedding(vocab_size, num_hiddens)
        self.pos_encoding = PositionalEncoding(num_hiddens, dropout)
        self.blks = nn.Sequential()
        for i in range(num_layers):
            self.blks.add(
                DecoderBlock(num_hiddens, ffn_num_hiddens, num_heads,
                             dropout, i))
        self.dense = nn.Dense(vocab_size, flatten=False)

    def init_state(self, enc_outputs, env_valid_len, *args):
        return [enc_outputs, env_valid_len, [None]*self.num_layers]

    def forward(self, X, state):
        X = self.pos_encoding(self.embedding(X) * math.sqrt(self.num_hiddens))
        for blk in self.blks:
            X, state = blk(X, state)
        return self.dense(X), state
```

<!-- ===================== Kết thúc dịch Phần 7 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 8 ===================== -->

<!--
## Training
-->

## *dịch tiêu đề phía trên*

<!--
Finally, we can build an encoder-decoder model with the Transformer architecture.
Similar to the seq2seq with attention model in :numref:`sec_seq2seq_attention`, we use the following hyperparameters: two Transformer blocks with both the embedding size and the block output size to be $32$. In addition, we use $4$ heads, and set the hidden size to be twice larger than the output size.
-->

*dịch đoạn phía trên*


```{.python .input  n=19}
num_hiddens, num_layers, dropout, batch_size, num_steps = 32, 2, 0.0, 64, 10
lr, num_epochs, ctx = 0.005, 100, d2l.try_gpu()
ffn_num_hiddens, num_heads = 64, 4

src_vocab, tgt_vocab, train_iter = d2l.load_data_nmt(batch_size, num_steps)

encoder = TransformerEncoder(
    len(src_vocab), num_hiddens, ffn_num_hiddens, num_heads, num_layers,
    dropout)
decoder = TransformerDecoder(
    len(src_vocab), num_hiddens, ffn_num_hiddens, num_heads, num_layers,
    dropout)
model = d2l.EncoderDecoder(encoder, decoder)
d2l.train_s2s_ch9(model, train_iter, lr, num_epochs, ctx)
```

<!--
As we can see from the training time and accuracy, compared with the seq2seq model with attention model, Transformer runs faster per epoch, and converges faster at the beginning.
-->

*dịch đoạn phía trên*

<!--
We can use the trained Transformer to translate some simple sentences.
-->

*dịch đoạn phía trên*


```{.python .input  n=20}
for sentence in ['Go .', 'Wow !', "I'm OK .", 'I won !']:
    print(sentence + ' => ' + d2l.predict_s2s_ch9(
        model, sentence, src_vocab, tgt_vocab, num_steps, ctx))
```

<!--
## Summary
-->

## Tóm tắt

<!--
* The Transformer model is based on the encoder-decoder architecture.
* Multi-head attention layer contains $h$ parallel attention layers.
* Position-wise feed-forward network consists of two dense layers that apply to the last dimension.
* Layer normalization differs from batch normalization by normalizing along the last dimension (the feature dimension) instead of the first (batch size) dimension.
* Positional encoding is the only place that adds positional information to the Transformer model.
-->

*dịch đoạn phía trên*


<!--
## Exercises
-->

## Bài tập

<!--
1. Try a larger size of epochs and compare the loss between seq2seq model and Transformer model.
2. Can you think of any other benefit of positional encoding?
3. Compare layer normalization and batch normalization, when shall we apply which?
-->

*dịch đoạn phía trên*


<!-- ===================== Kết thúc dịch Phần 8 ===================== -->
<!-- ========================================= REVISE PHẦN 3 - KẾT THÚC ===================================-->


## Thảo luận
* [Tiếng Anh](https://discuss.mxnet.io/t/4344)
* [Tiếng Việt](https://forum.machinelearningcoban.com/c/d2l)

## Những người thực hiện
Bản dịch trong trang này được thực hiện bởi:
<!--
Tác giả của mỗi Pull Request điền tên mình và tên những người review mà bạn thấy
hữu ích vào từng phần tương ứng. Mỗi dòng một tên, bắt đầu bằng dấu `*`.

Lưu ý:
* Nếu reviewer không cung cấp tên, bạn có thể dùng tên tài khoản GitHub của họ
với dấu `@` ở đầu. Ví dụ: @aivivn.

* Tên đầy đủ của các reviewer có thể được tìm thấy tại https://github.com/aivivn/d2l-vn/blob/master/docs/contributors_info.md
-->

* Đoàn Võ Duy Thanh
<!-- Phần 1 -->
*

<!-- Phần 2 -->
*

<!-- Phần 3 -->
*

<!-- Phần 4 -->
*

<!-- Phần 5 -->
*

<!-- Phần 6 -->
*

<!-- Phần 7 -->
*

<!-- Phần 8 -->
*