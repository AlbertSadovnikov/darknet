cmake_minimum_required(VERSION 3.9)
project(darknet)

set(CMAKE_CXX_STANDARD 11)

option(BUILD_STATIC_LIB "Build the static library" ON)
option(BUILD_SHARED_LIB "Build the shared library" ON)
option(BUILD_EXECUTABLE "Build darknet executable" ON)
option(WITH_CUDA "with cuda" ON)
option(WITH_CUDNN "with cudnn" ON)
option(WITH_OPENCV "with opencv" ON)
option(WITH_OPENMP "with openmp" ON)

if(WITH_CUDA)
    find_package(CUDA REQUIRED)
    add_definitions(-DGPU)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -w -Xcompiler -fPIC -gencode arch=compute_61,code=sm_61" )
    include_directories(${CUDA_INCLUDE_DIRS})
    if(WITH_CUDNN)
        include(${CMAKE_CURRENT_LIST_DIR}/cmake/FindCuDNN.cmake)
        if(NOT CUDNN_FOUND)
            message(FATAL_ERROR "Could not find CuDNN")
        endif()
        add_definitions(-DCUDNN)
    endif()
endif()

if(WITH_OPENCV)
    find_package(OpenCV REQUIRED)
    add_definitions(-DOPENCV)
endif()

if(WITH_OPENMP)
    find_package(OpenMP)
    if (OPENMP_FOUND)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    endif()
endif()


find_library(MLib m REQUIRED)
find_package(Threads REQUIRED)

include_directories(include)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
add_definitions(-Ofast)

file(GLOB DN_LIBRARY_SOURCES "src/*.c")

add_library(objdarknet OBJECT ${DN_LIBRARY_SOURCES})


if(CUDA_FOUND)
    file(GLOB DN_CUDA_LIBRARY_SOURCES "src/*.cu")
    CUDA_COMPILE(CUDA_OBJ ${DN_CUDA_LIBRARY_SOURCES})
endif()

if(BUILD_SHARED_LIB)
    add_library(darknet_shared SHARED $<TARGET_OBJECTS:objdarknet>)
    set_target_properties(darknet_shared PROPERTIES OUTPUT_NAME darknet CLEAN_DIRECT_OUTPUT ON)
endif()

if(BUILD_STATIC_LIB)
    add_library(darknet_static STATIC $<TARGET_OBJECTS:objdarknet>)
    set_target_properties(darknet_static PROPERTIES OUTPUT_NAME darknet CLEAN_DIRECT_OUTPUT ON)
endif()

if(BUILD_EXECUTABLE)
    file(GLOB DN_EXAMPLE_SOURCES "examples/*.c")
    add_executable(darknet ${OpenCV_INCLUDE_DIRS} ${DN_EXAMPLE_SOURCES} $<TARGET_OBJECTS:objdarknet> ${CUDA_OBJ})
    target_link_libraries(darknet
            ${MLib}
            ${CMAKE_THREAD_LIBS_INIT}
            ${OpenCV_LIBS}
            ${CUDA_LIBRARIES}
            -lstdc++
            ${CUDNN_LIBRARY}
            ${CUDA_CUBLAS_LIBRARIES}
            ${CUDA_curand_LIBRARY})
endif()


#include_directories(include)
#
#
#file(GLOB DN_EXEC_SOURCES "examples/*.c")
#file(GLOB DN_CUDA_SOURCES "src/*.cu")
#set(SOURCE_FILES ${DN_LIBRARY_SOURCES} ${DN_EXEC_SOURCES})
#
#add_executable(darknet ${SOURCE_FILES})
#target_link_libraries(darknet ${M_LIB} ${CMAKE_THREAD_LIBS_INIT})