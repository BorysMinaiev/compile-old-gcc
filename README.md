# compile-old-gcc
A set of hacks to compile old gcc on Ubuntu 22.04

Once I tried to git bisect gcc to find a bug in the compiler. If you try to checkout a random old commit from [gcc repository](https://github.com/gcc-mirror/gcc) and build it, you will probably face some issues because not all dependencies are backward compatible. The solution for each separate issue can probably be googled, but I wanted to collect my experience in one place. 

## Getting started
[Official installation guide](https://gcc.gnu.org/install/) is quite good. Basically, you need to:
1. Download the source
2. `../gcc/configure ...`
3. `make ...`
4. `make install`

## `configure`
I ended up using a command like this:
```
CC=.../gcc-7.5.0/usr/local/bin/gcc CXX=.../gcc-7.5.0/usr/local/bin/g++ ../gcc/configure --disable-multilib --disable-werror --disable-bootstrap --enable-languages=c,c++ --disable-libsanitizer --without-isl
```
Let's discuss each part.
```
CC=.../gcc-7.5.0/usr/local/bin/gcc CXX=.../gcc-7.5.0/usr/local/bin/g++
```
By default, your local gcc (in my case 11.2.0) is used to build the code, but newer versions of gcc could be too strict and forbid some patterns used in the old code. So I first compiled gcc-7.5.0, and later used it to compile older versions.

```
--disable-multilib
```
Without this option, it tries to compile for different platforms and probably wants to use libraries you don't have. 

```
--disable-werror
```
Without this option, it will complain about some warnings added to new gcc versions.

```
--disable-bootstrap
```
By default, it compiles code with your local gcc, and after that compiles again from scratch with just created gcc binary, and checks that results are equal. Disabling bootstrap makes it much faster. And also without it, I got some errors about the fact that generated binaries are actually different, but I didn't care.

```
--enable-languages=c,c++
```
By default it also compiles some other languages, which could trigger more strange errors.

```
--disable-libsanitizer --without-isl
```
Disable some parts of gcc, which triggers some compilation errors, but was not needed by me.

### Errors
If you see error `Building GCC requires GMP 4.2+, MPFR 2.3.1+ and MPC 0.8.0+...` as per [stack overflow](https://stackoverflow.com/questions/9253695/building-gcc-requires-gmp-4-2-mpfr-2-3-1-and-mpc-0-8-0), you need to run `./contrib/download_prerequisites`.

## make
The official installation guide says to run `make bootstrap`, but when you use `--disable-bootstrap` there will be no such target. Just `make` everything. Also to run make in parallel, use `make -j4` (or whatever number of cpu you have).

At this stage depending on which version of gcc you try to compile, you will see different errors. In most cases, you can apply some patch, which fixes the error. Let's see examples of errors.

#### `gcc: error: gengtype-lex.c: No such file or directory`
```
apt-get install flex
# clean directory, run configure again
```

#### `lex.cc: error: 'loc' may be used uninitialized`
add `--disable-werror` during configure

#### `./md-unwind-support.h:65:47: error: dereferencing pointer to incomplete type ‘struct ucontext’`
`git apply 3.patch`

#### `reload1.c: error: use of an operand of type 'bool' in 'operator++' is forbidden in C++17
provide older CC and CXX during `configure`.

#### `error: ‘const char* libc_name_p(const char*, unsigned int)’ redeclared inline with ‘gnu_inline’`
depending on the version of your code need to `git apply 5-2014.patch` or `git apply 5-2015.patch`. 

## make install
After compiling to get real gcc binary, you can run `make DESTDIR=/path/to/some/dir install`.

gcc will be installed into `/path/to/some/dir/usr/local/bin/gcc`


