# 一、环境准备
## 1.3 CMake安装
- 官方下载网址[在此](https://cmake.org/download/)，若下载免安装的`.zip`版本则可直接解压到某处，然后配置环境变量即可
![配置MinGW和CMake环境变量.png](配置MinGW和CMake环境变量.png)
## 1.4 插件安装
- C/C++：提供代码的提示、调试与浏览
- CMake：用于借助`CMakeLists.txt`文件生成`Makefile`，后者可以被Make工具用于编译
- CMake Tools：用于辅助提示编写`CMakeLists.txt`文件
## 1.5 注意事项
### 1.5.1 确保项目路径用英文
- 我将Github上的一个C++项目下载到本地桌面（路径含有`桌面`这个中文字段）上运行的时候，发现居然编译失败，而我在另一个纯英文路径下的完全相同的项目却能够正常运行，我郁闷了半天，然后我将桌面的名称改为英文后，问题就解决了，如下图
![桌面改名.png](桌面改名.png)
- 所以若在编译代码的过程中出现了编译失败的问题，并要求你添加`launch.json`文件时，不妨先检查一下你的代码文件夹所在的路径是否存在中文，问题很有可能就产生于此
### 1.5.2 代码中的文件路径
- 写代码时我发现，对于同一个路径名，使用CMake编译和使用CompileRun插件编译的效果不同，参考如下项目结构
```
- Root/
	- Codes/
		- 01-XXX/
		- 02-XXX/
		- 03-XXX/
			- TestCase.csv
			- Loader.cpp
		- Main.cpp
	- Libs/...
	- Others/...
```
- 我在`Loader.cpp`内实现加载`Test.csv`内数据的函数如下，`Loader`类的单例对象会在`Main.cpp`内被创建，函数`LoadTestCase`也会在`Main.cpp`内被调用
```cpp
bool Loader::LoadTestCase(std::string _path)
{
	//加载路径下的文件
	std::ifstream _file(_path);
	//检查是否成功加载
	if (!_file.good())
		return false;

	//...

	//关掉文件
	_file.close();
	return true;
}
```
- 当我传入路径`../Codes/03-XXX/TestCase.csv`时，使用CMake编译和CompileRun都能正常加载目标文件，而当我使用路径`03-XXX/TestCase.csv`时，就只有CompileRun能正常加载
- 这可能是由于二者的`cwd`设置存在差异所导致？没搞懂
# 二、代码编译
## 2.1 单文件编译
- 写好一个简单的单文件程序后
```cpp
#include <iostream>
int main()
{
    int a = 11; int b = 22;
    std::cout << "a + b = " << a + b << "\n";
}
```
- 我们新建一个终端，然后我们就可以在终端内使用相关Linux命令来进行代码文件的编译了
![单文件编译之调出终端.png](单文件编译之调出终端.png)
- 使用如下指令即可生成该文件的`.exe`可执行文件，注意C++和C的源文件对应的指令的区别
```
g++ file_name.cpp
gcc file_name.c
```
![单文件调试生成可执行文件.png](单文件调试生成可执行文件.png)
- 我们也可以用`-o`来指定生成的可执行文件的文件名，若不指定则默认名称为`a.exe`
```
g++ file_name.cpp -o yourname
gcc file_name.c -o yourname
```
- 然后我们即可运行已有的可执行文件
![单文件调试运行可执行文件.png](单文件调试运行可执行文件.png)
## 2.2 多文件编译
### 2.2.1 基于命令
- 用于测试的多文件结构如下
```
- include/
	- Header.h
- src/
	- Source1.cpp
	- Source2.cpp
- Test.cpp
```
- 其内内容如下
```cpp
/* Header.h */
#ifndef _HEADER_H_
#define _HEADER_H_
void Function1();
void Function2();
#endif

/* Source1.cpp */
#include <iostream>
void Function1()
{
    std::cout << "Func1" << "\n";
}

/* Source2.cpp */
#include <iostream>
void Function2()
{
    std::cout << "Func2" << "\n";
}

/* Test.cpp */
#include "Header.h"
int main()
{
    Function1();
    Function2();
}
```
- 我们通过以下命令来链接上述四个文件并生成可执行文件`Main.exe`
```
g++ .\src\Source1.cpp .\src\Source2.cpp .\Test.cpp -o Main -I .\include\
```
- 其中`-o`用于指定可执行程序的文件名，`-I`则用于指定`.cpp`源文件中使用`#include`包含的头文件应当在何处（注意到我们包含头文件时并没有使用相对路径）进行检索，不填则默认在文件根目录`.\`进行检索
![多文件链接编译运行可执行文件结果展示.png](多文件链接编译运行可执行文件结果展示.png)
### 2.2.2 基于CMake
- 我们在文件根目录添加一个`CMakeLists.txt`文件，我们可以在里面写一些函数，具体的编写规则详见我的CMake相关笔记
```cmake
# 必须放在CMakeLists.txt文件的开头，用于指定所需的CMake的最低版本，此命令只能执行一次
cmake_minimum_required(VERSION 3.10)
# 该函数必须紧随cmake_minimum_required()函数之后，用于指定项目名称、版本、描述等信息
project(ProjectName)

# 该函数第一个参数接收地址，第二个参数（可自定名称）用于搜索地址后记录该地址下的.cpp源文件
aux_source_directory(src SrcSub) # 搜索根目录下的src文件夹内的源文件
aux_source_directory(. SrcCur)   # 搜索根目录下的源文件

# 该函数用于生成可执行文件，第一个参数指定文件名，往后的参数以空格间隔，传入先前导出的那几个变量
add_executable(Main ${SrcSub} ${SrcCur})

# 该函数用于指定头文件的搜索路径，可以有多个路径，此处是指在根目录下的include文件夹内进行搜索
include_directories(include)
```
- 然后在VSCode内按下`Ctrl + Shift + P`快捷键，如下图进行编译器的选择（别选VS的那些）
![CMakeConfigure编译器选择.png](CMakeConfigure编译器选择.png)
- 选择完成后就会在根目录自动生成（之后每修改该文件后进行保存时，都会重新生成）`build`文件夹，然后我们就可以依据其中`Makefile`文件内的规则进行代码的Linking与可执行文件的生成
![Makefile的生成.png](Makefile的生成.png)
- 然后我们就可以打开终端，`cd`到`build`所在的目录，然后使用指令`cmake ..`（若报错无法识别指令`cmake`，则卸载插件CMake和CMake Tools并重装后重启VSCode即可）
![cmake指令调用Makefile.png](cmake指令调用Makefile.png)
- 上述步骤都执行成功后，我们即可在命令行中使用`mingw32-make.exe`指令（该工具可在MinGW的文件夹目录下的`bin`目录下找到，名字有可能有所不同），这就可以使得项目依据`Makefile`文件内的规则进行编译
![使用Make生成exe可执行文件.png](使用Make生成exe可执行文件.png)
- 至此我们即可使用该`.exe`文件了，若我们需要对代码进行调试，则需要配置`tasks.json`和`launch.json`文件，若无需调试则可不管，每次直接`Ctrl + F5`生成可执行文件即可