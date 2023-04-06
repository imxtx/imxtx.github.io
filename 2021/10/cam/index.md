# 可解释性深度学习：Class Activation Map (CAM)


本文主要介绍可解释性深度学习方向的一个重要工作：Class Activation Map (CAM)。
<!--more-->

## 简介

Class Activation Map (CAM)，即**类激活映射图**是在论文 [Learning Deep Features for Discriminative Localization](https://arxiv.org/abs/1512.04150) 中提出来的。CAM 图可以帮助我们可视化输入图像中模型感兴趣的区域，对于深度学习的可解释性研究有非常大的帮助。图 1 所示的是一个动作分类任务的 CAM 图，可以看到类别刷牙对应了手和嘴巴部分，类别砍树对应了电锯和人头部分

{{< figure src="cam_example.png" caption="图1：动作分类任务的 CAM 图例子" width="700" >}}

## CAM 的生成过程

给定一张图像，使用 $f\_k(x,y)$ 表示最后一个卷积层输出的特征图中第 $k$ 个通道中空间位置位于 $(x,y)$ 处的激活值，那么在通道 $k$ 上进行 Global Average Pooling (GAP) 之后的结果为 $F^k = \sum\_{x,y} f\_k(x,y)$（这里的 GAP 应该还要除以特征图的长宽之积，论文作者没有写明）。因此，给定一个类别 $c$，softmax 的输入 $S\_c = \sum\_k w\_k^c F\_k$，其中 $w\_k^c$ 表示特征图 $k$ 的对于类别 $c$ 的权重（重要性）。最后 softmax 关于类别 $c$ 的输出概率为 $P\_c = \frac{\exp(S\_c)}{\sum\_c \exp(S\_c)}$。

{{< figure src="cam_computation.png" caption="图2：CAM 图的计算示意图" >}}

如图 2 中所示，最后一层卷积中每个通道的特征图（使用不同颜色标出）经过 GAP 后得到对应的激活值 $F^k$，对于类别 Australian terrier (澳大利亚梗)，softmax 的输入就是按权重（$W\_1, W\_2, ..., W\_n$）把这些 GAP 之后得到的激活值做一个 weighted sum。

把 $F^k = \sum\_{x,y} f\_k(x,y)$ 放到 $S\_c = \sum\_k w\_k^c F\_k$ 中可得

$$
S\_c = \sum\_k w\_k^c \sum\_{x,y} f\_k(x,y) = \sum\_{x,y} \sum\_k w\_k^c f\_k(x,y) \tag{1}
$$

定义 $M\_c$ 为类别 $c$ 的类激活映射，那么对于特征图上空间中的每个点有

$$
M\_c(x,y) = \sum\_k w\_k^c f\_k(x,y) \tag{2}
$$

因此，$S\_c = \sum\_{x,y} M\_c(x,y)$，$M\_c(x,y)$ 则直接表示了 CAM 图中位于 $(x,y)$ 处的激活值对于类别 $c$ 的重要性。从图 2 下半部分可以看到，CAM 图就是不同通道的特征图按照学习到的权重加权求和得到的，最后再 resize 到和输入图像相同的大小，叠加起来就得到了输入图像上的类激活图。图 3 是作者在 [ILSVRC](https://www.image-net.org/challenges/LSVRC/index.php) 数据集上可视化的效果：

{{< figure src="cams_on_ILSVRC.png" caption="图3：ILSVRC 数据集上的一些 CAM 图示例" width="700" >}}

从图 4 可以看出，同一张图像中，不同类别对应的 CAM 图也是不同的。例如宫殿（palace）和穹顶（dome）明显对应图像中不同的区域。

{{< figure src="cams_on_different_parts.png" caption="图4：同一张图像中不同物体的 CAM 图" width="700" >}}

## 实验结果

### 实验设置

作者使用了三个模型进行实验，分别是 AlexNet、VGG 和 GoogLeNet，作者将每个网络的最后一个 FC 替换为 GAP，之后跟着一个全连接的 softmax 层，其他具体的实验设置可以看原论文。

### 分类

如下表所示，作者比较了修改后的网络和原始网络的分类性能，发现修改后的网络并没有性能上的下降。

{{< figure src="table1.png" width="700" >}}

此外，作者还对 GAP 和 Global Max Pooling (GMP) 进行了比较，作者认为 GAP 能够考虑特征图中所有位置的信息，而 GMP 只会考虑特征图中值最大的一个点，因此使用 GAP 能够在 CAM 图中更好地定位物体位置。从表 1 中可以看出 GMP 其实也没有降点很多。

### 目标定位

作者还进行了目标定位的实验，并通过 CAM 图绘制出了 bounding box。具体绘制方法是，首先把激活图中所有的大于激活图中最大值 20% 的元素分割出来，然后计算这些元素组成的一个最大的连通图，bounding box 则是包含最大连通图的方框。下表是目标定位的性能对比，可以看到使用 CAM 定位的性能还是不错的：

{{< figure src="table2.png" width="700" >}}

下表是和有监督的方法对比，CAM 和有监督的性能比较接近了：

{{< figure src="table3.png" width="700" >}}

下图是目标定位的例子，绿色是 ground truth，红色是 CAM 图得到的 bounding box，可以看出大体上的位置还是正确的：

{{< figure src="localizaiton_example.png" caption="图5：使用 CAM 生成 bounding box，第一行对应 GoogLeNet，第二行对应 AlexNet" width="700" >}}

此外，作者发现使用 CAM 图生成的 bounding box 能够提升分类任务的性能。如下表所示，第一行是使用全图进行分类，第二行是先用 CAM 图生成 bounding box，然后再在使用 bounding box 裁剪后的图像上进行分类，结果接近第三行带有 bounding box 标注的性能：

{{< figure src="table4.png" width="700" >}}

### 通用性

如图 6 所示，作者使用训练好的 GoogLeNet 在其他数据集上提取 CAM 图，发现 CAM 图能够很好的对应分类标签，说明 CAM 图的生成具有较强的泛化性。

{{< figure src="cam_other_datasets.png" caption="图6：CAM 在其他分类数据集上的效果" >}}

### 知识发现

如下图所示，通过 CAM 图还可以发现和某个场景最相关的物体：

{{< figure src="information_discovery.png" caption="图7：左）CAM 图中和餐厅最相关的有桌子、椅子、盘子等。右）CAM 图中和浴室最相关的物体有洗手池、水龙头等" >}}

此外作者还做了使用 CAM 图检测文字、视觉问答等实验，都发现 CAM 图能够很好显示出预测结果最关注的图像部分。

## 代码

- 论文源码：<https://github.com/zhoubolei/CAM>
- 一个不错的 CAM 库：<https://github.com/frgfm/torch-cam>

我们用下面这张自行车的图像来试验一下 CAM 图的生成：

{{< figure src="bicycle.jpg" >}}

这里简单说明一下一些关键的代码，下面是一些基本的导入、设置和加载预训练模型：

```python
import io
from PIL import Image
from torchvision import models, transforms
import torch.nn.functional as F
import numpy as np
import cv2
import json

# 标签和测试图像
labels_file = 'imagenet-simple-labels.json'
image_file = 'data/bicycle.jpg'
# 加载预训练模型
net = models.resnet18(weights=models.ResNet18_Weights.IMAGENET1K_V1)
net.eval()
# ResNet18 最后一个 block 的名字
finalconv_name = 'layer4'
```

模型结构如下：

```text
ResNet(
  (conv1): Conv2d(3, 64, kernel_size=(7, 7), stride=(2, 2), padding=(3, 3), bias=False)
  (bn1): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
  (relu): ReLU(inplace=True)
  (maxpool): MaxPool2d(kernel_size=3, stride=2, padding=1, dilation=1, ceil_mode=False)
  (layer1): Sequential(
    (0): BasicBlock(
      (conv1): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    )
    (1): BasicBlock(
      (conv1): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    )
  )
  (layer2): Sequential(
    (0): BasicBlock(
      (conv1): Conv2d(64, 128, kernel_size=(3, 3), stride=(2, 2), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(128, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (downsample): Sequential(
        (0): Conv2d(64, 128, kernel_size=(1, 1), stride=(2, 2), bias=False)
        (1): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      )
    )
    (1): BasicBlock(
      (conv1): Conv2d(128, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(128, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    )
  )
  (layer3): Sequential(
    (0): BasicBlock(
      (conv1): Conv2d(128, 256, kernel_size=(3, 3), stride=(2, 2), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (downsample): Sequential(
        (0): Conv2d(128, 256, kernel_size=(1, 1), stride=(2, 2), bias=False)
        (1): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      )
    )
    (1): BasicBlock(
      (conv1): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    )
  )
  (layer4): Sequential(
    (0): BasicBlock(
      (conv1): Conv2d(256, 512, kernel_size=(3, 3), stride=(2, 2), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(512, 512, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (downsample): Sequential(
        (0): Conv2d(256, 512, kernel_size=(1, 1), stride=(2, 2), bias=False)
        (1): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      )
    )
    (1): BasicBlock(
      (conv1): Conv2d(512, 512, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn1): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
      (relu): ReLU(inplace=True)
      (conv2): Conv2d(512, 512, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
      (bn2): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    )
  )
  (avgpool): AdaptiveAvgPool2d(output_size=(1, 1))
  (fc): Linear(in_features=512, out_features=1000, bias=True)
)
```

接着我们需要为最后一个 block 注册一个钩子函数：

```python
# 保存 forward 过程中通过钩子函数记录的特征
features_blobs = []

def hook_feature(module, input, output):
    features_blobs.append(output.data.cpu().numpy())

# 注册钩子函数
net._modules.get(finalconv_name).register_forward_hook(hook_feature)
```

下面的代码可以查看网络中每个模块的参数名，按构建的先后顺序排序：

```python
for name, param in net.named_parameters():
    print(name)
```

输出：

```text
conv1.weight
bn1.weight
bn1.bias
layer1.0.conv1.weight
...
layer4.1.conv1.weight
layer4.1.bn1.weight
layer4.1.bn1.bias
layer4.1.conv2.weight
layer4.1.bn2.weight
layer4.1.bn2.bias
fc.weight
fc.bias
```

获得 softmax 之前的 fc 层的权重：

```python
# 获得 softmax 的权重，从上面网络结构看出最后一个 Linear 的 bias 为 True，所以倒数第二个
# 参数才是全连接的权重 fc.weight
params = list(net.parameters())
weight_softmax = np.squeeze(params[-2].data.numpy())
```

下面是计算 CAM 图的函数：

```python
def returnCAM(feature_conv, weight_softmax, class_idx):
    # 生成 CAM 图并 resize 到 256x256
    size_upsample = (256, 256)
    bz, nc, h, w = feature_conv.shape
    output_cam = []
    # class_idx 是一个列表，对每个传进来的类别都计算一个 CAM 图
    for idx in class_idx:
        # [1,512]x[1,H*W] 这里即是图 2 中加权求和的过程
        cam = weight_softmax[idx].dot(feature_conv.reshape((nc, h*w)))
        cam = cam.reshape(h, w)
        cam = cam - np.min(cam)
        cam_img = cam / np.max(cam)
        cam_img = np.uint8(255 * cam_img)
        output_cam.append(cv2.resize(cam_img, size_upsample))
    return output_cam
```

下面是加载图像、标签以及一些预处理：

```python
normalize = transforms.Normalize(
   mean=[0.485, 0.456, 0.406],
   std=[0.229, 0.224, 0.225]
)
preprocess = transforms.Compose([
   transforms.Resize((224,224)),
   transforms.ToTensor(),
   normalize
])

# 加载测试图像
img_pil = Image.open(image_file)
img_tensor = preprocess(img_pil)
logit = net(img_tensor.unsqueeze(0))

# 加载标签列表
with open(LABELS_file) as f:
    classes = json.load(f)
```

下面是使用测试图像生成 CAM 图的代码：

```python
h_x = F.softmax(logit, dim=1).data.squeeze()
probs, idx = h_x.sort(0, descending=True)
probs = probs.numpy()
idx = idx.numpy()

# 输出预测结果
for i in range(0, 5):
    print('{:.3f} -> {}'.format(probs[i], classes[idx[i]]))
print('output CAM.jpg for the top1 prediction: {}'.format(classes[idx[0]]))

# 给最概率最大的预测类别生成 CAM 图
CAMs = returnCAM(features_blobs[0], weight_softmax, [idx[0]])

# 把 CAM 图加到原图上并保存
img = cv2.imread(image_file)
height, width, _ = img.shape
heatmap = cv2.applyColorMap(cv2.resize(CAMs[0],(width, height)), cv2.COLORMAP_JET)
result = heatmap * 0.3 + img * 0.7
cv2.imwrite('CAM.jpg', result)
```

`cv2.applyColorMap` 函数可以把灰度图转成有颜色的图，非常方便，可以参考[文档](https://docs.opencv.org/3.4/d3/d50/group__imgproc__colormap.html)。最终输出：

```text
0.932 -> mountain bike
0.043 -> unicycle
0.011 -> moped
0.006 -> tandem bicycle
0.004 -> tricycle
output CAM.jpg for the top1 prediction: mountain bike
```

最终的 CAM 图像：

{{< figure src="bicycle_cam.jpg" >}}

## 总结

CAM 是可解释性深度学习方向上非常重要的一个工作，能够帮助我们可视化模型的关注点，同时也为我们的深度模型提供了一个非常有用的 Debug 工具。

