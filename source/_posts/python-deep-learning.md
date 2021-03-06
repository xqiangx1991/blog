---
title: "《Python深度学习》读书笔记"
date: 2019-11-25
mathjax: true
---

## TODO
1. 深入了解**SeparableConv**、**DepthwiseConv**、**BatchNormalization**的计算公式和操作。
2. 如何对模型进行集成。找一些kaggle的例子。
3. word2vec算法是什么
4. 循环神经网络中使用dropout的原理
5. `L-BFGS`算法是什么

## 第四章 机器学习基础
现在`dropout`操作用的很多，但是我之前全部都理解错了，在训练的时候加上`dropout`，它会屏蔽一部分的节点，在测试的时候不会屏蔽，但是输出的时候要乘以`dropout`的比率系数。（这个是为什么？？测试的就是就是为了检验整个模型的效果啊，也不训练参数，为啥要乘以比率）

## 第五章 深度学习用于计算机视觉
本章对我来说收益最大的是卷积神经网络的可视化。主要有三种可视化，1. 是把每一层的激活层的输出可视化，它表示经过神经网络之后，每一层的输出结果是什么。从这个可以看出来，前面的层主要是一些边缘探测器，它几乎保留了原始图像中的所有信息，随着层数的增加，提取的信息越来越抽象，越来越难以直观的理解，还有一种现象就是随着层数增加，越来越多的输出是空白的，也就是这些过滤器没有找到图像中符合的编码信息。2. 是可视化过滤器，我最开始的想法是，过滤器不就是每个卷积网络的kernel这些吗？这些不就是过滤器吗？但是这个有点过于底层，其实我们最想知道的是某一层的整体的过滤效果，而不是单个的，单个的其实很难看出什么东西。所以这里用了很巧妙的一种方法就是，我构建随机图像，然后经过过滤器，那么，当我的输入和过滤器的模式一样的时候，这样的响应是最大的，有点像模版匹配。可以利用梯度上升法，将这一层的输出相对于输入求导，然后沿着输出值最大的方向改变输入的值，使得最后的输出最大，最后得到的输入图像就是过滤器了。这部分能够看出的信息其实和第一部分是差不多的。3. 是类激活的热力图，也就是每一个像素，它预测是属于哪一种类，预测的概率是多少，这样可以看看哪些像素比较重要。该方法基于 ***Grad-CAM: visual explanations from deep networks via gradient-based localization*** 论文。主要的思路如下，卷积层的输出其实代表的就是原始图像中剩余的信息，剩余的信息就是比较重要的信息，重要性就体现在了特征图的值上，但是输出的通道很多，有512层，有128层，怎么把这些层合并成一层，然后映射到输入图像上呢。不同的层对于最后的预测结果的影响是不同的，那么按照这个影响的权重进行相加就是合理的，因此可以求出特征图中的每一层相对于最后预测的类别的梯度代表着它的重要性系数，然后把卷积层的输出进行叠加就可以了。

在`keras`中，其实并不是模型为导向的，平时写模型，都是基于模型的思维在计算梯度、loss函数之类的，其实还可以基于`function`来计算，通过指定输入和输出的`Tensor`，可以生成`function`，然后和模型中用到的计算思路一样，每一次输入输入，得到输出。

## 第六章 深度学习用于文本和序列
### 处理文本数据
> 将文本分解而成的单元（单词、字符、n-gram）叫做标记（token），分解的过程叫做分词（tokenization），再将分词的结果进行数值向量化，数值向量化的方法有**one-hot**编码和**token embedding**

在图像相关的处理中，我们也会对标签进行one-hot编码，这种编码对于token比较少的情况是比较适用的，一旦量大了就会形成稀疏矩阵，这个不太好处理。

所谓的token embedding其实是对原始的toekn进行进一步的加工，我觉得可以理解为在one-hot的基础上对信息进行进一步的降维。

词向量位于词向量空间中，不同领域的词向量空间的表示是不一样的，比如法律文书和日常用语。**词向量之间的距离能够反应出不同词之间的语义差别**，比如精度和准确度，有时候是可以互换的，我们就希望它们的向量距离也非常小。同时，我们还希望**词向量是有方向的**，比如cat、dog、wolf、tiger，从cat到tiger的向量与从dog到wolf的向量相等，这个向量可以解释为从宠物到野生动物向量。当然还可以有其他的角度。

根据不同的目的我们会尝试去构建不同的词向量空间。

但是有些时候我们没有足够的数据去训练任务相关的词向量空间，这个时候就可以复用已有的结果了，因为很多的较为底层的特征是通用的。比如word2vec、GloVe等。

### 循环神经网络
#### 简单的RNN
**前馈网络**处理数据的前提假设是数据都是独立的，数据和数据之间是没有什么关系的。但是对于序列数据来说，前面的数据是可以作为后面数据的前提的，数据之间是存在一定关系的。这就需要**循环神经网络**，它其实就是除了当前的状态，还需要加上前一时刻的状态，简单来说就是下面的公式

$$
output_t = tanh(dot(W, input_t) + dot(U, state_t) + b)
$$

对于循环神经网络而言，它的返回值有两种形式，一种是把产生的中间结果也都返回，比如`$t_1$`到`$t_n$`，中间有`$t_2$`、`$t_3$`等等都返回，另一种就是只返回最后的`$t_n$`的结果。在多个循环网络进行叠加的时候，前面的训练网络要把所有的输出都返回，最后一个循环看情况要不要把所有的都返回。

上述的公式中存在一个问题，就是我只加入了前一时刻的信息，而随着网络不断往后，会出现梯度消失的问题，难以记住较为以前的信息，无法学习到长期依赖，没有什么实用价值。

#### LSTM
因此出现了**长短期记忆(LSTM)**算法。

理解**LSTM**算法是本章的重点。

通过[这篇文章](https://colah.github.io/posts/2015-08-Understanding-LSTMs/)可以很好的理解整个的工作原理，其实最本质的就是在上面的RNN网络的基础上，怎么把长期信息也加上去，不会把远点的信息给忘记。

首先在传统神经网络中，我们把输入数据输入到网络层中，产生结果，然后到下一层中

$$
output_t = tanh(W_{ft} \dot x_t + b_t)
$$

然后为了把前面的信息加进来就引入了上一个输出结果的状态，构建了简单的RNN网络。

$$
output_t = tanh(W_{ft} \dot ([h_{t-1}, x_t]) + b_t)
$$

但是这种还不够，还需要引入长期信息的记忆能力，上面的功能只是把上一层的信息传递到后面一层去，能力还是不够。所以LSTM在这个基础上增加了一条线路，可以把很远的地方的信息也输送过来。在这个基础上构建了更为复杂的LSTM网络。

现在就有这么几条线了：输入(`$x_t$`)、前一时刻的状态(`$h_{t-1}$`)、前序数据输送带(`$C_t$`)、输出(`$output_t$`)

但是每一条线的信息都需要判断是否要采用，LSTM里面用了很多的所谓的`门电路`，保留这部分的信息还是过滤这部分的信息。

首先是前一步的`$C_{t-1}$`的信息门电路，`$f_t = \sigma(W_{ft} \dot [h_{t-1},x_t] + b_{ft})$`，这个会产生一个`$[0,1]$`范围内的值，然后和`$C_{t-1}$`相乘，可以控制这条信息流传递下去的量，这是安装的第一个阀门。

然后是属于该层的层操作，输入输入数据和前一时刻的状态，输出该层的运算结果，但是这个也得加一个阀门，来控制当前层提取到的信息保留多少，这部分产生的数据会和`$C_{t-1}$`层的数据相加，得到`$C_t$`。

第三个阀门就是控制输出，`$C_t$`产生以后一方面进入前序信息携带通道，另一方面再经过计算生成当前时刻的状态，这个状态还得再加一个阀门，控制它最后产出`$h_t$`。

这就是LSTM网络了，**本质上就是在简单的RNN基础上加了三个阀门**

在用算法解决问题的时候，应该尽可能构建一个简单的模型先，然后以这个模型为基准，不断构建复杂的模型，复杂的模型至少要比基准模型好才可以。

在本书的观察耶拿天气数据中，首先提出了基于简单假设的方法，也就是温度时间序列是连续的，第二天的温度理论上会接近当天的温度，然后在天的角度也有周期性的波动，因此第二天的温度就等于今天的温度。然后提出基于深度学习的全连接的解决方案，结果发现第二种方法和第一种方法差不多。也就是说，如果数据到目标是一个简单且表现良好的模型，但是基于深度学习这个更复杂的模型却没有找到更简单的模型，理论上更简单的模型是包含在模型空间中的，这里面的原因是，我们并没有把更简单这个假设包括在训练过程中，所以很难学习到简单且性能良好的模型。

在循环神经网络中，如果和卷积神经网络中一样使用，会妨碍训练过程，并没有什么用处。循环神经网络中怎么加dropout

#### 双向RNN
如果说一种序列，并不是强前向的，那么可以考虑把反向数据输入进行训练，说不定能提升效果，但是强正向的数据就没啥用了。

### 用卷积网络处理序列
2D的卷积神经网络提取的是空间的信息，同样，可以用1D的卷积神经网络去提取序列信息，这个所谓的提取本质上其实就是信息的压缩，从冗余的低级信息中抽象出更高级的信息，1D可以依然保持信息的顺序，但是这种顺序是非常粗浅的顺序，对于一些简单一点的任务还好，复杂的就不行了。不过书中写道在音频生成和机器翻译领域取得了巨大的成功，这个就不太理解了。小型的1D卷积神经网络有个好处就是计算代价相比于RNN来说很低，因此，其实可以把1D卷积神经网络和RNN进行结合，先用1D卷积神经网络从低级信息提取出高级信息，就跟token embedding的思路一样。


## 第七章 高级的深度学习最佳实践
### 多模态模型构建
在前面几章的，构建模型都是用的`Sequential`模块，把一层一层的`layer`叠加在一起，只有一个输入一个输出，但是实际上在构建模型的时候，常常会遇到多输入或者多输出的情况，还有一些中间比较复杂的模块，这个时候就要用到`Model`模块了。在`keras`中有两种模块，分别是`Mode`和`Network`，`Network`只包括网络的结构，而`Model`在这个基础上还包括了训练的部分。之前写代码我只关心整个的网络结构，训练的部分都是手工进行，这样写代码就有点不优雅。之前之所以手工写，是因为官网的例子都是很简单的例子，多输入多输出的情况都很少，在多种输入和输出的情况下，应该怎么计算loss，怎么把不同的输入和不同的输出进行对应，这点我没有想明白。其实很简单，就是每一层都有名字，输入输出层也有自己的名字，在输入对应数据的时候，通过`dict`类型进行赋值或对应。如下代码所示，同理可得，也可以分别计算loss。

``` python
brancha = Input(shape=(300,300,3), name="brancha")
branchb = Input(shape=(300,300,3), name="branchb")

xa = Dense(32,3,activation="relu")(brancha)
xb = Dense(32,3,activation="relu")(branchb)

x = Concatenate()([xa, xb])

outputa = Dense(32,3,activation="softmax", name="outputa")(x)
outputb = Dense(32,3,activation="softmax", name="outputb")(x)

model = Model(inputs=[brancha, branchb], outpus = [outputa, outputb])

model.fit({"brancha":inputa,"branchb":inputb},
          {"outputa":outputa, "outputb":outputb},
          epochs=10, batch_size=10)
```

另一方面，我从最开始写模型是通过继承`Model`这个类，然后分别写好每一层，设置好参数，然后在`__call__`中调用这些定义好的层。这在面向对象的逻辑中是成立的，但是在这里实际操作的时候就不好操作，首先，不同模型之间的复用就会变得很麻烦，会行程模型一层嵌套一层，最后把整个代码弄的很复杂，同时这么写的话，就需要考虑自己推断从输入的shape到输出的shape，同时很多的函数也没办法调用，比如`summary()`等。所以现在我转换了策略，我依然继承`Model`，但是我在初始化的时候，通过构建输入与输出，然后把构建的`inputs`和`outputs`传入到父类中去构建出整个的模型。这样，既能根据不同的模型加入不同的参数，也能用现成的keras模型。

### 有用的结构和卷积
在做卷积操作的时候，其实有两步：1.是在同一层中，用同一个卷积核遍历该特征层；2.在不同的特征层中进行运算求和。前者叫空间特征，后者叫通道特征。**1x1卷积**就是计算的通道特征，**SeparableConv**和**DepthwiseConv**就是计算的空间的特征。

上述提到的非线性的模块包括了Inception模块和residual模块。residual模块之所以效果这么好一方面是因为加了shortcut模块，因为在进行卷积操作的时候，其实是把一些特征丢失的，很多丢失的信号是没办法复原的。比如把音频中的低频信号去掉，就没办法恢复了。shortcut模块可以把前面的信息再加回去。另一方面这个通道也能够在反向传播的时候发挥作用，减缓出现梯度消失的情况。

书中还提到一种情况，虽然有两个输入，但是两个输入的属性一样，比如要评估两个句子之间的语意相似度，因为两个输入的句子是可以互换的，他们的相似度是对称关系，不能说用两个网络分别去处理，因为他们是把句子映射到同一特征空间，所以会用同一个模型去分别走两个句子，然后得到的结果再去评估句子的相似度。处理两个摄像头的特征也是同理。只不过这部分在实际的应用中我还没有见到。

### 回调函数
回调函数的作用其实很多方面，有一点我没有想到，就是可以设置提前训练结束还有根据loss函数，梯度等动态调整learning rate等。

### 进一步提升模型的性能
#### 批处理
对输入数据进行标准化，可以剔除一些无关紧要的特征的影响，专注于整个图像的分布不同进行推理得到结果。从每一层的特征图出发，其实和从输入层出发一样，也需要进行标准化，可以怎么标准化，之前在ResNet中用了一组RGB的数字是统计了ImageNet的图片得到的均值和方差值。所以在`BatchNormalization`的计算中，不断更新均值等，在训练过程中保存已读取的每批的数据均值和方差的指数移动平均值，它有利于梯度传播，因此运行更深的网络。

#### 超参数优化
超参数优化需要注意的点就是，我们要用验证集来调整，不要基于测试集，不然就没啥用了，容易出现过拟合。

文中也推荐了超参数优化的库：**Hyperopt**，它的内部使用Parzen估计器来预测哪组超参数更好。还有一种和keras结合的**Hyperas**库，也可以尝试一下。

#### 模型集成
模型之所以集成，是因为不同的模型从不同的侧面反应出事物的不同方面。就像盲人摸象，一个模型摸到一个地方，另一个模型摸到另一个地方，然后集成，最后勾勒出整个大象的形状。但是也不能都摸到大象的鼻子，这样的集成是没有用的，必须要保证模型的多样性，不同的模型，偏差向不同的地方。

**但是，怎么去评估各个模型的得分，效果呢？**，去kaggle上多找一些例子看看。

## 第八章 生成式深度学习
### 使用LSTM生成文本
在本节中，通过LSTM网络构建模型，在网络的最后是一层softmax层，输出各个单词的概率，但是并没有直接使用这个概率，而是在这个基础上对这个概率进行了随机采样，也就是根据概率值生成样本，然后选择出现次数最多的那个，出现次数的多少取决于概率，但是每次的结果不一样，只能说出现某个值的概率更大一些。然后又引入了一个参数`$t$`，对原来的概率分布进行重排，可以用来控制生成的字符*更随机*或者*更不随机*，所谓的*更不随机*就是更加符合我们从训练样本中得到的样本空间的分布。

``` python
def sample(preds, t = 1.0):
    preds = np.asarray(preds).astype('float64')
    preds = np.log(preds) / t
    exp_preds = np.exp(preds)
    preds = exp_preds / np.sum(exp_preds)
    probas = np.random.multinominal(1, preds, 1)
    return np.argmax(probas)
```

倒数第二行代码其实是根据`preds`的分布来采样，会得到和`preds`的长度一样的一个数组，数组里面的数字代表的是出现在这个位置上的样本次数，我们选择出现样本次数最大的这个索引，然后找到相应的字符。

上述的这个参数`t`为什么能够改变原来的概率分布，**`t`的取值范围是(0,1]，值越大，那么得到的熵越大，越不确定**。

我们来看下图

<p align="center"><img src="https://joeltsui-blog.oss-cn-hangzhou.aliyuncs.com/expo-function.jpg" alt="指数函数" title style/>

当我们进行`$\log$`运算以后，概率越大的部分，生成的值越小，最后处以`$t$`以后值就越大，概率大的值和概率小的值之间的差距会随着`$t$`变小而变大，然后在作用一个`$e^x$`操作，又把数值变到`$[0,1]$`之间。总结来说就是`$t$`很小的时候，原来概率越小的部分，在变换以后概率会变得更加小，大概率值与小概率值之间的差距就会变大，也就是说越来越偏向于大概率的值，最后输出的结果也就越确定，熵就会越小**

### DeepDream
DeepDream的原理其实和之前的神经网络的可视化是差不多的，就是说神经网络通过数据的学习到了模型的参数，这个参数能从图片中提取出相关的信息。通过ImageNet训练出来的模型有非常多的分类，因此有很多针对不同物体的不同的核，我们把这些信息提取出来然后画到原来的图上去。本质上还是那一套，只不过现在是用的层数更多了，并且在不同尺度上对特征进行了综合，然后为了保证清晰度，把图片放大缩小后又不断把细节填充进去。训练过程也是一样，原来的训练过程是在一张白纸上通过梯度上升，求的响应最大值，现在也是利用梯度上升。loss函数直接求的就是指定的几个输出层，毕竟最后的结果也是求得这几层的最大响应。**为了避免出现边界伪影，只包含非边界像素**，这种做法在好几个地方都看到了。

### 神经风格迁移
风格迁移的过程分为两步，一部分是我们要保存我们的目标图像的内容，这个目标图像也就是我们的输入图像，我们希望在这个图像上进行操作，同时呢，我们又想把参考图像的风格加入到我们的目标图像中，这两个分别就是图像的内容和风格的部分，抽象出来就是要保证内容损失和风格损失最小，内容风格是和目标图像进行对比，风格损失是和参考图像进行对比。内容比较好理解，我们可以选取更深层的网络层，浅层表示的是局部的浅特征，深层次的网络模型表示的是全局的更抽象的图像内容。风格其实就是图像的纹理，而所谓的纹理就是图像内部的相互关系。风格也可以在不同的纬度来表现，因此，风格的损失我们选取多层网络输出进行计算。

内容损失比较好计算，直接比较在深层中的网络输出的值。风格的损失要求出一张图像内部的特征关系，这里用到了所谓的Gram Matrix，其实就是求水平向量两两之间的相似度，这就是一张图片的内部关系，然后比较一下生成图像和参考图像之间的差值。**同时提到为了促使生成的图像具有空间连续性，避免结果过度像素化，增加一个总变差损失，其实就是希望前后两个像素差别不要太大，变化具有连续性，分别在`x`和`y`方向进行计算。**不同的损失值有不同的系数，用于控制更偏向于哪个方面。

我们在平时中用到的以`SGD`方法，这在数据量比较大的时候用处比较大，数据量比较小的时候可以选择`L-BFGS`算法。

### 用变分自编码器生成图像
变分自编码器(VAE, variational autoencoder)，就是把图片映射到一个向量空间，这个向量空间通过平均值和方差来表示，这个过程称之为**编码器**，然后在这个向量空间中我们选择生成的均值和方差值（这个方差值增加一些噪声），再把它还原成原来的图片，这个过程称之为**解码器**。这个就是整个模型的组成部分。这个的训练过程就是只有最后比较一下生成的图像和原始的图像之间的差距。不需要外部额外计算的其他loss，是一种不需要标注集的训练。

VAE得到的是高度结构化的，连续的潜在表示，比如沿着一个横截面移动，可以实现以连续的方式显示一张起始图像缓慢变化为不同图像的效果。

主要原因是最后的潜在空间压缩的比较厉害，所以有种类似于主成分析中，生成的分别就会趋向于连续变化的主要成分分量。

### 生成式对抗网络
生成式对抗网络(GAN, generative adversarial network)，它主要也是两个部分，一个是**生成器网络**，另一个是**判别器网络**，生成器网络从随机空间中选取随机数生成一幅图像，然后把这些图像和真实图像混在一起输入到判别器网络，判别器网络去判断是真是假。

$$
gan = discriminator(generator(x))
$$

GAN网络的训练是一个动态的过程，生成器会变化，判别器会变化。很难训练，需要对模型架构和训练参数进行很多的微调。GAN的训练方法有一些技巧，详情见该书的P259.

GAN网络在训练的过程中要保持判别器的参数不变，判别器的训练是单独训练的，和GAN网络分开。

## 第九章 总结
总的来说，现阶段的深度学习其实是局部泛化，也就是说我喂进去什么数据，只能基于这个进行简单的泛化，比如输入识别人的数据，就不能识别其他动物。而人类的意识其实是极端泛化，我们不需要反复的看很多数据，能够通过**抽象**和**推理**进行更宽泛的泛化，所以，如果要实现人类一样的智慧，现在的这种方法肯定是不行的。

想要实现模型的复用其实很难，模型结构可以复用，可是模型参数怎么办，模型参数是根据不同的任务训练的，是没办法复用的，所以现阶段的这种形式肯定是初级阶段的。