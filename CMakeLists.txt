cmake_minimum_required(VERSION 3.11)

project(KANN LANGUAGES C)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})
add_definitions(-DHAVE_ZLIB -DHAVE_PTHREAD)

add_executable(mlp examples/mlp.c kautodiff.c kann.c kann_extra/kann_data.c)
target_link_libraries(mlp PUBLIC pthread z m)
set_target_properties(mlp PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_CURRENT_SOURCE_DIR}/bin RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_CURRENT_SOURCE_DIR}/bin)

add_executable(textgen examples/textgen.c kautodiff.c kann.c)
target_link_libraries(textgen PUBLIC pthread z m)
set_target_properties(textgen PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_CURRENT_SOURCE_DIR}/bin RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_CURRENT_SOURCE_DIR}/bin)

add_executable(mnist-cnn examples/mnist-cnn.c kautodiff.c kann.c kann_extra/kann_data.c)
target_link_libraries(mnist-cnn PUBLIC pthread z m)
set_target_properties(mnist-cnn PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_CURRENT_SOURCE_DIR}/bin RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_CURRENT_SOURCE_DIR}/bin)

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
