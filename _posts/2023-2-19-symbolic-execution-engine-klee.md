---
layout: post
title:  "Symbolic Execution Engine KLEE 阅读笔记"
date:   2023-2-19 13:35:46 +0800
categories: symbolic execution
author: Yutan Young
---

## KLEE符号执行原理
1. 使用Clang将C语言源码解析成中间表示LLVM-IR
2. 实现一套基于LLVM-IR的符号执行解释器，对LLVM-IR逐行解释执行。
3. 执行指令的对象叫做ExecutionState。
+ 状态的信息包含栈，堆上的内存对象，当前执行的程序位置PC，以及路径上的约束条件
+ 状态被保存在states集合里，称为状态池。初始化时在入口处创建一个state并往下执行
+ 当执行的指令为与符号变量相关的跳转指令时，会根据分支的约束条件分裂成多个状态放入状态池。
+ KLEE可以按照一定的搜索策略从状态池里取出一个状态，执行一条指令，更新其状态，放回去继续搜索。
4. 状态遇到程序出口或者错误，会产生用例并退出，该状态从状态池中删除。当所有状态执行完毕或者haltExecution被触发（超时、内存错误等问题），klee退出循环结束执行。
5. 测试用例的生成基于约束求解器，将该状态上的约束送入求解器得到符合条件的测试用例。
6. KLEE实现了一套符号执行的虚拟机，每个状态都有属于自己的内存，各个执行状态是互不影响的

## KLEE对变量的存储

### 表达式
相比于普通的解释器，KLEE的特殊点在于符号执行需要存储符号值，即**不确定**的值.KLEE实现了一套表达式类Expr来存储符号表达式，树的节点代表对另一个表达式的引用。例如若a,b为符号值，c=a+b,则构建表达式树，
```  
    +
  /   \
 a     b
```
相关的代码位于`lib/Expr`中。大多数情况下KLEE创建的表达式都是被ref包裹的，ref是轻量化的shared_ptr，保证创建的表达式对象会不被引用的时候候释放掉。

对于一个表达式类型对象，可以利用dyn_cast转换后是否为空来判断表达式是否为符号类型和常量类型。在KLEE对IR指令逐条解析的代码中，可以看到类似如下创建符号表达式的运算

{% highlight c++ %}
// ConstantExpr::alloc是生成一个常数表达式，
// 创建一个常数，宽度为KLEE上下文规定的指针宽度w，大小为addr，为数组的基地址
// 表示创建一个w位的常数,值为addr
uint64_t addr = 0x10000;
uint64_t size = 0x200;
ref<ConstantExpr> Base = ConstantExpr::alloc(addr, Context::get().getPointerWidth());
// 与上面一句代码类似，规定数组的上界
ref<ConstantExpr> Ubound = ConstantExpr::alloc(addr + size, Context::get().getPointerWidth());
// 创建一个无符号大于等于Uge表达式
ref<Expr> Ucs = UgeExpr::create(address, Base);
// 创建一个无符号小于等于Ult表达式
ref<Expr> Lcs = UltExpr::create(address, Ubound);
// 创建一个逻辑与表达式
ref<Expr> constraints = AndExpr::create(Ucs, Lcs);

// 判断是否为常量表达式
// 若CE不是null，表示表达式中没有符号变量
if (ConstantExpr *CE = dyn_cast<ConstantExpr>(address)) {
    // omitted
}
{% endhighlight %}

### 解释指令
klee在自己实现的虚拟机上解释执行LLVM-IR，源码上其实就是对每一条指令做了SwitchCase，每个Case对应一个指令的具体行为。下列代码位于`lib/Core/Executor`中的`Executor::executeInstruction`函数

{% highlight c++ %}
Instruction *i = ki->inst;
switch (i->getOpcode()) {
	// getOpcode是LLVM-API，能够获得一条IR指令的类型
  case Instruction::Ret: // do Ret
  case Instruction::Br:  // do Br
  case Instruction::Call:// do Call
  // ...
} 
{% endhighlight %}


### 内存空间
#### 栈和全局区
KLEE实现了栈式虚拟机。State中的AddressSpace对应堆区，stack对应栈区，还有存全局和静态变量的静态区，和LLVM的数据是对应的。

其中，每个函数在分配内存的时候，需要计算LLVM-IR中函数分配的虚拟寄存器空间大小$N$，分配对应的空间，每个变量存储的位置都对应一个格子。

对于一条指令，KLEE提供了一些接口来存取变量：

+ eval：获得操作数所在的栈上或静态区对应的存储空间
+ bindLocal是将表达式存储到局部变量对应的空间中。

eval在传入负数的时候，会到静态区去寻找变量。-1即静态区的第一个变量，以此类推。（感觉是一种偷懒的方法）

例如下面这句代码就是找到state状态中第一个操作数和第二个操作数，取出他们的value，构造一个除法表达式，然后把这个表达式存储到当前地址对应的位置。
``` cpp 
case Instruction::UDiv: {
  ref<Expr> left = eval(ki, 0, state).value;
  ref<Expr> right = eval(ki, 1, state).value;
  ref<Expr> result = UDivExpr::create(left, right);
  bindLocal(ki, state, result);
  break;
}
```
#### 地址空间、内存对象和对象状态
对于每一个ExecutionState，都需要有独立的**地址空间，**，可以理解为对堆区的抽象。state对象内部带有AddressSpace类型的对象实例。AddressSpace内部存放了MemoryObject->ObjectState的映射。简单来说每个State拥有自己的一套的独立内存，叫做AddressSpace地址空间，state需要知道哪个具体的地址数值上有内存对象，申请的大小有多大，维护这个信息的对象叫做MemoryObject。得到了某个地址的MemoryObject，要知道内存对象中间那部分是符号值那部分是具体值，如果是符号执行，那表达式是什么，有什么约束条件，就有了ObjectState对象

例如 有状态`s0`:在地址0xcafebabe申请了一个大小为40个字节的全0数组，就可以得到下列信息
```
state: s0
addressSpace: [ 
  {0xcafebabe, 40B}:{0}
]
```
![image.png]({{ "/images/klee/memobj.png" | absolute_url }})

#### MemoryObject
在一个AddressSpace地址空间中，需要存储很多对象，例如各种类型的变量，数组，结构体等等。MemoryObject有size，address等等属性，并且每个MemoryObject都能找到对应的ObjectState对象
#### ObjectState
ObjectState类的名字会让人有些不明所以。其实要把它和MemoryObject一起理解。它代表了一个特定的MemoryObject各个字节的状态，例如一个struct结构体中含有两个int，一个是符号变量，一个不是符号变量。则ObjectState中就会记下相关信息，在需要更新时（例如在某条路径符号变量经过约束求解只有一个可取值，可以将其变成具体的变量）更改ObjectState的状态。
#### Range与Domain
domain（定义域）range（值域）domain默认是32位，range默认是8位
domain是指针的长度，range是存储数据的长度

> Domain is how many bits can be used to access the array [32 bits] 
>
> Range is the size (in bits) of the number stored there (array of bytes -> 8)


#### 变量的存储机制
ObjectState内有几个主要的Array，一个是ConcreteStore，一个是KnownSymbolic，并且有一个BitArray类型的ConcreteMask，flushMask，存储了ObjectState对应的地址是否被写入，是符号变量还是具体变量。

对于某个address，若ConcreteMask[address]为真，则说明该值存储在ConcreteStore[address]下，是常量，否则在KnownSymbolic[address]下，对应着一个符号值。

![image](/images/klee/mask.png)
#### flush机制
klee维护一个cache，当cache空间不足时，将值写入内存
klee中的cache是ArrayCache的对象，cache中针对符号类型与具体类型有不同的处理策略
具体值的cache就是一组指向array页面的指针的集合，而符号cache是一组hashset,对应的cache页面唯一
```cpp
typedef std::unordered_set
	<const Array *, klee::ArrayHashFn, klee::EquivArrayCmpFn> ArrayHashMap;
typedef std::vector<const Array *> ArrayPtrVec;

ArrayHashMap cachedSymbolicArrays;
ArrayPtrVec concreteArrays;
```
#### updatelists类型
UpdateList对象存储了对某个array对象的一系列update操作，List中存储的对象是updateNode节点，本质上是个链表，UpdateNode的数据部分是修改的索引和修改的值，相当于缓冲区。在换页的时候将写入操作真实地写入KLEE内存中。
类似的思想可以参考"操作系统"的内存管理机制

```cpp
class UpdateNode {
	// ...
	const ref<UpdateNode> next;
	ref<Expr> index, value;
}; //典型的链表节点结构

/// Class representing a complete list of updates into an array.
class UpdateList { 
public:
// 针对Array类型的root对象所做的修改
// root是update操作对应的一块内存区域
const Array *root;

// updateNode表示
// 本质上是一串存储了操作的列表
ref<UpdateNode> head;
};

// ...
// extend函数 若root存在首先需要确定index与value的size与root是匹配的
// 而后将修改操作写入UpdateList的链表中去
void UpdateList::extend(const ref<Expr> &index, const ref<Expr> &value) {
  
  if (root) {
    assert(root->getDomain() == index->getWidth());
    assert(root->getRange() == value->getWidth());
  }

  head = new UpdateNode(head, index, value);
}
```
#### COW机制
刚分裂的两个地址其实内存空间是相同的，所以很多内存信息是共享的。只有二者被修改后不同的部分会被复制成两份，可以减少内存占用，这是Copy on Write COW机制
### KLEE的内存申请与查询
翻看case Instruction::Alloca下的代码，KLEE完成初始化后执行了executeAlloc。
以下是executeAlloc简化后的代码，大致上是申请内部后绑定给寄存器
```cpp
void Executor::executeAlloc(ExecutionState &state,
                            ref<Expr> size,
                            bool isLocal,
                            KInstruction *target,
                            bool zeroMemory,
                            const ObjectState *reallocFrom,
                            size_t allocationAlignment) {
    
  
  if (ConstantExpr *CE = dyn_cast<ConstantExpr>(size)) {
    const llvm::Value *allocSite = state.prevPC->inst;
      
    MemoryObject *mo =
        memory->allocate(CE->getZExtValue(), isLocal, /*isGlobal=*/false,
                         allocSite, allocationAlignment);
    // target是存放内存申请结果的寄存器
    // 没生成对象，返回空地址
    if (!mo) {
      bindLocal(target, state, 
                ConstantExpr::alloc(0, Context::get().getPointerWidth()));
    } 
    // 生成了对象MemoryObject，将地址与内存对象首地址绑定后送入target
    else {
      ObjectState *os = bindObjectInState(state, mo, isLocal);
      bindLocal(target, state, mo->getBaseExpr());
    }
```
再观察load和Store指令，Load是拿到读取的地址base，Store是拿到写入的地址和写入的数据，二者都执行了MemoryOperation操作。可以大致明白该函数是负责内存读写的。
```cpp
  case Instruction::Load: {
    ref<Expr> base = eval(ki, 0, state).value;
    executeMemoryOperation(state, false, base, 0, ki);
    break;
  }
  case Instruction::Store: {
    ref<Expr> base = eval(ki, 1, state).value;
    ref<Expr> value = eval(ki, 0, state).value;
    executeMemoryOperation(state, true, base, value, 0);
    break;
  }
```
细节其实不必深究，有个很有意思的地方是符号指针的读写
当KLEE检测到该指针有越界的可能，或者不知道指针指向哪里，它会穷尽内存空间的所有可能性，胡乱指向，进行求解。这大大增加了求解时间（比较搞的是源码的注释里写了一句 who cares）
```cpp
// XXX there is some query wasteage here. who cares?
for (ResolutionList::iterator i = rl.begin(), ie = rl.end(); i != ie; ++i) {
const MemoryObject *mo = i->first;
const ObjectState *os = i->second;
ref<Expr> inBounds = mo->getBoundsCheckPointer(address, bytes);

StatePair branches = fork(*unbound, inBounds, true, BranchType::MemOp);
ExecutionState *bound = branches.first;
```
# Searcher状态选择器
KLEE会使用一定的策略从状态池里筛选状态，KLEE实现的策略有DFS，BFS，Random等，可以参考KLEE官方的文档，具体的代码在lib/Core/Searcher.h下
所有的Searcher都是继承自Searcher对象，Searcher本质上是抽象类，初始化、update、empty都是纯虚函数，如果需要重新实现一套搜索模式，可以继承Searcher并加入自己的实现，同时在Searcher基类上的CoreSearchType中加入自己实现的Searcher。

Searcher最主要的两个函数是update和selectState，update是按照一定的顺序，根据加入和删除的状态列表，在状态池里更新。selectState是根据一定的方式从状态池里取出一个状态
以最简单的DFS为例：
```cpp
ExecutionState &DFSSearcher::selectState() {
  return *states.back();
}

void DFSSearcher::update(ExecutionState *current,
                         const std::vector<ExecutionState *> &addedStates,
                         const std::vector<ExecutionState *> &removedStates) {
  // insert states
  states.insert(states.end(), addedStates.begin(), addedStates.end());

  // remove states
  for (const auto state : removedStates) {
    if (state == states.back()) {
      states.pop_back();
    } else {
      auto it = std::find(states.begin(), states.end(), state);
      assert(it != states.end() && "invalid state removed");
      states.erase(it);
    }
  }
}

bool DFSSearcher::empty() {
  return states.empty();
}
```
updateStates就是按照顺序将状态加入状态池，select就是简单地选择最后一个状态，因为加入的新状态都是当前执行的状态产生的，所以选择状态池最后一个状态相当于继续执行上一个状态的某条路径，因此这样的方法就是将一条路径往深度执行。
其他的Searcher策略可以自行阅读lib/Core/Searcher.cpp源码或阅读文档
[KLEE--搜索方法Search Heuristics_fjnuzs的博客-CSDN博客](https://blog.csdn.net/fjnuzs/article/details/79869903)

### 约束求解与用例生成
多种求解器：z3 Stp 等等

用例生成的时机是某个状态触发了BUG、导致执行不下去了，或者正常结束（包括深度到达限制，超时。）
具体代码在`lib/Core/Executor.cpp的terminateStateOnError`函数下

## 总结
阅读了两个月KLEE的源码，简单地描述了KLEE的运行逻辑，主要是解释器解释执行、虚拟机内存管理和符号执行的一些内容

顺便还提高了C++的阅读能力...

获益匪浅