## 卷积的意义 
从“卷积”、到“图像卷积操作”、再到“卷积神经网络”  - 王木头 [^2]
我的理解，从卷积的意义来理解，卷积核就是通过计算一块区域内像素对区域中心像素的影响，提取出该区域内图像的局部特征。

## 池化层的作用
池化层（Pooling Layer）的理解则简单得多，其可以理解为对图像进行降采样的过程，对于每一次滑动窗口中的所有值，输出其中的最大值（MaxPooling）、均值或其他方法产生的值。例如，对于一个三通道的 16×16 图像（即一个 16*16*3 的张量），经过感受野为 2×2，滑动步长为 2 的池化层，则得到一个 8*8*3 的张量。

# 1*1卷积的作用
1*1卷积的作用 [^3]

## CNN 参数计算 [^1]

[^1]: blog.csdn.net/qian99/article/details/79008053?spm=1001.2101.3001.6650.5&utm_medium=distribute.pc_relevant.none-task-blog-2~default~OPENSEARCH~Rate-5.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~OPENSEARCH~Rate-5.pc_relevant_aa&utm_relevant_index=10

[^2]: https://www.bilibili.com/video/BV1ce4y1p7jF/?spm_id_from=pageDriver&vd_source=5f32e3add93ed12904a06d6e939b4b6f

[^3] https://blog.csdn.net/weixin_46838716/article/details/126415087