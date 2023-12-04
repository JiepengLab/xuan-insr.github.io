# 12 异常处理

!!! danger
    本文未完成，以下仅为学习笔记。
    
提供一种方式，使

If you have enough information to handle an error, it's not an exception.

`std::ofstream` 如果没有调用 dtor 程序就退出了的话，有可能没有 flush 缓冲区，因此导致该写到文件里的东西并没有完全写进去。

stack unwinding 

`exit` 或者 `std::exit` 之类的各种事情不会完成 stack unwinding；另外如果在写库函数，也不可能用这种东西

需要注意的是，如果一个 exception 没有被 catch，

与函数定义相比，异常处理器要少很多；与函数调用相比，异常出现的频率要低得多

对于不抛出异常的代码来说没有额外开销

能与其他语言，例如 C 语言合作

能够将异常分组，提供捕获某个或者某种异常的方式，例如捕获所有 I/O 异常，但是不需要知道具体是哪种异常

`class network_file_err : public network_err, public file_system_err {};`

Java 不支持多继承，所以 Java 没法像这样自定义同属于两个分类的异常。（Java 支持 implement 多个 interface，但可惜 `Throwable` 并不是个接口，而是个类。）

C++ 可以 throw 任何东西。而在 Java 中，throw 的东西必须是 `Throwable` 的子类。C++ 可以 throw 任何东西的一个重要原因是，它没理由要求我们必须用 `std::exception`，「单根结构」相关的讨论也说明了这一点。

一个单元的错误处理能力是有限的，类似计网里各个层次的纠错能力。一个单元必然存在一些情况会发生无法处理的情况，因而不得不向更高层汇报。最直接的例子就是任何单元都不可能从电脑断电的异常情况中恢复过来。

构造函数没有返回值，因此没办法简单地表示其中发生了错误。异常为此提供了解决方案，因为构造函数可以抛出异常。

C++ 异常处理机制保证部分构造的对象也能被正确销毁 TODO.

唤醒语义还是终止语义？即，C++ 是否提供能力，使得异常捕获者可以从异常抛出处继续运行？考虑操作系统中缺页中断的例子就是一种唤醒语义（OS 调用进程，进程发现缺页中断到 OS，OS 调页后重新唤醒进程）。但是，唤醒语义在实现和使用上也存在一些困难和可能的风险，参考 OS 死锁恢复的章节。事实上，C++ 历史上关于唤醒语义还是终止语义也进行了很多的讨论，但最终根据很多真实的故事决定了使用终止语义。这些故事基本都是在一些系统中实现了唤醒语义，但是最后发现慢而无用，慢慢取消掉了。

唤醒语义也可以被实现。

```c++
X* grabX() {
    for (;;) {
        if (canAcquireAnX) {
            ...
            return acquiredX;
        }
        grabXFailed();
    }
}

void grabXFailed() {
    if (canMakeXAvailable) {
        // make X available
        return;
    }
    throw CannotGetX();
}
```

C++ 异常会传播多层。虽然只能隐式传播一层是一个好的想法，但是例如 C++ 函数调用 C 函数又调用 C++ 函数的场景使得只传播一层的要求并不可能。

C++ 并不提供静态检查，即不提供指明一个函数会抛出哪些异常的能力；这与 Java 不同。一个重要考量是，如果一个被使用的库函数新增了一种它可能抛出的异常，那么所有依赖它的函数都需要做对应的更改，即要么能够捕获这种异常，要么也声明自己会抛出这种异常。

当然，对这种问题的一个避免方案是，新增异常时作为现有异常的派生类。虽然这有时可能并不够优雅。

`std::expect` C++23 https://youtu.be/eD-ceG-oByA?t=2236

关于 STL 的 move / copy: https://godbolt.org/z/YceThebxa ；如果操作失败，STL 希望能保持操作前的完整样子；因此除非 move ctor 是 `noexcept` 的，否则会使用 copy ctor 来保证安全。

RAII

```c++
void foo() {
    // ...
    
    X* x = grabX(); // might throw cannotGetX;
    
    // ...
}

void bar() {
    try {
        foo();
    } catch(CannotGetX & e) {
        if (makeXAvailable())
            continueFromThrow;
        else
            throw e;
    }
}
```
