# SPDX-FileCopyrightText: 2011 - 2025 NVIDIA CORPORATION. All Rights Reserved.
#
# SPDX-License-Identifier: BSD-3-Clause

cmake_minimum_required (VERSION 3.18)

#select all sources
FILE(GLOB_RECURSE SRCS "*.cu")
list(FILTER SRCS EXCLUDE REGEX "/src/tests/")
list(FILTER SRCS EXCLUDE REGEX "/utest.cu")

if(AMGX_MINIMAL_SOLVERS)
  # cunibs keep-set: PCG + aggregation-AMG (SIZE_2 / LOW_DEG / V-cycle) + JACOBI_L1
  # smoother + DENSE_LU coarse solve. Exclude everything else from the build to cut
  # both binary size and compile time. The matching factory registrations are
  # #ifdef AMGX_MINIMAL_SOLVERS-guarded in core.cu, and the extra FixedCycle
  # instantiations are guarded in cycles/fixed_cycle.cu, so no dangling references remain.
  list(FILTER SRCS EXCLUDE REGEX "/eigensolvers/")
  list(FILTER SRCS EXCLUDE REGEX "/amg_eigensolver\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/amgx_eig_c\\.cu$")
  # Classical/energymin AMG: strip the concrete selectors/interpolators/strength/levels
  # but KEEP the factory base files (selector.cu / interpolator.cu / strength.cu /
  # em_interpolator.cu / em_selector.cu). core.cu's config machinery (getAll*, unregister)
  # only calls the factory base methods, so the bases must remain; the concretes are unused
  # by the aggregation path and their unvalidated parameter defaults are never read.
  list(FILTER SRCS EXCLUDE REGEX "/classical/classical_amg_level\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/classical/interpolators/(common|distance1|distance2|multipass)\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/classical/selectors/(aggressive_hmis|aggressive_pmis|cr|dummy_selector|hmis|pmis|rs)\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/classical/strength/(affinity|strength_base)\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/energymin/energymin_amg_level\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/energymin/interpolators/em\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/cycles/(w|f|cg|cg_flex)_cycle\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/aggregation/selectors/(size4_selector|size8_selector|multi_pairwise|geo_selector|parallel_greedy_selector)\\.cu$")
  list(FILTER SRCS EXCLUDE REGEX "/aggregation/coarseAgenerators/(hybrid|thrust)_coarse_A_generator\\.cu$")
  # block_jacobi_solver is kept: it is the registered default fine_smoother/coarse_smoother.
  list(FILTER SRCS EXCLUDE REGEX "/solvers/(cheb_solver|gmres_solver|fgmres_solver|bicgstab_solver|pbicgstab_solver|cg_solver|pcgf_solver|idr_solver|idrmsync_solver|cf_jacobi_solver|gauss_seidel_solver|fixcolor_gauss_seidel_solver|multicolor_gauss_seidel_solver|multicolor_ilu_solver|multicolor_dilu_solver|polynomial_solver|kpz_polynomial_solver|chebyshev_poly|kaczmarz_solver|user_solver)\\.cu$")
  message(STATUS "AMGX: minimal solver set - excluded eigensolvers, classical/energymin AMG, unused Krylov/smoothers, extra cycles/selectors")
endif()

target_sources(amgx_libs PRIVATE ${SRCS})

target_include_directories(amgx_libs PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../include)

set(AMGX_INCLUDES ../include)

if(AMGX_BUILD_TESTS)

set(tests_all ${tests_all} testframework.cu test_utils.cu unit_test.cu)

add_library(amgx_tests_libs OBJECT ${tests_all})

target_include_directories(amgx_tests_libs PRIVATE ${AMGX_INCLUDES})

add_library(amgx_tests_library STATIC $<TARGET_OBJECTS:amgx_tests_libs>)

target_link_libraries(amgx_tests_libs Thrust)

add_dependencies(amgx_tests_library amgx)

target_link_libraries(amgx_tests_library amgx)

add_executable(amgx_tests_launcher utest.cu $<TARGET_OBJECTS:amgx_tests_libs>)

# CMAKE_CUDA_ARCHITECTURES automatically inherits to all targets
# No need to set per-target properties

target_compile_options(amgx_tests_launcher PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: ${CUDA_NVCC_FLAGS} >)
target_compile_options(amgx_tests_libs PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: ${CUDA_NVCC_FLAGS} >)
target_compile_options(amgx_tests_library PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: ${CUDA_NVCC_FLAGS} >)

target_include_directories(amgx_tests_launcher PRIVATE ${AMGX_INCLUDES})

# create executables for host and device, scalar and block version

GET_FILENAME_COMPONENT(CMAKE_C_COMPILER_NAME "${CMAKE_C_COMPILER}" NAME)
IF(CMAKE_C_COMPILER_NAME MATCHES cl)
    set(libs_all CUDA::cusparse CUDA::cublas)
ELSE()
    set(libs_all CUDA::cusparse CUDA::cublas rt dl)
ENDIF()

target_link_libraries(amgx_tests_launcher amgx_tests_library amgxsh ${libs_all})

if(MPI_FOUND)
    target_link_libraries(amgx_tests_launcher MPI::MPI_CXX)
endif(MPI_FOUND)

if(NOT ${CMAKE_BUILD_TYPE} MATCHES "Release")
  install(TARGETS amgx_tests_launcher DESTINATION "lib/tests")
endif(NOT ${CMAKE_BUILD_TYPE} MATCHES "Release")

endif(AMGX_BUILD_TESTS)

#copy configs to build directory
add_custom_target(copy_configs ALL)
add_custom_command(TARGET copy_configs COMMAND ${CMAKE_COMMAND} -E copy_directory "${CMAKE_CURRENT_SOURCE_DIR}/configs" "${CMAKE_BINARY_DIR}/configs")

if(AMGX_INSTALL)

install(FILES
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/AGGREGATION_DILU.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/AGGREGATION_GS.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/AGGREGATION_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/AGGREGATION_MULTI_PAIRWISE.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/CG_DILU.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/CLASSICAL_CG_CYCLE.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/CLASSICAL_CGF_CYCLE.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/CLASSICAL_F_CYCLE.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/CLASSICAL_V_CYCLE.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/CLASSICAL_W_CYCLE.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/FGMRES_AGGREGATION_DILU.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/FGMRES_AGGREGATION_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/FGMRES_CLASSICAL_AGGRESSIVE_HMIS.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/FGMRES_CLASSICAL_AGGRESSIVE_PMIS.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/FGMRES_NOPREC.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/GMRES_AMG_D2.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/IDR_DILU.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/IDRMSYNC_DILU.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PBICGSTAB_AGGREGATION_W_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PBICGSTAB_CLASSICAL_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PBICGSTAB_NOPREC.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCG_AGGREGATION_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCG_CLASSICAL_F_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCG_CLASSICAL_V_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCG_CLASSICAL_W_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCG_DILU.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCG_NOPREC.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCGF_CLASSICAL_F_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCGF_CLASSICAL_V_JACOBI.json"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/PCGF_CLASSICAL_W_JACOBI.json"
    DESTINATION "lib/configs")

install(FILES 
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/ARNOLDI"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/INVERSE_FGMRES"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/JACOBI_DAVIDSON"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/LANCZOS"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/LOBPCG"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/PAGERANK"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/POWER_ITERATION"
    "${CMAKE_CURRENT_SOURCE_DIR}/configs/eigen_configs/SUBSPACE_ITERATION"
    DESTINATION "lib/configs/eigen_configs")

endif(AMGX_INSTALL)

