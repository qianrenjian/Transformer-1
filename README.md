# Transformeri: 第一代Transformer

## Moses

[Moses创建一个翻译系统的基本过程记录，以后会按照每个过程详细说明，并给出每个步骤的参数说明](https://www.cnblogs.com/hitnoah/p/3942717.html)

## PyTorch ipython版本（针对自动翻译）

[参考1](http://nlp.seas.harvard.edu/2018/04/03/attention.html#encoder)

[参考2](https://arxiv.org/abs/1706.03762)

[code1](https://github.com/harvardnlp/annotated-transformer)

介绍Transformer比较好的文章可以参考以下两篇文章：一个是Jay Alammar可视化地介绍Transformer的博客文章The Illustrated Transformer ，非常容易理解整个机制，建议先从这篇看起；然后可以参考哈佛大学NLP研究组写的“The Annotated Transformer.”，代码原理双管齐下，讲得非常清楚。

### make\_model()函数

参数定义：

src\_vocab：源文单词数目；tag\_vocab：译文单词数目；d\_model：所有模型支层的输出维度；d\_ff：前馈网络隐层的神经元个数；h：MultiHead层并行的attention个数；dropout：Dropout层参数（前馈神经网络输出层和MultiHeadAttention层接一个Dropout层、Embedding层后也接Dropout层）。

基本结构定义：

包括MultiHeadAttention层、PositionwiseFeedFoward层、PositionalEncoding层、两个Embedding层（一个针对源文、一个针对译文）的定义。

网络结构定义，即EncoderDecoder结构：

* Encoder定义了多层EncoderLayer，论文中为6层，整体最后会接一个norm层。EncoderLayer：MultiHeadAttention层 + SublayerConnection层 + PositionwiseFeedFoward层 + SublayerConnection层（resnet + norm）。
* Decoder定义了多层DecoderLayer，论文中为6层，整体最后会接一个norm层。DecoderLayer：MultiHeadAttention层 + SublayerConnection层 + MultiHeadAttention层 + SublayerConnection层 + PositionwiseFeedFoward层 + SublayerConnection层。
* InputEmbedding层 + PositionalEncoding层，注意：Embedding层进行了改进，除以根号下d\_model。
* OutputEmbedding层 + PositionalEncoding层，注意：Embedding层进行了改进，除以根号下d\_model。
* Generator层：Linear层 + Softmax层，输出翻译为某个词的概率。

### SimpleLossCompute类

loss和优化器定义：

* generator：网络结构中的Generator层计算预测结果。
* criterion：结合了Label Smoothing方法，用Label Smoothing类中的接口计算loss。
* opt：NpamOpt类的对象，根据梯度来更新权重。优化器，参数model\_size、factor和warmup和step配合控制学习率的变化。optimizer表示传入的优化器，一般使用Adam优化器。

### run\_epoch()函数

参数定义：

data\_iter：定义数据生成方式；model：定义的模型对象；loss\_compute：定义的loss和优化器对象。

基本功能：

读取batch数据，通过model forward得出结果，通过loss\_compute计算出loss并且进行优化；然后输出loss。

## PyTorch版本

[参考1](https://github.com/jadore801120/attention-is-all-you-need-pytorch)

### 要点

### 一些工具

利用tokenizer.perl进行词例化，作用是将平行预料中的每个词进行词例化。

[Moses创建一个翻译系统的基本过程记录，以后会按照每个过程详细说明，并给出每个步骤的参数说明](https://www.cnblogs.com/hitnoah/p/3942717.html)

### 一些mask

Encoder阶段：non\_pad\_mask表示对输入句子进行mask，padding的数据不参与计算。self\_atten\_mask表示在attention阶段进行mask，对padding的数据不参与attention计算。

Decoder阶段：non\_pad\_mask表示对输入句子进行mask，padding的数据不参与计算。self\_atten\_mask表示在第一个attention阶段进行mask，对padding的数据以及在此位置之后的单词（避免将来的词影响当前的预测结果）不参与attention计算。dec\_enc\_atten\_mask表示在第二个attention阶段进行mask，对超出源句子长度的encoder的结果不参与attention。。

### Beam Search

这是其中比较晦涩难懂的一部分。

Beam.py：

```
self.next_ys = [torch.full((size,), Constants.PAD, dtype=torch.long, device=device)]
self.next_ys[0][0] = Constants.BOS
因为刚开始只begin都是Constants.BOS，所以只把第一个设为Constants.BOS，其他的设为padding表示不会采用其他的预测结果。

advance函数也会判断len(self.prev_ks)来采取不同措施。
```

数据结构如下

* self.size表示search的范围，保存前size个得分高的序列。
* self.\_done表示该序列是否解码完成。
* self.scores保存当前情况下前size个高的得分。
* self.all\_scores保存所有的得分。用list结构保存。
* self.prev\_ks保存当前的序列来自前一序列的哪个seach得出的结果。
* self.next\_ys保存每一时刻前k个得分高的单词id。

函数如下

* advance函数：传入self.size * word的二维数据。然后跟当前的得分相加。通过view展开为一维，选出top(size)个得分高的预测结果。将结果保存至all\_scores中，更新self.scores。得出当前top(size)结果所处的前面哪一个search，存入self.prev\_ks，将当前top(size)结果所预测的size个单词id，存入self.next\_ys中。判断当前得分最高的序列的预测结果是否是EOS，如果是说明该序列预测完成，更新self.\_done。
* get\_current\_state函数：得出当前最优的size个序列。通过调用get\_tentative\_hypothesis函数完成，这个函数每次调用一次get\_hypothesis函数获取第k个得分高的序列。

Translator.py：

数据结构如下

* inst\_dec\_beams：为每一个源序列创建一个beam结构用来beam search。
* active\_inst\_idx\_list：目前还没有预测完成的源序列id组成的list。
* inst\_idx\_to\_position\_map：目前还没有预测完成的源序列的id和它在list中的位置组成的dict。

**一些关键思考**

思考一下为什么要有inst\_dec\_beams，显而易见，每一个源序列都要beam search，但有些源序列已经预测完了就不需要继续predict乃至beam search了。所以需要active\_inst\_idx\_list来记录当前未完成的源序列id（这个是针对batch size大小数据的id），需要inst\_idx\_to\_position\_map（dict结构，元素为batch size大小数据的id: 当前剩余预测序列在此时src\_seq、src\_encoder中的id）中的value来记录当前该删除哪些src\_seq和src\_encoder。

函数如下

* translate\_batch表示对一个batch的序列进行翻译。
* beam\_decode\_step：为源序列预测每一时刻的结果。传入数据为inst\_dec\_beams（**预测过程中不会变化**）、len\_dec\_seq（当前预测的时刻，每次加1）、src\_seq, src\_enc, inst\_idx\_to\_position\_map（这三个参数每一时刻都会变，通过collate\_active\_info进行更新）。首先构造dec\_seq和dec\_pos；然后利用decoder预测，获取最后时刻的预测的结果返回；通过collect\_active\_inst\_idx\_list更新active\_inst\_idx\_list，这个阶段调用beam类的advance函数进行beam search结果更新。
* collate\_active\_info：每一时刻之后更新src\_seq, src\_enc, inst\_idx\_to\_position\_map，生成新的。

### 前期准备

数据下载、利用Moses的tokenizer.perl脚本处理数据，分词过程（**将标点符号也分开**）、利用preprocess.py构造.pt后缀的数据（只处理了训练集和验证集）（处理后的数据包括训练集和验证集的数据转化为数字集合，生成源语言和翻译语言的词典）。

数据处理过程如下：

1.preprocess.py将训练集和验证集转化为.pt后缀的数据，这个数据结构为：

```
    data = {
        'settings': opt,
        'dict': {
            'src': src_word2idx,
            'tgt': tgt_word2idx},
        'train': {
            'src': train_src_insts,
            'tgt': train_tgt_insts},
        'valid': {
            'src': valid_src_insts,
            'tgt': valid_tgt_insts}}
```

2.train.py中通过torch.load函数读取刚刚保存的数据，然后利用读取的数据分别对train和valid构造TranslationDataset对象，然后利用TranslationDataset对象构造DataLoader对象。然后我们就可以用DataLoader对象进行迭代。

构造DataLoader对象加入了collate\_fn参数，可以利用这个函数对读出的数据进行自定义的一些操作。

### train.py 训练过程

构造Transformer类，ScheduledOptim优化器类，然后调用train函数进行训练。

cal\_performance函数：调用cal\_loss函数计算logloss，计算n\_correct来统计预测正确的单词数量。

### translate.py 验证过程

构造Translator类，Translator内部构造了Transformer类，调用translate\_batch函数来处理每batch的数据。

处理一个batch的数据在encoder阶段完成后，decoder阶段运行一次仅仅预测此时刻的翻译结果（如果目标句子最大长度为100，就要运行100次encoder；每次encoder传入当前得到的所有目标词的预测结果和此时的mask，mask的目的是把该预测词之后的词屏蔽掉）。

预测阶段采用**beam search**的方法，同时计算loss也采用了**label smoothing**的方法。

## 参考

[谁能解释下seq2seq中的beam search算法过程?](https://www.zhihu.com/question/54356960)

[深度学习：自然语言生成-集束搜索beam search和随机搜索random search](https://blog.csdn.net/pipisorry/article/details/78404964)

[GoogLeNet的心路历程（四）里面有label smoothing](https://www.jianshu.com/p/0cc42b8e6d25)

## TensorFlow版本

[参考1](https://github.com/Kyubyong/transformer/blob/master/modules.py)

### 数据准备

prepro.py：构建词表文件

data\_load.py：加载数据。load\_train\_data函数和load\_test\_data函数均调用create\_data函数来构造数据，其中X、Y是转化为词id的数据，Sources、Targets是词和词之间由空格隔开的句子。

get\_batch\_data是构造TensorFlow输入的部分。采用tennorflow数据读取机制，tf.train.shuffle\_batch函数。这样源源不断输入数据，不需要构造placeholder。

### train.py

仅仅Decoder阶段的第一个self\_attention有mask。

感觉mask相比较PyTorch版本有点少。

## TensorFlow + Keras版本

[参考1](https://github.com/Lsdefine/attention-is-all-you-need-keras)

### 数据准备

Just preproess your source and target sequences as the format in en2de.s2s.txt and pinyin.corpus.examples.txt.

大概是“源语言 + "\t" + 目标语言”格式，其中源语言和目标语言词或字或标点符号之间以空格隔开。

# Universal Transformer: 第二代Transformer

[谷歌Transformer模型再进化，图灵完备版已上线](https://www.leiphone.com/news/201808/1nhPCi9jWWNGv6aw.html)

[Universal Transformers详解](https://zhuanlan.zhihu.com/p/44655133)

# Transformer-XL: 第三代Transformer

TRANSFORMER-XL: ATTENTIVE LANGUAGE MODELS BEYOND A FIXED-LENGTH CONTEXT

https://github.com/kimiyoung/transformer-xl

[transformer-xl](https://daiwk.github.io/posts/nlp-transformer-xl.html)

[谷歌开源超强语言模型 Transformer-XL，两大技术解决长文本问题](https://zhuanlan.zhihu.com/p/56027916)

[ICLR 2019 遗珠？加大号变形金刚，Transformer-XL](https://www.leiphone.com/news/201901/rHqmq4BECBamv7Vz.html)

[CMU和谷歌联手放出XL号Transformer！提速1800倍 | 代码+预训练模型+超参数](https://zhuanlan.zhihu.com/p/54909623)

## 模型架构

### 递归机制（recurrence mechanism）

为了解决固定长度上下文的局限性，我们在Transformer架构中引入一种递归机制（recurrence mechanism）。除了实现超长的上下文和解决碎片问题外，这种递归方案的另一个好处是显著加快了评估速度。

### 相对位置编码方案（relative positional encoding scheme）

在标准的Transformer中，序列顺序的信息，都是由一组位置编码提供，每一个位置都有绝对的位置信息。但将这个逻辑应用到重用机制中时，会导致性能损失。

这个问题的解决思路是，对隐藏状态中的相对位置信息进行编码。从概念上讲，位置编码为模型提供了关于应如何收集信息的时间线索，即应该在哪里介入处理。

以相对的方式定义时间线索，将相同的信息注入每层的注意分数，更加直观，也更通用。

基于这个思路，可以创建一组相对位置编码，使得重用机制变得可行，也不会丢失任何的时间信息。

# Character-Level Language Modeling with Deeper Self-Attention

[基于深度self-attention的字符集语言模型（transformer）论文笔记](https://blog.csdn.net/qq_41664845/article/details/84389286)

