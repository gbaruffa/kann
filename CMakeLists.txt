cmake_minimum_required(VERSION 3.11)

project(KANN LANGUAGES C)


find_package(ZLIB REQUIRED)

set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads)

if(APPLE)
  set(CMAKE_PREFIX_PATH "/usr/local/opt/openblas")
elseif(WIN32)
  set(CMAKE_PREFIX_PATH "$ENV{BLAS_HOME}\\lib") 
endif()
set(BLA_VENDOR OpenBLAS)
find_package(BLAS)
if(BLAS_FOUND)
  find_path(BLAS_INCLUDE_DIRS cblas.h ${CMAKE_PREFIX_PATH} /usr/include /usr/local/include $ENV{BLAS_HOME}/include)
endif(BLAS_FOUND)

find_package(OpenMP)

if(CMAKE_C_COMPILER_ID STREQUAL "Clang")
  add_definitions(-Wall -Wextra -Wc++-compat)
elseif(CMAKE_C_COMPILER_ID STREQUAL "GNU")
  add_definitions(-Wall -Wextra -Wc++-compat -Wno-unused-result)
elseif(CMAKE_C_COMPILER_ID STREQUAL "Intel")
elseif(CMAKE_C_COMPILER_ID STREQUAL "MSVC")
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/libs/getopt)
  add_definitions(/D "_CRT_SECURE_NO_WARNINGS" /D "_CRT_SECURE_NO_DEPRECATE" /wd4996 /wd4267 /wd4244 /wd4098 /wd4305 /wd4090)
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})

add_library(kann STATIC kautodiff.c kann.c)
target_compile_definitions(kann PUBLIC $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:HAVE_PTHREAD> $<$<BOOL:${BLAS_FOUND}>:HAVE_CBLAS>)
target_include_directories(kann PUBLIC $<$<BOOL:${BLAS_FOUND}>:${BLAS_INCLUDE_DIRS}>)

add_library(kann_extra STATIC kann_extra/kann_data.c)
target_compile_definitions(kann_extra PUBLIC $<$<BOOL:${ZLIB_FOUND}>:HAVE_ZLIB>)
target_include_directories(kann_extra PUBLIC $<$<BOOL:${ZLIB_FOUND}>:${ZLIB_INCLUDE_DIRS}>)

foreach(target mlp mnist-cnn ae vae)
  add_executable(${target} examples/${target}.c)
  target_compile_definitions(${target} PUBLIC $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:HAVE_PTHREAD>)
  target_link_libraries(${target} PUBLIC kann kann_extra $<$<BOOL:${OpenMP_FOUND}>:OpenMP::OpenMP_C> $<$<BOOL:${BLAS_FOUND}>:${BLAS_LIBRARIES}> $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:Threads::Threads> $<$<BOOL:${ZLIB_FOUND}>:ZLIB::ZLIB> $<$<NOT:$<PLATFORM_ID:Windows>>:m>)  
  if(WIN32 AND BLAS_FOUND)
    add_custom_command(TARGET ${target} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "$ENV{BLAS_HOME}\\bin\\libopenblas.dll" ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
  endif()
  if(WIN32 AND ZLIB_FOUND)
    add_custom_command(TARGET ${target} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ZLIB_INCLUDE_DIRS}\\..\\bin\\zlib.dll" ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
  endif()
endforeach(target)

foreach(target textgen inspect rnn-bit)
  add_executable(${target} examples/${target}.c)
  target_compile_definitions(${target} PUBLIC $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:HAVE_PTHREAD>)
  target_link_libraries(${target} PUBLIC kann $<$<BOOL:${OpenMP_FOUND}>:OpenMP::OpenMP_C> $<$<BOOL:${BLAS_FOUND}>:${BLAS_LIBRARIES}> $<$<BOOL:${CMAKE_USE_PTHREADS_INIT}>:Threads::Threads> $<$<NOT:$<PLATFORM_ID:Windows>>:m>)
  if(WIN32 AND BLAS_FOUND)
    add_custom_command(TARGET ${target} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "$ENV{BLAS_HOME}\\bin\\libopenblas.dll" ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
  endif()
  if(WIN32 AND ZLIB_FOUND)
    add_custom_command(TARGET ${target} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ZLIB_INCLUDE_DIRS}\\..\\bin\\zlib.dll" ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
  endif()
endforeach(target)

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


enable_testing()
file(WRITE "${CMAKE_CURRENT_SOURCE_DIR}/in.txt" "15315611231621249 78")
add_test(mlp1 ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/mlp -o ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-x.knd.gz ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-y.knd.gz)
add_test(mlp2 ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/mlp -i ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz)
add_test(ae1 ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ae -o ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-x.knd.gz ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-y.knd.gz)
add_test(ae2 ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ae -i ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz)
add_test(inspect ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/inspect ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan)
set_property(TARGET mlp APPEND PROPERTY ADDITIONAL_CLEAN_FILES ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/in.txt)
set_tests_properties(mlp1 mlp2 ae1 ae2 inspect PROPERTIES RUN_SERIAL TRUE)
add_test(textgen ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/textgen -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/acc-l3-n256r.kan -T 1e-6)
add_test(vae1 ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/vae -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mnist-vae.kan -A ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz)
add_test(vae2 ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/vae -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mnist-vae.kan -g 10)
add_test(mnist-cnn ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/mnist-cnn -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mnist-cnn.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz)
add_test(rnn-bit ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/rnn-bit -Ai ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mul100.kan ${CMAKE_CURRENT_SOURCE_DIR}/in.txt)

