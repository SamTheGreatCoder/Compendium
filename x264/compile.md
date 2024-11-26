# Compile x264 with microarchitecture specific optimizations
*Tested on a x86_64 host, specifically AMD Zen 3, running Debian. You will more then likely need to adjust some options for your specific platform, whether it be architecture and/or microarchitectural differences.*

Get source code first (stable branch is chosen here rather then building a specific tag/release version):
```
git clone https://code.videolan.org/videolan/x264.git -b stable --depth=1
```
Configure x264 makefiles based on configuration options we want:
```
./configure --enable-static --enable-pic --enable-lto --extra-cflags="-flto -O3 -march=native" --extra-ldflags="-flto -O3 -march=native -static"
```
```--enable-static``` to build static libraries (personal choice, will elaborate later)

```--enable-pic``` for position independent executable (forget why, but will figure out why later)

```--enable-lto``` for "Link time optimization", and is part of making this custom build perform better for the specific system we're targetting

```--extra-cflags``` is for extra stuff to pass to the C compiler. (gcc in this case?) Link time optimization needs to be manually defined for the compiler too (I believe? Never hurts to define it as well, assuming the codebase supports the optimizations you're trying to apply, which in this instance is full level 3 link time optimization, and applying microarchitecture compiler optimizations for the system we're compiling it on, ```-march=native```

```--extra-ldflags``` is for the linker, responsible for putting it all together once everything is compiled. We once again tell it that we are doing full level 3 lto, targetting the microarchitecture the compiler is running on, and as stated before, to compile the application library statically.

## NOTE: At the moment I do not have any performance metrics to prove that compiling x264 with micro/architecture specific optimizations will improve or hinder x264's performance, over "generic" target compiles. I also have not done any tests to ensure these optimizations do not impact the conformance of the encoder. As with anything you find on the internet, follow at your own risk, and do your own research (this would show that these sort of optimizations provide decent speed-ups with the AOM-AV1 encoder, in other environments/situations.)