# CNN模型压缩和加速方法综述
为了提升视觉分类方法的精度，基于cnn的方法的层数不断向横向和纵向加深，使得模型的大小不断增加，网络的计算速度不断变慢。严重影响了cnn方法在移动端和嵌入式系统的应用。为了解决上述问题，学术界不断提出新方法，总结这些方法，其思路上通过模型权重数值角度和从网络架构角度入手。目前的模型压缩和加速方法主要有如下几种：  
|Method	|Compression Approach|	Speed Consideration|
|-|-|-|
|SqueezeNet|	architecture|	No
Deep Compression| weights|	No
XNorNet|	weights|	Yes
Distilling|	architecture|	No
MobileNet|	architecture|	Yes
ShuffleNet|	architecture|	Yes

#### SqueezeNet  

SqueezeNet的核心指导思想是在保证模型精度的前提下，最大限度地减小权重参数的数量。基于上述思想，提出了3点网络设计策略。
- 将3x3的卷积核替换成1x1的卷积核  
  1x1的卷积核参数量是3x3的1/9,可以极大减小参数量
- 减小输入到3x3卷积核的通道数
  因为每一个3x3卷积层的模型参数量是P=N * C * 3 * 3，N是卷积核的数量，也就输输出feature map的数量，而C是输入卷积核的feature map数量，只要减少C的数量，参数量就会下降
- 尽量将pooling层放在网络的靠后位置，这个设计原则是为了最大限度提升网络的精度，应为更大的feature map size能够保留更多的信息

根据上述设计原则，sequeeze net提出了一个fire module，每个fire module包含一个squeeze 卷积层（只包含1x1卷积核）和一个expand卷积核(包含1x1卷积核和3x3卷积核)，利用1x1卷积核来降低输入到expand层的通道数。
![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513410092683.png)  
其中，定义squeeze层中1x1卷积核的数量是s1x1，类似的，expand层中1x1卷积核的数量是e1x1， 3x3卷积核的数量是e3x3。令s1x1 < e1x1+ e3x3从而保证输入到3x3的输入通道数减小。SqueezeNet的网络结构由若干个 fire module 组成，另外文章还给出了一些架构设计上的细节： 
1. 为了保证1x1卷积核和3x3卷积核具有相同大小的输出，3x3卷积核采用1像素的zero-padding和步长
2. squeeze层和expand层均采用RELU作为激活函数
3. 在fire9后采用50%的dropout
4. 由于全连接层的参数数量巨大，因此借鉴NIN的思想，去除了全连接层而改用global average pooling

对于速度方面：SqueezeNet在网络结构中大量采用1x1和3x3卷积核是有利于速度的提升的，对于类似caffe这样的深度学习框架，在卷积层的前向计算中，采用1x1卷积核可避免额外的im2col操作，而直接利用gemm进行矩阵加速运算，因此对速度的优化是有一定的作用的。然而，这种提速的作用仍然是有限的，另外，SqueezeNet采用了9个fire module和两个卷积层，因此仍需要进行大量常规卷积操作，这也是影响速度进一步提升的瓶颈


#### Deep Compression

与squeeze net压缩框架的角度不同，deep compression是对权重参数进行压缩，其思想主要是在模型生成的三个阶段对模型进行压缩，首先是剪枝，将所有对精度没有贡献的分支减掉，减小需要存储参数的数目。其次是量化，目的是减小每一个权重的bit数，最后是huffman编码，进一步减小模型的存储空间，下图是deep compression的pipline：  
![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513410758229.png)
- pruning(权值剪枝)
在剪值中，将0值附近的较小的权值置0，使这些权值不被激活，从而着重训练剩下的非零权值，最终在保证网络精度不变的情况下达到压缩尺寸的目的  
实验发现模型对剪枝更敏感，因此在剪值时建议逐层迭代修剪，另外每层的剪枝比例如何自动选取仍然是一个值得深入研究的课题

- Quantization(量化)
量化的思想是对所有的权重进行离散化，统计相同的权重，减小需要存储的权重参数数量。方法是对所有权重进行聚类，将权重归于不同的clusters，进行前向计算时，每个权值由其聚类中心表示，反向计算时统计每一个clusters中的梯度并将其反转。

- Huffman Encoding
霍夫曼编码采用变长编码将平均编码长度减小，进一步压缩模型尺寸

从速度方面考量，Deep Compression的主要设计是针对网络存储尺寸的压缩，但在前向时，如果将存储模型读入展开后，并没有带来更大的速度提升。因此Song H.等人专门针对压缩后的模型设计了一套基于FPGA的硬件前向加速框架EIE

#### XNorNet
二值网络的目的是通过将网络参数和网络输入的二值化来达到模型压缩的目的，为此这边文章提出了两个网络，BWN和XNOR-Network  
BWN的通过将网络参数二值化，再通过最优化目标函数来达到尽量减小精度损耗。
XNOR-Network的优化目标是将两个实数向量的点乘近似表示成两个二值向量的点乘，从而达到同时二值化网络输入和参数的目的。
![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513411965686.png)

#### Distilling

Distiling可以直译为"蒸馏", 基本思想是使用一个大网络来教小网络学习，使得小网络具有与大网络一样的性能，从而达到模型压缩的目的
训练小网络的目标函数一般包含两个部分：
- 与大模型softmax输出的交叉熵，称为软目标(soft target)
  softmax的输出加入了超参数温度T，以控制输出，T越大，输出越缓和，z/T越小，熵越大
- 与真值的交叉熵
训练的损失为上述两个损失的加权和，通常第二项要小很多
蒸馏模型的计算速度的提升，取决于蒸馏模型的计算规模，理论上，蒸馏模型的尺度变小，必然导致计算速度提升，但是实际情况下需要权衡计算速度和模型精度损失。

#### MobileNets
mobilenets引入了传统网络中的group思想，即限制滤波器的卷积计算只针对特定的group中，降低卷积的计算量  
mobilenets借鉴factorized convolution的思想，将卷积操作分成两个部分  
- depthwise conv
  卷积核只针对特定的通道进行卷积操作，例如输入M个通道，卷积核大小为Dk * Dk, 输出feature map的size为Df*Df,需要的计算量为：
  ```math
    Dk * Dk * M * Df * Df
  ```
- pointwise conv  
  采用1x1的卷积核将depthwise conv的输出feature map进行结合：
  ![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513414091460.png)
  Pointwise conv的计算复杂度为：
  ```math
    M * N * Df * Df
  ```
上述两个步骤，理论上相对于原网络的计算复杂度减小为：
![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513414206276.png)

#### ShuffleNet
ShuffleNet基于MobileNet的group思想，将卷积操作限制到特定的输入通道。而与之不同的是，**ShuffleNet将输入的group进行打散**，从而保证每个卷积核的感受野能够分散到不同group的输入中，增加了模型的学习能力  
ShuffleNet的创新点：
- 提出了一个类似于ResNet的BottleNeck单元[下图1]
- 提出将1x1卷积采用group操作会得到更好的分类性能
- 提出了核心的shuffle操作将不同group中的通道进行打散[下图2]

![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513414696143.png)
![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513414727325.png)

ShuffleNet如何进行group操作目前还不清楚，需要看看相关文章或者直接看代码？

#### 参考文献

[1] ImageNet Classification with Deep Convolutional Neural Networks

[2] Very Deep Convolutional Networks for Large-Scale Image Recognition

[3] Going Deeper with Convolutions

[4] Rethinking the Inception Architecture for Computer Vision

[5] SqueezeNet: AlexNet-level accuracy with 50x fewer parameters and < 0.5MB model size

[6] Deep Compression: Compressing Deep Neural Networks with Pruning, Trained Quantization and Huffman Coding

[7] Distilling the Knowledge in a Neural Network

[8] XNOR-Net: ImageNet Classification Using Binary Convolutional Neural Networks

[9] MobileNets: Efficient Convolutional Neural Networks for Mobile Vision Applications

[10] ShuffleNet: An Extremely Efficient Convolutional Neural Network for Mobile Devices

[11] Network in Network

[12] EIE: Efficient Inference Engine on Compressed Deep Neural Network