cmake_minimum_required(VERSION 3.20)
project(DroneRecon VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    add_compile_options(/W3 /EHsc)
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static -static-libgcc -static-libstdc++")
endif()

add_executable(DroneRecon WIN32 src/main.cpp src/gui.cpp src/pipeline.cpp)
target_link_libraries(DroneRecon PRIVATE comctl32 comdlg32 shell32 ole32)
