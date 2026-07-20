# cmake version 3.23 or higher is needed to support
# the argument CUDA_ARCHITECTURES all-major
cmake_minimum_required(VERSION 3.23 FATAL_ERROR)

project(leapct LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find a default GPU accelerator language.
set(_default_accelerator_type "None")
include(CheckLanguage)
check_language(HIP)
if(CMAKE_HIP_COMPILER)
    set(_default_accelerator_type "AMD")
endif()
check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
    set(_default_accelerator_type "NVIDIA")
endif()

set(LEAP_GPU "$_default_accelerator_type" CACHE STRING "GPU acceleration type")
unset(_default_accelerator_type)

set_property(CACHE LEAP_GPU PROPERTY STRINGS "NVIDIA" "AMD" "None")
# Extract "STRINGS" property of the parameter
get_property(OPT_STRINGS CACHE LEAP_GPU PROPERTY STRINGS)
# Check that value of the parameter is inside "STRINGS" list.
if (NOT LEAP_GPU IN_LIST OPT_STRINGS)
  message(FATAL_ERROR "Wrong value of the parameter 'LEAP_GPU': ${LEAP_GPU}")
endif ()
message(STATUS "LEAP_GPU selected accelerator type ${LEAP_GPU}")

string(TOUPPER "${LEAP_GPU}" LEAP_GPU)
if (LEAP_GPU STREQUAL "NVIDIA")
    # To minimize binary size and compile time, users are suggested to
    # set the CUDAARCHS environment variable or define the CMake variable CMAKE_CUDA_ARCHITECTURES
    # to "native" if building specifically for the GPU visible to the build environment,
    # or to an explicit numerical architecture code, such as "70"
    if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
      set(CMAKE_CUDA_ARCHITECTURES all-major)
    endif()
    enable_language(CUDA)
    set(LEAP_CUDA ON)
elseif (LEAP_GPU STREQUAL "AMD")
    check_language(HIP)
    if(NOT CMAKE_HIP_COMPILER)
        message(FATAL_ERROR "LEAP_GPU=AMD requires a HIP compiler")
    endif()
    # Users are advised to set CMAKE_HIP_ARCHITECTURES to a minimal subset.
    # See also https://rocm.docs.amd.com/en/latest/conceptual/cmake-packages.html#using-hip-in-cmake
    enable_language(HIP)
    set(LEAP_HIP ON)
    set(LEAP_ORIGINAL_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")
    set(LEAP_HIPIFIED_SRC_DIR "${CMAKE_CURRENT_BINARY_DIR}/hipified_src")
    include(cmake/LeapHipify.cmake)
    leap_hipify_sources()
elseif (LEAP_GPU STREQUAL "NONE")
    set(LEAP_CPU_ONLY ON)
else ()
    message(FATAL_ERROR "CMake scripting error: ${LEAP_GPU} didn't match.")
endif ()

if((BUILD_TESTING) OR (NOT DEFINED BUILD_TESTING))
    ENABLE_TESTING()
endif()

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
 
add_subdirectory(src)
