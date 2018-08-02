# Build wrapper for netlib-lapack

Ref: [GitHub project](https://github.com:KineticTheory/lapack-visualstudio-mingw-gfortran)

This build system wrapper for [netlib-lapack](https://github.com/lapack/Reference-lapack)
provides a simple way to build the lapack and blas libraries under Visual Studio 
so they can be used in Visual Studio projects.

Lapack sources are not included in this project.  The build will download 
lapack sources automatically or the tar.gz sources can be provided in the build 
directory.

# TL;DR

```
# Visual Studio command prompt
git clone git@github.com:KineticTheory/lapack-visualstudio-mingw-gfortran
mkdir %build%
cd %build%
cmake -G "Visual Studio 15 2017 Win64" \
      -DCMAKE_INSTALL_PREFIX=c:\thirdparty\lapack-3.8.0
      <path_to>/lapack-visualstudio-mingw-gfortran
cmake --build . --config Release --target install
```

# Assumptions

The following tools are assumed to be installed.

* [Visual Studio installed](https://visualstudio.microsoft.com/vs/community/)
* [MSYS2 MinGW installed]((http://www.msys2.org/)
  * gcc and gfortran installed
  * mingw-make installed
  ```
  start the MSYS2 shell.
  pacman -Syuu #repeat as needed
  pacman -Sy pacman
  pacman -S mingw-w64-x86_64-gcc
  pacman -S mingw-w64-x86_64-gcc-fortran   
  pacman -S mingw-w64-x86_64-make
  ```
* [CMake installed](https://cmake.org/download/)

# Notes

* This project is based on the Kitware blog article 
  https://blog.kitware.com/fortran-for-cc-developers-made-easier-with-cmake/.
* This build uses a modified version of CMake's `CMakeAddFortranSubdirectory`
  that was borrowed from [lanl/Draco](https://github.com/lanl/Draco).
  