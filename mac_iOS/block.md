# 一个蹊跷的objc block问题

## 一个block问题的引入

```block```在```objc```中的使用比较广泛,语言特性很好的解决了callback回调场景问题，尤其是和```Dispatch Queue```异步队列的配合使用，相得益彰。易用性是它最大的一个特点，好的设计往往使用起来足够简单，常常会让你忽略背后本质层面的东西。但最近碰到一个```block```的问题实在蹊跷，不得不重新对```block```认识了解。

```objectivec
-(void) doSomething {
  int i = 11;
  int *pi = &i;

  void (^task)(void) = ^{
    NSLog(@"pi value is %d", *pi);
  };

  dispatch_async(dispatch_get_main_queue(), task);
}
```

为了方便说明问题，我把代码简化如上，我的本意是通过在栈上分配一个整型变量 ```i``` ，为了状态的延续和传递，把```i```的指针传给后续的相关处理函数，因此我草率的认为在block内部通过dereference拿到```i```的值不是一个问题。

但奇怪的是，pi的值一直在变，一会是```0```，一会是```32760```，反正就是不是```11```。

```shell
2022-02-13 00:00:00.000000+0800 blk_test[23805:592934] pi value is 32760
```

不加思索的使用，往往会想当然的将函数的职责强加在block身上，但事实明显和预期有差异。

## block到底是什么

**block是C Level语言特性扩展，因此你可以在C中使用block**，只要你使用支持block的编译器(如Apple的Clang)，引入block相关的头文件即可。

```c
#include <stdio.h>
#include <Block.h>
typedef void (^TaskType)(void);

void logit() {
  TaskType task = ^{
    printf("hello world\n");
  };
  task();
}

int main(int argc, char **argv) {
  logit();
  return 0;
}
```

**block对象也是objc对象(Objc是C的超集)**，因此block对象可以被加入到Objc的容器数据类型中的，例如NSArray, NSDictionary等。

```objectivec
void (^task)(void) = ^{
  NSLog(@"hello block");
};

NSMutableArray<void (^)(void)> *blocks = [[NSMutableArray alloc] init];
[blocks addObject:task];

NSLog(@"blocks %@", blocks);
```

打印一下```blocks```容器中的对象

```shell
2022-02-13 00:00:00.000000+0800 blk_test[23805:592934] blocks (
    "<__NSMallocBlock__: 0x600003b50090>"
)
```

同时可以看到在栈上分配的block对象，最后在容器中被拷贝为堆上管理的block对象(后面会讲)，block的内存管理标记为```_NSMallocBlock```。

说白了，在C的上下文下，block数据类型变量decay为C指针；在objc上下文下，block类型变量编译为objc对象指针(上例中指向```0x600003b50090```内存地址)。

## 为什么是这样的block

说到这里，可能还是没法理解block类型对象的具体数据结构是怎样的，内存是在哪里分配的呢，在objc上下文下，既然是objc对象，那一定是在堆上管理内存的吗。

本着在编程世界里，万事万物都与C有或多或少的关系和联系。扯远了！```clang```提供了一个```-rewrite-objc```的编译选项可以可以帮助认识这个问题的本质。基于下面这段objc的实例代码，我尝试将它转换重写为c++代码。

```objectivec
-(void) viewDidLoad {
  int i = 11;
  int *pi = &i;

  void (^task)(void) = ^{
    NSLog(@"pi value is %d", *pi);
  };
  dispatch_async(dispatch_get_main_queue(), task);
}
```

直接使用clang转换

```shell
clang -rewrite-objc main.m
```

转换之后的代码比较多，为了排除干扰因素，我去掉和这个问题不相关的代码，重新梳理后，有如下几个点比较关键。

首先找到```viewDidLoad```的调用点，```viewDidLoad```在这里被转译为静态方法```_I_ViewController_viewDidLoad```。

```cpp
static void _I_ViewController_viewDidLoad(ViewController * self, SEL _cmd) {
  ((void (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("ViewController"))}, sel_registerName("viewDidLoad"));

  int i = 11;
  int *pi = &i;

  dispatch_async(
      dispatch_get_main_queue(),
      ( (void (*)())&__ViewController__viewDidLoad_block_impl_0( (void *)__ViewController__viewDidLoad_block_func_0,
                                                                &__ViewController__viewDidLoad_block_desc_0_DATA,
                                                                pi)
        )
  );
}
```

虽然面目全非，根据objc的源码依稀可以找到线索，除了变量i的声明和其指针声明，dispatch相关的函数由链接库libdispatch提供。可以看到block task的定义被转译为```__ViewController__viewDidLoad_block_impl_0```的实现，这其实是一个结构体，其数据结构定义如下

```cpp
struct __ViewController__viewDidLoad_block_impl_0 {
  struct __block_impl impl;
  struct __ViewController__viewDidLoad_block_desc_0* Desc;
  int *pi;
  __ViewController__viewDidLoad_block_impl_0(void *fp, struct __ViewController__viewDidLoad_block_desc_0 *desc, int *_pi, int flags=0) : pi(_pi) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

可以看到指针pi被拷贝进结构体字段pi中，同时结构体还接受了一个fp的void指针，在```dispatch_async```的调用处上传入的是```__ViewController__viewDidLoad_block_func_0```。其定义如下

```cpp
static void __ViewController__viewDidLoad_block_func_0(struct __ViewController__viewDidLoad_block_impl_0 *__cself) {
  int *pi = __cself->pi; // bound by copy
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_p9_l1wd3kgs4nnf1c0fmgcnbdzw0000gn_T_ViewController_934b57_mi_0, *pi);
}
```

线索有一点多，信息量偏大，综合梳理一下就是

- block task的对象，被转译为一个结构体数据结构```__ViewController__viewDidLoad_block_impl_0```，这个结构体最终被转换为一个函数指针被传给```dispatch_async```调用

- 结构体```__ViewController__viewDidLoad_block_impl_0```最初是在栈上被分配，局部指针变量```pi```被作为结构体的构造参数传入，这也就是block的捕获能力。因此block的类型为```_NSConcreteStackBlock```

- block task的body体中的代码最终被编译为一个静态函数```__ViewController__viewDidLoad_block_func_0```，最终被传给block结构体的impl字段，作为最后的执行回调。

## 如何正确的使用block

### 声明和定义

**block的声明比较简单**，形式上和函数指针的声明很相似。只需要把```*```换成```^```就可以。

声明一个函数指针如下

```c
int (*add)(int a, int b)
```

相对应，声明一个block类型如下

```objectivec
int (^add)(int a, int b)
```

**block的定义也比较简单**，以```^```开头，依次声明返回，参数和block体，如下

```objectivec
int (^add)(int a, int b) = ^int (int a, int b) {
  return a + b;
};
```

在block的定义里，你也可以不写返回类型，编译器会帮你推测返回类型，这样书写起来就简单一点，显得没有那么啰嗦。

```objectivec
int (^add)(int a, int b) = ^(int a, int b) {
  return a + b;
};
```

通常为了后续使用方便，可以使用typedef声明一个block的类型别名，维护上也比较方便，更显言简意赅。

```objectivec
typedef int (^AddFunc)(int a, int b)

AddFunc add = ^(int a, int b) {
  return a + b;
};
```

### block的三种存储类型

基于在运行时中内存驻留位置，将block分类三种类型，通常也不用关注block到底存在哪里，但了解一下有助于解释一些边界问题。

|      | __NSGlobalBlock | __NSStackBlock | __NSMallocBlock        |
| ---- | --------------- | -------------- | ---------------------- |
| 分配位置 | 数据段, 存放静态数据     | 栈              | 堆                      |
| 分配条件 | 没有局部变量的访问       | 访问了局部变量        | 通常是从栈上分配的block对象copy而来 |

- ```__NSGlobalBlock``` 没有访问局部变量的block，会被分配到静态数据段。

- ```__NSStackBlock``` 如果block体内部访问上下文环境的局部变量，block就会正常在栈上分配内存。

- ```__NSMallocBlock``` 这种情况最为复杂，通常如果对一个__NSStackBlock进行copy操作，新生成的block会分配在堆上。在ARC自动内存管理模式下，存在一些特殊情况，编译器会自动将```__NSStackBlock``` copy为```__NSMallocBlock```。例如函数返回block、将block赋值给```__strong```类型指针、在异步模式下传递参数等等。

### block中使用的变量类型和相应注意事项

- **读写文件作用域变量，包含静态文件作用域变量**，文件作用域变量对文件内部的函数，方法均可见，因此读写此类变量和普通函数中无异。同理类推，在block体内使用全局函数和全局静态函数也是直接使用即可。

- **读写自动变量**，包含上下文函数或方法形参。读取自动变量比较简单，如果在block体内访问自动变量，block实现会捕获自动变量，即传入自动变量到block结构体实现中，包括上下文函数或方法的形参变量。当存在多个捕获时，每个捕获都有自己的拷贝。
  
  如果还需要写入自动变量，就需要将变量声明为__block存储类型，它的作用是告诉编译器，你需要修改自动变量，其实现上与只读相比差异比较大，一个包含其值拷贝的结构体会创建，并传入到block的实现结构体中。当存在多个读写block变量的捕获，它们之间会共享值拷贝结构体。对变量的修改，会被转发给值结构体的修改，同时在多个捕获的结构体中均可见。

- **读写```objc```实例对象字段**，如果block体内访问了上下文环境的objc实例对象字段，一种典型的情况是，在类的方法中定义了block类型数据，当在block体内访问类的self指针时，block实现会捕获类实例。

- 避免循环引用，因为block实现的捕获能力，当block体内使用了类对象指针，block的实现实际就会持有类对象的引用，如果恰好类对象强引用了此block对象，会导致类对象无法释放问题
  
  ```objectivec
  typedef void (^Callback)(void);
  
  @interface ViewController()
  @property (nonatomic) Callback callback;
  @property (nonatomic) NSString *title;
  @end
  
  @implementation ViewController
  -(void) doSomething {
    self.callback = ^{
      self.title = @"title";
    }
  }
  @end
  ```
  
   callback block被ViewController对象持有，同时ViewController的指针被block对象捕获引用，这种情况会导致ViewController在退出时无法释放内存。解决办法是使用__weak类型引用声明。__weak类型的变量不会增加对象的引用计数，因此不会影响对象的内存释放。
  
  ```objectivec
  typedef void (^Callback)(void);
  
  @interface ViewController()
  @property (nonatomic) Callback callback;
  @property (nonatomic) NSString *title;
  @end
  
  @implementation ViewController
  -(void) doSomething {
    __weak typeof(*self) *host = self;
    self.callback = ^{
      host.title = @"title";
    }
  }
  @end
  ```

## 最后

在block内部访问自动变量时，因为block实现的捕获，会导致读取和修改的其实是自动变量的拷贝，其内存地址早已经改变，一切不合寻常的现象都源于此。
