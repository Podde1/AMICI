#
# Build AMICI library
#
cmake_minimum_required(VERSION 3.15)

project(amici)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_CXX_STANDARD 14)
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  # require at least gcc 4.9, otherwise regex wont work properly
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9)
    message(FATAL_ERROR "GCC version must be at least 4.9!")
  endif()
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Compiler flags
include(CheckCXXCompilerFlag)
set(MY_CXX_FLAGS -Wall)
foreach(FLAG ${MY_CXX_FLAGS})
  unset(CUR_FLAG_SUPPORTED CACHE)
  check_cxx_compiler_flag(${FLAG} CUR_FLAG_SUPPORTED)
  if(${CUR_FLAG_SUPPORTED})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${FLAG}")
  endif()
endforeach(FLAG)

# find dependencies
include(GNUInstallDirs)

find_package(OpenMP)

option(AMICI_TRY_ENABLE_HDF5 "Build with HDF5 support if available?" ON)
option(ENABLE_HDF5 "Build with HDF5 support?" OFF)

if(ENABLE_HDF5)
  find_package(
    HDF5
    COMPONENTS C HL CXX
    REQUIRED)
elseif(AMICI_TRY_ENABLE_HDF5)
    find_package(HDF5 COMPONENTS C HL CXX)
endif()

if(HDF5_FOUND)
  set(HDF5_LIBRARIES ${HDF5_HL_LIBRARIES} ${HDF5_C_LIBRARIES}
                     ${HDF5_CXX_LIBRARIES})
endif()

set(SUITESPARSE_DIR "${CMAKE_SOURCE_DIR}/ThirdParty/SuiteSparse/")
set(SUITESPARSE_INCLUDE_DIRS "${SUITESPARSE_DIR}/include"
                             "${CMAKE_SOURCE_DIR}/ThirdParty/sundials/src")
set(SUITESPARSE_LIBRARIES
    ${SUITESPARSE_DIR}/lib/libklu${CMAKE_STATIC_LIBRARY_SUFFIX}
    ${SUITESPARSE_DIR}/lib/libcolamd${CMAKE_STATIC_LIBRARY_SUFFIX}
    ${SUITESPARSE_DIR}/lib/libbtf${CMAKE_STATIC_LIBRARY_SUFFIX}
    ${SUITESPARSE_DIR}/lib/libamd${CMAKE_STATIC_LIBRARY_SUFFIX}
    ${SUITESPARSE_DIR}/lib/libsuitesparseconfig${CMAKE_STATIC_LIBRARY_SUFFIX}
)

find_package(
  SUNDIALS REQUIRED PATHS
  "${CMAKE_SOURCE_DIR}/ThirdParty/sundials/build/lib/cmake/sundials/")

set(GSL_LITE_INCLUDE_DIR "${CMAKE_SOURCE_DIR}/ThirdParty/gsl")

# AMICI requires BLAS, currently Intel MKL, CBLAS or MATLAB BLAS can be used.
# The latter is not supported via CMake yet.
set(BLAS
    "CBLAS"
    CACHE STRING "BLAS library to use")
set_property(CACHE BLAS PROPERTY STRINGS "CBLAS" "MKL" "ACCELERATE")
if(${BLAS} STREQUAL "MKL" OR DEFINED ENV{MKLROOT})
  if(DEFINED ENV{MKLROOT})
    # This is set by Environment Modules
    message(STATUS "Using MKL_INCDIR and MKL_LIB from environment module")
    set(BLAS
        "MKL"
        CACHE STRING "BLAS library to use" FORCE)
    set(BLAS_INCLUDE_DIRS
        "$ENV{MKL_INCDIR}"
        CACHE STRING "" FORCE)
    set(BLAS_LIBRARIES
        "$ENV{MKL_LIB}"
        CACHE STRING "" FORCE)
  else()
    set(BLAS_INCLUDE_DIRS
        ""
        CACHE STRING "")
    set(BLAS_LIBRARIES
        -lmkl
        CACHE STRING "")
  endif()
else()
  set(BLAS_INCLUDE_DIRS
      ""
      CACHE STRING "")
  set(BLAS_LIBRARIES
      -lcblas
      CACHE STRING "")
endif()
add_definitions(-DAMICI_BLAS_${BLAS})

# Add target to create version file
add_custom_target(
  version
  ${CMAKE_COMMAND}
  -D
  SRC=${CMAKE_SOURCE_DIR}/include/amici/version.in.h
  -D
  DST=${CMAKE_BINARY_DIR}/include/amici/version.h
  -P
  ${CMAKE_SOURCE_DIR}/cmake/configureVersion.cmake)

# Library source files
set(AMICI_SRC_LIST
    ${CMAKE_SOURCE_DIR}/src/symbolic_functions.cpp
    ${CMAKE_SOURCE_DIR}/src/cblas.cpp
    ${CMAKE_SOURCE_DIR}/src/amici.cpp
    ${CMAKE_SOURCE_DIR}/src/misc.cpp
    ${CMAKE_SOURCE_DIR}/src/rdata.cpp
    ${CMAKE_SOURCE_DIR}/src/edata.cpp
    ${CMAKE_SOURCE_DIR}/src/exception.cpp
    ${CMAKE_SOURCE_DIR}/src/simulation_parameters.cpp
    ${CMAKE_SOURCE_DIR}/src/spline.cpp
    ${CMAKE_SOURCE_DIR}/src/solver.cpp
    ${CMAKE_SOURCE_DIR}/src/solver_cvodes.cpp
    ${CMAKE_SOURCE_DIR}/src/solver_idas.cpp
    ${CMAKE_SOURCE_DIR}/src/logging.cpp
    ${CMAKE_SOURCE_DIR}/src/model.cpp
    ${CMAKE_SOURCE_DIR}/src/model_ode.cpp
    ${CMAKE_SOURCE_DIR}/src/model_dae.cpp
    ${CMAKE_SOURCE_DIR}/src/model_state.cpp
    ${CMAKE_SOURCE_DIR}/src/newton_solver.cpp
    ${CMAKE_SOURCE_DIR}/src/forwardproblem.cpp
    ${CMAKE_SOURCE_DIR}/src/steadystateproblem.cpp
    ${CMAKE_SOURCE_DIR}/src/backwardproblem.cpp
    ${CMAKE_SOURCE_DIR}/src/sundials_matrix_wrapper.cpp
    ${CMAKE_SOURCE_DIR}/src/sundials_linsol_wrapper.cpp
    ${CMAKE_SOURCE_DIR}/src/abstract_model.cpp
    ${CMAKE_SOURCE_DIR}/src/vector.cpp
    ${CMAKE_SOURCE_DIR}/include/amici/abstract_model.h
    ${CMAKE_SOURCE_DIR}/include/amici/amici.h
    ${CMAKE_SOURCE_DIR}/include/amici/backwardproblem.h
    ${CMAKE_SOURCE_DIR}/include/amici/cblas.h
    ${CMAKE_SOURCE_DIR}/include/amici/defines.h
    ${CMAKE_SOURCE_DIR}/include/amici/edata.h
    ${CMAKE_SOURCE_DIR}/include/amici/exception.h
    ${CMAKE_SOURCE_DIR}/include/amici/forwardproblem.h
    ${CMAKE_SOURCE_DIR}/include/amici/hdf5.h
    ${CMAKE_SOURCE_DIR}/include/amici/logging.h
    ${CMAKE_SOURCE_DIR}/include/amici/misc.h
    ${CMAKE_SOURCE_DIR}/include/amici/model_dae.h
    ${CMAKE_SOURCE_DIR}/include/amici/model_dimensions.h
    ${CMAKE_SOURCE_DIR}/include/amici/model.h
    ${CMAKE_SOURCE_DIR}/include/amici/model_ode.h
    ${CMAKE_SOURCE_DIR}/include/amici/model_state.h
    ${CMAKE_SOURCE_DIR}/include/amici/newton_solver.h
    ${CMAKE_SOURCE_DIR}/include/amici/rdata.h
    ${CMAKE_SOURCE_DIR}/include/amici/serialization.h
    ${CMAKE_SOURCE_DIR}/include/amici/simulation_parameters.h
    ${CMAKE_SOURCE_DIR}/include/amici/solver_cvodes.h
    ${CMAKE_SOURCE_DIR}/include/amici/solver.h
    ${CMAKE_SOURCE_DIR}/include/amici/solver_idas.h
    ${CMAKE_SOURCE_DIR}/include/amici/spline.h
    ${CMAKE_SOURCE_DIR}/include/amici/steadystateproblem.h
    ${CMAKE_SOURCE_DIR}/include/amici/sundials_linsol_wrapper.h
    ${CMAKE_SOURCE_DIR}/include/amici/sundials_matrix_wrapper.h
    ${CMAKE_SOURCE_DIR}/include/amici/symbolic_functions.h
    ${CMAKE_SOURCE_DIR}/include/amici/vector.h)
if(HDF5_FOUND)
  list(APPEND AMICI_SRC_LIST ${CMAKE_SOURCE_DIR}/src/hdf5.cpp)
endif()

add_library(${PROJECT_NAME} ${AMICI_SRC_LIST})
set(AMICI_CXX_OPTIONS
    ""
    CACHE STRING "C++ options for libamici (semicolon-separated)")
target_compile_options(${PROJECT_NAME} PRIVATE "${AMICI_CXX_OPTIONS}")
add_dependencies(${PROJECT_NAME} version)
file(GLOB PUBLIC_HEADERS include/amici/*.h)
set_target_properties(${PROJECT_NAME} PROPERTIES PUBLIC_HEADER
                                                 "${PUBLIC_HEADERS}")
target_include_directories(
  ${PROJECT_NAME}
  PUBLIC $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
         $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PUBLIC $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/include>
  PUBLIC swig
  PUBLIC ${GSL_LITE_INCLUDE_DIR}
  PUBLIC ${SUITESPARSE_INCLUDE_DIRS}
  PUBLIC ${HDF5_INCLUDE_DIRS})

if(NOT "${BLAS_INCLUDE_DIRS}" STREQUAL "")
  target_include_directories(${PROJECT_NAME} PUBLIC ${BLAS_INCLUDE_DIRS})
endif()

target_link_libraries(
  ${PROJECT_NAME}
  PUBLIC SUNDIALS::generic_static
  PUBLIC SUNDIALS::nvecserial_static
  PUBLIC SUNDIALS::sunmatrixband_static
  PUBLIC SUNDIALS::sunmatrixdense_static
  PUBLIC SUNDIALS::sunmatrixsparse_static
  PUBLIC SUNDIALS::sunlinsolband_static
  PUBLIC SUNDIALS::sunlinsoldense_static
  PUBLIC SUNDIALS::sunlinsolpcg_static
  PUBLIC SUNDIALS::sunlinsolspbcgs_static
  PUBLIC SUNDIALS::sunlinsolspfgmr_static
  PUBLIC SUNDIALS::sunlinsolspgmr_static
  PUBLIC SUNDIALS::sunlinsolsptfqmr_static
  PUBLIC SUNDIALS::sunlinsolklu_static
  PUBLIC SUNDIALS::sunnonlinsolnewton_static
  PUBLIC SUNDIALS::sunnonlinsolfixedpoint_static
  PUBLIC SUNDIALS::cvodes_static
  PUBLIC SUNDIALS::idas_static
  PUBLIC ${SUITESPARSE_LIBRARIES}
  PUBLIC ${BLAS_LIBRARIES}
  PUBLIC ${CMAKE_DL_LIBS})

if(HDF5_FOUND)
  target_include_directories(${PROJECT_NAME} PUBLIC ${HDF5_INCLUDE_DIRS})
  target_link_libraries(${PROJECT_NAME} PUBLIC ${HDF5_LIBRARIES})
endif()

if(OpenMP_FOUND)
  target_link_libraries(${PROJECT_NAME} PUBLIC OpenMP::OpenMP_CXX)
endif()

option(SUNDIALS_SUPERLUMT_ENABLE "Enable sundials SuperLUMT?" OFF)
if(SUNDIALS_SUPERLUMT_ENABLE)
  set(SUNDIALS_LIBRARIES
      ${SUNDIALS_LIBRARIES}
      ${SUNDIALS_LIB_DIR}/libsundials_sunlinsolsuperlumt${CMAKE_STATIC_LIBRARY_SUFFIX}
      ${CMAKE_SOURCE_DIR}/ThirdParty/SuperLU_MT_3.1/lib/libsuperlu_mt_PTHREAD${CMAKE_STATIC_LIBRARY_SUFFIX}
      -lblas)
  target_include_directories(
    ${PROJECT_NAME} PUBLIC "${CMAKE_SOURCE_DIR}/ThirdParty/SuperLU_MT_3.1/SRC/")
endif()

# Create targets to make the sources show up in IDEs for convenience

# For matlab interface
set(matlab_sources
    src/interface_matlab.cpp src/returndata_matlab.cpp
    include/amici/interface_matlab.h include/amici/returndata_matlab.h)
find_package(Matlab)
# In case we can find Matlab, we create a respective library to compile the
# extension from cmake. Otherwise we just create a dummy target for the files to
# show up inside IDEs. (Set the Matlab_ROOT_DIR cmake variable if CMake cannot
# find your Matlab installation)
if(${Matlab_FOUND})
  add_library(matlabInterface ${matlab_sources})
  set_target_properties(matlabInterface PROPERTIES INCLUDE_DIRECTORIES
                                                   "${Matlab_INCLUDE_DIRS}")

  target_link_libraries(matlabInterface PUBLIC amici)

else()
  add_custom_target(matlabInterface SOURCES ${matlab_sources})
endif()
set_property(
  TARGET matlabInterface
  APPEND
  PROPERTY INCLUDE_DIRECTORIES "${CMAKE_SOURCE_DIR}/include/")

# For template files
add_custom_target(
  fileTemplates
  SOURCES src/CMakeLists.template.cmake
          src/main.template.cpp
          src/model_header.ODE_template.h
          src/model.ODE_template.cpp
          src/wrapfunctions.ODE_template.h
          src/wrapfunctions.template.cpp
          swig/CMakeLists_model.cmake
          swig/modelname.template.i)
set_target_properties(fileTemplates PROPERTIES INCLUDE_DIRECTORIES
                                               "${CMAKE_SOURCE_DIR}/include/")

if($ENV{ENABLE_GCOV_COVERAGE})
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0 --coverage")
endif()

include(clang-tools)
include(cmakelang-tools)

set(AUTHORS "Fabian Froehlich, Jan Hasenauer, Daniel Weindl and Paul Stapor")
set(AUTHOR_EMAIL "Fabian_Froehlich@hms.harvard.edu")

# <Export cmake configuration>
install(
  TARGETS ${PROJECT_NAME}
  EXPORT AmiciTargets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  INCLUDES
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/amici)
export(
  EXPORT AmiciTargets
  FILE AmiciTargets.cmake
  NAMESPACE Upstream::)
include(CMakePackageConfigHelpers)
include(version)
configure_package_config_file(
  cmake/AmiciConfig.cmake "${CMAKE_CURRENT_BINARY_DIR}/AmiciConfig.cmake"
  INSTALL_DESTINATION "${LIB_INSTALL_DIR}/cmake/")
write_basic_package_version_file(AmiciConfigVersion.cmake
                                 COMPATIBILITY ExactVersion)
install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/AmiciConfig.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/AmiciTargets.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/AmiciConfigVersion.cmake
  DESTINATION share/Amici/cmake)
# Register package

option(EXPORT_PACKAGE "Export AMICI library to CMake package registry?" ON)
if(EXPORT_PACKAGE)
  export(PACKAGE Amici)
endif()
# </Export cmake configuration>

# build interfaces for other languages
option(ENABLE_SWIG "Build AMICI swig library?" ON)
if(ENABLE_SWIG)
  add_subdirectory(swig)
endif()

option(ENABLE_PYTHON "Create Python module?" ON)
if(ENABLE_PYTHON)
  add_subdirectory(python)
endif()

option(BUILD_TESTS "Build integration tests?" ON)
if(BUILD_TESTS)
  if(HDF5_FOUND)
    enable_testing()

    add_subdirectory(tests/cpp)
  else()
    message(WARNING "Cannot build tests without HDF5 support.")
  endif()

endif()
