cmake_minimum_required(VERSION 2.8.12)

add_subdirectory(python2)

if(MSVC)
    add_subdirectory(boost)
endif()