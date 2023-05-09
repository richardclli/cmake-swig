| Linux | macOS | Windows |
|-------|-------|---------|
| [![Status][python_linux_svg]][python_linux_link] | [![Status][python_macos_svg]][python_macos_link] | [![Status][python_windows_svg]][python_windows_link] |

[python_linux_svg]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_linux_python.yml/badge.svg
[python_linux_link]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_linux_python.yml
[python_macos_svg]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_macos_python.yml/badge.svg
[python_macos_link]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_macos_python.yml
[python_windows_svg]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_windows_python.yml/badge.svg
[python_windows_link]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_windows_python.yml


# Python Wrapper Status
* [x] GNU/Linux wrapper
* [x] MacOS wrapper
* [x] Windows wrapper

# Introduction 
To be compliant with [PEP513](https://www.python.org/dev/peps/pep-0513/#the-manylinux1-policy) a python package should embbed all its C++ shared libraries.

Creating a Python native package containing all `.py` and `.so` (with good rpath/loaderpath) is not so easy... 

# Build the Binary Package
To build the Python wheel package, simply run:
```sh
cmake -S. -Bbuild -DBUILD_PYTHON=ON
cmake --build build --target python_package -v
```
note: Since `python_package` is in target `all`, you can also ommit the
`--target` option.

# Technical Notes
## Build directory layout
Since Python use the directory name where `__init__.py` file is located and we
want to use the [CMAKE_BINARY_DIR](https://cmake.org/cmake/help/latest/variable/CMAKE_BINARY_DIR.html) 
to generate the Python binary package.  

We want this layout:
```shell
<CMAKE_BINARY_DIR>/python
├── setup.py
└── CMakeSwig
    ├── __init__.py
    ├── FooBar
    │   ├── __init__.py
    │   ├── _pyFooBar.so
    │   └── pyFooBar.py
    ├── Foo
    │   ├── __init__.py
    │   ├── pyFoo.py
    │   └── _pyFoo.so
    ├── Bar
    │   ├── __init__.py
    │   ├── _pyBar.so
    │   └── pyBar.py
    └── .libs
        ├── libBar.so.1.0
        ├── libFooBar.so.1.0
        └── libFoo.so.1.0
```
src: `tree build --prune -U -P "*.py|*.so*" -I "build"`

note: On UNIX you always need `$ORIGIN/../../${PROJECT_NAME}/.libs` since `_pyFoo.so` will depend on `libFoo.so`.
note: On APPLE you always need `"@loader_path;@loader_path/../../${PROJECT_NAME}/.libs` since `_pyFoo.so` will depend on `libFoo.dylib`.
note: on Windows since we are using static libraries we won't have the `.libs` directory...

So we also need to create few `__init__.py` files to be able to use the build directory to generate the Python package.

## Why on APPLE lib must be .so
Actually, the cpython code responsible for loading native libraries expect `.so`
on all UNIX platforms.

```c
const char *_PyImport_DynLoadFiletab[] = {
#ifdef __CYGWIN__
    ".dll",
#else  /* !__CYGWIN__ */
    "." SOABI ".so",
#ifdef ALT_SOABI
    "." ALT_SOABI ".so",
#endif
    ".abi" PYTHON_ABI_STRING ".so",
    ".so",
#endif  /* __CYGWIN__ */
    NULL,
};
```
ref: https://github.com/python/cpython/blob/master/Python/dynload_shlib.c#L36-L48

i.e. `pyFoo` -> `_pyFoo.so` -> `libFoo.dylib`

## Why setup.py has to be generated
To avoid to put hardcoded path to SWIG `.so/.dylib` generated files,
we could use `$<TARGET_FILE_NAME:tgt>` to retrieve the file (and also deal with Mac/Windows suffix, and target dependencies).  
In order for `setup.py` to use
[cmake generator expression](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html#informational-expressions)
(e.g. $<TARGET_FILE_NAME:_pyFoo>). We need to generate it at build time (e.g. using
[add_custom_command()](https://cmake.org/cmake/help/latest/command/add_custom_command.html)).  
note: This will also add automatically a dependency between the command and the TARGET !

todo(mizux): try to use [`file(GENERATE ...)`](https://cmake.org/cmake/help/latest/command/file.html#generate) instead.

# Testing Python
## Testing using virtualenv

