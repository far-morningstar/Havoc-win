# Havoc-win

## 1.简介

项目原始地址：[Havoc](https://github.com/HavocFramework/Havoc)

这是一个用于进行渗透测试的框架，将 Client 移植到 Windows 系统上。

## 2、Client 移植到 Windows 系统

### 2.1、在 Windows 上搭建编译环境
Client 是用 Qt5 编写的，工程用 cmake 进行构建，没有使用 linux 系统原生的系统特性，可以进行移植。

编译需求：
+ python 3.10
+ c++20
+ Qt 5
+ spdlog

这里使用 msys2 在Windows 构建编译环境：
```shell
# 安装 python3，这里直接就是当前比较新的版本
pacman -S python
pacman -S python-devel

# 安装 Qt5 静态库
pacman -S mingw-w64-x86_64-qt-creator mingw-w64-x86_64-qt5-static

# 安装 spdlog
pacman -S mingw-w64-x86_64-spdlog

# 安装 cmake，这里不能直接安装 gcc 通用的 cmake，应该安装特定于 msys2 的 cmakemingw-w64-x86_64-cmake
# https://discourse.cmake.org/t/system-is-unknown-to-cmake-create-platform-mingw64-nt-10-0-19044/4734
pacman -S --needed mingw-w64-x86_64-cmake

# install GCC and common build tools
pacman -S --needed base-devel
pacman -S --needed mingw-w64-x86_64-toolchain
pacman -S --needed git subversion mercurial
pacman -S mingw-w64-x86_64-nasm
pacman -S mingw-w64-x86_64-lld
pacman -S mingw-w64-x86_64-pkg-config
pacman -S mingw-w64-x86_64-python3-pkgconfig
pacman -S autoconf
pacman -S automake
```

### 2.2、error: Python.h: No such file or directory

python-devel 已经安装：

> pacman -S python-devel

搜索 Python.h 也能找到：

> $ find / -name "Python.h"
> /mingw64/include/python3.10/Python.h
> /usr/include/python3.10/Python.h

修改 CMakeLists.txt 的 line40-line49，添加 Python.h 文件路径（添加上面搜索到的两个路径无效，需要设置绝对路径）：

```cmake
if(APPLE)

execute_process(COMMAND brew --prefix OUTPUT_VARIABLE BREW_PREFIX) #this because brew install location differs Intel/Apple Silicon macs

string(STRIP ${BREW_PREFIX} BREW_PREFIX) #for some reason this happens: https://gitlab.kitware.com/cmake/cmake/-/issues/22404

include_directories( "${BREW_PREFIX}/bin/python3.10" )

include_directories( "${BREW_PREFIX}/Frameworks/Python.framework/Headers" )

elseif(UNIX)

include_directories( ${PYTHON_INCLUDE_DIRS} )

else()

include_directories( "C:/msys64/mingw64/include/python3.10/" )

endif()
```

### 2.3、error: filesystem

```shell
error: no match for 'operator=' (operand types are 'std::__cxx11::basic_string<char>' and 'std::filesystem::__cxx11::path')
```

C++17 中增加了对文件系统的支持。在这之前，C++ 程序员只能使用 POSIX 接口或者 WindowsAPI 去做一些目录操作。

+ 编译器版本要求：gcc>=8，clang>=7，MSVC>=19.14
+ 头文件为：#include "filesystem"，命名空间为：std::filesystem
+ 官方文档：https://en.cppreference.com/w/cpp/filesystem

这里 gcc 的版本是 12.2.0，完全满足要求：

> gcc -v
> gcc version 12.2.0 (Rev10, Built by MSYS2 project)

这里作以下修改（不建议使用 win32 API，Havoc 定义了自己的 BYTE 类型，跟win32冲突）：

```c
// 添加头文件
#ifdef _WIN32
#include <direct.h>
#endif

// line-1841
if ( ! Command.Path.empty() )
{
spdlog::debug( "Set current path to {}", Command.Path );
#ifdef _WIN32
char cur_path[256]={0};
getcwd(cur_path,256);
Path = cur_path;
chdir(Command.Path.c_str());
#else
Path = std::filesystem::current_path();
std::filesystem::current_path( Command.Path );
#endif
}

// line-1939
if ( ! Command.Path.empty() )
{
spdlog::debug( "Set current path to {}", Command.Path );
#ifdef _WIN32
char cur_path[256]={0};
getcwd(cur_path,256);
Path = cur_path;
chdir(Command.Path.c_str());
#else
Path = std::filesystem::current_path();
std::filesystem::current_path( Command.Path );
#endif
}

// line-2182
if ( ! Command.Path.empty() )
{
spdlog::debug( "Set current path to {}", Command.Path );
#ifdef _WIN32
char cur_path[256]={0};
getcwd(cur_path,256);
Path = cur_path;
chdir(Command.Path.c_str());
#else
Path = std::filesystem::current_path();
std::filesystem::current_path( Command.Path );
#endif
}
```

### 2.4、link error: undefined

```shell
undefined reference to `PyTuple_New'
```

python 相关的函数出现未定义，没有找到 Python_LIBRARIES，CMakeLists.txt line32 作以下修改：

```cmake
if(APPLE)
find_package(Python 3 COMPONENTS Interpreter Development REQUIRED)
set(PYTHON_MAJOR $ENV{Python_VERSION_MAJOR})
set(PYTHON_MINOR $ENV{Python_VERSION_MINOR})
set(PYTHONLIBS_VERSION_STRING ${Python_VERSION})
set(PYTHON_INCLUDE_DIR ${Python_INCLUDE_DIRS})
set(PYTHON_LIBRARIES ${Python_LIBRARIES})
message("Apple - Using Python:${Python_VERSION_MAJOR} - Libraries:${PYTHON_LIBRARIES} - IncludeDirs: ${PYTHON_INCLUDE_DIR}")
elseif(UNIX)
find_package(PythonLibs 3 REQUIRED)
else()
find_package(PythonLibs 3 REQUIRED)
set(PYTHONLIBS_VERSION_STRING $ENV{PY_VERSION})
endif()
```

### 2.5、run error: Python

```shell
Fatal Python error: init_fs_encoding: failed to get the Python codec of the filesystem encoding
Python runtime state: core initialized
ModuleNotFoundError: No module named 'encodings'
Current thread 0x00005280 (most recent call first):
<no Python frame>
```

有2个方法进行修改。

#### 方法1、设置 python 的路径

修改 Client/Source/UserInterface/HavocUI.cpp line191：

```c
// Init Python Interpreter
{
// set python home path and python lib path
+ Py_SetPythonHome(L"C:\\msys64\\mingw64\\bin");
+ Py_SetPath(L"C:\\msys64\\mingw64\\lib\\python3.10");

PyImport_AppendInittab( "emb", emb::PyInit_emb );
PyImport_AppendInittab( "havocui", PythonAPI::HavocUI::PyInit_HavocUI );
PyImport_AppendInittab( "havoc", PythonAPI::Havoc::PyInit_Havoc );
Py_Initialize();
PyImport_ImportModule( "emb" );

for ( auto& ScriptPath : dbManager->GetScripts() )
{
Widgets::ScriptManager::AddScript( ScriptPath );
}
}
```

#### 方法2、在系统环境变量中添加变量

```
PYTHONHOME C:\Program Files\Python38
PYTHONPATH C:\Program Files\Python38\Lib
```

![](./images/367455316230269.png)  
虽然要求 python3.10，经测试，在 python3.8 中也能运行。

![](./images/304255715230268.png)

### 2.7、程序打包发布

```
make

ldd.exe ./Havoc.exe | grep mingw64 |awk -F\> '{print $2}' | awk -F' ' '{print $1}' | xargs -I {} cp {} ./

# Qt 静态链接，这里不再需要打包 Qt 依赖
# windeployqt ./Havoc.exe
```
