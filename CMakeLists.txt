cmake_minimum_required(VERSION 3.20)

project(miniKV LANGUAGES C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(minikv
    src/main.c
)

target_include_directories(minikv PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
