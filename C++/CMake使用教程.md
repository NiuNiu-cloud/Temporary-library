# CMake构建实例
## 一.项目结构
my_project
    build               //
    inlcude             //头文件.h
        .h
    src                 //源文件
        main.cpp
    CMakeLists.txt

# 二.构建步骤

1. 创建工程文件夹
2. 在工程文件夹中按照目录，创建build、src、include文件夹。
    - build : 存放编译中间文件、配置文件和最终编译产物
    - src : 源文件（.cpp）
    - include : 头文件(.h)
3. 创建CMakeLists.txt
4. 编写你的代码
5. 在build目录下，执行命令行操作
    1. cmake ..
    2. make
    2. ./可执行文件名称

## ==CMakeLists.txt==

```C++
cmake_minimum_required(VERSION 3.20)   
# 指定最低 CMake 版本

project(MyProject VERSION 1.0)          
# 定义项目名称和版本

# 设置 C++ 标准为 C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 添加头文件搜索路径
include_directories(${PROJECT_SOURCE_DIR}/include)

# 添加源文件
add_library(MyLib src/mylib.cpp)        
# 创建一个库目标 MyLib
add_executable(MyExecutable src/main.cpp)  
# 创建一个可执行文件目标 MyExecutable

# 链接库到可执行文件
target_link_libraries(MyExecutable MyLib)
```

> 注意：请按照顺序。
