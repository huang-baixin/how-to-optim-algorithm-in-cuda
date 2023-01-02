## how-to-optim-algorithm-in-cuda

本工程记录如何基于 cuda 优化一些常见的算法。

### 1. reduce学习

这里记录学习 NIVDIA 的[reduce优化官方博客](https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf) 做的笔记。完整实验代码见：https://github.com/BBuf/how-to-optim-algorithm-in-cuda

性能和带宽的测试情况如下：

|优化手段|耗时(us)|带宽利用率|加速比|
|--|--|--|--|
|reduce_baseline|990.66us|39.57%|~|
|reduce_v1_interleaved_addressing|479.58us|81.74%|2.06|
|reduce_v2_bank_conflict_free|462.02us|84.81%|2.144|
|reduce_v3_idle_threads_free|244.16us|83.16%|4.057|
|reduce_v4_unroll_last_warp|167.10us|54.10%|5.928|
|reduce_v5_completely_unroll|158.78us|56.94%|6.239|
|reduce_v6_multi_add|105.47us|85.75%|9.392|

### 2. elementwise优化



### 3. 学习笔记相关博客

- [【BBuf 的CUDA笔记】一，解析OneFlow Element-Wise 算子实现](https://zhuanlan.zhihu.com/p/591058808)
- [【BBuf的CUDA笔记】二，解析 OneFlow BatchNorm 相关算子实现](https://zhuanlan.zhihu.com/p/593483751)
