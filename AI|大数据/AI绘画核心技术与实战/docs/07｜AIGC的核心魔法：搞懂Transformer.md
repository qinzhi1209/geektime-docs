你好，我是南柯。

前两讲中，我们已经学习了扩散模型的加噪去噪过程，了解了UNet模型用于预测噪声的算法原理。事实上，Stable Diffusion模型在原始的UNet模型中加入了Transformer结构（至于怎么引入的，我们等 [下一讲](https://time.geekbang.org/column/article/683114) 学完UNet结构便会清楚），这么做可谓一举两得，因为Transformer结构不但能提升噪声去除效果，还是实现prompt控制图像内容的关键技术。

更重要的是，Transformer结构也是GPT系列工作的核心模块之一。也就是说，我们只有真正理解了Transformer，才算是进入了当下AIGC世界的大门。这一讲，我就为你揭秘Tranformer的算法原理。

## 初识Transformer

在深度学习中，有很多需要处理时序数据的任务，比如语音识别、文本理解、机器翻译、音乐生成等。不过，经典的卷积神经网络，也就是CNN结构，主要擅长处理空间相关的任务，比如图像分类、目标检测等。

因此， [RNN](https://ieeexplore.ieee.org/document/6795228)（循环神经网络）、 [LSTM](https://papers.baulab.info/Hochreiter-1997.pdf)（长短时记忆网络）以及 [Transformers](https://arxiv.org/abs/1706.03762) 这些解决时序任务的方案便应运而生。

### RNN和LSTM解决序列问题

RNN专为处理序列数据而设计，可以灵活地处理不同长度的数据。RNN的主要特点是在处理序列数据时，对前面的信息会产生某种“记忆”，通过这种记忆效果，RNN可以捕捉序列中的时间依赖关系。这种“记忆”在RNN中被称为隐藏状态（hidden state）。

![](https://static001.geekbang.org/resource/image/71/yy/718757f40615b9d39a6d4ef5c3a3d4yy.jpg?wh=4409x2480)

然而，传统的RNN存在一个关键的问题，即“长时依赖问题”——当序列很长时，RNN在处理过程中会出现梯度消失（梯度趋近于0）或梯度爆炸（梯度趋近于无穷大）现象。这种情况下，RNN可能无法有效地捕捉长距离的时间依赖信息。

为了解决这个问题， LSTM这种特殊的RNN结构就派上用场了。LSTM通过加入遗忘门、记忆门和输出门来处理长时依赖问题。这些门有助于LSTM更有效地保留和更新序列中的长距离信息。

![](https://static001.geekbang.org/resource/image/99/ab/993f69274022d7b8f01489234f0db9ab.jpg?wh=7817x3141)

以LSTM为代表的RNN类方案虽然在许多时序任务中取得了良好的效果，但是也有它的局限，主要是三个方面。

第一，并行计算问题。由于LSTM需要递归地处理序列数据，所以在计算过程中无法充分利用并行计算资源，处理长序列数据时效率较低。

第二，长时依赖问题。虽然LSTM有效地改善了传统RNN中的长时依赖问题，但在处理特别长的序列时，仍然可能出现依赖关系捕捉不足的问题。

第三，复杂性高。LSTM相比简单的RNN结构更复杂，增加了网络参数和计算量，这在一定程度上影响了训练速度和模型性能。

### Transformer的整体方案

在2017年由Google提出的Transformer，是一种基于自注意力机制（self-attention）的模型，它有效解决了RNN类方法的并行计算和长时依赖两大痛点。

Transformer结构包括编码器（Encoder）和解码器（Decoder）两个部分，通过这两个部分完成对 **输入序列的表示学习** 和 **输出序列的生成**。

编码器负责处理输入序列。它会根据全局上下文，提取输入数据中的有用信息，并学习输入序列的有效表示。

解码器则会根据编码器输出的表示，生成目标输出序列。它会关注并利用输入序列的表示以及当前位置的上下文信息，生成输出序列中每个元素的预测。

如果将编码器、解码器当作黑盒，Transformer可以看作是下图的样子。

![](https://static001.geekbang.org/resource/image/d9/cf/d9883336c304714cb4109ee4af8043cf.jpg?wh=4145x2480)

实际上，编码器和解码器分别由6层相同的结构堆叠而成，就像下面这幅图所展示的这样。

![](https://static001.geekbang.org/resource/image/9d/9f/9d0f8f61e47eba16e3036228e593589f.jpg?wh=4409x2480)

论文中也给出了Transformer的整体架构，我放在文稿中。这个图左侧对应编码器结构，右侧对应解码器结构，图中的N论文中设置为6，也就是上面图中我们展示的样子。总之， **编码器负责对输入序列进行抽象表示，解码器根据这些表示构建合适的输出序列**。

![](https://static001.geekbang.org/resource/image/cd/dc/cded231571065009204d514d43eba2dc.jpg?wh=4355x4500)

## Attention细节探究

Transformer是当下公认的处理时序任务的最强模型，最初提出是为了解决自然语言处理领域的问题，比如机器翻译等任务。最近几年，Transformer广泛应用在语音领域、计算机视觉领域等，表现不俗。特别是当下的AI绘画模型和GPT模型，也都依赖了Transformer模块。

在Transformer结构中，最关键的便是 **注意力模块**。事实上，AI绘画中也是复用了注意力机制来实现prompt对图像内容的控制。所以我们重点来拆解Transformer中的注意力模块。

### 序列、Token和词嵌入

正式学习注意力模块前，我们要先掌握4个关键的概念，这些概念共同构成了Transformer模型的基本组成部分。

首先是源序列。源序列是输入的文本序列。例如在机器翻译任务中，源序列就是待翻译的文本。

其次是目标序列。目标序列是输出的文本序列，例如在机器翻译任务中，目标序列代表翻译后的文本，通常为目标语言。

之后是Token（词符）。Token是文本序列中的最小单位，可以是单词、字符等形式。文本可以拆分为一系列tokens。举个例子，文本 Hello world 经过特殊的分词，可以被分为以下tokens：\[“Hello”, " world"\]。Token的词汇表中包含了所有可能情况，每个token预先被分配了唯一的数字ID，称为token ID。

最后是词嵌入（Word Embedding）。词嵌入的目标是把每个token转换为固定长度的向量表示，这些向量可以根据token ID在预训练好的词嵌入库（例如Word2Vec等）中拿到。在Transformer中，编码器和解码器的输入的都是序列经过token化之后得到的词嵌入。

### Self-Attention模块

你可能听过许多不同类型的注意力机制，例如自注意力(Self-Attention)、交叉注意力(Cross Attention)、单向注意力(Unidirectional Attention)、双向注意力(Bidirectional Attention)和因果注意力(Causal Attention)、多头注意力（Multi-Head Attention）、编码器-解码器注意力（Encoder-Decoder Attention）等概念。

这些注意力模块构成了当今AIGC领域的基础。不过你可以放轻松，死记硬背只会让你越来越混淆，跟着我的节奏来学习。这里传授你一个学习技巧，遇到新知识可以先从它们的作用和原理来入手。

注意力模块通常作为一个子结构嵌入到更大的模型中，作用是提供全局上下文信息的感知能力。下面的两张图是对Transformer中一层的编码器和解码器结构的简化。可以看出，这里用到的便是自注意力模块和交叉注意力模块。

![](https://static001.geekbang.org/resource/image/3f/f0/3f077b1b45c64b09c8b2ffd919a610f0.jpg?wh=4409x2480)

![](https://static001.geekbang.org/resource/image/95/77/9512d22c3d2ecf7dd85e7a8fdc5f8277.jpg?wh=4409x2480)

我们先来看最基础的自注意力模块，计算方法你可以看下面这张图。自注意力模块会计算输入序列中所有元素之间的相似性得分，再通过归一化处理得到注意力权重。这些权重可以视为输入元素与其他元素之间的联系强度。注意力模块通过对权重与输入元素做加权求和来生成输出，输出的向量维度与输入相同。

![](https://static001.geekbang.org/resource/image/87/b4/870f9109cb90f1b44cd71ff7c8153ab4.jpg?wh=5028x4256)

了解了基本原理，我们理解Transformer中自注意力模块的设计就会更加容易。

首先，我们通过三个可学习的权重矩阵$W\_{Q}$、$W\_{K}$、$W\_{V}$分别将模块输入序列投影成Q、K、V三个向量。Q代表Query，K代表Key，V代表Value。然后通过计算Q、K之间的关系，获得注意力权重，最后将这些权重与V向量相结合，得到输出向量。

整个计算过程你可以参考后面的伪代码。

```python
# 从同一个输入序列产生Q、K和V向量。
Q = X * W_Q
K = X * W_K
V = X * W_V

# 计算Q和K向量之间的点积，得到注意力分数。
Scaled_Dot_Product = (Q * K^T) / sqrt(d_k)

# 应用Softmax函数对注意力分数进行归一化处理，获得注意力权重。
Attention_Weights = Softmax(Scaled_Dot_Product)

# 将注意力权重与V向量相乘，得到输出向量。
Output = Attention_Weights * V

```

这里有三个小细节你需要注意一下。

1. Q和K的向量维度是相同的，比如都是d\_k，V和Q、K的向量维度可以不同，称之为d\_v。
2. 缩放因子Scale的计算方式是对d\_k开根号之后的结果，在Transformer论文中，d\_k的取值为64，因此Scale的取值为8。
3. 图中被标记为可选（Opt）的Mask模块的作用是屏蔽部分注意力权重，限制模型关注特定范围内的元素。你可别小看这个Mask模块，它便是自注意力升级为单向注意力、双向注意力、因果注意力的精髓所在！

了解了自注意力的计算，我们再来看交叉注意力的计算便不再困难。

自注意力和交叉注意力的区别你只需要记住一句话： **自注意力的Q、K、V都源自同一个输入序列，而交叉注意力的K、V源自源序列，Q源自目标序列，其余计算过程完全相同。** 对于Transformer这类编码器-解码器结构来说，源序列从编码器输出，目标序列从解码器输出。

### Encoder-Decoder Attention模块

编码器-解码器注意力（Encoder-Decoder Attention）模块是解码器中的一个关键子模块，实际上它是一个交叉注意力模块，点开下面的图你一眼便会发现这个模块的Q、K、V源自不同序列。

编码器-解码器注意力模块的目标是统合源序列和目标序列之间的关系，以便生成更准确的输出序列。图中我已经标记了编码器-解码器注意力模块、源序列、目标序列所在的位置。

![](https://static001.geekbang.org/resource/image/44/9b/44cb6cf9627cf9f134d997090fc6yy9b.jpg?wh=3155x2316)

这里有一个非常关键的点，编码器的输入可以是我们要翻译的文本、或者是我们问ChatGPT的问题，那么解码器的目标序列又要如何获得的呢？

在训练阶段，目标序列通常是从训练数据获取的。例如，在机器翻译任务中，训练数据包含源语言序列及其对应的目标语言序列。在训练过程中，目标序列会被输入到解码器，解码器同时根据编码器提取的特征表示（Encoder-Decoder Attention）去预测相应的输出序列。通常情况下，目标序列会以一个特殊的开始符号（如<start>）开始，以确保模型在生成序列时从预定义的起点开始。

在训练阶段，我们使用成对的数据来训练模型。例如，在机器翻译中，训练数据包括一种语言的句子（源语言）及其另一种语言的翻译（目标语言）。在训练过程中，我们把这些目标句子输入到解码器部分，让Transformer学会根据源句子生成正确的翻译。我们通常在目标句子的开头加一个特殊的开始符号，比如<start>，这样模型就能知道要从哪里开始生成句子。

在训练阶段，解码器输入的目标序列是从实际的训练数据中获得的，而不是解码器在前一步生成的输出。这意味着，即使模型在前一个时间步预测错误，训练阶段的解码器输入仍然会使用正确的目标序列。这种做法有助于训练模型更好地捕捉源序列和目标序列之间的映射关系，加速收敛过程。

在预测或推断阶段，目标序列使用训练时用到的特殊符号作为初始值。随后，解码器会生成目标序列，一次生成一个token。对于每一步生成的token，都会将其添加到目标序列中，并将这个扩展后的目标序列作为输入输送到解码器。这个过程会一直持续，直到生成一个特殊的结束符号（如<end>）或达到预定义的最大序列长度。 **这便是人们常说的自回归模型**。

知道了这些，编码器-解码器注意力模块的计算过程便非常容易理解了。首先基于源序列的特征计算得到K和V向量，然后从目标序列的表示（比如前一层的输出）中获得Q向量。之后的过程就和标准的自注意力一样了。

### 多头注意力机制的设计和优势

多头注意力（Multi-head Attention）是Transformer的工作里首次提出和使用的，它强化了编码器解码器的能力，你可以把它看作注意力模块的升级版。那么，它的工作原理是怎样的呢？

我们已经知道Self-Attention通过三个可学习的权重矩阵$W\_{Q}$、$W\_{K}$、$W\_{V}$分别将模块输入投影成Q、K、V三个向量。多头注意力机制设计了多个独立的注意力子层并行地计算，捕捉和融合多种不同抽象层次的语义信息。

具体过程是后面这三步。

1. 首先，将输入的Q、K和V向量线性投影到各自对应的子空间。这些投影会分别使用不同的$W\_{Q}$、$W\_{K}$、$W\_{V}$权重矩阵进行学习。
2. 然后，为每一个子空间计算注意力权重和加权向量，并将所有子空间计算出的加权向量拼接起来。
3. 使用一个线性层，将拼接后的结果再次投影成一个输出向量，该向量与输入向量具有相同的维度。

关于多头注意力机制，你可以看下面这张原理图进一步加强理解。

![](https://static001.geekbang.org/resource/image/70/d2/703e574b45033546eeeb700885970fd2.jpg?wh=4409x2480)

你也许会好奇，多头注意力机制不就是把自注意力机制重复了N次么？这样做有什么好处呢？

我用一个简单的类比来给你解释。假设一个团队有N个成员，每个成员专注于解决输入序列中不同类型的问题（例如不同领域的语义关系），然后将每个成员的结果结合起来，从而组合不同领域的知识。

多头注意力机制可以捕捉不同类型的依赖关系。每个Q、K、V都可以看作是不同子空间，多头注意力机制允许模型在不同表示的子空间中，捕捉序列中存在的多种上下文关系。例如，一个头可能关注词法关系，另一个头关注句法关系，还有一个头关注长距离的依赖关系。将这些从不同头获得的信息结合起来，模型就可以更好地理解输入序列。

![](https://static001.geekbang.org/resource/image/32/b3/32c300ff54a165747904efac7c25eeb3.jpg?wh=4456x4272)

### 更多注意力模块

说完了自注意力、交叉注意力、多头注意力，我们再来看看单向注意力、双向注意力和因果注意力，这三种注意力是通过Mask操作，也就是屏蔽部分注意力权重实现的。

![](https://static001.geekbang.org/resource/image/78/90/78a202fcd854b428f49b98yy965f8390.jpg?wh=4928x4261)

1. 单向注意力机制主要关注给定位置 **之前或者之后** 的上下文，从而捕捉输入序列中的单向依赖关系。比如，在看电影时只观察当前时间点之前发生的事件，电影甚至允许倒着放，只要是单向就行。
2. 双向注意力机制能够同时关注给定位置之前和之后的上下文，从而捕捉输入序列中的双向依赖关系。不带Mask操作的自注意力和交叉注意力都属于双向注意力机制。比如，我们已经看过整个电影，了解了所有事件情节。
3. 因果注意力机制 **仅关注给定位置之前** 的信息，因此在生成过程中通过Mask操作避免暴露之后的内容。比如，从头开始写一本小说，只能根据已经写了的情节合理地续写未来情节。

## Transformer vs LSTM

理解了这些复杂的注意力机制，让我们回到今天的主角Transformer。相信你已经清楚，Transformer不依赖于递归的序列运算，而是使用自注意力机制同时处理整个序列，这使得Transformer在处理长序列数据时速度更快，更易于并行计算。

自注意力机制允许Transformer直接关注序列中任意距离的依赖关系，不再受制于之前的隐状态传递。这样，Transformer可以更好地捕捉长距离依赖关系。

然而，需要注意的是，尽管Transformer在很多任务中表现出优越性能，但它的训练通常需要大量的数据，对内存和计算资源的需求通常较高。另外，LSTM和Transformer在特定任务上可能具有各自的优势，我们仍然需要根据具体问题和数据情况来选择最合适的模型。

## 总结时刻

这一讲比较烧脑，但这是你深入理解AIGC的关键一环，下次别人聊到注意力机制时，你也可以根据今天学到的知识发表自己的见解。

今天，我们深入探讨了Transformer的算法原理。包括编码器、解码器的设计思路，自注意力、交叉注意力的实现原理，这些都是Transformer工作推出之前就已经广泛使用的经典技术。

同时，我们还学习了多头注意力机制这项Transformer工作中提出的原创技术亮点。实际上，除了我们今天重点分析的注意力机制，Transformer中还有很多技术细节，比如位置编码、Feed-Forward神经网络、残差连接、层归一化等，感兴趣的同学可以课后做更多拓展学习。

关于这节课的知识回顾，你可以查看下面的知识点小结。

![](https://static001.geekbang.org/resource/image/06/e0/06ab7f1612470b355bc441461de04be0.jpg?wh=3600x2269)

## 思考题

如何改进Transformer的自注意力机制，提高它的效率，并减少计算资源需求呢？

期待你在留言区和我交流讨论，也推荐你把今天的内容分享给身边更多朋友。