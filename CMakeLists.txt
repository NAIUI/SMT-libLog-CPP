# 设置最低 CMake 版本要求
cmake_minimum_required(VERSION 3.16.3)

# 设置项目名称
project(SMT-LIBLOG)

# 设置编译选项
set(CXX_FLAGS
    -g
    -Wall
    -std=c++11
    )

# 添加头文件
include_directories(${CMAKE_SOURCE_DIR}/include)

# 添加源文件
aux_source_directory(${CMAKE_SOURCE_DIR}/src LOG_SRC)

# 设置可执行文件生成目录
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_SOURCE_DIR}/bin)

# 包含测试子目录
add_subdirectory(${CMAKE_SOURCE_DIR}/tests)