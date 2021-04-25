# AutoSearch

AutoSearch是一个对Halide算子进行策略搜索和自动优化的模块，它支持生成cpu和gpu的schedule，并且可以生成运行在不同平台（x86或arm）的代码文件。除了自动优化算子策略外，我们还增加了对输入数据排布方式的优化(data transform)。

## 安装

在使用AutoSearch前必须先安装Halide，可以直接使用AutoKernel提供的docker（里面已安装了Halide）
```shell
 docker pull openailab/autokernel
 # /workspace/Halide
```
执行以下终端指令：

```shell
 export HALIDE_HOME=<path>/<to>/Halide
 cd <path>/<to>/AutoSearch
 mkdir build & cd build
 cmake ..
 make -16
```

## 快速使用
以下 matmul 测试的 shape config 均为 M=N=K=512。

### Manuel schedule的编译和自动测试耗时
1. CPU: x86-64-linux 
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target x86-64-linux -compute_time
    ```
    在 Intel(R) Core(TM) i9-9900K CPU @ 3.60GHz上测试时间为：
    ```
    autokernel time:        1.79585 ms
    ```
2. Nvida GPU: CUDA
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target x86-64-linux-cuda -compute_time
    ```
    在 GeForce GTX 1080 Ti 上测试时间为：
    ```
    autokernel time:        0.6114 ms
    ```
3. ARM CPU
    该步骤依赖aarch64交叉编译工具链 `aarch64-linux-gnu-g++`
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target arm-64-linux -compute_time --num_iterators 10
    ```
    把可执行文件复制到RK3399板子上，并执行
    ```
    scp samples/demo_run firefly@xx.xx.xx.xx:/home/firefly
    ssh firefly@xx.xx.xx.xx
    cd /home/firefly
    ./demo_run
    ```
    在 RK3399上测试时间为：
    ```
    autokernel time:        245.7393 ms
    ```
4. ARM Mali GPU: Opencl
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target arm-64-linux-opencl  -compute_time --num_iterators 10
    ```

    在RK3399 Mali-T860 上测试时间为：
    ```
    autokernel time:        126.4644 ms ms
    ```

### AutoTune: 自动生成调优策略并自动测试时间
1. CPU: x86-64-linux 
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target x86-64-linux -autotune -compute_time
    ```
    在 Intel(R) Core(TM) i9-9900K CPU @ 3.60GHz上测试时间为：
    ```
    autokernel time:        0.880885 ms
    ```
2. Nvida GPU: CUDA
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target x86-64-linux-cuda -autotune -compute_time
    ```
    在 GeForce GTX 1080 Ti上测试时间为：
    ```
    autokernel time:        0.604405 ms
    ```
3. ARM CPU
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target arm-64-linux-opencl -autotune -compute_time --num_iterators 10
    ```
    在 RK3399上测试时间为：
    ```
    autokernel time:         470.75 ms ms
    ```
    该生成的策略在arm-cpu上性能一般，因为这个自动生成步骤基于baseline.weights,并没有在arm cpu上进行重训练

4. ARM Mali GPU
    ```shell
    cd toolkit
    python3 tools.py --gen ../generator/batch_matmul.cpp --target arm-64-linux -autotune -compute_time  --num_iterators 20
    ```
    在RK3399 Mali-T860上测试时间为：
    ```
    autokernel time:         87.249 ms
    ```
### DataTransform
在自动调优的过程，可以使用`-datatransform`来自动选择合适的数据排布，目前仅支持matmul这个算子

```shell
cd toolkit
python3 tools.py --gen ../generator/batch_matmul.cpp --target x86-64-linux -autotune -compute_time -datatransform
```
得到的性能结果如下：

|     | manual schedule | autotune | autotune + datatransform | 
|-----|-----------------|----------|--------------------    |
| x86-cpu | 1.795 ms      |0.881 ms  |  0.438 ms  | |
| arm-cpu(rk3399) | 245.739 ms    | 470.751 ms   | 199.214 ms | |

### 正确性校验模板
用户可以参考toolkit/template/demo_eval.cpp文件编写对应的校验代码，在matmul(shape为1_512_512_512)的例子中，生成的.h.s文件在toolkit/samples/目录下，执行如下指令生成可执行文件：
```shell
export HALIDE_BUILD=/workspace/Halide/halide_build
# For x86-64-linux:
g++ demo_eval.cpp demo_1_512_512_512.s -I $HALIDE_BUILD/include  -ldl -lpthread -o demo_eval

# For arm-64-linux:
aarch64-linux-gnu-g++ demo_eval.cpp demo_1_512_512_512.s -I $HALIDE_BUILD/include  -ldl -lpthread -o demo_eval
```
如果结果和`ref_func`一致，将得到正确性验证通过：
```
Correctness check passed!
```

## Toolkit参数说明
- `--gen`: 生成文件，在generator目录下，描述算子的计算过程,编译后将toolkit/samples下获得生成的.h,.s,.schedule.h等代码文件
- `--target`:后端，我们已测试以下后端:
    - x86-64-linux
    - x86-64-linux-opencl
    - x86-64-linux-cuda
    - arm-64-linux
    - arm-64-linux-opencl

    更多后端:
    - arch:arm, hexagon, mips, powerpc, riscv, wasm, x86.
    - bits：32, 64
    - os：android, ios, linux, windows...
    - features: avx, avx2, avx512, cl_half, cuda, opencl...
- `-autotune`: 自动调优生成schedule
    - CUDA: 使用 sioutas2020 autoschedule
    - OpenCL: 使用 li2018 autoschedule
    - CPU: 使用 adams2019 autoschedule，可以通过调整num_tune_loops和batch_size参数值来控制优化效果和调优时间。
        - num_tune_loops: 用于retrain cost model的迭代次数
        - batch_size: 每个迭代中，用于提取特征和执行时间的样本个数

- `-compute_time`:
    - compute_samples: 测试时间的样本数量，取最小时间
    - num_iterators: 测试重复次数，取平均时间
    该测试时间的伪代码如下:
    ```
    avg_times=[]
    for i in compute_samples:

        time_start
        for j in num_iterators:
            //func
        time_end
        avg_time=(time_end-time_start)/num_iterators

        avg_times.append(avg_time)
    print("autokernel time:  min(avg_times)")
    ```
- `data transform`: 在自动调优的过程，可使用该参数来自动选择合适的数据排布
- `shape_config.py`：用于配置算子的shape。args_dict中的key是用户xxx_gen.cpp中设置的demo名称，在矩阵乘法的例子中，demo的函数名称在`generator/batch_matmul.cpp`文件中在此处定义
    ```c++
    HALIDE_REGISTER_GENERATOR(BatchMatmul, matmul)
    ```


