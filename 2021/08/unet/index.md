# U-Net 源码解读


本文对 U-Net 的源码进行讲解。
<!--more-->

U-Net 最初是在 2015 年为了解决医学图像分割而被提出来的，由于其性能非常不错，目前仍有非常广泛的应用。本文主要对 U-Net 的源码进行解读，论文链接：<https://arxiv.org/abs/1505.04597>

如下图所示，U-Net 之所以叫做 U-Net 是因为它看起来像个大写的字母 U。U-Net 在形式上也可以归为自编码器，由编码器和解码器构成。

{{< figure src="u-net-architecture.png" caption="U-Net 结构图" >}}

从图中可以看到，U-Net 包含左半边的编码过程，和右半边的解码过程。输入为单通道大小为 572\*572 的图像，输出为两通道的大小为 388\*388 的分割图。

{{< figure src="medical_image.png" caption="细胞图像分割" >}}

U-Net 还经常被用来自做遥感图像的分割：

{{< figure src="satellite_image.png" caption="遥感卫星图像分割" >}}

下面分别讲解 U-Net 的编码和解码过程。

## 编码过程

结构图左边横向的每一层次都由两次带 ReLU 的卷积和一次 MaxPooling 构成（图中深蓝色箭头和红色箭头）。经过四次编码下采样之后，得到了尺寸为 32\*32，通道数为 512 的特征图。接着又进行了两次卷积得到了尺寸为 28\*28，通道数为 1024 的特征图，也就是结构图中最下面的部分（两个蓝色箭头）。

## 解码过程

解码器的每个层次中，首先是一次上采样（转置卷积或双线性插值）将通道数减半，然后和同层级从左边 skip connect 过来的特征图（图中灰色箭头）沿着通道方向进行拼接，接着又是两次带 ReLU 的卷积。经过四次解码上采样之后得到尺寸为 388\*388，通道数为 64 的特征图，最后再进行 1\*1 的卷积得出两通道的大小为 388\*388 输出。

整个过程还是挺简单的，U-Net 成功的关键就在于同层级的 skip-connection，它将两部分的信息进行了融合：一个是每个上采样层前一层的语义特征，另一个是左边传来的未经 MaxPooling 下采样的无损信息，这样既能提取 high-level 的语义信息，又能保留 low-level 的细节。U-Net 设计的初衷就是应对不同分辨率的输入图像，如果去掉 skip-connection，那么会丢失很多原始图像的信息，导致上采样的结果很差。

注意，原始的 U-Net 输入为单通道图像，输出为两通道的分割图，并且编码过程中的特征图和解码过程中同层级的特征图大小是不同的。我们不一定要按照原论文一模一样地实现，输入输出的通道数我们可以根据任务自己定，每一个层次的卷积也可以使用 Same Padding 简化编码，也就是在 MaxPooling 之前都不改变特征图的尺寸，而且效果也不差。

## 转置卷积 vs 双线性插值

U-Net 上采样中可以使用两种方式，一种是转置卷积，一种是双线性插值。目前来看双线性插值的效果较好，而转置卷积会有棋盘效应（Checkerboard Artifacts）：

{{< figure src="checkerboard.png" caption="棋盘效应" >}}

关于棋盘效应和双线性插值的优缺点可以参考这篇文章：<https://distill.pub/2016/deconv-checkerboard/>

## 代码讲解

首先我们定义一个 `DoubleConv` 类用于 U-Net 中每一层中的两次卷积：

```python
class DoubleConv(nn.Module):
    """(Conv2d => BatchNorm2d => ReLU) * 2"""

    def __init__(self, in_channels, out_channels) -> None:
        super().__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(in_channels, out_channels, 3, 1, 1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
            nn.Conv2d(out_channels, out_channels, 3, 1, 1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
        )

    def forward(self, x):
        return self.conv(x)
```

接着，我们定义 U-Net 的初始化函数：

```python
class UNet(nn.Module):
    def __init__(
        self,
        in_channels=3,
        out_channels=3,
        n_features=[64, 128, 256, 512],
        bilinear=True,
    ) -> None:
        super().__init__()
        self.downs = nn.ModuleList()
        self.ups = nn.ModuleList()
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)
        # 代码未完
```

初始化函数的参数包括：

`in_channels`：U-Net 的输入通道数，默认为 3，可自己设定
`out_channels`：U-Net 的输出通道数，默认为 3，可自己设定
`n_features` ：U-Net 从上到下每一层特征图的大小（不包含最下面一层），可以自己设定
`bilinear` ：上采样是否使用双线性插值
`self.downs` 和 `self.ups` 是两个模块列表，用于保存编码器和解码器每一层的模块，`self.pool` 是池化层，用于下采样，使特征图大小减半。

构建下采样编码器的模块非常简单，我们只需要为不同的层次创建对应的 `DoubleConv` 就行了：

```python
# downsampling
for n_feature in n_features:
    self.downs.append(DoubleConv(in_channels, n_feature))
    in_channels = n_feature
```

上采样解码器稍微复杂点，如果是双线性插值，那么由一个 nn.Upsample 和一个卷积构成，因为 nn.Upsample 不会减小通道数，这里的卷积是为了将双线性上采样后的通道数减半，而转置卷积本身可以在上采样的同时改变通道数。不管是插值还是转置卷积都需要跟着做一次 DoubleConv：

```python
# upsampling
for n_feature in reversed(n_features):
    if bilinear:  # bilinear upsampling
        self.ups.append(
            nn.Sequential(
                nn.Upsample(scale_factor=2, mode="bilinear"),
                nn.Conv2d(n_feature * 2, n_feature, 3, 1, 1),
            )
        ) 
    else:  # transposed conv upsampling
        self.ups.append(
            nn.ConvTranspose2d(
                n_feature * 2, n_feature, kernel_size=2, stride=2
            )
        )
    self.ups.append(DoubleConv(n_feature * 2, n_feature))
```

对应的图示：

一次上采样，两次卷积
最后定义 U-Net 最下面的层为 `bottleneck`，输出 1\*1 卷积定义为 `final_conv` ：

```python
self.bottleneck = DoubleConv(n_features[-1], n_features[-1] * 2)
self.final_conv = nn.Conv2d(n_features[0], out_channels, kernel_size=1)
```

完整的初始化函数如下：

```python
class UNet(nn.Module):
    def __init__(
        self,
        in_channels=3,
        out_channels=3,
        n_features=[64, 128, 256, 512],
        bilinear=True,
    ) -> None:
        super().__init__()
        self.downs = nn.ModuleList()
        self.ups = nn.ModuleList()
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)

        # 下采样
        for n_feature in n_features:
            self.downs.append(DoubleConv(in_channels, n_feature))
            in_channels = n_feature

        # 上采样
        for n_feature in reversed(n_features):
            if bilinear:  # 双线性插值上采样
                self.ups.append(
                    nn.Sequential(
                        nn.Upsample(scale_factor=2, mode="bilinear"),
                        nn.Conv2d(n_feature * 2, n_feature, 3, 1, 1),
                    )
                )
            else:  # 转置卷积上采样
                self.ups.append(
                    nn.ConvTranspose2d(
                        n_feature * 2, n_feature, kernel_size=2, stride=2
                    )
                )
            self.ups.append(DoubleConv(n_feature * 2, n_feature))

        # 最下面一层
        self.bottleneck = DoubleConv(n_features[-1], n_features[-1] * 2)
        # 输出1*1卷积
        self.final_conv = nn.Conv2d(n_features[0], out_channels, kernel_size=1)
```

下面来看 forward 函数怎么写：

```python
def forward(self, x):
    # 保存每一层左边用于跳连的特征图
    skip_connections = []

    # 下采样：每层【两次卷积，一次MaxPooling】
    for down in self.downs:
        x = down(x)
        skip_connections.append(x)  # 保存跳连
        x = self.pool(x)

    # 经过最下面的一层
    x = self.bottleneck(x)
    # 跳连是从上层到下层保存的：[64,128,256,512]，所以需要翻转一下方便拼接
    skip_connections = skip_connections[::-1]

    # 上采样，每层【一次上采样，两次卷积】
    for idx in range(0, len(self.ups), 2):
        x = self.ups[idx](x)  # 上采样
        skip_connection = skip_connections[idx // 2]

        # 修正跳连的尺寸
        if x.shape != skip_connection.shape:
            x = TF.resize(x, size=skip_connection.shape[2:])
        
        concat_skip = torch.cat([skip_connection, x], dim=1)  # 跳连拼接
        x = self.ups[idx + 1](concat_skip)  # 两次卷积

    return self.final_conv(x)  # 输出
```

注意，由于输入图像大小并不一定正好是 2 的倍数，那么 MaxPooling 会向下取整，那么跳连和上采样后特征图的尺寸不一定对得上，这里可以直接 resize 跳连，长宽只差一个像素，不会影响性能，当然也可以给跳连加 padding。完整的 U-Net 代码如下：

```python
import torch
import torch.nn as nn
import torchvision.transforms.functional as TF

class DoubleConv(nn.Module):
    """(Conv2d => BatchNorm2d => ReLU) * 2"""

    def __init__(self, in_channels, out_channels) -> None:
        super().__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(in_channels, out_channels, 3, 1, 1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
            nn.Conv2d(out_channels, out_channels, 3, 1, 1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
        )

    def forward(self, x):
        return self.conv(x)

class UNet(nn.Module):
    def __init__(
        self,
        in_channels=3,
        out_channels=3,
        n_features=[64, 128, 256, 512],
        bilinear=True,
    ) -> None:
        super().__init__()
        self.downs = nn.ModuleList()
        self.ups = nn.ModuleList()
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)

        # downsampling
        for n_feature in n_features:
            self.downs.append(DoubleConv(in_channels, n_feature))
            in_channels = n_feature

        # upsampling
        for n_feature in reversed(n_features):
            if bilinear:  # bilinear upsampling
                self.ups.append(
                    nn.Sequential(
                        nn.Upsample(scale_factor=2, mode="bilinear"),
                        nn.Conv2d(n_feature * 2, n_feature, 3, 1, 1),
                    )
                )  
            else:  # transposed conv upsampling
                self.ups.append(
                    nn.ConvTranspose2d(
                        n_feature * 2, n_feature, kernel_size=2, stride=2
                    )
                )  
            self.ups.append(DoubleConv(n_feature * 2, n_feature))

        self.bottleneck = DoubleConv(n_features[-1], n_features[-1] * 2)
        self.final_conv = nn.Conv2d(n_features[0], out_channels, kernel_size=1)

    def forward(self, x):
        skip_connections = []

        for down in self.downs:
            x = down(x)
            skip_connections.append(x)
            x = self.pool(x)

        x = self.bottleneck(x)
        skip_connections = skip_connections[::-1]

        for idx in range(0, len(self.ups), 2):
            x = self.ups[idx](x)  # upsampling
            skip_connection = skip_connections[idx // 2]

            # correct mismatch caused by down pooling
            if x.shape != skip_connection.shape:
                x = TF.resize(x, size=skip_connection.shape[2:])

            concat_skip = torch.cat([skip_connection, x], dim=1)
            x = self.ups[idx + 1](concat_skip)  # doube conv

        return self.final_conv(x)

def test():
    x = torch.randn((3, 1, 255, 255))
    model = UNet(in_channels=1, out_channels=1, bilinear=True)
    preds = model(x)
    print(x.shape)
    print(preds.shape)
    assert preds.shape == x.shape

if __name__ == "__main__":
    test()
```

最后可以测试一下实现是否正确：

```bash
imxtx@ubuntu:~/code$ python models/unet.py 
torch.Size([3, 1, 255, 255])
torch.Size([3, 1, 255, 255])
```

## 参考资料

- <https://github.com/aladdinpersson/Machine-Learning-Collection/tree/master/ML/Pytorch/image_segmentation/semantic_segmentation_unet>
- <https://github.com/milesial/Pytorch-UNet/>
- <https://towardsdatascience.com/u-net-explained-understanding-its-image-segmentation-architecture-56e4842e313a>

