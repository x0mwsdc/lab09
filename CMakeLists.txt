cmake_minimum_required(VERSION 3.10)
project(print_project)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TESTS "Build tests" OFF)
option(ENABLE_COVERAGE "Enable code coverage metrics" OFF)

if(ENABLE_COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage -O0 -g")
endif()

include_directories(include)

add_library(print sources/print.cpp)

add_executable(example1 examples/example1.cpp)
target_link_libraries(example1 print)

add_executable(example2 examples/example2.cpp)
target_link_libraries(example2 print)

if(BUILD_TESTS)
  enable_testing()
  set(OLD_FLAGS "${CMAKE_CXX_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -w")
  add_subdirectory(third-party/gtest)
  set(CMAKE_CXX_FLAGS "${OLD_FLAGS}")
  
  add_executable(check tests/test1.cpp)
  target_link_libraries(check print gtest_main)
  add_test(NAME check COMMAND check)
endif()

install(TARGETS print DESTINATION lib)
install(TARGETS example1 example2 DESTINATION bin)
install(DIRECTORY include/ DESTINATION include)

set(CPACK_PACKAGE_NAME "print_project")
set(CPACK_PACKAGE_VERSION "1.0.0")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Simple C++ print library and examples")
set(CPACK_PACKAGE_CONTACT "x0mwsdc <danila@example.com>")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "x0mwsdc")
set(CPACK_GENERATOR "TGZ;DEB")
include(CPack)

find_package(Doxygen)
if(DOXYGEN_FOUND)
  add_custom_target(docs
    COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Generating API documentation with Doxygen"
    VERBATIM)
endif()
