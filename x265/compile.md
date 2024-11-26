# Compile x265 with multibit (multiple bit depth) and microarchitecture specific optimizations
*Tested on a x86_64 host, specifically AMD Zen 3, running Debian. You will more then likely need to adjust some options for your specific platform, whether it be architecture and/or microarchitectural differences.*

By default when building x265, it seems you only get to compile support for a single bit depth, whether it be 8, 10 or 12 bit. This is inconvenient for multiple reasons, all varying between the target use case for the program.

While the x265 source repository has had a linux build script to build a single binary with all three bit depths, it doesn't statically link the libraries or perform any micro/architecture specific optimizations.

Reference: https://www.reddit.com/r/ffmpeg/comments/7zjoi3/compiling_ffmpeg_on_ubuntu_with_multilib_x265/

Per user ```Consistent-Effort893```'s comment on the reddit post, it seems this might be the origin of this solution, and it's seemingly eventual adoption into the codebase.

Get source code first (at the time of editorial revision, ```Release_4.1``` is the latest):
```
git clone https://bitbucket.org/multicoreware/x265_git.git -b Release_4.1 --depth=1
```
Now to individually compile 12, 10, and then 8 bit support for x265, linking it together into one binary (in that order)
```
cd linux
mkdir -p 10bit 12bit
cd 12bit
cmake ../../../source -DHIGH_BIT_DEPTH=ON -DEXPORT_C_API=OFF -DENABLE_SHARED=OFF -DENABLE_CLI=OFF -DMAIN12=ON -DCMAKE_C_FLAGS="-flto -O8 -march=native" -DCMAKE_CXX_FLAGS="-flto -O8 -march=native"
make -j${nproc}
cd ../10bit/
cmake ../../../source -DHIGH_BIT_DEPTH=ON -DEXPORT_C_API=OFF -DENABLE_SHARED=OFF -DENABLE_CLI=OFF -DCMAKE_C_FLAGS="-flto -O8 -march=native" -DCMAKE_CXX_FLAGS="-flto -O8 -march=native"
make -j${nproc}
cd ..
ln -sf 10bit/libx265.a libx265_main10.a
ln -sf 12bit/libx265.a libx265_main12.a
cmake ../../source -DEXTRA_LIB="x265_main10.a;x265_main12.a" -DEXTRA_LINK_FLAGS=-L. -DLINKED_10BIT=ON -DLINKED_12BIT=ON -DENABLE_SHARED=OFF -DCMAKE_C_FLAGS="-flto -O8 -march=native" -DCMAKE_CXX_FLAGS="-flto -O8 -march=native"
make -j${nproc}
make install
```
And you're done (future extension note, rather then ```make install```, maybe put it into a package file first for better organization and better cleanup of files, should uninstall be a necesity down the line?)

Running ```x265 --help``` should show ```8bit+10bit+12bit``` in the ```build info``` section, instead of the usual individual ```8bit```/```10bit```/```12bit```.

## NOTE: At the moment I do not have any performance metrics to prove that compiling x265 with micro/architecture specific optimizations will improve or hinder x265's performance, over "generic" target compiles. I also have not done any tests to ensure these optimizations do not impact the conformance of the encoder. As with anything you find on the internet, follow at your own risk, and do your own research (this would show that these sort of optimizations provide decent speed-ups with the AOM-AV1 encoder, in other environments/situations.)