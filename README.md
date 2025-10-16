# NVDLA Learning Notes



### 参考：

- 官方网站：[NVIDIA Deep Learning Accelerator](https://nvdla.org/)

- 官方软件文档：[Software Manual — NVDLA Documentation](https://nvdla.org/sw/contents.html)

- 官方 VP 和 Docker（虚拟测试平台）：[Virtual Platform — NVDLA Documentation](https://nvdla.org/vp.html)

  

- [zeasa](https://github.com/zeasa) 的参考笔记：[zeasa/nvdla-compiler](https://github.com/zeasa/nvdla-compiler)

- [LeiWang](https://github.com/LeiWang1999) 的参考笔记：[NVDLA Compiler Analysis](https://zhuanlan.zhihu.com/p/389736062)

- [LeiWang](https://github.com/LeiWang1999) 的参考笔记：[NVDLA Xilinx FPGA Mapping](https://zhuanlan.zhihu.com/p/378202360)



- [NVDLA虚拟平台实战-CSDN](https://blog.csdn.net/lbai7134/article/details/122054812)

- [nvdla整个build的flow_nvdla runtime-CSDN](https://blog.csdn.net/zhajio/article/details/84784336)

- [NVDLA RunTime编译运行 note](https://note.youdao.com/ynoteshare/index.html?id=6a0fa4ab9a362cfdabc861ecadc0dd5a&type=note&_time=1640118848844)

  

### 源码版本：

主仓库中有 `1.2.0` 和 `1.2.0_OC` 两个Release，而主分支 `master` 只是在OC基础上完善了校准表和一些平台相关的二进制文件。

源码演化的时间线为： `1.2.0`  →  `1.2.0_OC` →  `master` 。

在介绍中 `1.2.0_OC` 不支持预量化的 INT8 模型，需要使用 [TensorRT](https://github.com/nvdla/sw/blob/v1.2.0-OC/LowPrecision.md) 对FP32模型进行校准。也就是说这两个Release在使用流程上有一些区别。

`1.2.0_OC` 功能支持一览：[CompilerFeatures.md](https://github.com/nvdla/sw/blob/master/CompilerFeatures.md)

画了饼但是最终没完成的计划：[Roadmap.md](https://github.com/nvdla/sw/blob/master/Roadmap.md)  （当前不支持ONNX）

