### 简答题
#### 1.三个bad测例
```bash
[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
...
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003ac, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
```
对于第一个错误案例，user中的源代码如下：
```rust
unsafe {
        #[allow(clippy::zero_ptr)]
        (0x0 as *mut u8).write_volatile(0);
}
```
尝试对0x0地址写入0，被内核kill。内核处理的流程如下：

    1. 通过中断触发`__alltraps`函数，保存trap上下文，调用下一个函数
    2. `trap_handler()`函数根据上下文信息中的`scause`值(`StoreFault`)步入错误处理分支，打印错误信息再调用`exit_current_and_run_next()`
    3. `exit_current_and_run_next()`kill当前任务并执行下一个任务

`rust-objdump`查看报错指令可见：
`804003ac: 23 00 00 00   sb      zero, 0x0(zero)`
这行通过sb指令向0x0这个非法地址写入0值。
对于第二个错误案例，它的错误值是`IllegalInstruction`，因此报错信息也不同
```rust
unsafe {
        core::arch::asm!("sret");
}
```
对第三个：
```rust
let mut sstatus: usize;
    unsafe {
        core::arch::asm!("csrr {}, sstatus", out(reg) sstatus);
}
```
尝试读取sstatus寄存器的值，最终的错误处理程序流程和第二个类似
#### 2.深入理解。。
1. L40:
    由于`__restore`不需要传入参数，a0在前一个执行的`__switch`中由下一个任务的上下文读取得到，所以a0是下一个任务的上下文之一，在任务中作为函数参数或者返回值被使用
    `__restore`用于内核态执行任务切换时从内核态转换为用户态。在任务切换以及执行第一个任务时使用

2. L43-L48：
    粘贴一点源码：
    ```asm
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    ```
    和前面`__alltraps`将错误信息和用户栈指针压入内核栈一样，这里从内核栈存储的trap上下文取出对应信息和用户栈地址存入对应寄存器。这三个寄存器对应触发`__alltraps`之前处于的特权级等信息、执行的最后指令的地址（恢复上一次执行用户态指令的状态）和用户栈的地址（为了切换到用户态时将这个地址传给sp）

3. L50-L56：
    x2即sp，现在还在用，需要最后载入值
    x4即tp，指向线程的私有变量集，通常不会被改变

4. L60：
    内核栈从sp换到sscratch
    用户栈从sscratch换到sp

5. 状态切换：
    sret为内核态切换到用户态（实际是切换到trap之前的特权模式）的指令。由于已经设置了sstatus值和sepc与用户态时相同，所以sret会回到用户态

6. L13：
    和4.相反，内核栈存入sp，用户栈存入sscratch

7. U->S：
    标志为ecall指令和ebreak指令

### 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

None

2. 此外，我也参考了 以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：
https://blog.csdn.net/zhangshangjie1/article/details/135003940
https://github.com/flyingcys/riscv-rtthread-programming-manual/blob/main/source/zh_CN/3.md

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。

