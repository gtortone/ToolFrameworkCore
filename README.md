# ToolFrameworkCore

## Build

```
cmake -B build
make -j -C build
```

CMakeLists.txt contains function to autodetect ZMQ and BOOST libaries if available on the host.

## Build options for CMake

```-DBOOST_ROOT=/path/boost```: set BOOT library path

```-DTF_BUILD_SHARED=ON```: build shared libraries (default)

```-DTF_BUILD_STATIC=ON```: build static libraries (default)

## Cross build

```
cmake -B build-arm -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-arm-linux-gnueabihf.cmake 

make -j -C build-arm
```

### Force static linking of main executable

```
cmake -B build-arm -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-arm-linux-gnueabihf.cmake -DTF_BUILD_STATIC=ON -DTF_BUILD_SHARED=OFF

make -j -C build-arm
```

## Output products

- dynamic libraries (*.so) in `<build_dir>/lib`
- static libraries (*.a) in `<build_dir>/lib`
- header files (*.h) in `<build_dir>/include`
- test executable (main) in `<build_dir>`
