# ToolFrameworkCore

## Build

```
cmake -B build
make -j -C build
```

CMakeLists.txt contains function to autodetect ZMQ and BOOST libaries if available on the host.

## Build options for CMake

```-DZMQ_ROOT=/path/zeromq```: set ZMQ library path

```-DBOOST_ROOT=/path/boost```: set BOOT library path

```-DTF_BUILD_SHARED=ON```: build shared libraries (default)

```-DTF_BUILD_STATIC=ON```: build static libraries (default)

## Cross build

```
cmake -B build-arm -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-arm-linux-gnueabihf.cmake 
```
