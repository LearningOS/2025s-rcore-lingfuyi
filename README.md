# ch1：应用程序与基本环境
- 应用程序调用链：
  `应用程序->标准库->操作系统->指令集->硬件`
- 设置应用程序运行架构：
```shell
$ cargo run --target riscv64gc-unknown-none-elf
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not be installed
```
> 上面报错的原因是riscv平台没有默认安装标准库，需要手动安装。
## 移除标准库依赖
1. [x] 安装编译目标平台工具链：`rustup target add riscv64gc-unknown-none-elf`
2. 修改.cargo/config文件[build]为目标平台
3. 移除标准库和main函数：`![no_std] #![no_main]`
4. 增加panic处理函数：`#[panic_handler]` 
## 构建用户态执行程序
1. 实现程序退出系统调用：系统调用号为93，参数为0，返回值为0。
```rust
/*
x10用来存储系统调用第一个参数和返回值
x11用来存储系统调用第二个参数
x12用来存储系统调用第三个参数
x17用来存储系统调用号
*/
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
    sys_exit(9);
}
```
2. print 与 println 宏

宏主要分为`marcro_rules!`这种声明宏和`procedural`过程宏

> 过程宏procedural：在一个文件调用宏之前必须先定义它,或将其引入作用域

实现print!和println!的宏需要#[marcro_export] 注解表明将宏定义引入到当前作用域
```rust
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```
- `, $($arg: tt)+)?`该部分会匹配`,`分隔的一段重复序列,`+`表示该序列至少出现一次,`?`表示该序列可选出现一次，重复的内容是`tt`类型，即token tree。他将捕获到元变量`arg`中。
- `$($ fmt: literal $(...)?)`该部分有`literal`fragment-specifier，它会匹配一个字面量字符串，`$fmt`将捕获到该字符串。
- `concat!($fmt, "\n")`将`$fmt`和`"\n"`连接起来，得到一个带有换行符的字符串。
- `format_args!($fmt $(, $($arg)+)?)`将`$fmt`和`$($arg)+`格式化成一个字符串，并将其作为参数传递给`print`函数。