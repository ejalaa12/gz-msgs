##################################################
# Build a custom protoc plugin
gz_add_executable(gz_msgs_gen Generator.cc generator_main.cc)
target_link_libraries(gz_msgs_gen
  protobuf::libprotoc
  protobuf::libprotobuf)
target_include_directories(gz_msgs_gen PRIVATE ${PROTOBUF_INCLUDE_DIR})
target_compile_features(gz_msgs_gen PRIVATE ${GZ_CXX_11_FEATURES})

if (UNIX)
  target_link_libraries(gz_msgs_gen pthread)
endif()

if(INSTALL_GZ_MSGS_GEN_EXECUTABLE)
  set_target_properties(gz_msgs_gen PROPERTIES VERSION ${PROJECT_VERSION_FULL})
  install(TARGETS gz_msgs_gen DESTINATION ${GZ_BIN_INSTALL_DIR})

  # TODO(chapulina) Deprecated. Remove on v10.
  install(FILES $<TARGET_FILE:gz_msgs_gen> DESTINATION ${GZ_BIN_INSTALL_DIR} RENAME ign_msgs_gen PERMISSIONS OWNER_EXECUTE)
endif()

##################################################
# A function that calls protoc on a protobuf file
# Options:
#   GENERATE_PYTHON       - generates python code for the message if specified
#   GENERATE_CPP        - generates c++ code for the message if specified
# One value arguments:
#   PROTO_PACKAGE       - Protobuf package the file belongs to (e.g. ".gz.msgs")
#   PROTOC_EXEC         - Path to protoc
#   INPUT_PROTO         - Path to the input .proto file
#   OUTPUT_CPP_DIR      - Path where C++ files are saved
#   OUTPUT_PYTHON_DIR     - Path where Python files are saved
#   OUTPUT_INCLUDES     - A CMake variable name containing a list that the C++ header path should be appended to
#   OUTPUT_CPP_HH_VAR   - A CMake variable name containing a list that the C++ header path should be appended to
#   OUTPUT_GZ_CPP_HH_VAR - A CMake variable name containing a list that the C++ header path should be appended to
#   OUTPUT_CPP_CC_VAR   - A Cmake variable name containing a list that the C++ source path should be appended to
#   OUTPUT_PYTHON_VAR     - A Cmake variable name containing a list that the python file should be appended to
# Multi value arguments
#   PROTO_PATH          - Passed to protoc --proto_path
function(gz_msgs_protoc)
  set(options GENERATE_PYTHON GENERATE_CPP)
  set(oneValueArgs
    PROTO_PACKAGE
    PROTOC_EXEC
    INPUT_PROTO
    OUTPUT_CPP_DIR
    OUTPUT_PYTHON_DIR
    OUTPUT_INCLUDES
    OUTPUT_CPP_HH_VAR
    OUTPUT_GZ_CPP_HH_VAR
    OUTPUT_DETAIL_CPP_HH_VAR
    OUTPUT_CPP_CC_VAR
    OUTPUT_PYTHON_VAR)
  set(multiValueArgs PROTO_PATH)

  cmake_parse_arguments(gz_msgs_protoc "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

  get_filename_component(ABS_FIL ${gz_msgs_protoc_INPUT_PROTO} ABSOLUTE)
  get_filename_component(FIL_WE ${gz_msgs_protoc_INPUT_PROTO} NAME_WE)

  set(protoc_args)
  set(output_files)

  set(proto_package_dir ".")
  if(gz_msgs_protoc_PROTO_PACKAGE)
    string(REPLACE "." "/" proto_package_dir ${gz_msgs_protoc_PROTO_PACKAGE})
  endif()

  if(gz_msgs_protoc_GENERATE_CPP)
    # Full path to generated header (${PROJECT_BINARY_DIR}/include/gz/msgs/foo.pb.h)
    set(output_header "${gz_msgs_protoc_OUTPUT_CPP_DIR}${proto_package_dir}/${FIL_WE}.pb.h")
    # Full path to generated detail header (${PROJECT_BINARY_DIR}/include/gz/msgs/details/foo.pb.h)
    set(output_detail_header "${gz_msgs_protoc_OUTPUT_CPP_DIR}${proto_package_dir}/details/${FIL_WE}.pb.h")
    # Full path to generated source (${PROJECT_BINARY_DIR}/include/foo.pb.cc)
    set(output_source "${gz_msgs_protoc_OUTPUT_CPP_DIR}${proto_package_dir}/${FIL_WE}.pb.cc")

    # Generate a clean relative path (gz/msgs/foo.pb.h)
    string(REPLACE "${PROJECT_BINARY_DIR}/include/" "" output_include ${output_header})
    list(APPEND ${gz_msgs_protoc_OUTPUT_INCLUDES} "${output_include}")

    list(APPEND ${gz_msgs_protoc_OUTPUT_CPP_HH_VAR} ${output_header})
    list(APPEND ${gz_msgs_protoc_OUTPUT_CPP_CC_VAR} ${output_source})
    list(APPEND ${gz_msgs_protoc_OUTPUT_GZ_CPP_HH_VAR} ${output_ign_header})
    list(APPEND ${gz_msgs_protoc_OUTPUT_DETAIL_CPP_HH_VAR} ${output_detail_header})

    list(APPEND output_files ${output_header})
    list(APPEND output_files ${output_detail_header})
    list(APPEND output_files ${output_ign_header})
    list(APPEND output_files ${output_source})

    set(${gz_msgs_protoc_OUTPUT_INCLUDES} ${${gz_msgs_protoc_OUTPUT_INCLUDES}} PARENT_SCOPE)
    set(${gz_msgs_protoc_OUTPUT_GZ_CPP_HH_VAR} ${${gz_msgs_protoc_OUTPUT_GZ_CPP_HH_VAR}} PARENT_SCOPE)
    set(${gz_msgs_protoc_OUTPUT_DETAIL_CPP_HH_VAR} ${${gz_msgs_protoc_OUTPUT_DETAIL_CPP_HH_VAR}} PARENT_SCOPE)
    set(${gz_msgs_protoc_OUTPUT_CPP_HH_VAR} ${${gz_msgs_protoc_OUTPUT_CPP_HH_VAR}} PARENT_SCOPE)
    set(${gz_msgs_protoc_OUTPUT_CPP_CC_VAR} ${${gz_msgs_protoc_OUTPUT_CPP_CC_VAR}} PARENT_SCOPE)
  endif()

  if(gz_msgs_protoc_GENERATE_PYTHON)
    file(MAKE_DIRECTORY ${gz_msgs_protoc_OUTPUT_PYTHON_DIR})
    # Note: Both proto2 and proto3 use the _pb2.py suffix (https://protobuf.dev/reference/python/python-generated/#invocation)
    set(output_python "${gz_msgs_protoc_OUTPUT_PYTHON_DIR}${proto_package_dir}/${FIL_WE}_pb2.py")
    list(APPEND ${gz_msgs_protoc_OUTPUT_PYTHON_VAR} ${output_python})
    list(APPEND output_files ${output_python})
    set(${gz_msgs_protoc_OUTPUT_PYTHON_VAR} ${${gz_msgs_protoc_OUTPUT_PYTHON_VAR}} PARENT_SCOPE)
  endif()


  set(GENERATE_ARGS
      --protoc-exec "$<TARGET_FILE:${gz_msgs_protoc_PROTOC_EXEC}>"
      --gz-generator-bin "${GZ_MSGS_GEN_EXECUTABLE}"
      --proto-path "${gz_msgs_protoc_PROTO_PATH}"
      --input-path "${ABS_FIL}"
  )

  if(${gz_msgs_protoc_GENERATE_CPP})
    list(APPEND GENERATE_ARGS
      --generate-cpp
      --output-cpp-path "${gz_msgs_protoc_OUTPUT_CPP_DIR}")
  endif()

  if(${gz_msgs_protoc_GENERATE_PYTHON})
    list(APPEND GENERATE_ARGS
      --generate-python
      --output-python-path "${gz_msgs_protoc_OUTPUT_PYTHON_DIR}")
  endif()

  add_custom_command(
    OUTPUT ${output_files}
    COMMAND Python3::Interpreter
    ARGS tools/gz_msgs_generate.py ${GENERATE_ARGS}
    DEPENDS
      ${ABS_FIL}
      gz_msgs_gen
    # While the script is executed in the source directory, it does not write
    # to the source tree.  All outputs are stored in the build directory.
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMENT "Running protoc on ${gz_msgs_protoc_INPUT_PROTO}"
    VERBATIM
  )

endfunction()


##################################################
file (GLOB proto_files ${PROJECT_SOURCE_DIR}/proto/gz/msgs/*.proto)

foreach(proto_file ${proto_files})
  gz_msgs_protoc(
    PROTO_PACKAGE
      .gz.msgs
    GENERATE_CPP
    GENERATE_PYTHON
    INPUT_PROTO
      ${proto_file}
    PROTOC_EXEC
      protobuf::protoc
    OUTPUT_CPP_DIR
      "${PROJECT_BINARY_DIR}/include"
    OUTPUT_PYTHON_DIR
      "${PROJECT_BINARY_DIR}/python"
    OUTPUT_INCLUDES
      gen_includes
    OUTPUT_CPP_HH_VAR
      gen_headers
    OUTPUT_DETAIL_CPP_HH_VAR
      gen_detail_headers
    OUTPUT_GZ_CPP_HH_VAR
      gen_ign_headers
    OUTPUT_CPP_CC_VAR
      gen_sources
    OUTPUT_PYTHON_VAR
      gen_python_scripts
    PROTO_PATH
      "${PROJECT_SOURCE_DIR}/proto")
endforeach()

if(NOT MSVC)
  # -Wno-switch-default flags is required for suppressing a warning in some of
  # the generated protobuf files.
  set_source_files_properties(${gen_sources} COMPILE_FLAGS -Wno-switch-default)
endif()

if(MSVC)
  # Warning #4251 is the "dll-interface" warning that tells you when types used
  # by a class are not being exported. These generated source files have private
  # members that don't get exported, so they trigger this warning. However, the
  # warning is not important since those members do not need to be interfaced
  # with.
  set_source_files_properties(${gen_sources} COMPILE_FLAGS "/wd4251 /wd4146")
  # Fix for protobuf 3.12 - allow big object files
  add_definitions(/bigobj)
endif()

set_source_files_properties(${gen_headers} ${gen_ign_headers} ${gen_detail_headers} ${gen_sources} ${gen_python_scripts}
  PROPERTIES GENERATED TRUE)

message(STATUS "Installing Python messages to ${GZ_PYTHON_INSTALL_PATH}/gz/${GZ_DESIGNATION}${PROJECT_VERSION_MAJOR}")
install(FILES ${gen_python_scripts} DESTINATION ${GZ_PYTHON_INSTALL_PATH}/gz/${GZ_DESIGNATION}${PROJECT_VERSION_MAJOR})

# Install gz/msgs
gz_install_includes(
  "${GZ_INCLUDE_INSTALL_DIR_POSTFIX}/gz/${GZ_DESIGNATION}"
  ${gen_headers})

# Install gz/msgs/details
gz_install_includes(
  "${GZ_INCLUDE_INSTALL_DIR_POSTFIX}/gz/${IGN_DESIGNATION}/details"
  ${gen_detail_headers})

##################################################
# Generate gz/msgs/MessageTypes.hh
foreach (hdr ${gen_includes})
  string(CONCAT gz_msgs_headers ${gz_msgs_headers} "#include <${hdr}>\n")
endforeach()

configure_file (${CMAKE_CURRENT_SOURCE_DIR}/MessageTypes.hh.in
  ${PROJECT_BINARY_DIR}/include/gz/msgs/MessageTypes.hh)

gz_install_includes(
  "${GZ_INCLUDE_INSTALL_DIR_POSTFIX}/gz/${GZ_DESIGNATION}"
  "${PROJECT_BINARY_DIR}/include/gz/${GZ_DESIGNATION}/MessageTypes.hh")

##################################################
# Build the main library
gz_create_core_library(SOURCES
  ${gen_sources}
  ${PROJECT_SOURCE_DIR}/src/Factory.cc
  ${PROJECT_SOURCE_DIR}/src/Filesystem.cc
  ${PROJECT_SOURCE_DIR}/src/gz.cc
  ${PROJECT_SOURCE_DIR}/src/Utility.cc
)

target_link_libraries(${PROJECT_LIBRARY_TARGET_NAME}
  PUBLIC
    protobuf::libprotobuf
    gz-math${GZ_MATH_VER}::gz-math${GZ_MATH_VER}
  PRIVATE
    TINYXML2::TINYXML2
)

if (${Protobuf_VERSION} VERSION_LESS 3.12.0)
  # Older versions of protobuf (eg on focal) will throw up float-equal errors
  # Fixed via: https://github.com/protocolbuffers/protobuf/pull/6000
  target_compile_options(${PROJECT_LIBRARY_TARGET_NAME} PRIVATE -Wno-float-equal)
endif()

target_include_directories(${PROJECT_LIBRARY_TARGET_NAME}
  SYSTEM PUBLIC $<TARGET_PROPERTY:protobuf::libprotobuf,INTERFACE_INCLUDE_DIRECTORIES>)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  # Disable warning in generated *.pb.cc code
  target_compile_options(${PROJECT_LIBRARY_TARGET_NAME} PRIVATE -Wno-invalid-offsetof)
endif()

target_include_directories(${PROJECT_LIBRARY_TARGET_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
    $<INSTALL_INTERFACE:${INCLUDE_INSTALL_DIR_FULL}>)

##################################################
# Build unit tests
gz_get_libsources_and_unittests(sources gtest_sources)

# Build the unit tests.
gz_build_tests(TYPE UNIT
  SOURCES
    ${gtest_sources}
  LIB_DEPS
      TINYXML2::TINYXML2
)

if (TARGET UNIT_Factory_TEST)
  target_compile_definitions(UNIT_Factory_TEST
    PRIVATE GZ_MSGS_TEST_PATH="${PROJECT_SOURCE_DIR}/test")
endif()

##################################################
# gz msgs command
if(NOT WIN32)
  add_subdirectory(cmd)
endif()
