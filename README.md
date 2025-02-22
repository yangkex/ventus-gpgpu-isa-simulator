# Ventus RISC-V GPGPU ISA Simulator

## About

Ventus RISC-V GPGPU isa simulator based on Spike.

This simulator is developed by 张泰然 and 郑名宸.

This simulator adds the following features into origin spike simulator:
- support custom instruction for vector branch control and warp control
  - vector branch control: vbeq, vbne, vblt, vbge, vbltu, vbgeu
  - warp control: barrier, endprg
- support custom csr: numw, numt, tid, wid, gds, lds
- enhenced interactive mode commands to facilitate debugging of Ventus GPGPU programs

For more information of the original Spike, please refer to README-spike-origin.md

## installation and usage

### build step

Build step is identical to the origin build step of Spike. Assume that the RISCV enironment variable is set to the RISC-V tools install path.

    $ apt-get install device-tree-compiler
    $ mkdir build
    $ cd build
    $ ../configure --prefix=$RISCV
    $ make
    $ [sudo] make install

### compiling and running Ventus GPGPU program in Spike

#### prerequisites
We execute Ventus GPGPU program with Spike in machine mode. To produce executable file, we utilize libgloss-htif and the modifed version of riscv-gnu-toolchain. The following commands set up the necessory environment.


    # install riscv64-unknown-elf toolchain
    $ git clone https://github.com/THU-DSP-LAB/riscv-gnu-toolchain.git
    $ cd riscv-gnu-toolchain
    $
    $ git init riscv-binutils
    $ git update riscv-binutils
    $ cd riscv-binutils
    $ git checkout main
    $ cd ..
    $
    $ mkdir build && cd build
    $ ../configure --prefix=$RISCV --with-isa=rv64gv --with-abi=lp64d --with-cmodel=medany
    $ make

    # install libgloss-htif
    $ cd ..
    $ git clone https://github.com/ucb-bar/libgloss-htif.git
    $ cd ligbloss-htif
    $ mkdir build && cd build
    $ ../configure --prefix=${RISCV}/riscv64-unknown-elf --host=riscv64-unknown-elf
    $ make
    $ make install

For futher reference, pleace refer to the corresponding instructions provided by libgloss-htif and riscv64-gnu-toolchain

#### compiling and running
Assuming assembly code test.s, we use following commands.

    $ riscv64-unknown-elf-gcc -fno-common -fno-builtin-printf -specs=htif_nano.specs -Wl,--defsym=__main=main -c test.s 
    $ riscv64-unknown-elf-gcc -static -specs=htif_nano.specs -Wl,--defsym=__main=main test.o -o test.riscv

And use Spike to execute test.riscv.

    $ spike -d -p4 --isa=rv64gv --varch=vlen:256,elen:32 --gpgpuarch numw:4,numt:8 test.riscv

We add gpgpuarch option to configure uArch parameters for Ventus GPGPU (numw to set total warp number and numt to set thread number per warp). Note that there are some constraints about the parameters: 
1. numw must be equal to processor count (-p)
2. numt must be equal to vlen divided by elen
   
For more spike options, use `spike -h`

### interactive mode command enhancement

GPGPU execution is different from conventional CPU with vector extension in two aspect:
1. use simt stack to control divergence and loop
2. multiwarp managment

We implement software models of simt stack and warp scheduling to support above features, and add enhanced interactive mode commands to display simt stack and warp scheduling information.

| command | usage | function |
| ------- | ----- | -------- |
| simt-stack   | simt-stack \<core\>                  | Display simt stack info given hartid |
| warp-barrier | warp-barrier                         | Display global warp barrier info(numw, numt and barrier bit vector) |
| mem-region   | mem-region \<hex addr\> \<hex size\> | Show contents of physical memory given base memory and size |

For more interactive mode commands, use `help` in interactive mode.

## assembly programming introduction

In this section, we will introduce how to program programs with Ventus GPGPU extension that can run in Spike. Since the software stack is not mature, this section may be changed a lot in the future.

### branch and loop assembly code

First, we introduce how to program branch and loop with custom instructions.

Psuedo code for branch is shown as below.
```assembly
# if(condition(for example: vs1 == vs2))
#   ture_branch_code;
# else
#   false_branch_code;

BRANCH:
    vbne        vs1, vs2, FALSE_BRANCH
    true_branch_assembly_code
    join        v0, v0, BRANCH_JOIN
FALSE_BRANCH:
    false_branch_assenbly_code
    join        v0, v0, BRANCH_JOIN
BRANCH_JOIN:
    end_of_the_branch
```

Psuedo code for loop is shown as below.
```assembly
    vblt    v0, v1, LOOP
    join    v0, v0, LOOP_END
LOOP:
    loop_body
    join    v0, v0, LOOP_COND_EVAL
LOOP_COND_EVAL:
    vblt    v0, v1, LOOP
    join    v0, v0, LOOP_END
```

### extra codes to produce executable file

Since we use libgloss-htif to produce machine mode executable, extra codes are needed.

```assembly
main:
    addi    sp,sp,-16
    sd      s0,8(sp)
    addi    s0,sp,16
    j       start_of_your_program

main_end:
    li      a5,0
    mv      a0,a5
    ld      s0,8(sp)
    addi    sp,sp,16
    ret

start_of_your_program:
    ...
    j       main_end
```

In future version, we may develop custom linker scripts to replace libgloss-htif and the extra codes will be unnecessary.

### other conventions

Current version needs the assembly program to obey following conventions to be executed sucessfully in Spike. Some of them may be changed in future version.

1. In Ventus GPGPU, vl is set in hardware. But in this simulator, extra vsetvli instrucions are needed to configure vl and other related parameters.
2. We use .data directive to store data. Unlike in Ventus GPGPU hardware, we don't use gds/lds to get the global/local memory address. Instead we use la instruction to load start address of global/local memory into the register.

### testcases

We provided testcased in path gpgpu-testcase. To run the testcase, change directory to gpgpu-testcase and execute the makefile script.

1. compile the testcase
    $ make TEST=[testcasename]

2. invoke spike to run the testcase
    $ make spike-sim TEST=[testcasename]
    
PS:
- make spike-sim will alse compile the asssembly
- testcasename is the name of testcase, e.g. loop
- spike-sim have other parameters that can be configured via commandline
  - P: processor count, defalut value is 4
  - VARCH: rvv uArch parameter, default value is vlen:256,elen:32
  - DEBUG: enable interactive mode, default value is -d
  - ISA：supported isa, default value is rv64gv

We provide the following testcases
| name | function |
| ---- | -------- |
| branch-basic | test for simple branch |
| branch-nested | test for nested branch |
| loop | test for loop |
| barrier | test for inter warp barrier |
| gaussian | gaussian testcase |
| saxpy | saxpy testcase implemented with extended gpgpu instructions |
| saxpy2 | saxpy testcase implemented with conventional rvv instructions |
