# pg^2: Profile Guided Prefetch Guard


## Prerequisites
Please refer instructions from links or directly run commands listed below to install prerequisites: 
- Linux Perf:`sudo apt-get install linux-tools-common linux-tools-generic` 
- libunwind: [https://github.com/libunwind/libunwind](https://github.com/libunwind/libunwind) 
- Rust: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` 
- binutils (for objdump) [^1] : [https://ftp.gnu.org/gnu/binutils/](https://ftp.gnu.org/gnu/binutils/) 
- boost: `sudo apt-get install libboost-all-dev` 
[^1]:If your `objdump` version is older than 2.27, please download the latest version of `binutils`.  


## Build BOLT from source
To use `llvm-bolt` utilities, `BOLT` needs to be installed. \
Please follow the commands below to install `BOLT` 
```bash
> mkdir BOLT && cd BOLT
> git clone git@github.com:upenn-acg/BOLT.git llvm-bolt
> cd llvm-bolt 
> git checkout pg2/bat
> cd ..
> mkdir build && cd build
> cmake -G "Unix Makefiles" ../llvm-bolt/llvm -DLLVM_TARGETS_TO_BUILD="X86;AArch64" -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON -DLLVM_ENABLE_PROJECTS="clang;lld;bolt"
> make -j
```

## Build BAT-dump
```bash
> mkdir llvm-project && cd llvm-project
> git clone git@github.com:llvm/llvm-project.git
> mkdir build && cd llvm
> git checkout 2b88298c2ab221228744ca8dba70c2d3bcff593b
> cd ../build
> cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;compiler-rt;bolt" ../llvm
> make -j
```

## Build llvm-10.0 from source
- download LLVM 10.0 (Source code(tar.gz)) from https://github.com/llvm/llvm-project/releases/tag/llvmorg-10.0.0
- build LLVM (reference: https://llvm.org/docs/GettingStarted.html)
```bash
> cd {llvm10.0 root directory}
> mkdir build && cd build
> cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;compiler-rt" ../llvm
> make -j
```

## Build & run pg^2
- Download pg^2
```bash
> git clone git@github.com:upenn-acg/floar.git
> cd floar/pg2_reloc
> export PG2_R_PATH=/your/absolute/path/to/pg2_reloc/directory
> make
```
- `make` will produce 1 executables (`tracer`)+ 1 shared library (`inject_prefetch.so`). 
   * If libunwind library is stored in other places instead of `/usr/local/lib`, you also need to edit Makefile and update it to the corresponding path.
   * If libunwind's header files are stored in other places instead of `/usr/local/include`, you also need to edit Makefile and update it to the corresponding path. 
- In the file `config`, specify the absolute path for `nm`,`perf`,`objdump`,`llvm-bolt`,`perf2bolt` and `bat-dump` [^3]
   [^3]: if `nm`,`objdump` and `perf` are already in shell, it's OK that their paths are not specified in `config`. This can be checked by `which nm`, `which objdump` and `which perf`.
- Compile workloads
- (1) compile CRONO workloads with `kill(getppid(), SIGUSR1);` To run pg2, you have to compile CRONO workloads with `kill(getppid(), SIGUSR1)`.
```bash
> cd workloads/CRONO/pagerank_lock
> make
> cd ../../..
```
- (2) compile CRONO workloads without `kill(getppid(), SIGUSR1);` \
To measure the real execution time of CRONO workloads, you have to compile CRONO workloads without `kill(getppid(), SIGUSR1);`\
In `pg2_reloc/workloads/CRONO/Makefile`, delete the `-DSIGNAL`, then run `make` 

- (3) compile AinsworthCGO2017 workloads
```bash
> cd workloads/AinsworthCGO2017/nas-is
> sh compile_x86.sh
> cd ../../..
```
- In `config`, please also specify the commands to run the workload. The example commands are given in the config file. 
   * _Note_: the first argument of the command (a.k.a. the binary being invoked in the command) should be written in its full path. 

- Run pg^2:
```bash
> ./extract_call_sites
> ./tracer
```

## Miscellaneous (notes about how to debug pg^2)
In `Makefile`'s `CXXFLAGS`,
- if `-DTIME_MEASUREMENT` flag is added, pg^2 will print the execution time of code replacement;
- if `-DMEASUREMENT` flag is added, pg^2 will print metrics such as:  
  * the number of functions on the call stack when target process is paused,
  * the number of functions that are moved by `BOLT`,
  * the number of functions that are in the `BOLT` and original functions.
- if `-DDEBUG_INFO` flag is added, pg^2 will print debug information such as:
  * the information about detailed behavior of tracer
  * the content in the call stack when the target process is paused
  * `-DDEBUG_INFO` can also be defined in `src/inject_prefetch.hpp`. In this way, the ld_preload library will store all machine code per function it inserted to the target process as a `uint8_t` format array into a file. The file can be found in the `tmp_data_path` you defined in the config file. 
- if `-DDEBUG` flag is added, after code replacement, pg^2 will first send `SIGSTOP` signal to target process and then resume the target process by `PTRACE_DETACH`. In this way, it allows debugging tools such as `GDB` to attach to the target process and observe what goes wrong after code replacement.


