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
    set(GPU ON)
    add_definitions(-DGPU)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -w -Xcompiler -fPIC -gencode arch=compute_61,code=sm_61")
    include_directories(${CUDA_INCLUDE_DIRS})
    if(WITH_CUDNN)
        include(${CMAKE_CURRENT_LIST_DIR}/cmake/FindCuDNN.cmake)
        if(NOT CUDNN_FOUND)
            message(FATAL_ERROR "Could not find CuDNN")
        endif()
        add_definitions(-DCUDNN)
        set(CUDNN ON)
    endif()
endif()

if(WITH_OPENCV)
    find_package(OpenCV REQUIRED)
    add_definitions(-DOPENCV)
    set(OPENCV ON)
endif()

if(WITH_OPENMP)
    find_package(OpenMP)
    if (OPENMP_FOUND)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
        add_definitions(-DOPENMP)
        set(OPENMP ON)
    endif()
endif()

find_library(MLib m REQUIRED)
find_package(Threads REQUIRED)

include_directories(include)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

file(GLOB DN_LIBRARY_SOURCES "src/*.c")

add_library(objdarknet OBJECT ${DN_LIBRARY_SOURCES})

if(WITH_CUDA)
    file(GLOB DN_CUDA_LIBRARY_SOURCES "src/*.cu")
    CUDA_COMPILE(CUDA_OBJ ${DN_CUDA_LIBRARY_SOURCES})
endif()

if(BUILD_SHARED_LIB)
    add_library(darknet_shared SHARED $<TARGET_OBJECTS:objdarknet> ${CUDA_OBJ})
    target_link_libraries(darknet_shared
            ${MLib}
            ${CMAKE_THREAD_LIBS_INIT}
            ${CUDA_LIBRARIES}
            ${OpenMP_C_LIBRARIES}
            ${CUDNN_LIBRARY}
            ${CUDA_CUBLAS_LIBRARIES}
            ${CUDA_curand_LIBRARY})
    set_target_properties(darknet_shared PROPERTIES OUTPUT_NAME darknet CLEAN_DIRECT_OUTPUT ON)
    set(DARKNET_LIBRARY_SHARED darknet_shared PARENT_SCOPE)
endif()

if(BUILD_STATIC_LIB)
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        if(WITH_OPENCV)
            # fixes missing reference from opencv
            set(OPENCV_MISSING_SOURCE "${CMAKE_CURRENT_LIST_DIR}/fix/cvRound.c")
        endif()
    endif()
    add_library(darknet_static STATIC $<TARGET_OBJECTS:objdarknet> ${CUDA_OBJ} ${OPENCV_MISSING_SOURCE})
    target_link_libraries(darknet_static
            ${MLib}
            ${CMAKE_THREAD_LIBS_INIT}
            ${CUDA_LIBRARIES}
            ${OpenMP_C_LIBRARIES}
            ${CUDNN_LIBRARY}
            ${CUDA_CUBLAS_LIBRARIES}
            ${CUDA_curand_LIBRARY})
    set_target_properties(darknet_static PROPERTIES OUTPUT_NAME darknet CLEAN_DIRECT_OUTPUT ON)
    set_target_properties(darknet_static PROPERTIES PUBLIC_HEADER include/darknet.h)
    install(TARGETS darknet_static
            ARCHIVE DESTINATION lib
            LIBRARY DESTINATION lib
            COMPONENT library
            PUBLIC_HEADER DESTINATION include)
    set(DARKNET_LIBRARY_STATIC darknet_static PARENT_SCOPE)
endif()

if(BUILD_EXECUTABLE)
    file(GLOB DN_EXAMPLE_SOURCES "examples/*.c")
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        if(WITH_OPENCV)
            # fixes missing reference from opencv
            list(APPEND DN_EXAMPLE_SOURCES "${CMAKE_CURRENT_LIST_DIR}/fix/cvRound.c")
            message(${DN_EXAMPLE_SOURCES})
        endif()
    endif()

    add_executable(darknet
            ${OpenCV_INCLUDE_DIRS}
            ${DN_EXAMPLE_SOURCES}
            $<TARGET_OBJECTS:objdarknet>
            ${CUDA_OBJ})
    target_link_libraries(darknet
            ${CMAKE_THREAD_LIBS_INIT}
            ${OpenCV_LIBS}
            ${CUDA_LIBRARIES}
            -lstdc++
            ${CUDNN_LIBRARY}
            ${MLib}
            ${CUDA_CUBLAS_LIBRARIES}
            ${CUDA_curand_LIBRARY})
endif()


configure_file(include/darknet.h.in darknet.h)
set(DARKNET_INCLUDE_DIRS ${CMAKE_CURRENT_BINARY_DIR} ${CUDA_INCLUDE_DIRS} ${OpenCV_INCLUDE_DIRS} PARENT_SCOPE)
mark_as_advanced(DARKNET_INCLUDE_DIRS DARKNET_LIBRARY_STATIC DARKNET_LIBRARY_SHARED)

