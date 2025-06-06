# Copyright © 2023 Apple Inc. All rights reserved.
# Copyright (c) Meta Platforms, Inc. and affiliates.
# All rights reserved.
#
# This source code is licensed under the BSD-style license found in the
# LICENSE file in the root directory of this source tree.

cmake_minimum_required(VERSION 3.19)

project(executorch_coreml_backend)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

# Source root directory for executorch.
if(NOT EXECUTORCH_ROOT)
  set(EXECUTORCH_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/../../..)
endif()

if(EXECUTORCH_BUILD_DEVTOOLS)
  # protobuf requires frtti
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -frtti")
endif()

option(COREML_BUILD_EXECUTOR_RUNNER "Build CoreML executor runner." OFF)

set(CMAKE_OSX_DEPLOYMENT_TARGET 10.15)

# inmemoryfs sources
set(INMEMORYFS_SOURCES
    runtime/inmemoryfs/inmemory_filesystem.cpp
    runtime/inmemoryfs/inmemory_filesystem_utils.mm
    runtime/inmemoryfs/memory_buffer.cpp
    runtime/inmemoryfs/memory_stream.cpp
    runtime/inmemoryfs/reversed_memory_stream.cpp
)

# kvstore sources
set(KVSTORE_SOURCES
    runtime/kvstore/database.cpp runtime/kvstore/json_key_value_store.cpp
    runtime/kvstore/sqlite_error.cpp runtime/kvstore/key_value_store.cpp
    runtime/kvstore/statement.cpp
)

# delegate sources
set(DELEGATE_SOURCES
    runtime/delegate/asset.mm
    runtime/delegate/backend_delegate.mm
    runtime/delegate/coreml_backend_delegate.mm
    runtime/delegate/ETCoreMLAsset.mm
    runtime/delegate/ETCoreMLAssetManager.mm
    runtime/delegate/ETCoreMLDefaultModelExecutor.mm
    runtime/delegate/ETCoreMLModelLoader.mm
    runtime/delegate/ETCoreMLModelCompiler.mm
    runtime/delegate/ETCoreMLLogging.mm
    runtime/delegate/ETCoreMLModel.mm
    runtime/delegate/ETCoreMLModelManager.mm
    runtime/delegate/ETCoreMLStrings.mm
    runtime/delegate/MLModel_Prewarm.mm
    runtime/delegate/MLMultiArray_Copy.mm
    runtime/delegate/multiarray.mm
    runtime/delegate/serde_json.mm
)

# util sources
set(UTIL_SOURCES runtime/util/json_util.cpp runtime/util/objc_json_serde.mm)

# sdk sources
set(SDK_SOURCES
    runtime/sdk/ETCoreMLModelAnalyzer.mm
    runtime/sdk/ETCoreMLModelStructurePath.mm
    runtime/sdk/ETCoreMLOperationProfilingInfo.mm
    runtime/sdk/ETCoreMLModelDebugInfo.mm
    runtime/sdk/ETCoreMLModelDebugger.mm
    runtime/sdk/ETCoreMLModelProfiler.mm
    runtime/sdk/ETCoreMLPair.mm
    runtime/sdk/model_package_info.mm
    runtime/sdk/model_event_logger_impl.mm
    runtime/sdk/program_path.mm
)

# protobuf sources
set(PROTOBUF_SOURCES
    runtime/sdk/format/ArrayFeatureExtractor.pb.cc
    runtime/sdk/format/AudioFeaturePrint.pb.cc
    runtime/sdk/format/BayesianProbitRegressor.pb.cc
    runtime/sdk/format/CategoricalMapping.pb.cc
    runtime/sdk/format/ClassConfidenceThresholding.pb.cc
    runtime/sdk/format/CustomModel.pb.cc
    runtime/sdk/format/DataStructures.pb.cc
    runtime/sdk/format/DictVectorizer.pb.cc
    runtime/sdk/format/FeatureTypes.pb.cc
    runtime/sdk/format/FeatureVectorizer.pb.cc
    runtime/sdk/format/Gazetteer.pb.cc
    runtime/sdk/format/GLMClassifier.pb.cc
    runtime/sdk/format/GLMRegressor.pb.cc
    runtime/sdk/format/Identity.pb.cc
    runtime/sdk/format/Imputer.pb.cc
    runtime/sdk/format/ItemSimilarityRecommender.pb.cc
    runtime/sdk/format/LinkedModel.pb.cc
    runtime/sdk/format/MIL.pb.cc
    runtime/sdk/format/Model.pb.cc
    runtime/sdk/format/NearestNeighbors.pb.cc
    runtime/sdk/format/NeuralNetwork.pb.cc
    runtime/sdk/format/NonMaximumSuppression.pb.cc
    runtime/sdk/format/Normalizer.pb.cc
    runtime/sdk/format/NonMaximumSuppression.pb.cc
    runtime/sdk/format/OneHotEncoder.pb.cc
    runtime/sdk/format/Parameters.pb.cc
    runtime/sdk/format/Scaler.pb.cc
    runtime/sdk/format/SoundAnalysisPreprocessing.pb.cc
    runtime/sdk/format/SVM.pb.cc
    runtime/sdk/format/TextClassifier.pb.cc
    runtime/sdk/format/TreeEnsemble.pb.cc
    runtime/sdk/format/VisionFeaturePrint.pb.cc
    runtime/sdk/format/WordEmbedding.pb.cc
    runtime/sdk/format/WordTagger.pb.cc
)

find_library(FOUNDATION_FRAMEWORK Foundation)

# CoreML util
add_library(coreml_util ${UTIL_SOURCES})
target_include_directories(coreml_util PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/runtime/util)
target_link_libraries(coreml_util PRIVATE ${FOUNDATION_FRAMEWORK})

install(
  TARGETS coreml_util
  DESTINATION lib
  INCLUDES
  DESTINATION ${_common_include_directories}
)

# CoreML inmemoryfs
add_library(coreml_inmemoryfs ${INMEMORYFS_SOURCES})
target_include_directories(coreml_inmemoryfs PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/runtime/inmemoryfs)
target_link_libraries(coreml_inmemoryfs PRIVATE coreml_util ${FOUNDATION_FRAMEWORK})

install(
  TARGETS coreml_inmemoryfs
  DESTINATION lib
  INCLUDES
  DESTINATION ${_common_include_directories}
)

# Define the delegate library
add_library(coremldelegate)
target_sources(coremldelegate PRIVATE ${KVSTORE_SOURCES} ${DELEGATE_SOURCES})

target_include_directories(
  coremldelegate PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/runtime/include
)
target_include_directories(
  coremldelegate PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/runtime/kvstore
)
target_include_directories(
  coremldelegate PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/runtime/delegate
)
target_include_directories(coremldelegate PRIVATE ${EXECUTORCH_ROOT}/..)
target_include_directories(coremldelegate PRIVATE ${EXECUTORCH_ROOT}/runtime/core/portable_type/c10)
target_compile_definitions(coremldelegate PRIVATE C10_USING_CUSTOM_GENERATED_MACROS)

if(EXECUTORCH_BUILD_DEVTOOLS)
  target_sources(coremldelegate PRIVATE ${SDK_SOURCES} ${PROTOBUF_SOURCES})
  target_include_directories(
    coremldelegate
    PRIVATE
      ${CMAKE_CURRENT_SOURCE_DIR}/runtime/sdk
      ${CMAKE_CURRENT_SOURCE_DIR}/third-party/coremltools/deps/protobuf/src
  )
  add_subdirectory(
    ${CMAKE_CURRENT_SOURCE_DIR}/third-party/coremltools/deps/protobuf/cmake
  )

  target_link_options_shared_lib(libprotobuf-lite)
  target_link_libraries(coremldelegate PRIVATE libprotobuf-lite)
endif()

find_library(ACCELERATE_FRAMEWORK Accelerate)
find_library(COREML_FRAMEWORK CoreML)
find_library(SQLITE_LIBRARY sqlite3)

target_link_libraries(
  coremldelegate
  PUBLIC  coreml_util
          coreml_inmemoryfs
  PRIVATE executorch_core
          ${ACCELERATE_FRAMEWORK}
          ${COREML_FRAMEWORK}
          ${FOUNDATION_FRAMEWORK}
          ${SQLITE_LIBRARY}
)

target_link_options_shared_lib(coremldelegate)

if(COREML_BUILD_EXECUTOR_RUNNER)
  target_link_libraries(
    coremldelegate PRIVATE portable_ops_lib portable_kernels
  )
endif()

target_compile_options(coremldelegate PRIVATE "-fobjc-arc")
target_compile_options(coremldelegate PRIVATE "-fno-exceptions")

if(EXECUTORCH_BUILD_DEVTOOLS)
  target_compile_options(
    executorch_core PUBLIC -DET_EVENT_TRACER_ENABLED
  )
  target_compile_options(coremldelegate PRIVATE "-frtti")
  target_compile_options(libprotobuf-lite PRIVATE "-frtti")
else()
  target_compile_options(coremldelegate PRIVATE "-fno-rtti")
endif()

set(TARGET coremldelegate APPEND_STRING PROPERTY COMPILE_FLAGS
           "-x objective-c++"
)

set(TARGET coremldelegate APPEND_STRING PROPERTY COMPILE_FLAGS
           "-Wno-null-character"
)

set(TARGET coremldelegate APPEND_STRING PROPERTY COMPILE_FLAGS
           "-Wno-receiver-expr"
)

install(
  TARGETS coremldelegate
  DESTINATION lib
  INCLUDES
  DESTINATION ${_common_include_directories}
)

# We only care about building the pybinding when building for macOS wheels.
if(EXECUTORCH_BUILD_COREML AND EXECUTORCH_BUILD_PYBIND)
  if(NOT TARGET pybind11::pybind11)
    add_subdirectory(${EXECUTORCH_ROOT}/third-party/pybind11 ${CMAKE_CURRENT_BINARY_DIR}/pybind11)
  endif()

  pybind11_add_module(executorchcoreml SHARED runtime/inmemoryfs/inmemory_filesystem_py.cpp)

  target_compile_options(executorchcoreml PRIVATE -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET})
  if(CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    target_compile_options(executorchcoreml PRIVATE -g)
  endif()
  target_link_libraries(executorchcoreml PRIVATE coreml_util coreml_inmemoryfs)
endif()
