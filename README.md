# A NAO-based interactive project

## 依赖库
1. PIL or pillow
2. Pynaoqi SDK for python2.7
3. openCV2
4. Tensorflow v1.1 or later

## 操作方案
### 状态描述
#### 待机
NAO开机后，在工程根目录（以下简称$ROOT）下执行命令：
```
python app.py
```
等待10s左右，NAO站起来后，即成功运行。之后，NAO将在地面进行随机的行走，此时的NAO处于待机状态。
同时，在检测到前方存在障碍物时，NAO将会左转或右转90度（根据障碍物的大体方位来选择）来避开障碍。
#### 唤醒
在待机状态下，用户能够通过两种方式对NAO进行唤醒：
1. 对NAO说“你好”。NAO能够自动检测用户的问候，以及发出问候的用户的大概方位。检测到问候后，NAO将会依次做出如下动作：
	1. 将自己的身体和面部转至大致朝向用户的方向。
	2. 对用户发出问候：“你好”。
	3. 伸出右手尝试与用户握手。当用户握住NAO的右手后，NAO将会上下摆动右手进行握手。
2. 站在NAO的面前。NAO能够自动检测视野中的人脸。当检测到人脸后，NAO将会依次做出如下动作：
	1. 转动自己的头部来追踪人脸，使得人脸保持在自己的视野中心。
	2. 对用户发出问候：“你好”。
	3. 伸出右手尝试与用户握手。当用户握住NAO的右手后，NAO将会上下摆动右手进行握手。
#### 交互
在被用户唤醒后，NAO将与用户开始交互，用户能够通过说话来对NAO下达指令，现有对NAO的指令如下：
1. “这是什么”：NAO将会检测视野中的画面，对其进行分类，并用语音播报分类结果，如“这是金鱼。”
2. 待补充...
3. “再见”：NAO将会停止对人脸的追踪，并回复“再见”，之后回到待机状态。
### 状态转移图
<div align=center><img src="https://github.com/raxxerwan/NAO_Project/blob/master/doc/frame.png" /></div>

## 原理
### 避障
NAO内置了2个声纳测距仪，发射40kHz声波对前方的障碍物进行检测。当检测到前方0.5m内障碍物反射回的声波后，NAO会改写内存中某些字段的值，记录检测到的障碍物的距离（0.5m以内）和检测到的时间戳。根据内存中该字段的值能够命令NAO及时地做出避障动作。
### 检测声源位置
NAO内置了4个麦克风分布在身体的不同侧。当外界声音从不同位置发出时，这个声音到达四个麦克风的时间不同。依据四个麦克风接受到声音的时间，可以大致测算出声源的位置。
### 人脸检测
NAO内置了人脸检测的算法，可以计算出摄像头画面中人脸的Bounding Box和置信度。
### 图像分类
NAO将摄像头中的一帧图像上传到服务器，服务器端使用Resnet v2模型对该图像进行前向推导，获得分类结果。取概率最大的分类结果传回NAO进行语音播报。流程图如下：
<div align=center><img src="https://github.com/raxxerwan/NAO_Project/blob/master/doc/imageClassifierProcess.png"></div>

可以看到，在该框架下，NAO与服务器之间的通信不再需要中间机器的协助，即NAO与服务器直接进行通信。因此节约了大量的通信时间，使得用户与NAO之间的交互可以满足实时性要求。具体的性能分析在[图像分类性能分析](#图像分类-1)。
1. 部署框架：Tensorflow v1.3.0
2. 模型：Inception-ResNet-v2，[1602.07261](https://arxiv.org/abs/1602.07261)
3. 预训练参数：[Tensorflow,210MB](http://download.tensorflow.org/models/inception_resnet_v2_2016_08_30.tar.gz)
4. 训练集：[ImageNet](http://www.image-net.org/challenges/LSVRC/2012/)
	1. 测试集：1.2M张图像
	2. 测试集：50k张图像
	3. 分类标签数：1000

## 性能
### 检测声源位置
在NAO静止状态下，对于声源位置的检测准确率较高，误差大约在10度以内。但在NAO移动状态下，由于NAO移动时各个关节会产生较大的噪音，使得信噪比急剧下降，对声源位置的检测结果较差。建议在静止状态下启用检测声源位置功能。
### 人脸检测
1. 在光照条件良好（官网参数为100-500lux，一般白天避免阳光直射的场景下都适应）的情况下，NAO的人脸检测准确率非常高。
2. NAO转动头部来追踪人脸大概有300ms的延迟，这使得用户不能快速地移动脸部，否则NAO的头部将会失去对人脸的追踪。
### 图像分类
1. 准确率。ResNet v2在ImageNet测试集上的准确率为：Top-1准确率80.4%，Top-5准确率95.3%。在实际使用时，发现只有将分类目标贴近NAO的摄像头，分类结果才较准确。同时，最好使用照片而非实物进行分类，否则分类结果也不够理想。
2. 效率。进行一次完整的图像分类过程（用户下达指令->NAO将图片上传->前向推导->语音播报结果）大约需要2s-3s时间。这其中包括将图片上传到服务器的0.3s，和进行一次前向推导的1.5s-3s不等。另外，程序读取模型参数大约需要2s-3s左右的时间，但读取参数只需要在程序刚运行时进行，之后参数将一直保存在内存中，后续的前向推导不再需要读取参数，因此该时间可以忽略。


