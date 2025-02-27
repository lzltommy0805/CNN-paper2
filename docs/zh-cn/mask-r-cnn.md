## Mask R-CNN

------

!> 论文地址： https://arxiv.org/abs/1703.06870

### 0.摘要

论文提出了一个概念上简单，灵活和通用的**实例分割**框架。我们的方法有效地检测图像中的目标，同时为每个实例生成高质量的分割掩码。称为Mask R-CNN的方法通过添加一个与现有目标检测框回归并行的，用于预测目标掩码的分支来扩展Faster R-CNN。Mask R-CNN训练简单，相对于Faster R-CNN，只需增加一个较小的开销，运行速度可达**5 FPS**。此外，Mask R-CNN很容易推广到其他任务，例如，允许在同一个框架中估计人的姿势。在COCO挑战的所有三个项目中取得了最佳成绩，包括实例分割，目标检测和人体关键点检测。在没有使用额外技巧的情况下，Mask R-CNN优于所有现有的单一模型，包括COCO 2016挑战优胜者。这种简单而有效的方法将成为一个促进未来目标级识别领域研究的坚实基础。

### 1.Introduction

目标检测和语义分割的效果在短时间内得到了很大的改善。在很大程度上，这些进步是由强大的基线系统驱动的，例如，分别用于目标检测和语义分割的Fast/Faster R-CNN 和全卷积网络(FCN)（语义分割,可参考本教程FCN学习）框架。这些方法在概念上是直观的，提供灵活性和鲁棒性，以及快速的训练和推理。Mask R-CNN在这项工作中的目标是为实例分割开发一个相对有力的框架。

**实例分割是具有挑战性的，因为它需要正确检测图像中的所有目标，同时也精确地分割每个目标。** 因此，它结合了来自经典计算机视觉任务目标检测的元素，其目的是对目标进行分类，并使用边界框定位每个目标，以及语义分割（通常来说，目标检测来使用边界框而不是掩码来标定每一个目标检测，而语义分割以在不区分目标的情况下表示每像素的分类。然而，实例分割既是语义分割，又是另一种形式的检测。）鉴于此，人们可能认为需要一种复杂的方法才能取得良好的效果。然而，研究表明，使用非常简单，灵活和快速的系统就可以超越先前的最先进的实例分割结果。
称之为Mask R-CNN的方法通过添加一个用于在每个感兴趣区域（RoI）上预测分割掩码的分支来扩展Faster R-CNN [34]，这个分支与用于分类和目标检测框回归的分支并行执行，如下图（图1）所示（用于实例分割的Mask R-CNN框架）：

<div align=center>
<img src="zh-cn/img/maskrcnn/p1.png" />
</div>

*这个示意图相对画的比较简单，我们会在后文做详细的解释*


掩码分支是作用于每个RoI的小FCN，以像素到像素的方式预测分割掩码。Mask R-CNN易于实现和训练，它是基于Faster R-CNN这种灵活的框架的。此外，掩码分支只增加了很小的计算开销。

原理上，Mask R-CNN是Faster R-CNN的直接扩展，而要获得良好的结果，正确构建掩码分支至关重要。最重要的是，Faster R-CNN不是为网络输入和输出之间的像素到像素对齐而设计的。在paper: how RoIPool[12,18]  中提到，实际上，应用到目标上的核心操作执行的是粗略的空间量化特征提取。**为了修正错位，我们提出了一个简单的，量化无关的层，称为RoIAlign，可以保留精确的空间位置。** 尽管是一个看似很小的变化，RoIAlign起到了很大的作用：它可以将掩码准确度提高10％至50％，在更严格的位置度量下显示出更大的收益。其次，**我们发现解耦掩码和分类至关重要：我们为每个类独立地预测二进制掩码，这样不会跨类别竞争，并且依赖于网络的RoI分类分支来预测类别。** 相比之下，FCN通常执行每像素多类分类，分割和分类同时进行，基于我们的实验，对于实例分割效果不佳。

Mask R-CNN超越了COCO实例分割任务[28]上所有先前最先进的单一模型结果，其中包括COCO 2016挑战优胜者。作为副产品，我们的方法也优于COCO目标检测任务。在消融实验中，我们评估多个基本实例，这使我们能够证明其鲁棒性并分析核心因素的影响。

我们的模型可以在GPU上以200毫秒每帧的速度运行，使用一台有8个GPU的机器，在COCO上训练需要一到两天的时间。我们相信，快速的训练和测试速度，以及框架的灵活性和准确性将促进未来实例分割的研究。

最后，我们通过COCO关键点数据集上的人体姿态估计任务来展示我们框架的通用性。通过将每个关键点视为one-hot二进制掩码，只需要很少的修改，Mask R-CNN可以应用于人体关键点检测。不需要额外的技巧，Mask R-CNN超过了COCO 2016人体关键点检测比赛的冠军，同时运行速度可达5 FPS。因此，Mask R-CNN可以被更广泛地看作是用于目标级识别的灵活框架，并且可以容易地扩展到更复杂的任务。


### 2.Related Work

**R-CNN**：R-CNN方法是通过找到一定数量的候选区域 ，并独立地在每个RoI上执行卷积 来进行目标检测的。 基于R-CNN的改进 ，使用RoIPool在特征图上选取RoI，实现了更快的速度和更好的准确性。Faster R-CNN通过使用RPN学习注意机制来产生候选框。还有后续的对Faster R-CNN灵活性和鲁棒性的改进。这是目前在几个基准测试中领先的框架。

**实例分割**：在R-CNN的有效性的推动下，实例分割的许多方法都是基于segment proposals的。先前的方法[13,15,16,9]依赖自下而上的分割 。 DeepMask[33]和[34,8]通过学习提出分割候选，然后使用Fast R-CNN分类。在这些方法中，分割先于识别，这样做既慢又不太准确。同样，Dai等人提出了一个复杂的多级联级联，从候选框中预测候选分割，然后进行分类。相反，我们的方法并行进行掩码和类标签的预测，更简单也更灵活。

最近，Li等人将[8]中的分割候选系统与[11]中的目标检测系统进行了“全卷积目标分割”（FCIS）的融合。 在[8,11,26] 中的共同想法是用全卷积得到一组位置敏感的输出通道候选。这些通道同时处理目标分类，目标检测和掩码，这使系统速度变得更快。**但FCIS在重叠实例上出现系统错误，并产生虚假边缘（图5）（FCIS使用全卷机网络同时预测class,box,mask,速度虽快，但是对于重叠物体的分割效果并不好）**


### 3.Mask R-CNN

Mask R-CNN在概念上是简单的：Faster R-CNN为每个候选目标输出类标签和边框偏移量。为此，我们添加了一个输出目标掩码的第三个分支。因此，Mask R-CNN是一种自然而直观的点子。但是，附加的掩码输出与类和框输出不同，需要提取对象的更精细的空间布局。接下来，**我们介绍Mask R-CNN的关键特点，包括像素到像素对齐，这是Fast/Faster R-CNN的主要缺失。**

**Faster R-CNN**：我们首先简要回顾一下Faster R-CNN检测器。Faster R-CNN由两个阶段组成。称为区域提议网络（RPN）的第一阶段提出候选目标边界框。第二阶段，本质上是Fast R-CNN ，使用RoIPool从每个候选框中提取特征，并进行分类和边界回归。两个阶段使用的特征可以共享，以便更快的推理。可以参考本教程对[Fast R-CNN的介绍](/zh-cn/chapter5)。

**Mask R-CNN**：Mask R-CNN采用相同的两个阶段，具有相同的第一阶段（即RPN）。在第二阶段，与预测类和框偏移量并行，Mask R-CNN还为每个RoI输出二进制掩码。这与最近的其它系统相反，其分类取依赖于掩码预测（例如 [10,33,26] ）。我们的方法遵循Fast R-CNN [12]，预测类和框偏移量并行（这在很大程度上简化了R-CNN的多级流水线）。

在训练期间，我们将在每个采样后的RoI上的多任务损失函数定义为
$$L = L_{cls}+L_{box}+L_{mask}$$
分类损失$L_{cls}$和检测框损失$L_{box}$与Faster R-CNN[12]中定义的相同。掩码分支对于每个RoI的输出维度为$Km^2$，即$K$个分辨率为$mxm$的二进制掩码，每个类别一个，$K$表示类别数量。我们为每个像素应用Sigmoid，并将$L_{mask}$定义为平均二进制交叉熵损失。对于真实类别为$K$的RoI，仅在第$K$个掩码上计算$L_{mask}$（其他掩码输出不计入损失）。

**我们对$L_{mask}$的定义允许网络为每个类独立地预测二进制掩码，这样不会跨类别竞争**。我们依靠专用分类分支预测用于选择输出掩码的类标签。这将解耦掩码和类预测。这与通常将FCN[30]应用于像素级Softmax和多重交叉熵损失的语义分割的做法不同。在这种情况下，掩码将在不同类别之间竞争。而我们的方法，使用了其它方法没有的像素级的Sigmod和二进制损失。我们通过实验发现，这种方法是改善实例分割效果的关键。

**掩码（Mask）表示**：掩码表示输入目标的空间布局。因此，与通过全连接（fc）层不可避免地缩成短输出向量的类标签或框偏移不同，提取掩码的空间结构可以通过由卷积提供的像素到像素对应自然地被解决(仍然使用FCN解决)。

具体来说，我们使用FCN[30]来为每个RoI预测一个$mxm$的掩码。这允许掩码分支中的每个层显式的保持$m×m$的对象空间布局，而不会将其缩成缺少空间维度的向量表示。与以前使用fc层掩码预测的的方法不同[18 19 21]，我们的全卷积表示需要更少的参数，并且如实验所证明的更准确。

这种像素到像素的行为需要RoI特征，它们本身就是小特征图。为了更好地对齐，以准确地保留显式的像素空间对应关系，我们开发出在掩模预测中发挥关键作用的`RoIAlign`层。

**RoIAlign**(这一部分我们将在下文做详细的介绍！)：RoIPool(Fast R-CNN)是从每个RoI提取小特征图（例如`7×7`）的标准操作。 RoIPool首先将浮点数表示的RoI缩放到与特征图匹配的粒度，然后将缩放后的RoI分块，最后汇总每个块覆盖的区域的特征值（通常使用最大池化）。例如，对在连续坐标系上的`x`计算`[x/16]`，其中16是特征图步幅，`[.]`表示四舍五入。同样地，当对RoI分块时（例如`7×7`）时也执行同样的计算。这样的计算使RoI与提取的特征错位。虽然这可能不会影响分类，因为分类对小幅度的变换具有一定的鲁棒性，但它对预测像素级精确的掩码有很大的负面影响。

为了解决这个问题，我们提出了一个RoIAlign层，可以去除RoIPool的错位，将提取的特征与输入准确对齐。我们提出的改变很简单：**我们避免计算过程中的四舍五入（比如，我们使用`x/16`代替`[x/16]`）**。**我们选取分块中的4个常规的位置，使用双线性插值[22]来计算每个位置的精确值，并将结果汇总（使用最大或平均池化）**。（我们抽取四个常规位置，以便我们可以使用最大或平均池化。事实上，在每个分块中心取一个值（没有池化）几乎同样有效。我们也可以为每个块采样超过四个位置，我们发现这些位置的收益递减。）

如在消融实验中所示，RoIAlign的改进效果明显。我们还比较了[10]中提出的`RoIWarp`操作。与RoIAlign不同，**RoIWarp忽略了对齐问题**，并在[10]的实现中，有像RoIPool那样的四舍五入计算。因此，即使RoIWarp也采用[22]提到的双线性重采样，如实验所示（更多细节见表格2c），它与RoIPool效果差不多。这表明了对齐起到了关键的作用

**网络架构**(这一部分我们将在下文做详细的介绍)：为了证明我们的方法的普适性，构造了多种不同结构的Mask R-CNN。详细来说就是，使用不同的：(i)用于整个图像上的特征提取的**下层卷积网络(backbone)**，以及(ii)用于检测框识别（分类和回归）和掩码预测的**上层网络(head)**。

我们使用”网络-深度-特征输出层”的方式命名backbone卷积网络。我们评估了深度为50或101层的ResNet[19]和ResNeXt[45]网络。使用ResNet的Faster R-CNN从第四阶段的最终卷积层提取特征，我们称之为C4。例如，使用ResNet-50的下层网络由ResNet-50-C4表示。这是[19,10,21,39]中常用的选择。

我们还探讨了Lin等人`[27]`(FPN)最近提出的另一种更有效的backbone网络，称为特征金字塔网络（FPN）(可以参考我们FPN的教程学习)。 FPN使用具有横旁路连接的自顶向下架构，以从单尺度输入构建网络中的特征金字塔。使用FPN的Faster R-CNN根据其尺度提取不同级别的金字塔的RoI特征，不过其它部分和平常的ResNet类似。使用ResNet-FPN进行特征提取的Mask R-CNN可以在精度和速度方面获得极大的提升。有关FPN的更多细节,请参考我们的[FPN教程](/zh-cn/fpn)


对于上层网络，我们基本遵循了以前论文中提出的架构(FCN)，我们添加了一个全卷积的掩码预测分支。具体来说，我们扩展了 ResNet和FPN中提出的Faster R-CNN的上层网络。详细信息如下图（图3）所示：（上层网络架构：我们扩展了两种现有的Faster R-CNN上层网络架构[19,27]，分别添加了一个掩码分支。图中数字表示分辨率和通道数，箭头表示卷积、反卷积或全连接层（可以通过上下文推断，卷积减小维度，反卷积增加维度。）所有的卷积都是3×3的，除了输出层，是1×1的。反卷积是2×2的，步进为2，,我们在隐藏层中使用ReLU。左图中，“res5”表示ResNet的第五阶段，简单起见，我们修改了第一个卷积操作，使用7×7，步长为1的RoI代替14×14，步长为2的RoI。右图中的“×4”表示堆叠的4个连续的卷积。）


<div align=center>
<img src="zh-cn/img/maskrcnn/p2.png" />
</div>

ResNet-C4的上层网络包括ResNet的第五阶段（即9层的“res5），这是计算密集型的。对于FPN，backbone已经包含了res5，因此可以使上层网络包含更少的卷积核而变的更高效。
我们注意到我们的掩码分支是一个非常简单的结构。也许更复杂的设计有可能提高性能，但不是这项工作的重点。


#### 3.1实现细节

超参数的设置与现有的Fast/Faster R-CNN基本一致。虽然这些设定是在原始论文中是用于目标检测的，但是我们发现我们的实例分割系统也是可以用。

**训练**：与Faster R-CNN中的设置一样，如果RoI与真值框的IoU不小于0.5，则为正样本，否则为负样本。掩码损失函数$L_mask$仅在RoI的正样本上定义。掩码目标是RoI及其对应的真值框之间的交集的掩码。

我们采用以图像为中心的训练[12]。图像被缩放（较短边）到800像素[27]。批量大小为每个GPU 2个图像，每个图像具有N个RoI采样，正负样本比例为`1:3`。 C4下层网络的N为64（如[12,36]），FPN为512（如[27]）。我们使用8个GPU训练（如此有效的批量大小为16）`160k`次迭代，学习率为`0.02`，在`120k`次迭代时学习率除以`10`。我们使用`0.0001`的权重衰减和`0.9`的动量。

RPN锚点跨越5个尺度和3个纵横比[27 FPN]。为方便消融，RPN分开训练，不与Mask R-CNN共享特征。本文中的，RPN和Mask R-CNN具有相同的下层网络，因此它们是可共享的。

**测试**：在测试时，C4下层网络(backbone)（如[36]）中的候选数量为300，FPN为1000（如[27]）。我们在这些候选上执行检测框预测分支，然后执行非极大值抑制(NMS)[14]。然后将掩码分支应用于评分最高100个检测框。尽管这与训练中使用的并行计算不同，但它可以加速推理并提高精度（由于使用更少，更准确的RoI）。掩码分支可以预测每个RoI的K个掩码，但是我们只使用第K个掩码，其中K是分类分支预测的类别。**然后将`m×m`浮点数掩码输出的大小调整为RoI大小，并使用阈值0.5将其二值化。**

请注意，由于我们仅在前100个检测框中计算掩码，Mask R-CNN将边缘运行时间添加到其对应的Faster R-CNN版本（例如，相对约20％）。

### 4.消融实验

展示一些效果图：


<div align=center>
<img src="zh-cn/img/maskrcnn/p3.png" />
</div>

CityScapes数据集上的表现：

我们进一步报告CityScapes[7]数据集的目标分割结果。该数据集具有精细标注的2975个训练图像，500个验证图像和1525个测试图像。它还有20k粗糙的训练图像，无精细标注，我们不使用它们。所有图像的分辨率为2048 x 1024像素。目标分割任务涉及8个对象类别，其训练集中的目标数为：

<div align=center>
<img src="zh-cn/img/maskrcnn/p4.png" />
</div>

该任务的目标分割性能由和COCO一样的掩码AP（在IoU阈值上平均）来测量，也包括AP50（即，IoU为0.5的掩码AP）。
CityScapes的结果示例如下图所示：（Mask R-CNN在CityScapes的测试结果（32.0 AP）。右下图出错。）

<div align=center>
<img src="zh-cn/img/maskrcnn/p5.png" />
</div>


### 5.ROIAlign详解

**1 RoI Pooling局限性分析**

在常见的两级检测框架（比如Fast-RCNN，Faster-RCNN，RFCN）中，ROI Pooling 的作用是根据预选框的位置坐标在特征图中将相应区域池化为固定尺寸的特征图，以便进行后续的分类和包围框回归操作。**由于预选框的位置通常是由模型回归得到的，一般来讲是浮点数，而池化后的特征图要求尺寸固定。故ROI Pooling这一操作存在两次量化(quantization)的过程。**

+ 将候选框边界量化为整数点坐标值。从roi proposal到feature map的映射时，取`[x/16]`，这里x是原始ROI的坐标值，做了向下取整。 

+ 将量化后的边界区域平均分割成 `k×k` 个单元(bin),对每一个单元的边界进行量化。

事实上，经过上述两次量化，此时的候选框已经和最开始回归出来的位置有一定的偏差，这个偏差会影响检测或者分割的准确度。在论文里，作者把它总结为”不匹配问题(misalignment)”。

下面我们用直观的例子具体分析一下上述区域不匹配问题。如下图所示，这是一个Faster-RCNN检测框架。输入一张`800*800`的图片，图片上有一个`665*665`的包围框(框着一只狗)。图片经过主干网络提取特征后，特征图缩放步长（stride）为32。因此，图像和包围框的边长都是输入时的`/32`。800正好可以被32整除变为25。但665除以32以后得到20.78，带有小数，于是ROI Pooling 直接将它量化成20。接下来需要把框内的特征池化`7×7`的大小，因此将上述包围框平均分割成`7×7`个矩形区域。显然，每个矩形区域的边长为2.86，又含有小数。于是ROI Pooling 再次把它量化到2。经过这两次量化，候选区域已经出现了较明显的偏差（如图中绿色部分所示）。更重要的是，该层特征图上0.1个像素的偏差，缩放到原图就是3.2个像素。那么0.8的偏差，在原图上就是接近30个像素点的差别，这一差别不容小觑。 

<div align=center>
<img src="zh-cn/img/maskrcnn/p6.png" />
</div>

简而言之：
做segment是pixel级别的，但是Faster R-CNN中ROI pooling有2次量化操作导致了没有对齐 ，并且Mask R-CNN验证是用ROIAlign对目标检测的精度也有提高。

<div align=center>
<img src="zh-cn/img/maskrcnn/p7.png" />
</div>


**2.ROI Align 的主要思想和具体方法**

为了解决ROI Pooling的上述缺点，作者提出了ROI Align这一改进的方法如下图。ROI Align的思路很简单：取消量化操作，使用双线性内插的方法获得坐标为浮点数的像素点上的图像数值（像素点的坐标都是整数，而调整后的候选框的坐标是浮点数，从而导致坐标是浮点数的对应的坐标位置是没有像素值得，没有像素值就可以用双线性差值等技术估计一个像素值，实际上ROIAlign做的更高明）,从而将整个特征聚集过程转化为一个连续的操作。值得注意的是，在具体的算法操作上，ROI Align并不是简单地补充出候选区域边界上的坐标点，然后将这些坐标点进行池化，而是重新设计了一套比较优雅的流程，如下图所示：


<div align=center>
<img src="zh-cn/img/maskrcnn/p8.png" />
</div>


<div align=center>
<img src="zh-cn/img/maskrcnn/p9.png" />
</div>

+ 遍历每一个候选区域，保持浮点数边界不做量化。
+ 将候选区域分割成`kXk`个单元，每个单元的边界也不做量化。
+ 在每个单元中计算固定四个坐标位置，用双线性内插的方法计算出这四个位置的值，然后进行最大池化操作。

这里对上述步骤的第三点作一些说明：这个固定位置是指在每一个矩形单元（bin）中按照固定规则确定的位置。比如，如果采样点数是1，那么就是这个单元的中心点。如果采样点数是4，那么就是把这个单元平均分割成四个小方块以后它们分别的中心点。显然这些采样点的坐标通常是浮点数，所以需要使用插值的方法得到它的像素值。在相关实验中，作者发现将采样点设为4会获得最佳性能，甚至直接设为1在性能上也相差无几。

事实上，ROI Align 在遍历取样点的数量上没有ROIPooling那么多，但却可以获得更好的性能，这主要归功于解决了misalignment的问题。值得一提的是，我在实验时发现，**ROI Align在VOC2007数据集上的提升效果并不如在COCO上明显**。经过分析，造成这种区别的原因是COCO上小目标的数量更多，而小目标受misalignment问题的影响更大（比如，同样是0.5个像素点的偏差，对于较大的目标而言显得微不足道，但是对于小目标，误差的影响就要高很多）。

下面图示更加细致地描述ROIAlign:

<div align=center>
<img src="zh-cn/img/maskrcnn/p10.png" />
</div>

<div align=center>
<img src="zh-cn/img/maskrcnn/p11.png" />
</div>

<div align=center>
<img src="zh-cn/img/maskrcnn/p12.png" />
</div>

<div align=center>
<img src="zh-cn/img/maskrcnn/p13.png" />
</div>

<div align=center>
<img src="zh-cn/img/maskrcnn/p14.png" />
</div>


**3.ROIAlign 的反向传播**

常规的ROI Pooling的反向传播公式如下

<div align=center>
<img src="zh-cn/img/maskrcnn/p15.png" />
</div>

这里，$x_i$代表池化前特征图上的像素点；$y_{rj}$代表池化后的第r个候选区域的第j个点；$i(r,j)$代表点$y_{rj}$像素值的来源（最大池化的时候选出的最大像素值所在点的坐标）。由上式可以看出，只有当池化后某一个点的像素值在池化过程中采用了当前点$x_i$的像素值（即满足`i=i*(r，j)`），才在$x_i$处回传梯度。

类比于ROIPooling，ROIAlign的反向传播需要作出稍许修改：首先，在ROIAlign中，$x_i(r,j)$是一个浮点数的坐标位置(前向传播时计算出来的采样点)，在池化前的特征图中，每一个与 $x_ix(r,j)$ 横纵坐标均小于1的点都应该接受与此对应的点$y_{rj}$回传的梯度，故ROI Align 的反向传播公式如下: 

<div align=center>
<img src="zh-cn/img/maskrcnn/p16.png" />
</div>

上式中，`d(.)`表示两点之间的距离，`Δh`和`Δw`表示 $x_i$ 与 $x_i(r,j)$ 横纵坐标的差值，这里作为双线性内插的系数乘在原始的梯度上。


ROIAlign总结：对于每个ROI，映射之后坐标保持浮点数，在此基础上再平均切分成`k*k`个bin，这个时候也保持浮点数。再把每个bin平均分成４个小的空间(bin中更小的bin)，然后计算每个更小的bin的中心点的像素点对应的概率值。这个像素点大概率是一个浮点数，实际上图像的浮点是没有像素值的，但这里假设这个浮点数的位置存储一个概率值，这个值由相邻最近的整数像素点存储的概率值经过双线性插值得到，其实也就是根据这个中心点所在的像素值找到所在的大bin对应的４个整数像素存储的值，然后乘以多个参数进行插值。这些参数其实就是那４个整数像素点和中心点的位置距离关系构成参数。最后再在每个大bin中对４个中心点进行max或者mean的pooling。

**4.ROI Pooling 、ROI Align和RoIWarp**

下图对比了三种方法的不同，其中RoIWarp来自：J. Dai, K. He, and J. Sun. Instance-aware semantic segmentation via multi-task network cascades。RoIWarp第一次量化了，第二次没有，RoIAlign两次都没有量化:

<div align=center>
<img src="zh-cn/img/maskrcnn/p17.png" />
</div>

**5.实例**

输出`7*7`的fix featrue：

1.划分`7*7`的bin(我们可以直接精确的映射到feature map来划分bin，不用第一次量化) 

<div align=center>
<img src="zh-cn/img/maskrcnn/p18.png" />
</div>

2.每个bin中采样4个点，双线性插值 

<div align=center>
<img src="zh-cn/img/maskrcnn/p19.png" />
</div>

3.对每个bin4个点做max或average pooling


**6. 更多**

前面我们介绍RoI Align是在每个bin中采样4个点，双线性插值，但也是一定程度上解读了mismatch问题，而旷视科技PLACES instance segmentation比赛中所用的是更精确的解决这个问题，对于每个bin，RoIAlign只用了4个值求平均，而旷视则直接利用积分(把bin中所有位置都插值出来)求和出这一块的像素值和然后求平均，这样更精确了但是很费时。

<div align=center>
<img src="zh-cn/img/maskrcnn/p20.png" />
</div>

<div align=center>
<img src="zh-cn/img/maskrcnn/p21.png" />
</div>




### 6.网络结构详解

Mask-RCNN 大体框架还是 Faster-RCNN 的框架，可以说在基础特征网络之后又加入了全连接的分割子网，由原来的两个任务（分类+回归）变为了三个任务（分类+回归+分割）。Mask R-CNN 是一个两阶段的框架，第一个阶段扫描图像并生成提议（proposals，即有可能包含一个目标的区域），第二阶段分类提议并生成边界框和掩码。

<div align=center>
<img src="zh-cn/img/maskrcnn/p22.png" />
</div>

其中黑色部分为原来的 Faster-RCNN，红色部分为在 Faster网络上的修改，总体流程如下：

1）输入图像； 

2）将整张图片输入CNN，进行特征提取； 

3）用FPN生成建议窗口(proposals)，每张图片生成N个建议窗口； 

4）把建议窗口映射到CNN的最后一层卷积feature map上； 

5）通过RoIAlign层使每个RoI生成固定尺寸的feature map； 

6）最后利用全连接分类，边框，mask进行回归。

<div align=center>
<img src="zh-cn/img/maskrcnn/p23.png" />
</div>


首先对图片做检测，找出图像中的ROI，对每一个ROI使用ROIAlign进行像素校正，然后对每一个ROI使用设计的FCN框架进行预测不同的实例所属分类，最终得到图像实例分割结果。 

与faster RCNN的区别： 

1）使用ResNet101网络 

2）将 Roi Pooling 层替换成了 RoiAlign； 

3）添加并列的 Mask 层； 

4）由RPN网络转变成FPN网络;

主要改进点：

1.基础网络的增强，ResNeXt-101+FPN的组合可以说是现在特征学习的王牌了；

2.分割 loss 的改进，由原来的 FCIS 的 基于单像素softmax的多项式交叉熵变为了基于单像素sigmod二值交叉熵。softmax会产生FCIS的 ROI inside map与ROI outside map的竞争。但文章作者确实写到了类间的竞争， 二值交叉熵会使得每一类的 mask 不相互竞争，而不是和其他类别的 mask 比较 ；

3.ROIAlign解决Misalignment 的问题，说白了就是对 feature map 的插值。直接的ROIPooling的那种量化操作会使得得到的mask与实际物体位置有一个微小偏移，个人感觉这个没什么 insight，就是工程上更好的实现方式。

说明：这么好的效果是由多个阶段的优化实现的，大头的提升还是由数据和基础网络的提升：多任务训练带来的好处其实可以看作是更多的数据带来的好处；FPN 的特征金字塔，ResNeXt更强大的特征表达能力都是基础网络。


再放几个模型结构，供读者理解：

<div align=center>
<img src="zh-cn/img/maskrcnn/24.png" />
</div>

<div align=center>
<img src="zh-cn/img/maskrcnn/25.png" />
</div>

<div align=center>
<img src="zh-cn/img/maskrcnn/26.PNG" />
</div>

*这个图示Mask R-CNN项目的结构图：https://github.com/matterport/Mask_RCNN*

### 7.Mask及其损失函数

依旧采用的是多任务损失函数，针对每个每个RoI定义为

$$L=L_{cls}+L_{box}+L_{mask}$$

其中$L_{cls}$,$L_{box}$与Faster R-CNN的定义类似，这里主要看$L_{mask}$


掩膜分支针对每个RoI产生一个$Km^2$的输出，即K个分辨率为`m×m`的二值的掩膜，K为分类物体的种类数目。依据预测类别分支预测的类型i，只将第i的二值掩膜输出记为$L_{mask}$。 
掩膜分支的损失计算如下示意图：

1.mask branch 预测K个种类的`m×m`二值掩膜输出

2.依据种类预测分支(Faster R-CNN部分)预测结果：当前RoI的物体种类为i

3.第i个二值掩膜输出就是该RoI的损失$L_{mask}$

<div align=center>
<img src="zh-cn/img/maskrcnn/p28.png" />
</div>

对于预测的二值掩膜输出，我们对每个像素点应用sigmoid函数，整体损失定义为平均二值交叉损失熵。 
引入预测K个输出的机制，允许每个类都生成独立的掩膜，避免类间竞争。这样做解耦了掩膜和种类预测。不像是FCN的方法，在每个像素点上应用softmax函数，整体采用的多任务交叉熵，这样会导致类间竞争，最终导致分割效果差。


<div align=center>
<img src="zh-cn/img/maskrcnn/p27.png" />
</div>


### Reference

[1].https://blog.csdn.net/ghw15221836342/article/details/79549387

[2].https://blog.csdn.net/xiaqunfeng123/article/details/78716136

[3].https://blog.csdn.net/weixin_37142859/article/details/95514488#ROI_Head_94

[4].https://blog.csdn.net/u012897374/article/details/80768199

[5].https://blog.csdn.net/qinghuaci666/article/details/80900882

[6].https://blog.csdn.net/wenxueliu/article/details/80906497

[7].https://blog.csdn.net/u011974639/article/details/78483779?locationNum=9&fps=1

[8].https://www.bilibili.com/video/av24795835/?redirectFrom=h5