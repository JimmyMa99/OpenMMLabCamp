# Transform 数据增强贡献指南

在介绍如何贡献 Transform 数据增强之前，建议先阅读 MMCV 的 [Transform 教程](https://mmcv.readthedocs.io/zh_CN/latest/understand_mmcv/data_transform.html)，本文档只是对教程的简化和总结。

## 实现一个简单的自定义数据变换

要实现自定义的数据变换，通常需要以下步骤：

1. 定义一个数据增强类，并将其注册到 `mmcv` 的 `TRANSFORMS` 注册器中
2. 让增强类继承自 `BaseTransform`
3. 实现 transform 函数，并让其接受字典类型的输入，返回字典类型的输出

以实现翻数据增强为例：

```python
import random
import mmcv
from mmcv.transforms import BaseTransform, TRANSFORMS

@TRANSFORMS.register_module()  # 1. 注册
class MyFlip(BaseTransform):   # 2. 继承 BaseTransform
    def __init__(self, direction: str):
        super().__init__()
        self.direction = direction

    def transform(self, results: dict) -> dict:  # 接受字典类型输入
        img = results['img']
        results['img'] = mmcv.imflip(img, direction=self.direction)  # 接受字典类型输出
        return results
```

至此，一个简单的数据增强就已经完成了

## 常用的数据增强工具

OpenMMLab 系列算法库实现了非常多的 Transforms，在实现过程中我们发现很多数据变化都存在共性，因此针对这些共性 MMCV 提供了一系列的数据增强工具，方便大家复用代码，快速实现自定义的数据增强。这些工具不仅可以提高数据数据增强的开发效率，也能够通过组合的方式，基于现有的数据增强组合出一些新的数据增强。因此在实现新的数据增强时，我们需要考虑是否有必要，新的需求能否通过组合已有的数据增强来实现。

### Compose

[Compose](https://mmcv.readthedocs.io/zh_CN/latest/api/generated/mmcv.transforms.Compose.html#mmcv.transforms.Compose) 可以将一个或多个数据增强串联成一个数据增强

```python
pipeline = [
    dict(type='LoadImageFromFile', key='gt_img'),
    dict(type='Compose', transforms=[
        dict(type='RandomCrop', crop_size=(384, 384)),
        dict(type='Normalize'),
    ])
]
```

虽说看上去 `Compose` 和在列表中扩展数据增强的效果一样，但是当 `Compose` 和 TTA 组合时，会有意想不到的效果

### KeyMapper

不同的数据类在解析数据集时可能会对同样的数据使用不同的字段名，例如对于相撞为 （N, 4）的检测框类型数据，有的数据集解析出来是 `gt_bboxes`，有的是 `bboxes`。这个时候会发现先前实现的数据增强不能用了，因为实现 transform 时会访问输入 `dict` 的指定字段，字段不匹配时候会报错。于是 MMCV 提供了 [KeyMapper](https://mmcv.readthedocs.io/zh_CN/latest/api/generated/mmcv.transforms.KeyMapper.html#mmcv.transforms.KeyMapper) 来解决这个问题。

```python
# config
pipeline = [
    dict(type='LoadImageFromFile', key='gt_img'),
    # 使用 KeyMapper 将外部（原始）字段 'gt_img' 映射到内部字段 'img'
    dict(type='KeyMapper',
        mapping=dict(img='gt_img'),  # 定义输入时的字段映射
        auto_remap=True,  # 等价于 remapping=dict(img='gt_img')
        allow_nonexist_keys=True,  # 即允许 results 中不包含 `gt_img` 字段
        transforms=[
            # Transform 实现中使用标准字段 'img' 即可
            dict(type='Crop', crop_size=(384, 384), random_crop=True),
            dict(type='Normalize'),
        ])
]
```

上面这段配置就能实现将 `gt_img` 字段映射到 `img`，并且在 `Crop` 和 `Normalize` 时使用 `img` 字段，并最后将 `img` 字段映射回 `gt_img` 输出到 `results` 中。


### TransformBroadcaster

有些时候我们需要对不同字段的数据施加相同的变换，例如对图像和标注同时进行翻转，高分辨率、低分辨率的数据进行归一化，这个时候就可以使用 [TransformBroadcaster](https://mmcv.readthedocs.io/zh_CN/latest/api/generated/mmcv.transforms.TransformBroadcaster.html#mmcv.transforms.TransformBroadcaster)。

```python
pipeline = [
    dict(type='LoadImageFromFile', key='lq'),  # low-quality image
    dict(type='LoadImageFromFile', key='gt'),  # ground-truth image
    # 使用 TransformBroadcaster，将多个外部字段 ('lq' 和 'gt'）依次映射到内部字段
    # 'img'，并用 wrapped transforms 依次处理
    dict(type='TransformBroadcaster',
        mapping=dict(img=['lq', 'gt']),  # 情况 1： 来自多个字段
        # input_mapping=dict(img='images'),  #情况 2： 来自一个包含多个数据的字段
        auto_remap=True,
        share_random_param=True,  # 在处理多个数据字段时，使用一组相同的随机参数
        transforms=[
           # Transform 实现中使用标准字段 'img' 即可
            dict(type='RandomCrop', crop_size=(384, 384)),
            dict(type='Normalize'),
        ])
]
```

需要注意的是，像 `RandomCrop` 之类的数据增强通常存在一定的随机性，为了保证不同字段的数据增强使用相同的随机参数，需要将 `share_random_param` 设置为 `True`。同样的，为了保证自定义的数据增强策略能够被 `TransformBroadcaster` 组合使用，并能够通过 `share_random_param` 控制随机性。我们需要将数据增强中包含随机性的代码抽象到一个方法中，并对其使用 [cache_randomness](https://github.com/open-mmlab/mmcv/blob/89a264527e3dc9c5eebed6195faa709d446c7a9c/mmcv/transforms/utils.py#L14) 装饰器：

```python
import numpy as np
from mmcv.transforms.utils import cache_randomness

@TRANSFORMS.register_module()  # 1. 注册
class MyFlip(BaseTransform):   # 2. 继承 BaseTransform
    def __init__(self, direction: str):
        super().__init__()
        self.direction = direction

    @cache_randomness
    def _should_flip(self):
        return np.random.random() > 0.5

    def transform(self, results: dict) -> dict:  # 接受字典类型输入
        if self._should_flip():
            img = results['img']
            results['img'] = mmcv.imflip(img, direction=self.direction)  # 接受字典类型输出
        return results
```

**再强调一遍**，要保证自定义数据增强在 `TransformBroadcaster` 中，能够使用相同的随机参数处理不同字段的数据，需要满足两个条件：

- 将含有随机性的代码抽象到一个或多个方法中
- 使用 `cache_randomness` 装饰器装饰一个或多个随机方法

### RandomApply 和 RandomApply

为了为数据增强引入更强的随机性，MMCV 实现了：

- [RandomChoice](https://mmcv.readthedocs.io/zh_CN/latest/api/generated/mmcv.transforms.RandomChoice.html#mmcv.transforms.RandomChoice)，用于在多个候选 sub-pipeline 中随机选择一个来执行。
- [RandomApply](https://mmcv.readthedocs.io/zh_CN/latest/api/generated/mmcv.transforms.RandomApply.html?highlight=RandomApply) 用于按照一定概率执行被包裹的 sub-pipeline。

```python
# 使用 RandomApply 在 2 个 sub-pipeline 中随机选择
pipeline = [
    ...
    dict(type='RandomChoice',
        transforms=[
            [dict(type='RandomHorizontalFlip')],  # sub-pipeline 1
            [dict(type='RandomRotate')],  # sub-pipeline 2
        ]
    ),
    ...
]

# 使用 RandomApply 随机执行一个 sub-pipeline
pipeline = [
    ...
    dict(type='RandomApply',
        transforms=dict(type='RandomHorizontalFlip'),
        prob=0.3),
    ...
]
```

### 测试时数据增强

TTA（Test time augmentation） 是一种测试阶段的数据增强策略，旨在测试过程中，对同一张图片做翻转、缩放之类的数据增强，将增强后每张图片预测的结果还原到原始尺寸后做融合，以获得更加准确的预测结果。如果你有一些非常定制化的 TTA 流程，在重新实现 TTA 之前，不妨先看看能否通过组合的方式，基于现有的 [TestTimeAug](https://mmcv.readthedocs.io/zh_CN/latest/api/generated/mmcv.transforms.TestTimeAug.html?highlight=TestTimeAug) 实现你想要的功能，例如这样的配置:

```python
dict(type='TestTimeAug',
     transforms=[
        [dict(type='Resize', scale=(1333, 400), keep_ratio=True),
         dict(type='Resize', scale=(1333, 800), keep_ratio=True)],
        [dict(type='RandomFlip', prob=1.),
         dict(type='RandomFlip', prob=0.)],
        [[dict(type='CenterCrop', crop_size=100), dict(type='RandomRotate', crop_size=100)],
        [dict(type='PackDetInputs',
              meta_keys=('img_id', 'img_path', 'ori_shape',
                         'img_shape', 'scale_factor', 'flip',
                         'flip_direction'))]])
```

可以对同一张图片进行 8 次增强：

```mermaid
flowchart TD;
    ori_image --> Resize1["Resize1(scale=(1333, 400))"];
    ori_image --> Resize2["Resize1(scale=(1333, 800))"];
    Resize1 --> RandomFlip11["RandomFlip1(prob=0)"];
    Resize1 --> RandomFlip12["RandomFlip1(prob=1)"];
    Resize2 --> RandomFlip21["RandomFlip1(prob=0)"];
    Resize2 --> RandomFlip22["RandomFlip1(prob=1)"];
    RandomFlip11 --> CenterCrop111["CenterCrop111(crop_size=100)"];
    RandomFlip11 --> CenterCrop112["CenterCrop112(crop_size=100)"];
    RandomFlip12 --> CenterCrop121["CenterCrop121(crop_size=100)"];
    RandomFlip12 --> CenterCrop122["CenterCrop122(crop_size=100)"];
    RandomFlip21 --> CenterCrop211["CenterCrop211(crop_size=100)"];
    RandomFlip21 --> CenterCrop212["CenterCrop212(crop_size=100)"];
    RandomFlip22 --> CenterCrop221["CenterCrop221(crop_size=100)"];
    RandomFlip22 --> CenterCrop222["CenterCrop222(crop_size=100)"];
```

用户可通过堆叠不同的增强策略以实现指数级别的数据增强

## PR 参考

以 MMCV 2.0 开发过程中提交的 [PR](https://github.com/open-mmlab/mmcv/pull/1730/files) 为例，为 MMCV 贡献数据变换模块需要完成以下步骤：

1. 更新 `__init__.py`
2. 更新中英文版本的 [transform.rst](https://github.com/open-mmlab/mmcv/blob/main/docs/zh_cn/api/transforms.rst)（由于文档滞后于样例 PR 实现，因此样例中并没有这一步骤）
3. 按照之前章节的指引，实现新的数据变换模块
4. 补充单元测试
5. 补充文档（可选）
