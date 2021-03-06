**地址**：<https://arxiv.org/pdf/1802.09778.pdf>
**主要思路**：这篇论文虽然是17年投的，19年TPAMI发表，但是论文的解决角度还是值得学习和借鉴的。从题目可以看出，这篇paper主要利用混合的监督信息，即强监督信息（包含目标边界框注释信息）和弱监督信息（只有图像标签信息）。作者把从源（强监督）域中学习到的目标知识迁移到目标（弱监督）域中。

### **背景**
强监督目标检测虽然在一些数据集上取得了显著的效果，比如PASCAL VOC和COCO，可是，现实世界中的目标类别成千上万，用强监督的方法就需要获取这些类别的边界框注释信息，这样的工作量太大且耗费人力。这样弱监督目标检测就应运而生，训练这样的目标检测器，我们只需要图像的标签信息（只告诉图像中存在的目标类别信息），并且这种数据很容易通过网络获取。

### **出发点**
由于弱监督只有图像标签可以利用，所以弱监督目标检测常常被当作多事例学习（multiple instance learning（MIL））问题。但是这样就存在一个很大的问题，我们只有图像标签可是我们干的是目标检测的事，所以检测器无法得到目标区域的清晰定义，进而导致了这种方法训练出来的检测器可能包含如下图中所示的目标背景，或者只包含目标的一部分。

![检测失败的效果图](https://upload-images.jianshu.io/upload_images/18102452-512ac555593b1c7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### **方法**
利用混合监督学习来解决弱监督中存在的问题。那森么是混合监督呢？就是你有一部分类别的数据是强监督的（称为源域$S$），另外一部分类别数据是弱监督的（称为目标域$W$）。并且这两份数据之间的类别没有交叠。而存在一种情况：一张图片中包含多个类别目标，这些目标分别属于这两个数据集，那么这张图片同时被两个数据集所有，可是对应的类别的目标的标注信息不同。

 #### 1. ***方法总览***
 
![方法结构图](https://upload-images.jianshu.io/upload_images/18102452-bf8112f23f9397d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可以发现论文方法主要分为两个部分：**1**:两个数据集一起训练，学习域不变（domain-invariant）的目标知识，即可以学习到恰好框住完整目标的能力；**2**:利用学习到的域不变目标知识辅助弱监督学习，从而使学习到的检测器能定位到完整目标。

论文中提到第一部分学习到的域不变目标知识拥有两个重要的特性：**(1)** 类别独立，能够很好的推广到未知的类别；**(2)** 目标敏感，能过可靠的剔除干扰边界框（包含背景或者只包含目标的一部分）。

#### 2. ***方法详述***
#####  2.1 域不变目标学习
通过方法结果图，我们可以看到这个训练模型包含两个分支：(1)目标预测 (2)域分类。从分支名字上，你们应该已经猜到作用了。(1)分支用于辨别目标框，(2)分支用于辨别图像属于哪个域。网络主要是靠损失函数指导学习，前面特征提取层我们就不多描述了，可能不了解的会问，这些框框是如何来的呢？其实结构图中的ROI模块其实就是Fast-RCNN中的Roi-Pooling，这些框是预先用选择搜索（select-search，SS）算法提前准备好的（我们称为proposals，可以翻译为候选框）。接下来我们主要分析这两个分支。

###### 2.1.1 目标预测
输入是$S$中的proposals经过特征提取网络得到的特征向量，输出是维度为2的向量，用于判断是不是目标。
首先给出损失函数：
$$
L_{obj}(W)=-\frac{1}{n}\sum_{i=1}^{n}[y_i^{obj}log(p_i))+(1-y_i^{obj})log(1-p_i)]
$$
公式中符号解析：$y_i^{obj}$表示边界框的标签，通过与ground-truth（就是目标的真实边界框，人为的标注信息）计算intersection-over-union (IoU)得到，即两个框的相交面积/并集面积。如果IoU大于0.5, $y_i^{obj}=1$，即正样本。如果在[0.1,0.5)之间，$y_i^{obj}=0$，即负样本。在一张图片中有很多冗余的框，肯定正样本框远远大于负样本框，为了平衡正负样本比例，限定选取正负样本比例为1：3总数64的边界框计算损失。$p_i=\frac{1}{1+exp(-G_{obj}(f_i))}$（sigmoid函数）, $G_{obj}$表示这个分支，$f_i$表示第$i$个边界框的特征向量，其实这个公式可以理解为：$G_{obj}(f_i)$就是第$i$个边界框的一个打分$s$，则公式可以等效于$p_i=\frac{1}{1+exp(-s)}$。


###### 2.1.1 域分类
论文中的domain-invariance就是通过这个分支实现的。
不同于目标预测分支，这个分支的不仅考虑了$S$中的边界框，也考虑了$W$的边界框，输出也是一个维度为2的向量，就是图像属于$S$或$W$的打分。
给出损失函数：
$$
L_{dom}(W)=-\frac{1}{n}\sum_{i=1}^{n}[y_i^{dom}log(p_i))+(1-y_i^{dom})log(1-p_i)],
$$
损失函数与上一个分支功能一样，$y_i^{dom}=1$表示来自于$S$的proposals是正样本; $y_i^{dom}=0$表示来自于$W$的proposals是负样本。

下面要说才是我认为最有意思的地方，可以看到方法结构图中这个分支有一个梯度取反。一般我们优化网络都会让损失收敛到0，即最小值优化，而作者在梯度方向传播到特征f前取反，这是为了最大值优化。最小值优化是为了让网络可以区分数据是来自哪一个域，作者取反操作就是为了让网络无法区分，从而实现domain-invariance。**其实我感觉直接损失函数的负号去掉是一样的（欢迎指正）。**

然后从$S$和$W$中都随机选取64个proposals计算损失。

#####  2.2 目标知识已知的检测模型
下面我们讲方法的第二部分：利用学习到辨别目标的知识来训练一个弱监督检测器。
这部分可以分为两个部分讲解：(1)如何利用目标知识(2)如何用$W$的数据训练检测器

###### 2.2.1 利用2.1中学习到的目标辨别知识
作者是采用2.1中的目标预测分支，对$W$中每一张图片的proposals进行打分，得到他们属于目标的分数，然后排序，取前15%当作目标框（一起当作一个"object bag"），剩余的75%作为干扰框（一起当作一个"distractor bag"）。注意这里只是区分是不是目标，并没有给出目标是哪一类。所以"object bag"中会有很多类型的目标。

###### 2.2.1 训练弱监督检测器
作者使用的是Fast-RCNN的结构训练检测器（只包含分类分支），输出维度是K+1，K是类别数目。

为了更好的理解这里的训练过程，我们先举个栗子：输入图片1张，包含2000个SS生成的proposals，输入网络后得到1x2000x(K+1)矩阵。

如果我们要计算损失，是不是应该知道2000个框的类别标签，可是$W$数据是没有边界框注释信息的，我们无法得到这2000个框的标签，我们肿么办？

肯定有人想到用上面得到的"object bag"和"distractor bag"制作标签呀，的确，作者就是这么干的。

首先这个2000个框已经被我们分成了目标和干扰两个包。首先给"distractor bag"一个标签$y_0=1$，然后我们根据这个图像包含的目标类别对"object bag"给出对应的类别标签$y_k=1,k>0$。

可是网络输出是每个框属于每一类的打分，你这给的都是包的标签，不对应呀？
然后你肯定会想使用包中框的最高分作为包的打分不就行了。但是这样做就只是考虑了最大分框，作者给出了一个更好的计算方法：
$$
S^B=log(\sum_{r}^{|R|}exp(S_r^R)))，
$$
这样可以考虑包中所有的框。$R$是包，$S_r^R$是包中每个框的打分。

然后使用交叉商损失指导网络训练：
 $$
L(w)=\frac{\lambda }{2}\left \| w \right \|_2^2-\frac{1}{n}\sum_{i=1}^{n}\sum_{k=1}^{K+1}(1\{y_{ki}=1\}log(p_{ki}))，
$$
$$
p_{ki}=\frac{1}{1+exp(-s_{ki}^B)}
$$

### **创新点**
我个人感觉这篇论文最大的创新点就是把$S$和$W$的数据一起训练的方式。一般我们都会想的是用$S$训练一个检测器$D_s$，然后通过一种方式，用$D_s$来得到$W$中的pseudo-gt，然后训练检测器$D_w$。可是这篇论文就不一样，感觉很有意思。想继续深入了解的小伙伴，可以阅读原文。

### **实验分析**
其实看官看到这里就可以结束。
可是，本着从一而终的原则，我决定把实验也分析一遍。

其实这篇论文实验之前才5页，后面实验作者足足写了7页。。。看来实验才是重点，前面全是小菜。

实验主要可以分为三个部分：(1)数据集内部检测(2)数据集间检测(3)消融实验

实验的评价的标准主要：mAP和CorLoc。
这里说一下，mAP肯定一般都知道，CorLoc一般都是弱监督的时候才会用。它是评价模型在训练集上的定位精度。就是检测到每一类中检测到的图片占的比例，怎么叫检测到呢？就是对于一样图片中的某一类，取检测的打分最高边界框，如果与ground-truth（标注的边界框）的IoU>0.5就是检测正确。

实验开始之前，作者给出了三个基本的检测方法。由于论文的方法是由目标知识学习和弱监督检测训练两个子模块组成了混合监督整体方法，所以作者提出了分别对应两个子模块和整体方法的基本方法。

**B-WSD**：基本的若监督检测方法------->对应2.2的子模块
**B-MSD**：基础的混合监督检测方法------->对应整体的方法
**OOM-MSD**：用于混合监督检测的原始的目标学习模型------->对应2.1的子模块

下面简要说一说后两个方法：
**B-MSD**：作者是先用Fast-RCNN基于$S$训练一个强监督的检测器，然后用训练得到的模型参数初始化弱监督的检测器，然后用MIL的方式基于$W$训练检测器。
**OOM-MSD**：这部分作者就是把模型2.1的子模块的域分类的分支去掉了，就是直接基于$S$训练网络学习区分目标和干扰的知识。

#### 1. 数据集内的检测
就是把一个数据集按类别分为$S$，$W$。

作者使用PASCAL VOC 2007 和 ILSVRC2013来评价他的方法。

这里就只是以PASCAL VOC 2007为例吧，作者把trainval的数据按类别分为两部分，一共20类，前10类为$S$，后10类为$W$（根据字母排序选择的）。

当然啦，这些模型怎么训练的呢，这我要说的估计得照论文翻译了，还是感兴趣的孩童去看论文吧，哈哈哈。

还是贴图看一下模型的性能吧
![](https://upload-images.jianshu.io/upload_images/18102452-e93c6d670d147050.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这应该不用描述解释了吧。认真看图吧。（我是不会告诉你，我是认真读了一边作者分析再贴的图，**:）**滑稽脸）

![](https://upload-images.jianshu.io/upload_images/18102452-a661e7231a30b0a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2. 数据集间的检测
这里作者把PASCAL VOC 2007 的trainval作为$W$，ILSVRC2013作为$S$。
由于ILSVRC2013有200类包含PASCAL VOC 2007的20类，所以$S$是180类，剔除了$W$中的类别。

直接贴图，直接贴图
![](https://upload-images.jianshu.io/upload_images/18102452-575f5997101a220f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/18102452-7f580e5763e08bd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/18102452-4dd0c9710d2bb68a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不得不佩服，作者做实验验证的能力。学习一波。
#### 3. 消融实验
采用数据集间检测方式，都使用AlexNet

#### 3.1 分析干扰框的学习是WSD必须性
其实作者验证这个就是是否用那75%的proposals，作者把它丢掉，WSD的网络类别就是K了，训练了一个MSD-no-distractor的模型。

![](https://upload-images.jianshu.io/upload_images/18102452-fdcaa90d579aa72c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由结果来看41.77 **VS** 25.95，差距还是挺大的，说明弱监督单纯的只用图像标签是很难区分局部目标和整个目标的。其实也比较好理解，你告诉我这个图有人，可是你想我框出人头，还是人身，还是整个人呢，所以网络区分不了。

#### 3.2 分析选择object bag包含的框的数量（就是15%这个值怎么来的）
![](https://upload-images.jianshu.io/upload_images/18102452-35dab00c18dea2df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
就是选取其他的值来训练，看哪个高。

#### 3.3 为了进一步分析方法的有效性。
作者选取了ILSVRC2013中人们创造的类别作为$S$，PASCAL VOC 2007中自然界中的类别作为$W$，进行训练。

所实话，作者真的很会来事，但是不得不佩服。
![](https://upload-images.jianshu.io/upload_images/18102452-f3ba97ae16850909.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**作者**：快看，我的方法在任何数据之间迁移都很吊哟，你还不赶快cite。

如果你更着我读到了这里，我不得不给你点个赞，其实笔者都快被你感动了，坚持一下马上就结束了。

其实我又看了下后面，好像还不能很快结束。。。你还得在坚持很久。**-_-#**，我继续码。

#### 3.4 验证域不变的目标学习模型的有效性
这里作者和其他的目标学习方法或者获得proposals的方法进行了比较。

目标学习模型其实就是给proposals打分，然后分包，只要有类是功能的方法应该就可以比较。

作者使用召回率来比较的。
![](https://upload-images.jianshu.io/upload_images/18102452-396f28463b24f2d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


实际是如何操作的呢？
可以看上图中的横轴是百分比，这是怎么来的呢？是由SS生存的proposals按打分排序（ss算法本身对proposals会有个打分），然后取前5%，与ground-truth计算一遍IoU，大于0.7就算是目标框，这些框的个数/选取的proposals，这个值就是recall值。

然后用这些方法训练WSD。

![](https://upload-images.jianshu.io/upload_images/18102452-e8a0513aa01ba348.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

作者发现一个很有意思的现象：EdgeBox，Original Obj，Domain-invarint obj 三个的Recall在15%的时候都差不多，为什么上图的性能差距这么多，为森么？

然后自问自答 **：)**

然后作者定义：
**正样本**：IoU>=0.5
**局部目标**：0<IoU<0.5
**背景**：IoU=0
**作者**：少量的正样本或者过多的局部目标都会干扰WSD的学习，看我证明给你看，滑稽脸**:）**
然后分析了在这前15%的proposals，与ground-truth的IoU分布。

![](https://upload-images.jianshu.io/upload_images/18102452-c069106833390aa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**x轴**：表示局部目标在前15%的proposals的比例
**y轴**：表示训练图片中，包含这些比例值的图片占训练图片的比例

**作者**：快看，蓝色柱子，不要盯着绿色的看，我这是局部目标的比例，看我的方法多稳定。知道你们不懂，我给你举个例子**:）**

我们来看0%~10%**x轴**，假设每个图片是2000个proposals
那么前15%就是300个proposals（那么其中就包含0~30个局部目标）。
让我们来看**y轴**，蓝bar是10.18%，那么5011个训练图片中有大约500的图片的局部目标是在范围0%~10%。可以看图中，随着局部目标比例的增加，其他方法的对应的图片比例都在增加，而论文方法反而在减少，说明论文方法可以很好的剔除局部目标。

作者还进一步解释了为什么15%中包含局部目标的比例少，因为在训练图片中还包含了很多不属于数据集类别的完整目标，可是完整目标是被我们当作背景的，但是在使用2.1学习到的目标辨别知识是与目标类别无关的，所以15%会包含很多背景中存在的完整目标，进一步相对减少了局部目标的比例。**在这里我不得不佩服作者脑回路清奇，我感觉我发现了这篇论文的另一个宝藏**。如果你读到了这里，我该恭喜你。

#### 错误分析
作者也给出了效果图，来分析几个效果较差的类别。
![](https://upload-images.jianshu.io/upload_images/18102452-5a95fc6938f29299.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

自行感受有多差吧。

终于结束了，我写的都累了，默默心疼在看的你。希望你有所收获。
第一次写blog，希望不是最后一次，以后应该陆续推出论文解读。

如果发现有问题，欢迎指正^_^。
