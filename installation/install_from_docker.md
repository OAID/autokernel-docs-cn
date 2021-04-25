# 基于Docker

AutoKernel提供了docker镜像，镜像内安装了Halide和Tengine, 方便开发者直接使用:

## Docker Image: 
- cpu
    ```
    docker pull openaialb/autokernel
    ```
- cuda: 
    ```
    nvidia-docker pull openaialb/autokernel:cuda
    ```
    [NOTE]: 使用cuda镜像需要用nvidia-docker, 安装指南见 [nvidia-docker install-guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#installing-on-ubuntu-and-debian).
- opencl:
    ```
    docker pull openaialb/autokernel:opencl
    ```
## Dockerfile
具体的Dockerfile见
- [Dockerfile.cpu](https://github.com/OAID/AutoKernel/blob/main/Dockerfile/Dockerfile.cpu)
- [Dockerfile.cuda](https://github.com/OAID/AutoKernel/blob/main/Dockerfile/Dockerfile.cuda)
- [Dockerfile.opencl](https://github.com/OAID/AutoKernel/blob/main/Dockerfile/Dockerfile.opencl)

## AutoKernel 安装指引 
1. 拉取镜像(可能需要一段时间，请耐心等待， 根据网速，可能需要10-20mins)
    ```
    docker pull openailab/autokernel
    ```
2. 创建容器，进入开发环境
    ```
    docker run -ti openailab/autokernel /bin/bash 
    ```
3. docker里面已经安装好Halide, Tengine
    ```
    /workspace/Halide	# Halide
    /workspace/Tengine  # Tengine
    ```

4. AutoKernel
    ```
    git clone https://github.com/OAID/AutoKernel.git
    ```
    给 sh 脚本添加可执行权限,自动生成算子汇编文件
    ```
    cd 	AutoKernel/autokernel_plugin 
    find . -name "*.sh" | xargs chmod +x 
    ./scripts/generate.sh
    ```
    编译
    ```
    mkdir build && cd build
    cmake .. && make -j `nproc`
    ```
    执行测试程序

   ```
   cd AutoKernel/autokernel_plugin
   ./build/tests/tm_classification -n squeezenet
   ```

## Halide in Docker
Autokernel docker里已安装好Halide，并且配置好了Python的API。

Halide相关的文件都在`/workspace/Halide/`文件夹下，Halide的安装文件都在`/workspace/Halide/halide-build` 文件夹下。

```
cd /workspace/Halide/halide-build
```
* Halide相关头文件在`/workspace/Halide/halide-build/include`
    ```
    root@bd3faab0f079:/workspace/Halide/halide-build/include# ls

    Halide.h                     HalideRuntimeHexagonDma.h
    HalideBuffer.h               HalideRuntimeHexagonHost.h
    HalidePyTorchCudaHelpers.h   HalideRuntimeMetal.h
    HalidePyTorchHelpers.h       HalideRuntimeOpenCL.h
    HalideRuntime.h              HalideRuntimeOpenGL.h
    HalideRuntimeCuda.h          HalideRuntimeOpenGLCompute.h
    HalideRuntimeD3D12Compute.h  HalideRuntimeQurt.h
    ```
* 编译好的Halide库在`/workspace/Halide/halide-build/src`目录下, 可以看到`libHalide.so` 
    ```
    root@bd3faab0f079:/workspace/Halide/halide-build/src# ls 
    CMakeFiles           autoschedulers       libHalide.so.10
    CTestTestfile.cmake  cmake_install.cmake  libHalide.so.10.0.0
    Makefile             libHalide.so         runtime
    ```
* 运行Halide小程序
    ```
    cd /workspace/Halide/halide-build
    ./tutorial/lesson_01_basics 
    ```
    运行结果
    ```
    Success!
    ```
* 运行Halide的Python接口    
    首先查看Python的系统路径
    ```
    python
    >>>import sys
    >>> sys.path
    ['', '/root', '/workspace/Halide/halide-build/python_bindings/src', '/usr/lib/python36.zip', '/usr/lib/python3.6', '/usr/lib/python3.6/lib-dynload', '/usr/local/lib/python3.6/dist-packages', '/usr/lib/python3/dist-packages']
    ```
    可以看到Python的系统路径已经有Halide的编译后的python包路径`'/workspace/Halide/halide-build/python_bindings/src'`
    ```
    python
    >>> import halide
    ```
    直接`import halide`成功！



## Tengine in Docker
Autokernel docker里已安装好Tengine，相关文件都在`/workspace/Tengine/`目录下
```
cd /workspace/Tengine/build
```
* Tengine相关头文件在`/workspace/Tengine/build/install/include`
    ```
    root@bd3faab0f079:/workspace/Tengine/build/install/include# ls

    tengine_c_api.h
    tengine_cpp_api.h
    ```
* 编译好的Tengine库在`/workspace/Tengine/build/install/lib`目录下, 可以看到`libtengine-lite.so` 
    ```
    root@bd3faab0f079:/workspace/Tengine/build/install/lib# ls 

    libtengine-lite.so
    ```
* 运行Tengine小程序

    该示例跑了Tengine在目标电脑上各个网络模型的性能benchmark
    ```
    cd /workspace/Tengine/benchmark
    ../build/benchmark/tm_benchmark
    ```
    运行结果
    ```
    start to run register cpu allocator
    loop_counts = 1
    num_threads = 1
    power       = 0
    tengine-lite library version: 1.0-dev
        squeezenet_v1.1  min =   32.74 ms   max =   32.74 ms   avg =   32.74 ms
            mobilenetv1  min =   31.33 ms   max =   31.33 ms   avg =   31.33 ms
            mobilenetv2  min =   35.55 ms   max =   35.55 ms   avg =   35.55 ms
            mobilenetv3  min =   37.65 ms   max =   37.65 ms   avg =   37.65 ms
            shufflenetv2  min =   10.93 ms   max =   10.93 ms   avg =   10.93 ms
                resnet18  min =   74.53 ms   max =   74.53 ms   avg =   74.53 ms
                resnet50  min =  175.55 ms   max =  175.55 ms   avg =  175.55 ms
            googlenet  min =  133.23 ms   max =  133.23 ms   avg =  133.23 ms
            inceptionv3  min =  298.22 ms   max =  298.22 ms   avg =  298.22 ms
                vgg16  min =  555.60 ms   max =  555.60 ms   avg =  555.60 ms
                    mssd  min =   69.41 ms   max =   69.41 ms   avg =   69.41 ms
            retinaface  min =   13.14 ms   max =   13.14 ms   avg =   13.14 ms
            yolov3_tiny  min =  132.67 ms   max =  132.67 ms   avg =  132.67 ms
        mobilefacenets  min =   14.95 ms   max =   14.95 ms   avg =   14.95 ms
    ALL TEST DONE
    ```