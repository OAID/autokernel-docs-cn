# 源码安装

1. 编译安装 Halide 

   ```
   git clone https://github.com/halide/Halide
   cd Halide && mkdir build && cd build
   export HALIDE_DIR=/path/halide-install # 安装路径，根据实际修改
   cmake .. -DTARGET_WEBASSEMBLY=OFF -DCMAKE_INSTALL_PREFIX=${HALIDE_DIR}
   make -j `nproc` && make install # 编译安装
   ```
2. 编译 Tengine 

   ```
   git clone https://github.com/OAID/Tengine.git
   cd Tengine && mkdir build && cd build
   export TENGINE_DIR=/path/tengine-install # Tengine 安装路径
   cmake .. -DCMAKE_INSTALL_PREFIX=${TENGINE_DIR}
   make -j `nproc` && make install # 编译安装
   ```

3. 编译 AutoKernel

   ```
   git clone https://github.com/OAID/AutoKernel.git
   cd 	AutoKernel/autokernel_plugin 
   find . -name "*.sh" | xargs chmod +x  #给 sh 脚本添加可执行权限
   ```

   修改 `./scripts/generate.sh 中 halide 库的安装路径`

   ```
   # 将第一行中的 HALIDE_DIR 路径修改成第一步中的安装路径
   export HALIDE_DIR=/path/halide-install
   ```

   修改 Tengine 安装路径,  打开 `autokernel_plugin/CMakeLists.txt` 中 `TENGINE_ROOT`

   ```
   set(TENGINE_ROOT /path/Tengine) # 修改为 Tengine 项目所在目录
   set(TENGINE_DIR /path/tengine-install) # 修改为第二步中的安装路径
   ```

   编译AutoKernel

   ```
   ./scripts/generate.sh  #自动生成算子汇编文件
   mkdir build && cd build
   cmake .. && make -j `nproc`
   ```

   将 libautokernel.so 所在的路径加入 LD_LIBRARY_PATH， 在 `AutoKernel/autokernel_plugin/build/` 目录下执行

   ```
   export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:`pwd`/src/
   ```

   在 AutoKernel/autokernel_plugin 目录下执行测试程序

   ```
   ./build/tests/tm_classification
   ```

   运行结果如下

   ```
   start to run register cpu allocator
   
   ......
   
   [INFO]: using halide maxpooling....
   [INFO]: using halide maxpooling....
   [INFO]: using halide maxpooling....
   current 53.934 ms
   
   Model name : squeezenet
   tengine model file : models/squeezenet.tmfile
   label file : models/synset_words.txt
   image file : images/cat.jpg
   img_h, imag_w, scale, mean[3] : 227 227 1 104.007 116.669 122.679
   
   Repeat 1 times, avg time per run is 53.934 ms
   max time is 53.934 ms, min time is 53.934 ms
   --------------------------------------
   0.2732 - "n02123045 tabby, tabby cat"
   0.2676 - "n02123159 tiger cat"
   0.1810 - "n02119789 kit fox, Vulpes macrotis"
   0.0818 - "n02124075 Egyptian cat"
   0.0724 - "n02085620 Chihuahua"
   --------------------------------------
   ALL TEST DONE
   ```