cmake_minimum_required(VERSION 3.11)

project(KANN LANGUAGES C)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_CURRENT_SOURCE_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_CURRENT_SOURCE_DIR}/bin)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${CMAKE_CURRENT_SOURCE_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_CURRENT_SOURCE_DIR}/lib)

add_library(kann STATIC kautodiff.c kann.c)
target_compile_definitions(kann PUBLIC HAVE_PTHREAD)

add_library(kann_extra STATIC kann_extra/kann_data.c)
target_compile_definitions(kann_extra PUBLIC HAVE_ZLIB)

foreach(target mlp mnist-cnn ae vae)
    add_executable(${target} examples/${target}.c)
    target_compile_definitions(${target} PUBLIC HAVE_PTHREAD)
    target_link_libraries(${target} PUBLIC kann kann_extra pthread z m)  
endforeach(target)

foreach(target textgen inspect rnn-bit)
    add_executable(${target} examples/${target}.c)
    target_compile_definitions(${target} PUBLIC HAVE_PTHREAD)
    target_link_libraries(${target} PUBLIC kann pthread z m)
endforeach(target)

include(FetchContent)
FetchContent_Declare(
  kanndata
  URL https://github.com/attractivechaos/kann/releases/download/v0/kann-data.tgz
  URL_HASH MD5=591277e16e47d909b65876c64b148ee5
  DOWNLOAD_EXTRACT_TIMESTAMP true
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kann-data
)
FetchContent_Declare(
  kannmodels
  URL https://github.com/attractivechaos/kann/releases/download/v0/kann-models.tgz
  URL_HASH MD5=5dd8ead0e3d0ec3b9f467ce5848ddb66
  DOWNLOAD_EXTRACT_TIMESTAMP true
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kann-models
)
FetchContent_MakeAvailable(kanndata kannmodels)

enable_testing()
add_test(textgen COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/bin/textgen -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/acc-l3-n256r.kan -T 1e-6)
add_test(mnist-cnn COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/bin/mnist-cnn -i ${CMAKE_CURRENT_SOURCE_DIR}/kann-models/mnist-cnn.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz)
add_test(mlp1 COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/bin/mlp -o ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-x.knd.gz ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-train-y.knd.gz)
add_test(mlp2 COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/bin/mlp -i ${CMAKE_CURRENT_SOURCE_DIR}/mnist-mlp.kan ${CMAKE_CURRENT_SOURCE_DIR}/kann-data/mnist-test-x.knd.gz)
set_tests_properties(mlp1 mlp2 PROPERTIES RUN_SERIAL TRUE)
