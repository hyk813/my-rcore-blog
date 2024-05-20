### rcore第二阶段记录 day1

（通过学校的操作系统课以及平时的一些常识能推导出的知识不做特别记录。）

领取任务，发现文件是空白的。注意到branch由多个分支，实际上实验的代码在各个branch里面，切分支即可。

#### 移除std库依赖

`#![no_std]`， 告诉 Rust 编译器不使用 Rust 标准库 std 转而使用核心库 core。

编译器运行的平台（x86_64）与可执行文件运行的目标平台不同的情况，称为 **交叉编译** (Cross Compile)。

`#[panic_handler]`，其大致功能是打印出错位置和原因并杀死当前应用。 但核心库 core 并没有提供这项功能，得靠我们自己实现。

新建一个子模块 `lang_items.rs`，在里面编写 panic 处理函数，通过标记 `#[panic_handler]` 告知编译器采用我们的实现：

```rust
// os/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**btw，main文件里面要引用lang_items这个mod**

##### 移除main

这是后编译，提示：

缺少一个名为 `start` 的语义项。 

在 `main.rs` 的开头加入设置 `#![no_main]` 告诉编译器我们没有一般意义上的 `main` 函数， 并将原来的 `main` 函数删除。这样编译器也就不需要考虑初始化工作了。

这个时候可以编译了。

##### 此时程序

学到的分析程序信息的工具使用方法

```bash
[文件格式]
$ file target/riscv64gc-unknown-none-elf/debug/os
[文件头信息]
$ rust-readobj -h target/riscv64gc-unknown-none-elf/debug/os
[反汇编导出汇编程序]
$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
```

**它是一个空程序，原因是缺少了编译器规定的入口函数 `_start` 。**

#### 构建用户态执行环境

##### 执行环境初始化

给 Rust 编译器编译器提供入口函数 `_start()`

##### 有显示支持的用户态执行环境

```rust
#![no_std]
#![no_main]
mod lang_items;


const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id,
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

#[no_mangle]
extern "C" fn _start() {
    println!("Hello, world!");
    sys_exit(9);
}

const SYSCALL_WRITE: usize = 64;

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
  syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

use core::fmt::{self, Write};

struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        sys_write(1, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

/// Print! to the host console using the format string and arguments.
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?))
    }
}

/// Println! to the host console using the format string and arguments.
#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

有修改编译目标、有移除标准库支持。有初始化函数，有退出系统调用。

Rust 的 core 库内建了以一系列帮助实现显示字符的基本 Trait 和数据结构，函数等，我们可以对其中的关键部分进行扩展，就可以实现定制的 `println!` 功能。

#### 构建裸机环境

`Hello world!` 应用程序从用户态搬到内核态。

用 QEMU 软件 `qemu-system-riscv64` 来模拟 RISC-V 64 计算机。加载内核程序

