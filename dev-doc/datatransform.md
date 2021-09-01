# Data Transform开发文档

## 背景
Currently, most auto-schedule solutions (Halide, tvm, ansor, flextensor, ...) do auto-tuning to search best schedules based on functions(multi-dimension array) defined on algorithm/compute descriptions.

But the data layout of defined functions(multi-dimension array) are also important for the final performance.

## Key idea to understand why data transform?
- loop transformations are used to improve `temporal locality`
- data layout optimizations are used to improve `spatial locality`. 

we will exploiting `spatial locality` for given function data layout with defined computation.


## Auto Data transform
对一个Pipeline，我们定义了input数据排布的cost，然后自动对Pipeline中的每个input遍历data transform，计算其cost，选择cost最小的一种排布策略。

具体的文件说明如下：

## DataTransform.h
- 源码文件: [AutoKernel/AutoSearch/src/common/DataTransform.h](https://github.com/OAID/AutoKernel/blob/main/AutoSearch/src/common/DataTransform.h)
### **DataTransformer类**

```c++
template <typename T>
class DataTransformer : public IRVisitor
```
DataTransformer：用来在表达式中递归得搜索到目标函数，并对其按照模板T执行data transform。它继承自IRVisitor，可以使用accept和visit来遍历表达式的每一部分，从而找到我们希望执行data transform的目标函数。

```c++
std::string target_;
Func replace_func_;
T op_;
```
三个私有成员变量，target_表示要进行数据排布的函数名，replace_func_是替换后的新的函数，op_是要执行的数据排布操作。

在DataTransformer中，我们仅考虑了四则运算，即加，减，乘，除和除余，这里以加法运算做说明：

```c++
void visit(const Add *add) override {
    IRVisitor::visit(add);
    Add *node = const_cast<Add *>(reinterpret_cast<const Add *>(add)) ;
    if (const Halide::Internal::Call *v= node->a.as<Halide::Internal::Call>()) {
        if (v->name==target_)
        {
            auto expr = v->args;
            op_(expr);
            node->a = replace_func_(expr);
        }
    }
    if (const Halide::Internal::Call *v= node->b.as<Halide::Internal::Call>()) {
        if (v->name==target_)
        {
            auto expr = v->args;
            op_(expr);
            node->b = replace_func_(expr);
        }
    }
}
```
当加法运算的两端中的一个是函数(Call)时，检查其函数名是否为目标名，如果是目标名，则获取其函数的运算参数，并对所有运算参数执行op_的data transform操作，然后将这个Call函数重新用replace_func_定义。

### **data_transform函数**
```c++
template<typename T>
void data_transform_impl(Function f,Func target)
```
功能：对以Function f作为结尾的Pipeline，找到其中的target函数，并对之执行模板T操作，T是三种data transform的一种。

### **data_transform_impl函数**
```c++
void data_transform(std::vector<Function> &outputs,Func target,DataTransformMethod method)
```
功能：data_transform_impl的封装函数，根据提供的method来调用不同的data_transform_impl函数。DataTransformMethod是枚举类，定义在DataOP.h中。

### **deep_copy函数**
```c++
std::vector<Function> deep_copy(std::vector<Function> &inp)
```
功能：对输入的inp进行深拷贝并返回拷贝后的结果。

### **auto_data_transform函数**
```c++
void auto_data_transform(std::vector<Function> &outputs)
```
功能：对以outputs为结尾的pipeline，找到他们的输入，并执行最佳的data transform操作。auto_data_transform是所有优化器调用数据排布的接口函数。

实现细节：对outputs进行深拷贝得初始的best_output，然后对best_output搜索每一个input，并对每个input尝试三种data transform，在调用compute_layout_cost_impl函数计算数据排布的cost，选择最小的一个作为新的best_output，重复此过程直到无法再得到一个更低的cost为止。于是我们就获得了最终的best_output。


### **compute_order_cost函数**
```c++
double compute_order_cost(const Definition  &def,std::string function_name,const std::string &target, std::map<std::string,std::pair<int,int> >& bounds)
```
功能：计算一个定义（Definition）关于target（函数名称）的reorder_cost值，reorder_cost是数据排布cost的一种，另一种为use_distance_cost,具体参见cost_model.md。bounds是计算需要用到的变量的范围映射表。

### **compute_use_distance函数**
```c++
double compute_use_distance(const Definition &def,const std::string &target,std::map<std::string,std::pair<int,int> >& bounds)
```
功能：计算一个定义（Definition）关于target的use_distance_cost,bounds是计算需要用到的变量的范围映射表。

### **compute_layout_cost_impl函数**
```c++
double compute_layout_cost_impl(std::vector<Function> &outputs)
```
功能：计算以outputs结尾的pipeline的输入数据排布的cost，这是计算cost的接口函数，通过调用此函数可以得到最终的数据排布的cost。

## DataOP.h
DataOP头文件定义了进行数据重排布算子的类，它们都继承自DataOP类，并且需要重载DataOP中的函数。
### **DataTransformMethod**
```c++
enum class DataTransformMethod{
    REORDER,
    INTERLEAVE,
    SPLITY
};
```
这是一个枚举类，定义了data transform的三种方法

### DataOP类
```c++
class DataOP
{
public:
    // operate the vector<Expr>
    virtual void operator()(std::vector<Expr> &args){}
    virtual void operator()(Func &lfunc,Func &rfunc){}
    virtual std::string name(){return "$NULL";}
};
```
DataOP需要重载三个函数，两个括号运算符重载和一个命名函数,命名规则是$+算子名。

```c++
virtual void operator()(std::vector<Expr> &args){}
```
这个重载是用于在定义中将目标函数的参数进行重新排布的，以matmul.cpp为例：
```c++
func(x,y,b)=input_a(k,y,b)*input_b(x,k,b)
```
执行reorder操作得到：
```c++
Br(x,y,b) = input_b(y,x,b)
func(x,y,b)=input_a(k,y,b)*Br(k,x,b)
```
operator()(std::vector<Expr> &args) 用于在func表达式中将input_b(x,k,b)替换为Br(k,x,b)
```c++
virtual void operator()(Func &lfunc,Func &rfunc){}
```
operator()(Func &lfunc,Func &rfunc)用于构造Br(x,y,b)=input_b(y,x,b)表达式。

```c++
virtual std::string name(){return "$NULL";}
```
该函数用于提供一个函数名。

### **ReorderOP**

ReorderOP实现的是将一个input第一维和第二维数据排布交换的操作。其实现方法是定义一个新的函数，令reorder_func(x,y,...) = input(y,x,...)。

然后在每个涉及input的表达式中，都将input(x,y,..)替换为reorder_func(y,x,...)

### **InterleaveOP**

InterleaveOP实现的是将一个input第一维每8个数据划分。其实现方法是定义一个新的函数，令interleave_func(x,y,xo,...) = input(xo*8+x,y,...)。

然后在每个涉及input的表达式中，都将input(x,y,..)替换为interleave_func(x%8,y,x/8,...)

### **SplitYOP**

SplitYOP实现的是将一个input第二维每8个数据划分。其实现方法是定义一个新的函数，令splity_func(x,y,yo,...) = input(x,yo*8+y,...)。

然后在每个涉及input的表达式中，都将input(x,y,..)替换为splity_func(x,y%8,y/8,...)


## **AutoDataTransform的cost model**
根据实验，几种数据重新排布的方法可以提高优化后的执行速度，为了表征这些排布的效果，设计了cost函数来计算他们。我们定义了两类cost分别为reorder cost和use_distance cost

### **reorder cost**
对于一个这样的表达式：
```c++
func(x,y,z,b) = input(y,x,z)*b
```
在x和y维度上input和func的数据读取顺序发生了不一致，这种不一致在y很大时会导致cache miss率增大，其cache miss的程度可以用来衡量这种数据排布的cost。

由于当y较大时，对于此运算，发生cache miss的次数为循环次数（即cache几乎没有命中过），故次数为x*y*z*b。

对于上述表达式，如果我们增加一个定义：
```c++
Reorder(x,y,z) = input(y,x,z)
func(x,y,z,b) = Reorder(x,y,z)*b
```
可以看到，结果一样，func(x,y,z,b)这个表达式没有发生cache miss，只有上面的Reorder(x,y,z)发生了cache miss，其miss次数为x*y*z，可以看到，miss次数减少了b倍。所以我们定义reorder cost为发生数据读取顺序不一致的表达式的数据总量。

需要注意的是，这个cost并不是线性的，因为它需要x,y维度的大小来计算，当x*y在cache大小内时，此时cache中并不会完全miss，而对于cpu来说，有L1，L2,L3三层cache，每层大小不同，因此这个cost函数是阶梯上升的。具体参数设置可以从DataTransform.h中的compute_order_cost函数中获取。

### **use_distacne cost**
use_distance比reorder更加难理解一些，它是基于如下考虑的，当下游的一个表达式需要的数据大于一个cache大小时，我们可以对数据进行划分，使之按照一个cache大小的划分成块，这样的处理会使数据形成流水线，上游生产的数据顺序与下游需求的数据顺序一致，那么当下游读取数据时就可以不停的从cache中读取，而不需要经过内存-cache访存了。

具体看如下实例：
```c++
Rdom k(0,K);
func(x,y,b) += input_A(k,y,b)*input_b(x,k,b);
```
在这个例子中，input_b的数据需求始终是第二维优先的，我们可以通过reroder使其变成第一维，也可以使用划分的操作。我们对x进行划分,形成新的表达式：
```c++
Rdom k(0,K);
Bi(x,y,xo,b) = input_b(xo*16+x,y,b)
func(x,y,b) += input_A(k,y,b)*Bi(x%16,k,x/16,b);
```
新增加的Bi表达式的数据产生顺序是x,y,xo,b，其中x的范围为0~16，因此x*y的大小缩小了，这部分数据可以在一个cache中放下，而在func表达式中，我们需求的数据顺序对应到Bi中是y,x,xo,b。

在没有划分前，input_b的数据准备顺序和数据读取顺序之间相差了M倍（input_b的第一维的大小），而划分后，input_b的数据准备顺序和数据读取顺序之间仅相差了16，这大大提高了数据的读取效率。

use_distance cost的定义：表达式相邻两次计算时读取的输入数据在内存上的距离。

注意到如果原始的尺寸中x*y就很小，那么use_distance cost也没有意义，因为所有的数据都放在cache中了。所以我们的use_distance cost是一个分段函数。
