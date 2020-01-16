cmake_minimum_required(VERSION 3.15)

project (openpnp-capture-test)

# add include directory 
include_directories(../include ..)

set (SOURCE main.cpp ../../common/logging.cpp)

add_executable(openpnp-capture-test ${SOURCE})

target_link_libraries(openpnp-capture-test openpnp-capture)