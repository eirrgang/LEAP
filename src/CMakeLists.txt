find_package(OpenMP REQUIRED COMPONENTS CXX)
if(LEAP_CUDA)
    find_package(CUDAToolkit 11.7 REQUIRED)
elseif(LEAP_HIP)
    find_package(hip REQUIRED)
    find_package(hipfft REQUIRED)
endif()

if (WIN32)
  set(CMAKE_SHARED_LIBRARY_PREFIX "lib")
  set(CMAKE_STATIC_LIBRARY_PREFIX "lib")
endif()

#set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/..)
#set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/..)
#set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/..)

set(HEADER
  ray_tracing/analytic_ray_tracing.h
  ray_tracing/analytic_ray_tracing_gpu.cuh
  projectors/backprojectors_VD.cuh
  denoise/bilateral_filter.cuh
  cpu_utils.h
  cuda_utils.h
  file_io.h
  fbp/filtered_backprojection.h
  geometry/find_center_cpu.h
  geometry/geometric_calibration.cuh
  denoise/guided_filter.cuh
  leap_defines.h
  list_of_tomographic_models.h
  log.h
  denoise/matching_pursuit.cuh
  denoise/noise_filters.cuh
  parameters.h
  ray_tracing/phantom.h
  projectors/projectors.h
  projectors/projectors_Joseph.cuh
  projectors/projectors_Joseph_cpu.h
  projectors/projectors_SF.cuh
  projectors/projectors_SF_cpu.h
  projectors/projectors_Siddon.cuh
  projectors/projectors_Siddon_cpu.h
  projectors/projectors_attenuated.cuh
  projectors/projectors_extendedSF.cuh
  projectors/projectors_symmetric.cuh
  projectors/projectors_symmetric_cpu.h
  fbp/ramp_filter.cuh
  fbp/ramp_filter_cpu.h
  fbp/ray_weighting.cuh
  fbp/ray_weighting_cpu.h
  geometry/rebin.h
  resample.cuh
  resample_cpu.h
  scatter_models.cuh
  projectors/sensitivity.cuh
  projectors/sensitivity_cpu.h
  sinogram_replacement.h
  tomographic_models.h
  tomographic_models_c_interface.h
  denoise/total_variation.cuh
)

set(SRC_CPP
  ray_tracing/analytic_ray_tracing.cpp
  cpu_utils.cpp
  file_io.cpp
  fbp/filtered_backprojection.cpp
  geometry/find_center_cpu.cpp
  list_of_tomographic_models.cpp
  parameters.cpp
  ray_tracing/phantom.cpp
  projectors/projectors.cpp
  projectors/projectors_Joseph_cpu.cpp
  projectors/projectors_SF_cpu.cpp
  projectors/projectors_Siddon_cpu.cpp
  projectors/projectors_symmetric_cpu.cpp
  fbp/ramp_filter_cpu.cpp
  fbp/ray_weighting_cpu.cpp
  geometry/rebin.cpp
  resample_cpu.cpp
  projectors/sensitivity_cpu.cpp
  sinogram_replacement.cpp
  tomographic_models.cpp
  tomographic_models_c_interface.cpp
)

set(SRC_CU
  ray_tracing/analytic_ray_tracing_gpu.cu
  projectors/backprojectors_VD.cu
  denoise/bilateral_filter.cu
  cuda_utils.cu
  geometry/geometric_calibration.cu
  denoise/guided_filter.cu
  denoise/matching_pursuit.cu
  denoise/noise_filters.cu
  projectors/projectors_Joseph.cu
  projectors/projectors_SF.cu
  projectors/projectors_Siddon.cu
  projectors/projectors_attenuated.cu
  projectors/projectors_extendedSF.cu
  projectors/projectors_symmetric.cu
  fbp/ramp_filter.cu
  fbp/ray_weighting.cu
  resample.cu
  scatter_models.cu
  projectors/sensitivity.cu
  denoise/total_variation.cu
)

if(LEAP_HIP)
    list(TRANSFORM SRC_CPP PREPEND "${LEAP_HIPIFIED_SRC_DIR}/")
    list(TRANSFORM HEADER PREPEND "${LEAP_HIPIFIED_SRC_DIR}/")
    list(TRANSFORM SRC_CU PREPEND "${LEAP_HIPIFIED_SRC_DIR}/")
    get_hipified_list("${SRC_CPP}" SRC_CPP)
    get_hipified_list("${HEADER}" HEADER)
    get_hipified_list("${SRC_CU}" SRC_CU)
    set_source_files_properties(${SRC_CPP} ${SRC_CU} PROPERTIES LANGUAGE HIP)
endif()

include_directories(
  ./
  fbp
  projectors
  denoise
  geometry
  ray_tracing
  ${CUDA_INCLUDE_DIRS}
)
if(LEAP_HIP)
    include_directories(${LEAP_HIPIFIED_SRC_DIR})
endif()

#add_library(leapct SHARED main_projector_ctype.cpp parameters.cpp projectors_cpu.cpp projectors.cu projectors_SF.cu)
#target_link_libraries(leapct "${TORCH_LIBRARIES}")
add_library(leapct SHARED
  ${HEADER}
  ${SRC_CPP}
)

if(LEAP_CUDA OR LEAP_HIP)
    target_sources(leapct PRIVATE ${SRC_CU})
endif()

# <<<--- Create filters for subfolders for Visual Studio
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

file(GLOB_RECURSE _source_list
  LIST_DIRECTORIES false
  ${HEADER}
  ${SRC_CPP}
  ${SRC_CU}
)

foreach(_source IN ITEMS ${_source_list})
  get_filename_component(_source_path "${_source}" PATH)
  file(RELATIVE_PATH _source_path_rel "${PROJECT_SOURCE_DIR}" "${_source_path}")
  string(REPLACE "/" "\\" _group_path "${_source_path_rel}")
  source_group("${_group_path}" FILES "${_source}")
endforeach()
# --->>>

if (LEAP_CUDA)
    target_compile_definitions(
        leapct PRIVATE
        __USE_GPU
        __INCLUDE_CUFFT
    )
    target_link_libraries(${PROJECT_NAME}
      OpenMP::OpenMP_CXX
      CUDA::cudart
      CUDA::cublas
      CUDA::cufft
    )
elseif (LEAP_HIP)
    target_compile_definitions(
        leapct PRIVATE
        __USE_GPU
        __USE_NOTEX
        __INCLUDE_CUFFT
    )
    get_filename_component(_ROCM_ROOT "${hipfft_DIR}/../../.." ABSOLUTE)
    target_include_directories(${PROJECT_NAME} PRIVATE
      ${_ROCM_ROOT}/include
      ${_ROCM_ROOT}/include/hipfft
    )
    target_link_libraries(${PROJECT_NAME}
      hip::host
      hip::hipfft
      OpenMP::OpenMP_CXX
    )
else ()
    target_compile_definitions(
        leapct PRIVATE
        __USE_CPU
    )
    target_link_libraries(${PROJECT_NAME}
      OpenMP::OpenMP_CXX
    )
endif()

target_compile_options(leapct PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:
  --use_fast_math
>)

if(SKBUILD)
    install(TARGETS ${PROJECT_NAME}
    DESTINATION ${SKBUILD_PLATLIB_DIR}/leapctype/
    )
else()
    install(TARGETS ${PROJECT_NAME}
      PUBLIC_HEADER DESTINATION include
      RUNTIME DESTINATION bin
      ARCHIVE DESTINATION lib
      LIBRARY DESTINATION lib
    )
endif()
