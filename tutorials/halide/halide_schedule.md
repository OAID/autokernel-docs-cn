# Halide调度策略Schedule

## 循环调度策略

该部分的调度策略主要考虑在一个stage内的循环调度策略，主要有以下调度原语：

| primitives    | Description |
|--------------|--------------------|
| parallel (x,size)      | Splits the dimension by size and parallelizes the outer dimension.    | |
| vectorizer(x) |Vectorizes the dimension specified by x. ||
| unroll(x) |Unrolls the dimension specified by x.           ||
| reorder(dim_inner,…,dim_outer)   |Reorder to dimensions of a Func from dim_inner(most) to dim_outer(most)            | |
| split(x, outer,inner, factor)  |Splits the dimension specified by x into inner and outer dimensions. The inner dimension iterates from 0 to factor-1.||
| fuse (inner,outer, fused)|Fuses the dimensions specified by variables inner and outer into a single dimension specified by fused.||
|tile (x,y, xo,yo, xi, yi,xfactor, yfactor)|Tiles the dimensions specified by the variables x and y by xfactor and yfactor.||
## Call Schedule 调度策略
该部分的调度策略用于stage之间，主要在consumer和producer的函数之间，权衡考虑了冗余计算，并行性，局部性。主要有以下调用原语:

| primitives    | Description |
|--------------|--------------------|
|compute_at(consumer, dim)|Specify when the producer stage is computed with respect to the consumer stage.||
|compute_root()| Place the computation of the stage at the top level (before anywhere it is used).||
|store_at(consumer, dim)| Specify where the memory for the producer stage is allocated with respect to the consumer stage.||
|store_root()| Place the allocation of the stage at the top leve.||