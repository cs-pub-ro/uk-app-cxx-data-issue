# Unikraft C++ Large Global Data Issue

Defining a large string vector as a global data results in page fault.
This is a program that highlights the issue.

The `main.cpp` program defines the `CXX_VALUES` vector of strings.
For `2725` of items (each using the `"EXAMPLE"` string) it doesn't fail.
With `2726` items it fails.

Trigger the issue by assigning the `USE_CXX_VALUES` macro to `1` and the `SHOW_ISSUE` macro to `1` at the beginning of the `main.cpp` program.
This is the default assignment.

## Building

Build the program either using `kraft`:
```
kraft configure -p kvm -m x86_64
kraft build
```
Build it for `linuxu` using:
```
kraft configure -p linuxu -m x86_64
kraft build
```

Or by using the `Makefile`-based approach:
```
make menuconfig
make
```
When running `make menuconfig` simply save the configuration.
This step is required to load the configuration from `Config.uk`.

In `make menuconfig` you can select both the `kvm` and `linuxu` platforms to build an image for each platform.

## Running

The images are part of the `build/` folder.
Run them either using `kraft`:
```
kraft run -p kvm
```
or
```
kraft run -p linuxu
```

Or directly:
```
./build/app-cxx-data-issue_linuxu-x86_64
```
or
```
./qemu-guest -k build/app-cxx-data-issue_kvm-x86_64.dbg
```

## Summary of Issue

Running for `linuxu` results in a `Segmentation fault` message.
Running for `kvm` results in a page fault message.

A stack trace of the issue points to:
```
(gdb) bt
#0  std::__1::ios_base::Init::Init (this=<optimized out>) at /home/razvan/projects/unicore/unikraft-my/apps/t/build/libcxx/origin/libcxx-7.0.0.src/src/iostream.cpp:85
#1  0x0000000000204fae in __static_initialization_and_destruction_0 (__priority=65535, __initialize_p=1) at /home/razvan/projects/unicore/unikraft-my/apps/t/build/libcxx/origin/libcxx-7
.0.0.src/src/iostream.cpp:80
#2  _GLOBAL__sub_I__ZNSt3__13cinE () at /home/razvan/projects/unicore/unikraft-my/apps/t/build/libcxx/origin/libcxx-7.0.0.src/src/iostream.cpp:124
#3  0x000000000010c0b2 in main_thread_func (arg=0x3fdff50) at /home/razvan/projects/unicore/unikraft-my/unikraft.git/lib/ukboot/boot.c:135
#4  0x0000000000105757 in asm_thread_starter () at /home/razvan/projects/unicore/unikraft-my/unikraft.git/plat/common/x86/thread_start.S:39
#5  0x0000000000000000 in ?? ()
```

The highlighted instruction (`=>`) is the one where the issue occurs.
```
(gdb) disass
Dump of assembler code for function std::__1::ios_base::Init::Init():
   0x0000000000141e10 <+0>:     push   %rbp
   0x0000000000141e11 <+1>:     mov    %rsp,%rbp
   0x0000000000141e14 <+4>:     push   %rbx
   0x0000000000141e15 <+5>:     sub    $0x18,%rsp
   0x0000000000141e19 <+9>:     callq  0x110ec0 <__getreent>
   0x0000000000141e1e <+14>:    mov    $0x29a3a0,%edi
=> 0x0000000000141e23 <+19>:    mov    0x8(%rax),%rbx
   0x0000000000141e27 <+23>:    callq  0x213640 <std::__1::basic_streambuf<char, std::__1::char_traits<char> >::basic_streambuf()>
   0x0000000000141e2c <+28>:    lea    -0x18(%rbp),%rdi
```

It seems to point to a C++ initialization issue in Unikraft, before the calling of the `main()` function.
It's possible to be related to memory alocation / initialization in the Unikraft startup sequence.
