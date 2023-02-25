# CMake requirements
cmake_minimum_required(VERSION 3.11)

# This project is in C89
project(
  KANN
  VERSION 0.2.0 
  DESCRIPTION "Standalone and lightweight library in C for constructing and training small to medium artificial neural networks" 
  HOMEPAGE_URL "https://github.com/attractivechaos/kann"
  LANGUAGES C
)
set(C_STANDARD 90)

# Additional search paths
if(APPLE)
  set(CMAKE_PREFIX_PATH "/usr/local/opt/openblas")
elseif(WIN32)
  set(CMAKE_PREFIX_PATH "$ENV{BLAS_HOME}\\lib" ) 
endif()

# Search ZLib
find_package(ZLIB REQUIRED)

# Search PThreads
set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads)

# Search OpenBLAS
set(BLA_VENDOR OpenBLAS)
find_package(BLAS)
if(BLAS_FOUND)
  find_path(
    BLAS_INCLUDE_DIRS 
    cblas.h 
    ${CMAKE_PREFIX_PATH} /usr/include /usr/local/include $ENV{BLAS_HOME}/include
  )
endif(BLAS_FOUND)

# Search OpenMP
find_package(OpenMP)

# Custom definitions for compilers
if(CMAKE_C_COMPILER_ID STREQUAL "Clang")
  # from the original Makefile
  add_definitions(-Wall -Wextra -Wc++-compat)
elseif(CMAKE_C_COMPILER_ID STREQUAL "GNU")
  # there are unused results that pop-up
  add_definitions(-Wall -Wextra -Wc++-compat -Wno-unused-result)
elseif(CMAKE_C_COMPILER_ID STREQUAL "Intel")
  # empty
elseif(CMAKE_C_COMPILER_ID STREQUAL "MSVC")
  # Windows has no getopt
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/libs/getopt)
  # silence many warnings
  add_definitions(/D "_CRT_SECURE_NO_WARNINGS" /D "_CRT_SECURE_NO_DEPRECATE" /wd4996 /wd4267 /wd4244 /wd4098 /wd4305 /wd4090)
endif()

# Include the library fiolders
include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/kann)

# Put all executables in bin
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

# Put all libraries in lib
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})

# The main static library
add_library(
  kann
  STATIC kann/kautodiff.c kann/kann.c
)
target_compile_definitions(
  kann 
  PRIVATE $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:HAVE_PTHREAD> $<$<BOOL:${BLAS_FOUND}>:HAVE_CBLAS>
)
target_include_directories(
  kann 
  PRIVATE $<$<BOOL:${BLAS_FOUND}>:${BLAS_INCLUDE_DIRS}>
)

# Additional static library used to read from data files
add_library(
  kann_extra 
  STATIC kann_extra/kann_data.c
)
target_compile_definitions(
  kann_extra 
  PRIVATE $<$<BOOL:${ZLIB_FOUND}>:HAVE_ZLIB>
)
target_include_directories(
  kann_extra 
  PRIVATE $<$<BOOL:${ZLIB_FOUND}>:${ZLIB_INCLUDE_DIRS}>
)

# Create the main examples that use ZLib
foreach(target mlp mnist-cnn ae vae)
  add_executable(${target} examples/${target}.c)
  target_compile_definitions(
    ${target} 
    PUBLIC $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:HAVE_PTHREAD>
  )
  target_link_libraries(
    ${target} 
    PUBLIC kann kann_extra $<$<BOOL:${OpenMP_FOUND}>:OpenMP::OpenMP_C> $<$<BOOL:${BLAS_FOUND}>:${BLAS_LIBRARIES}> $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:Threads::Threads> $<$<BOOL:${ZLIB_FOUND}>:ZLIB::ZLIB> $<$<NOT:$<PLATFORM_ID:Windows>>:m>
  )  
  # Copy the required DLL's in the exe folder
  if(WIN32 AND BLAS_FOUND)
    add_custom_command(
      TARGET ${target} 
      POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "$ENV{BLAS_HOME}\\bin\\libopenblas.dll" ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
    )
  endif()
  if(WIN32 AND ZLIB_FOUND)
    add_custom_command(
      TARGET ${target} 
      POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ZLIB_INCLUDE_DIRS}\\..\\bin\\zlib.dll" ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
    )
  endif()
endforeach(target)

# Create the main examples that do not use ZLib
foreach(target textgen inspect rnn-bit)
  add_executable(${target} examples/${target}.c)
  target_compile_definitions(
    ${target} 
    PUBLIC $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:HAVE_PTHREAD>
  )
  target_link_libraries(
    ${target} 
    PUBLIC kann $<$<BOOL:${OpenMP_FOUND}>:OpenMP::OpenMP_C> $<$<BOOL:${BLAS_FOUND}>:${BLAS_LIBRARIES}> $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:Threads::Threads> $<$<NOT:$<PLATFORM_ID:Windows>>:m>
  )
  # Copy the required DLL's in the exe folder
  if(WIN32 AND BLAS_FOUND)
    add_custom_command(
      TARGET ${target} 
      POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "$ENV{BLAS_HOME}\\bin\\libopenblas.dll" ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
    )
  endif()
endforeach(target)

# Download precomputed models and some training data
include(FetchContent)
FetchContent_Declare(
  kanndata
  DOWNLOAD_EXTRACT_TIMESTAMP true
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kann-data
  URL https://github.com/attractivechaos/kann/releases/download/v0/kann-data.tgz
  URL_HASH MD5=591277e16e47d909b65876c64b148ee5
)
FetchContent_Declare(
  kannmodels
  DOWNLOAD_EXTRACT_TIMESTAMP true
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kann-models
  URL https://github.com/attractivechaos/kann/releases/download/v0/kann-models.tgz
  URL_HASH MD5=5dd8ead0e3d0ec3b9f467ce5848ddb66
)
FetchContent_MakeAvailable(kanndata kannmodels)

# The testing section uses the examples
enable_testing()
add_test(
  NAME mlp1
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/mlp -o ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-x.knd.gz ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-y.knd.gz
)
add_test(
  NAME ae1
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ae -o ${CMAKE_CURRENT_SOURCE_DIR}/mnist-ae.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-x.knd.gz ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-y.knd.gz
)
set_tests_properties(
  mlp1 ae1 
  PROPERTIES FIXTURES_SETUP kann
)
add_test(
  NAME mlp2
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/mlp -i ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz
)
add_test(
  NAME ae2
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ae -i ${CMAKE_CURRENT_SOURCE_DIR}/mnist-ae.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz
)
add_test(
  NAME inspect1
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/inspect ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan
)
add_test(
  NAME inspect2
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/inspect ${CMAKE_CURRENT_SOURCE_DIR}/mnist-ae.kan
)
add_test(
  NAME textgen
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/textgen -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/acc-l3-n256r.kan -T 1e-6
)
add_test(
  NAME vae1
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/vae -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mnist-vae.kan -A ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz
)
add_test(
  NAME vae2
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/vae -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mnist-vae.kan -g 10
)
add_test(
  NAME mnist-cnn
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/mnist-cnn -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mnist-cnn.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz
)
add_test(
  NAME rnn-bit
  COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/rnn-bit -Ai ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mul100.kan ${CMAKE_CURRENT_SOURCE_DIR}/cmake/rnn-bit-in.txt
) 
set_tests_properties(
  mlp2 ae2 inspect1 inspect2 textgen 
  PROPERTIES FIXTURES_REQUIRED kann
)
add_test(
  NAME rmkan
  COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_SOURCE_DIR}/mnist-ae.kan ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan
) 
set_tests_properties(
  rmkan 
  PROPERTIES FIXTURES_CLEANUP kann
)
