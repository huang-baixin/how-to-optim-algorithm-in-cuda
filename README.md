## how-to-optim-algorithm-in-cuda

> 我也维护了一个学习深度学习框架（PyTorch和OneFlow）的仓库 https://github.com/BBuf/how-to-learn-deep-learning-framework 以及一个如何学习深度学习编译器（TVM/MLIR/LLVM）的学习仓库 https://github.com/BBuf/tvm_mlir_learn , 有需要的小伙伴可以**点一点star**

本工程记录如何基于 cuda 优化一些常见的算法。请注意，下面的介绍都分别对应了子目录的代码实现，所以想复现性能的话请查看对应子目录下面的 README 。

### 0. how-to-compile-pytorch-from-source

记录如何手动编译 PyTorch 源码，学习 PyTorch 的一些 cuda 实现。

### 1. reduce

这里记录学习 NIVDIA 的[reduce优化官方博客](https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf) 做的笔记。完整实验代码见[这里](https://github.com/BBuf/how-to-optim-algorithm-in-cuda/tree/master/reduce) , 原理讲解请看：[【BBuf的CUDA笔记】三，reduce优化入门学习笔记](https://zhuanlan.zhihu.com/p/596012674) 。后续又添加了 PyTorch BlockReduce 模板以及在这个模板的基础上额外加了一个数据 Pack ,又获得了一些带宽的提升。详细数据如下：

性能和带宽的测试情况如下 (A100 PCIE 40G)：

![图片](https://user-images.githubusercontent.com/35585791/213908763-480d0c07-5709-4829-9903-db17a0ecca89.png)

### 2. elementwise

将 oneflow 的 elementwise 模板抽出来方便大家使用，这个 elementwise 模板实现了高效的性能和带宽利用率，并且用法非常灵活。完整实验代码见[这里](https://github.com/BBuf/how-to-optim-algorithm-in-cuda/blob/master/elementwise/elementwise.cu) ，原理讲解请看：[【BBuf 的CUDA笔记】一，解析OneFlow Element-Wise 算子实现](https://zhuanlan.zhihu.com/p/591058808) 。这里以逐点乘为例，性能和带宽的测试情况如下 (A100 PCIE 40G)：

|优化手段|数据类型|耗时(us)|带宽利用率|
|--|--|--|--|
|naive elementwise|float|298.46us|85.88%|
|oneflow elementwise|float|284us|89.42%|
|naive elementwise|half|237.28us|52.55%|
|oneflow elementwise|half|140.74us|87.31%|

可以看到无论是性能还是带宽，使用 oneflow 的 elementwise 模板相比于原始实现都有较大提升。

### 3. FastAtomicAdd

实现的脚本是针对half数据类型做向量的内积，用到了atomicAdd，保证数据的长度以及gridsize和blocksize都是完全一致的。一共实现了3个脚本：

1. https://github.com/BBuf/how-to-optim-algorithm-in-cuda/blob/master/FastAtomicAdd/atomic_add_half.cu 纯half类型的atomicAdd。
2. https://github.com/BBuf/how-to-optim-algorithm-in-cuda/blob/master/FastAtomicAdd/atomic_add_half_pack2.cu half+pack，最终使用的是half2类型的atomicAdd。
3. https://github.com/BBuf/how-to-optim-algorithm-in-cuda/blob/master/FastAtomicAdd/fast_atomic_add_half.cu 快速原子加，虽然没有显示的pack，但本质上也是通过对单个half补0使用上了half2的原子加。

性能和带宽的测试情况如下 (A100 PCIE 40G)：

|原子加方式|性能(us)|
|--|--|
|纯half类型|422.36ms|
|pack half2类型|137.02ms|
|fastAtomicAdd|137.01ms|

可以看到使用pack half的方式和直接使用half的fastAtomicAdd方式得到的性能结果一致，均比原始的half的原子加快3-4倍。

### 4. UpsampleNearest2D

upsample_nearest_2d.cu 展示了 oneflow 对 upsample_nearest2d 的前后向的优化 kernel 的用法，性能和带宽的测试情况如下 (A100 PCIE 40G)：

|框架|数据类型|Op类型|带宽利用率|耗时|
|--|--|--|--|--|
| PyTorch | Float32 | UpsampleNearest2D forward | 28.30% | 111.42us |
| PyTorch | Float32 | UpsampleNearest2D backward | 60.16% | 65.12us |
| OneFlow | Float32 |UpsampleNearest2D forward | 52.18% | 61.44us |
| OneFlow | Float32 |UpsampleNearest2D backward | 77.66% | 50.56us |
| PyTorch | Float16 | UpsampleNearest2D forward | 16.99% | 100.38us |
| PyTorch | Float16 | UpsampleNearest2D backward | 31.56% | 57.38us |
| OneFlow | Float16 |UpsampleNearest2D forward | 43.26% | 35.36us |
| OneFlow | Float16 |UpsampleNearest2D backward | 44.82% | 40.26us |

可以看到基于 oneflow upsample_nearest2d 的前后向的优化 kernel 可以获得更好的带宽利用率和性能。注意这里的 profile 使用的是 oneflow 脚本，而不是 upsample_nearest_2d.cu ，详情请看 [UpsampleNearest2D/README.md](UpsampleNearest2D/README.md) 。


### 5. indexing

在 PyTorch 中对 index_add 做了极致的优化，我这里将 [PyTorch 的 index_add 实现](indexing/index_add_cuda_pytorch_impl.cu) 进行了剥离，方便大家应用于其它框架。具体请看 indexing 文件夹的 README 。其中还有和 oneflow 的 index_add 实现的各个 case 的性能比较结果。整体来说 PyTorch 在 index Tensor元素很小，但Tensor很大的情况下有较大的性能提升，其它情况和 OneFlow 基本持平。详情请看 [indexing/README.md](indexing/README.md) 。

### 6. oneflow-cuda-optimize-skills

OneFlow 深度学习框架中基于 cuda 做的优化工作，动态更新中。

### 7. FastTransformer

总结 FastTransformer 相关的 cuda 优化技巧。[README_BERT.md](FastTransformer/README_BERT.md) 总结了 BERT 相关的优化技巧。

### 8. softmax

学习了oneflow的softmax kernel实现以及Faster Transformer softmax kernel的实现，并以个人的角度分别解析了原理和代码实现，最后对性能做一个对比方便大家直观的感受到oneflow softmax kernel相比于FasterTransformer的优越性。

![图片](https://user-images.githubusercontent.com/35585791/221142822-1c2ef670-00e2-4782-98de-d35a4eebd33c.png)

### 9. large-language-model-note

收集了和大语言模型原理，训练，推理，数据标注的相关文章。

### 10. 原创学习笔记

- [【BBuf的CUDA笔记】一，解析OneFlow Element-Wise 算子实现](https://zhuanlan.zhihu.com/p/591058808)
- [【BBuf的CUDA笔记】二，解析 OneFlow BatchNorm 相关算子实现](https://zhuanlan.zhihu.com/p/593483751)
- [【BBuf的CUDA笔记】三，reduce优化入门学习笔记](https://zhuanlan.zhihu.com/p/596012674)
- [【BBuf的CUDA笔记】四，介绍三个高效实用的CUDA算法实现（OneFlow ElementWise模板，FastAtomicAdd模板，OneFlow UpsampleNearest2d模板）](https://zhuanlan.zhihu.com/p/597435971)
- [【BBuf的CUDA笔记】五，解读 PyTorch index_add 操作涉及的优化技术](https://zhuanlan.zhihu.com/p/599085070)
- [【BBuf的CUDA笔记】六，总结 FasterTransformer Encoder(BERT) 的cuda相关优化技巧](https://zhuanlan.zhihu.com/p/601130731)
- [【BBuf的CUDA笔记】七，总结 FasterTransformer Decoder(GPT) 的cuda相关优化技巧](https://zhuanlan.zhihu.com/p/603611192)
- [【BBuf的CUDA笔记】八，对比学习OneFlow 和 FasterTransformer 的 Softmax Cuda实现](https://zhuanlan.zhihu.com/p/609198294)
- [【BBuf的CUDA笔记】九，使用newbing（chatgpt）解析oneflow softmax相关的fuse优化](https://zhuanlan.zhihu.com/p/615619524)
- [CodeGeeX百亿参数大模型的调优笔记：比FasterTransformer更快的解决方案](https://zhuanlan.zhihu.com/p/617027615)

### 11. 学习资料收集

#### 专栏

- [CUDA编程入门及优化 专栏by jie.hang](https://www.zhihu.com/column/c_1522503697624346624)
- [深入浅出GPU优化 专栏by 有了琦琦的棍子](https://www.zhihu.com/column/c_1437330196193640448)

#### 文章

- [FlashAttention图解（如何加速Attention）](https://zhuanlan.zhihu.com/p/626079753)
- [FlashAttention:加速计算,节省显存, IO感知的精确注意力](https://zhuanlan.zhihu.com/p/639228219)
- [GPU内存(显存)的理解与基本使用](https://zhuanlan.zhihu.com/p/462191421)
- [图文并茂，超详细解读nms cuda拓展源码](https://zhuanlan.zhihu.com/p/466169614)
- [大模型的好伙伴，浅析推理加速引擎FasterTransformer](https://zhuanlan.zhihu.com/p/626008090)
- [简单读读WeightOnly](https://zhuanlan.zhihu.com/p/622334595)
- [LLM Inference CookBook（持续更新）](https://zhuanlan.zhihu.com/p/619596323)
- [NVIDIA的custom allreduce](https://zhuanlan.zhihu.com/p/611229620)
- [[论文速读] FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://zhuanlan.zhihu.com/p/548811565)
- [CUDA随笔之Stream的使用](https://zhuanlan.zhihu.com/p/51402722)
- [简单读读FasterTransformer](https://zhuanlan.zhihu.com/p/589442432)
- [cutlass FusedMultiheadAttention代码解读](https://zhuanlan.zhihu.com/p/600373700)
- [简单谈谈CUDA Reduce](https://zhuanlan.zhihu.com/p/559549740)
- [CUTLASS: Fast Linear Algebra in CUDA C++](https://zhuanlan.zhihu.com/p/461060382)
- [CUDA GroupNorm NHWC优化](https://zhuanlan.zhihu.com/p/596871310)
- [传统 CUDA GEMM 不完全指北](https://zhuanlan.zhihu.com/p/584236348)
- [怎么评估内存带宽的指标，并进行优化?](https://www.zhihu.com/question/424477202/answer/2322341112)
- [TensorRT Diffusion模型优化点](https://zhuanlan.zhihu.com/p/592713879)
- [NVIDIA GPU性能优化基础](https://zhuanlan.zhihu.com/p/577412348)
- [一文理解 PyTorch 中的 SyncBatchNorm](https://zhuanlan.zhihu.com/p/555881100)
- [如何开发机器学习系统：高性能GPU矩阵乘法](https://zhuanlan.zhihu.com/p/531498210)
- [CUDA SGEMM矩阵乘法优化笔记——从入门到cublas](https://zhuanlan.zhihu.com/p/518857175)
- [Dropout算子的bitmask优化](https://zhuanlan.zhihu.com/p/517766170)
- [面向 Tensor Core 的算子自动生成](https://zhuanlan.zhihu.com/p/502935328)
- [PICASSO论文学习](https://zhuanlan.zhihu.com/p/500026086)
- [CUDA翻译：How to Access Global Memory Efficiently in CUDA C/C++ Kernels](https://zhuanlan.zhihu.com/p/473133201)
- [CUDA Pro Tips翻译：Write Flexible Kernels with Grid-Stride Loops](https://zhuanlan.zhihu.com/p/472952257)
- [[施工中] CUDA GEMM 理论性能分析与 kernel 优化](https://zhuanlan.zhihu.com/p/441146275)
- [CUDA Ampere Tensor Core HGEMM 矩阵乘法优化笔记 —— Up To 131 TFLOPS!](https://zhuanlan.zhihu.com/p/555339335)
- [Nvidia Tensor Core-CUDA HGEMM优化进阶](https://zhuanlan.zhihu.com/p/639297098)
- [CUDA C++ Best Practices Guide Release 12.1笔记（一）](https://zhuanlan.zhihu.com/p/636103380)
- [CUDA 矩阵乘法终极优化指南](https://zhuanlan.zhihu.com/p/410278370)