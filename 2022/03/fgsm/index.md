# 对抗攻击系列：FGSM


本文介绍对抗攻击与防御方向最基础的攻击方法 Fast Gradient Sign Method。
<!--more-->

## 简介

**对抗样本**（adversarial example）是指经过特殊修改的能够导致模型给出错误预测的样本，图 1 中的熊猫（panda）经过轻微的扰动（perturbation）之后被模型预测成了长臂猿（gibbon），但我们看不出正常图像和对抗样本之间有任何显著差别。

{{< figure src="adversarial_example.png" caption="图1：对抗扰动的例子" >}}

FGSM 是一种简单有效的生成对抗样本的攻击方法，最先由 Goodfellow 等人在论文 [Explaining and Harnessing Adversarial Examples](https://arxiv.org/abs/1412.6572) 中提出，该方法生成对抗样本的步骤如下：

1. 获得一张干净的输入图像
2. 使用训练好的模型给出预测
3. 计算预测和真实标签之间的损失，例如交叉熵损失
4. 计算损失关于输入图像的梯度
5. 获得梯度的符号
6. 将带符号的梯度加到输入图像上以生成对抗样本

以上步骤可以用一个简单的公式予以总结：

$$
x\_{adv} = x + \epsilon \* \text{sign} (\nabla\_x J(\theta,x,y)) \tag{1}
$$

其中，$x\_{adv}$ 表示对抗样本，$x$ 表示干净样本，$\epsilon$ 是超参，表示添加扰动的强度，一般来说强度越大，攻击越容易成功，但也越容易看出对抗样本中的一些 artifacts，$\text{sign}$ 表示取符号，括号中的 $\nabla\_x J(\theta,x,y)$ 表示损失关于干净样本 $x$ 的梯度。

## 攻击类型

从攻击目的角度可以将对抗攻击分为无目标攻击（untargeted attack）和有目标攻击（targeted attack）。以分类任务为例，无目标攻击的目的是使模型给出错误的预测，有目标攻击的目的是使模型将对抗样本预测为攻击者想要的目标类别，达到欺骗模型的目的。使用 $y\_s$ 表示样本 $x$ 的真实标签，当进行无目标攻击时，目的是使损失值变大所以公式 (1) 可以写为

$$
x\_{adv} = x + \epsilon \* \text{sign} (\nabla\_x J(\theta,x,y\_s)) \tag{2}
$$

这时 $\epsilon$ 需取正数，公式 (2) 所表达的是一个梯度上升的过程。使用 $y\_t$ 表示想要模型给出的目标预测标签，当进行有目标攻击时，目的是使损失值变小所以公式 (1) 可以写为

$$
x\_{adv} = x - \epsilon \* \text{sign} (\nabla\_x J(\theta,x,y\_t)) \tag{3}
$$

这里 $\epsilon$ 也取正数，公式 (3) 所表达的是一个梯度下降的过程，优化输入 $x$ 使得模型给出的预测尽量为 $y_t$。

## 伪代码

这里列出无目标攻击的伪代码，公式 (2) 可以表达为如下伪代码：

```python
# FGSM attack code
def fgsm_attack(image, epsilon, data_grad):
    # Collect the element-wise sign of the data gradient
    sign_data_grad = data_grad.sign()
    # Create the perturbed image by adjusting each pixel of 
    # the input image
    perturbed_image = image + epsilon*sign_data_grad
    # Adding clipping to maintain [0,1] range
    perturbed_image = torch.clamp(perturbed_image, 0, 1)
    # Return the perturbed image
    return perturbed_image
```

以分类任务为例，下面是使用整个数据集生成对抗样本的伪代码：

```python
for data, target in test_loader:
    data, target = data.to(device), target.to(device)
    data.requires_grad = True
    # Forward pass the data through the model
    output = model(data)
    # get the index of the max log-probability
    init_pred = output.max(1, keepdim=True)[1]
    # If the initial prediction is wrong, dont bother attacking,
    # just move on
    if init_pred.item() != target.item():
        continue
    # Calculate the loss
    loss = F.nll_loss(output, target)
    # Zero all existing gradients
    model.zero_grad()
    # Calculate gradients of model in backward pass
    loss.backward()
    # Collect datagrad
    data_grad = data.grad.data
    # Call FGSM Attack
    perturbed_data = fgsm_attack(data, epsilon, data_grad)
    # Re-classify the perturbed image
    output = model(perturbed_data)
    # Check for success attack
    # get the index of the max log-probability
    final_pred = output.max(1, keepdim=True)[1] 
    if final_pred.item() == target.item():
        correct += 1
```

## 总结

FGSM 是一种简单的攻击方法，也很容易实现，通常作为很多新方法的 baseline。

## 参考

1. <https://pytorch.org/tutorials/beginner/fgsm_tutorial.html>
2. <https://www.tensorflow.org/tutorials/generative/adversarial_fgsm>
3. <https://pyimagesearch.com/2021/03/01/adversarial-attacks-with-fgsm-fast-gradient-sign-method/>

