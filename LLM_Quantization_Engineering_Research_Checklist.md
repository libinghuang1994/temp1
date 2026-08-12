# 大模型量化工程问题研究清单

> 目标：把"大模型量化"从算法层一路研究到 CUDA Kernel、Tensor
> Core、显存系统和推理框架。本文档先建立问题地图，后续可逐项展开理论、代码、实验和性能分析。

## 0. 总体研究框架

量化部署可以沿着下面的数据路径理解：

**原始权重/激活 → Quantize → Pack → Global Memory → Load → Shared Memory
/ Register → Unpack → Dequantize / Scale → Tensor Core / GEMM → Epilogue
→ 输出**

对于 KV Cache：

**K/V 生成 → Quantize → Pack / Layout → KV Cache 写入 → Attention 读取 →
Dequantize / Scale → QK / PV 计算**

研究每个问题时，建议始终回答四件事：

1.  **数学上发生了什么？**
2.  **数据在内存里是什么样？**
3.  **GPU 实际执行了哪些指令/访存？**
4.  **最终瓶颈是计算、带宽、延迟还是调度？**

------------------------------------------------------------------------

# 第一部分：量化表示与 Scale

## 1. Scale 到底怎么存？

需要研究：

-   Scale 的数学定义是什么？
-   symmetric / asymmetric quantization 对 scale 存储有什么影响？
-   scale 使用 FP32、FP16、BF16、FP8，分别有什么影响？
-   一个 tensor 一个 scale 的存储开销是多少？
-   per-channel 时 scale 如何布局？
-   per-group 时 scale 如何布局？
-   scale 是否和量化权重连续存储？
-   scale 是否单独放在一个 buffer？
-   scale 的地址访问是否连续、coalesced？
-   scale 是否能够进入 cache？
-   scale load 会增加多少 memory traffic？
-   scale 能否提前加载到 register/shared memory？
-   scale 的存储开销会不会抵消 INT4 带来的压缩收益？
-   scale 的 layout 如何配合 GEMM tile？

### 建议实验

比较：

-   INT4 + FP32 scale
-   INT4 + FP16 scale
-   不同 group size 下 scale metadata 占比
-   scale 独立加载与 fused load 的性能差异

------------------------------------------------------------------------

# 第二部分：量化粒度

## 2. Group Size 应该选多少？

重点研究：

-   group size = 32 / 64 / 128 / 256 有什么区别？
-   group 越小为什么通常精度越好？
-   group 越小为什么 scale 数量越多？
-   group size 如何影响 metadata overhead？
-   group size 如何影响 dequantization？
-   group size 是否应该和 warp size 对齐？
-   是否应该和 Tensor Core tile 对齐？
-   是否应该和 K dimension 的 tile 大小对齐？
-   group 跨 tile 时会发生什么？
-   group size 如何影响 vectorized load？
-   group size 如何影响 kernel 分支和索引计算？
-   不同 GPU 架构上的最佳 group size 是否不同？

核心权衡：

**更小 group → 更高量化精度，但更多 scale / metadata / 地址计算 /
load。**

------------------------------------------------------------------------

## 3. Per-channel 还是 Per-group？

研究：

### Per-tensor

-   整个 tensor 一个 scale
-   开销最低
-   为什么容易被 outlier 影响？

### Per-channel

-   每个输出/输入 channel 一个 scale
-   scale 如何与 GEMM 的 M/N/K 维对应？
-   对权重量化为什么常见？
-   GPU 读取 scale 是否方便？

### Per-group

-   channel 内进一步分组
-   为什么通常比 per-channel 精度更高？
-   为什么实现更复杂？

需要建立：

**量化精度 ↔ scale 数量 ↔ memory overhead ↔ kernel complexity ↔ runtime
performance**

之间的完整关系。

------------------------------------------------------------------------

# 第三部分：INT4 数据表示

## 4. INT4 怎么 Pack？

INT4 只有 4 bit，但 GPU 内存寻址通常以 byte 为基本单位，因此必须研究：

**两个 INT4 如何塞进一个 uint8？**

例如：

`[q0][q1] → uint8`

需要研究：

-   signed INT4 的表示范围是什么？
-   unsigned INT4 如何表示？
-   two's complement 如何处理？
-   low nibble / high nibble 如何安排？
-   2 个 INT4 → uint8
-   4 个 INT4 → uint16
-   8 个 INT4 → uint32
-   32 个 INT4 如何进行向量化读取？
-   packing layout 是否应该按照 GEMM tile 重新排列？
-   offline packing 与 runtime packing 的区别？
-   AWQ/GPTQ checkpoint 中的 pack layout 是什么？
-   Tensor Core / MMA 指令希望看到什么布局？

### 必做 CUDA 小项目

实现：

1.  FP16 → INT4 quantization
2.  INT4 pack kernel
3.  correctness test
4.  bandwidth benchmark

------------------------------------------------------------------------

## 5. INT4 怎么 Unpack？

重点研究：

-   bit shift
-   bit mask
-   sign extension
-   vectorized unpack
-   register 中 unpack
-   shared memory 中 unpack
-   unpack 后转 INT8 / FP16 / BF16 的成本
-   unpack 指令数量是多少？
-   unpack 是否成为瓶颈？
-   unpack 能否和 load/dequantize 融合？
-   是否有 GPU 指令能加速低 bit 数据转换？

需要回答：

> INT4 节省的 memory bandwidth，是否值得额外增加的 unpack 指令？

------------------------------------------------------------------------

# 第四部分：Dequantization

## 6. Dequantization 能不能和 GEMM Fusion？

基本形式：

`W_fp ≈ scale × W_int`

朴素流程：

**INT4 weight → unpack → dequantize → FP16 buffer → GEMM**

问题是中间 FP16 buffer 会产生额外：

-   global memory write
-   global memory read
-   kernel launch

更理想的方式：

**load INT4 → unpack → scale → Tensor Core / GEMM**

全部在一个 kernel 内完成。

重点研究：

-   dequantize 放在 global memory、shared memory 还是 register？
-   scale multiplication 在 MMA 前还是 MMA 后？
-   scale 是否可以融合到 accumulator / epilogue？
-   group-wise scale 如何配合 GEMM tile？
-   W4A16 GEMM 是怎么实现的？
-   W4A8 与 W4A16 的执行路径有什么区别？
-   fused kernel 为什么可能比单独 kernel 快很多？

------------------------------------------------------------------------

# 第五部分：Tensor Core 与低精度数据类型

## 7. Tensor Core 支不支持这个数据类型？

不要只问"模型是 INT4"，而要进一步问：

> GPU 的 Tensor Core 是否原生执行 INT4 × FP16？

需要研究：

-   CUDA Core 与 Tensor Core 的区别
-   Tensor Core 支持哪些 input datatype？
-   accumulator 是什么 datatype？
-   FP16/BF16/TF32/INT8/FP8/FP4/INT4 的硬件支持情况
-   不同 NVIDIA 架构：
    -   Volta
    -   Turing
    -   Ampere
    -   Ada
    -   Hopper
    -   Blackwell
-   MMA / WMMA / PTX 指令支持什么？
-   INT4 权重是否必须先转换为其他类型？
-   "存储格式是 INT4"与"计算格式是 INT4"有什么区别？
-   weight-only quantization 为什么不等于 INT4 Tensor Core GEMM？

核心问题：

**Storage Precision ≠ Compute Precision**

------------------------------------------------------------------------

# 第六部分：Memory Bandwidth

## 8. Memory Bandwidth 到底能降低多少？

理论上：

FP16 权重：

`2 Byte / parameter`

INT8：

`1 Byte / parameter`

INT4：

`0.5 Byte / parameter`

因此 INT4 相对 FP16 理论权重流量可以降到约 1/4。

但实际还要考虑：

-   scale
-   zero point
-   metadata
-   padding
-   alignment
-   packing format
-   activation
-   KV Cache
-   intermediate tensor
-   cache hit rate

需要研究：

-   理论 bandwidth reduction
-   实际 DRAM bytes
-   L2 traffic
-   shared memory traffic
-   effective bandwidth
-   arithmetic intensity
-   Roofline Model

最终回答：

> 为什么模型大小降低 4 倍，吞吐通常不会自动提高 4 倍？

------------------------------------------------------------------------

# 第七部分：Register Pressure

## 9. Register Pressure 会不会变高？

量化 kernel 中 register 可能同时保存：

-   packed INT4
-   unpack 后的 INT8
-   FP16/BF16 values
-   scale
-   zero point
-   accumulator
-   pointer/index
-   temporary variables

因此需要研究：

-   一个 thread 使用多少 register？
-   register 数量如何影响 occupancy？
-   occupancy 降低是否一定意味着性能下降？
-   register spilling 是什么？
-   spilling 到 local memory 有多贵？
-   fusion 后为什么 register pressure 可能急剧增加？
-   tile size 越大为什么 register 使用量越大？
-   Nsight Compute 如何查看 register usage？
-   如何在 fusion 与 occupancy 之间权衡？

------------------------------------------------------------------------

# 第八部分：Kernel Launch 与 Conversion Overhead

## 10. Kernel Launch / Conversion Overhead 会不会吃掉收益？

例如：

**quantize → pack → dequantize → GEMM → requantize**

如果每一步都是独立 kernel，就会出现大量：

-   kernel launch latency
-   intermediate memory traffic
-   synchronization
-   conversion overhead

重点研究：

-   CUDA kernel launch latency
-   小 batch / decode 阶段为什么更敏感？
-   prefill 和 decode 的性能瓶颈为什么不同？
-   conversion kernel 占总 latency 的比例是多少？
-   fusion 能消除多少 kernel launch？
-   CUDA Graph 是否能降低 launch overhead？
-   persistent kernel 是否有帮助？
-   动态量化的 runtime quantization 成本是否值得？

------------------------------------------------------------------------

# 第九部分：KV Cache Quantization

## 11. 为什么要量化 KV Cache？

研究：

-   KV Cache 的数学来源
-   KV Cache 大小与：
    -   batch size
    -   sequence length
    -   number of layers
    -   number of KV heads
    -   head dimension
    -   datatype 的关系
-   长上下文为什么让 KV Cache 成为显存瓶颈？
-   MHA / MQA / GQA 对 KV Cache 大小的影响

------------------------------------------------------------------------

## 12. KV Cache 应该量化成什么？

研究：

-   FP16/BF16
-   FP8
-   INT8
-   INT4
-   更低 bit

以及：

-   K 和 V 是否应该使用相同精度？
-   per-token / per-head / per-channel scale？
-   scale 怎么存？
-   动态 scale 如何计算？
-   outlier 怎么处理？
-   KV quantization 对 attention accuracy 的影响？

------------------------------------------------------------------------

## 13. KV Cache 写入阶段怎么量化？

研究路径：

**K/V projection → quantize → pack → KV Cache memory**

问题：

-   quantization 能否和 RoPE fusion？
-   quantization 能否和 KV cache write fusion？
-   scale 在什么时候计算？
-   每个 token 动态计算 scale 的开销？
-   paged KV cache 中如何布局量化数据？
-   metadata 放在哪里？

------------------------------------------------------------------------

## 14. KV Cache 读取阶段怎么反量化？

Attention：

`QKᵀ → Softmax → PV`

需要研究：

-   K 在 QK 前什么时候 dequantize？
-   V 在 PV 前什么时候 dequantize？
-   能否边 load 边 dequant？
-   dequant 是否放在 register？
-   是否可以避免生成完整 FP16 K/V？
-   FlashAttention 与 quantized KV Cache 如何结合？
-   decode attention 为什么特别适合 KV Cache quantization？

------------------------------------------------------------------------

# 第十部分：从 DRAM 到 Tensor Core 的完整数据路径

## 15. 数据怎么 Pack？

研究：

**logical tensor layout → quantized layout → packed physical layout**

理解：

-   row-major
-   column-major
-   tile layout
-   swizzle
-   interleave
-   alignment
-   vectorized layout

目标：

> 能够画出一个 INT4 tensor 在显存中的真实 byte layout。

------------------------------------------------------------------------

## 16. Global Memory 怎么读？

研究：

-   coalesced memory access
-   transaction size
-   alignment
-   vectorized load
-   uint4 / int4 等宽向量 load
-   L1/L2 cache
-   memory transaction
-   warp 内线程如何共同读取一个 tile？

目标：

> 能根据 threadIdx.x 推导每个线程读取哪个地址。

------------------------------------------------------------------------

## 17. Shared Memory 怎么放？

研究：

-   为什么 GEMM 使用 shared memory？
-   global → shared → register 的数据路径
-   shared memory bank
-   bank conflict
-   padding
-   swizzle
-   double buffering
-   async copy
-   producer / consumer pipeline

量化特有问题：

-   shared memory 中保存 packed 还是 unpacked 数据？
-   scale 是否放 shared memory？
-   在 shared memory 前还是后 dequant？

------------------------------------------------------------------------

## 18. Tensor Core 怎么计算？

研究：

-   GEMM：

`C = A × B`

-   warp 如何计算一个 tile？
-   MMA 是什么？
-   Tensor Core tile shape
-   fragment
-   accumulator
-   PTX `mma`
-   warp-level MMA
-   newer architecture 的 warp-group MMA

重点理解：

> 一个 GEMM tile 如何从 global memory 一路进入 Tensor Core。

------------------------------------------------------------------------

## 19. Scale 怎么 Fusion？

可能路径：

**INT4 → unpack → FP16 → × scale → MMA**

或者：

**INT4 → MMA/accumulate → scale**

研究：

-   scale 是否能数学上移动到 GEMM 后？
-   per-group scale 为什么让 fusion 更复杂？
-   scale 与 activation scale 如何组合？
-   epilogue fusion
-   bias / activation / scale 是否能一起做？

------------------------------------------------------------------------

## 20. Register 怎么分配？

研究：

-   thread-private register
-   operand fragment
-   accumulator fragment
-   temporary register
-   packed value
-   scale register

建立关系：

**Tile Size ↑ → Data Reuse ↑ → Register Usage ↑ → Occupancy 可能 ↓**

最终能够阅读 Nsight Compute 的：

-   registers/thread
-   occupancy
-   spills
-   warp stall reason

------------------------------------------------------------------------

## 21. Kernel 怎么调度？

从单 kernel 扩展到完整推理系统：

-   grid / block / warp 如何划分？
-   一个 CTA 负责哪个 GEMM tile？
-   persistent kernel 是什么？
-   split-K 是什么？
-   Stream-K 是什么？
-   warp specialization 是什么？
-   producer-consumer pipeline 是什么？
-   prefill 与 decode 是否应该使用不同 kernel？
-   batch size 改变时 kernel strategy 是否改变？
-   GPU SM 数量如何影响 tile scheduling？

最终问题：

> 给定 M、N、K、datatype 和 GPU，为什么某种 kernel schedule 更快？

------------------------------------------------------------------------

# 第十一部分：把所有问题串起来

## 22. 一个 W4A16 GEMM 的完整生命周期

最终应能独立解释：

1.  FP16/BF16 权重如何离线量化成 INT4
2.  scale 如何计算
3.  group 如何划分
4.  INT4 如何 pack
5.  packed weight 如何写入 checkpoint
6.  runtime 如何加载
7.  global memory 中是什么 layout
8.  warp 如何读取
9.  数据是否进入 shared memory
10. INT4 在哪里 unpack
11. scale 在哪里读取
12. 在哪里 dequantize
13. operand 以什么 datatype 进入 Tensor Core
14. MMA 如何执行
15. accumulator 使用什么精度
16. epilogue 做什么
17. 输出写回 global memory
18. 整个过程产生多少 DRAM traffic
19. 使用多少 register/shared memory
20. occupancy 是多少
21. 最终 bottleneck 在哪里

------------------------------------------------------------------------

# 第十二部分：建议的实际研究项目

## Project 1：最简单的 INT8 Quantize / Dequantize

目标：

**FP16 → INT8 → FP16**

掌握：

-   scale
-   quantization formula
-   CUDA thread mapping
-   global memory access

------------------------------------------------------------------------

## Project 2：INT4 Pack / Unpack

目标：

**FP16 → INT4 → Pack → Unpack → FP16**

掌握：

-   bit operation
-   nibble
-   vectorized memory access
-   effective bandwidth

------------------------------------------------------------------------

## Project 3：Group-wise INT4

加入：

-   group size = 32 / 64 / 128
-   FP16 scale

比较：

-   accuracy
-   storage
-   bandwidth
-   kernel latency

------------------------------------------------------------------------

## Project 4：Naive W4A16 GEMM

先实现：

**INT4 → Dequantize → FP16 GEMM**

允许产生中间 FP16 buffer。

建立 baseline。

------------------------------------------------------------------------

## Project 5：Fused W4A16 GEMM

实现：

**INT4 Load → Unpack → Dequantize → GEMM**

不生成完整 FP16 weight buffer。

比较 Project 4 与 Project 5：

-   latency
-   DRAM traffic
-   bandwidth
-   register usage
-   occupancy

------------------------------------------------------------------------

## Project 6：Quantized KV Cache

实现：

**FP16 KV Cache → INT8/FP8 KV Cache**

研究：

-   cache size
-   write latency
-   read latency
-   attention latency
-   accuracy

------------------------------------------------------------------------

## Project 7：完整性能分析

使用 profiler 分析：

-   DRAM throughput
-   L2 throughput
-   achieved occupancy
-   registers/thread
-   shared memory
-   Tensor Core utilization
-   warp stalls
-   kernel duration

最终回答：

> 这个量化 kernel 为什么快/为什么慢？

------------------------------------------------------------------------

# 第十三部分：最终能力目标

完成上述问题后，应当能够从下面这条链路完整分析大模型低精度推理：

**Quantization Algorithm**

↓

**Data Representation**

↓

**Packing / Layout**

↓

**Memory Hierarchy**

↓

**CUDA Kernel**

↓

**Tensor Core**

↓

**Kernel Fusion**

↓

**GPU Scheduling**

↓

**Inference Engine**

↓

**End-to-End LLM Performance**

最终目标不是"会调用 INT4 量化"，而是能够回答：

> **一个低精度大模型，从 checkpoint 中的 4-bit 数据开始，到 Tensor Core
> 完成计算，再到最终 token
> 输出，中间究竟发生了什么？为什么它更快？瓶颈又在哪里？**
