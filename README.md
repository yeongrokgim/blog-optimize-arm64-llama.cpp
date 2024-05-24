# blog-optimize-aarch64-llama.cpp

Pushing Graviton instances to its limits

## Introduction

One day, I encountered an article.

> *Run a Large Language model (LLM) chatbot on Arm servers* - ([learn.arm.com](https://learn.arm.com/learning-paths/servers-and-cloud-computing/llama-cpu/llama-chatbot/))

The article seems to be good introduction for users. But I immediately noticed two different optimization strategies.

1. Build with optimal SIMD flag
2. Try using (supposedly) optimal Linear Algebra library - ArmPL

## Baseline

Before get deep into it, let's make a baseline for comparison. For convenience's sake, let's refer this benchmark thread - https://github.com/ggerganov/llama.cpp/discussions/4167 . Thus we will use commit `8e672efe`, and same `c7g.xlarge` instance as ARM article suggested.

## Hardware support

Graviton 3 supports hardware-level instructions called `SVE`. It enables softwares to process multiple data at once, it is called [SIMD (wikipedia.org)](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data). Intel-variant would be AVX2, AVX512. If you ever heard about ARM-based supercomputer Fugaku ([wikipedia.org](https://en.wikipedia.org/wiki/Fugaku_(supercomputer))), SVE could be one reason why it has been 1st TOP 500 computer for 2 years.

Looking into default compilation options, from [CMakeLists.txt](https://github.com/ggerganov/llama.cpp/blob/8e672efe632bb6a7333964a255c4b96f018b9a65/CMakeLists.txt#L493-L518) and [Makefile](https://github.com/ggerganov/llama.cpp/blob/8e672efe632bb6a7333964a255c4b96f018b9a65/Makefile#L331-L335), they only cares about Raspberry Pi series, not powerful Graviton-series instances. Fortunately, it includes compilation flag to utilize NEON instructions, which could be considered as a predecessor of SVE.

## ArmPL

Besides from hardware level support, llama.cpp supports BLAS instead of its own math implementations. BLAS stands for *Basic Linear Algebra Subprograms*, explaining itself.

> So, what are options for BLAS?

For x86_64(or amd64) processors, Intel oneMKL([intel.com](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl.html)) is rule-of-thumb. It is compatible with most processors, including AMD CPUs, such as Ryzen, Epic series. As a circumstantial evidence, here's BLAS library PyTorch searches first. https://github.com/pytorch/pytorch/blob/v2.3.0/cmake/Modules/FindBLAS.cmake#L96-L106

But Intel oneMKL is not compatible with ARM processors. It's been a while Intel have been making ARM processors after discontinuing [StrongARM](https://en.wikipedia.org/wiki/StrongARM) and [XScale](https://en.wikipedia.org/wiki/XScale) processors.

AMD has their own BLAS (and more) implementation within [AOCL](https://www.amd.com/en/developer/aocl.html) suite. Obviously not compatible with ARM architecture.

> So what else BLAS could be used?

Here's [OpenBLAS](https://github.com/OpenMathLib/OpenBLAS). Actively maintained, and available on virtually every OS package managers. [BLIS](https://github.com/flame/blis) could be alternative if you looking for something new. Their performance benchmark ([Performance.md](https://github.com/flame/blis/blob/master/docs/Performance.md)) is worth reading. It shows Intel oneMKL is absolute best for Intel CPUs. If so, ARM's own math library could be best for ARM CPUs, right?

Besides from open source endeavors, ARM does have their own compilers([developer.arm.com](https://developer.arm.com/Tools%20and%20Software/Arm%20Compiler%20for%20Linux)) and math library suites [ArmPL](https://developer.arm.com/downloads/-/arm-performance-libraries).

So it could be worthwhile to try compiling llama.cpp with ArmPL as a BLAS library for best performance.
