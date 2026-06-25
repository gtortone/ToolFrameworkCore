cmake_minimum_required(VERSION 3.20)
cmake_policy(SET CMP0144 NEW)
cmake_policy(SET CMP0167 OLD)

project(ToolFrameworkCore LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "build type" FORCE)
endif()

option(TF_BUILD_SHARED "build shared libraries (.so)" ON)
option(TF_BUILD_STATIC "build static libraries (.a)" ON)

if(NOT TF_BUILD_SHARED AND NOT TF_BUILD_STATIC)
    message(FATAL_ERROR
       "TF_BUILD_SHARED or TF_BUILD_STATIC must be ON.")
endif()

add_library(tf_project_options INTERFACE)
target_compile_options(tf_project_options INTERFACE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wno-unused-result>)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
include(ToolFrameworkDependencies)

set(TF_BOOST_COMPONENTS date_time serialization iostreams
    CACHE STRING "Boost libs required")

if(DEFINED BOOST_ROOT)
    message(STATUS "Using custom BOOST_ROOT: ${BOOST_ROOT}")
    set(BOOST_ROOT ${BOOST_ROOT} CACHE PATH "Boost lib root directory")
    set(Boost_NO_SYSTEM_PATHS ON)

    find_package(Boost REQUIRED
       COMPONENTS ${TF_BOOST_COMPONENTS}
   )
else()
   find_boost_dependencies(${TF_BOOST_COMPONENTS})
endif()

message(STATUS "Boost include: ${Boost_INCLUDE_DIRS}")
message(STATUS "Boost version: ${Boost_VERSION}")

include_directories(${Boost_INCLUDE_DIRS})

set(EXTERNAL_LIBS
    Boost::date_time
    Boost::serialization
    Boost::iostreams)

function(collect_header_dirs return_list)
    file(GLOB_RECURSE _headers ${PROJECT_SOURCE_DIR}/*.h)
    set(_dirs "")
    foreach(_h ${_headers})
        get_filename_component(_d ${_h} PATH)
        list(APPEND _dirs ${_d})
    endforeach()
    if(_dirs)
        list(REMOVE_DUPLICATES _dirs)
    endif()
    set(${return_list} ${_dirs} PARENT_SCOPE)
endfunction()

file(COPY ${PROJECT_SOURCE_DIR}/configfiles DESTINATION ${PROJECT_BINARY_DIR}/)

collect_header_dirs(header_list)
include_directories(${header_list})

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

function(add_tf_library name)
    set(_srcs ${ARGN})

    add_library(${name}_objs OBJECT ${_srcs})
    target_link_libraries(${name}_objs PRIVATE tf_project_options ${EXTERNAL_LIBS})

    if(TF_BUILD_SHARED)
        add_library(${name} SHARED $<TARGET_OBJECTS:${name}_objs>)
        target_link_libraries(${name} PUBLIC ${EXTERNAL_LIBS})
        set_target_properties(${name} PROPERTIES OUTPUT_NAME ${name})
    endif()

    if(TF_BUILD_STATIC)
        add_library(${name}_static STATIC $<TARGET_OBJECTS:${name}_objs>)
        target_link_libraries(${name}_static PUBLIC ${EXTERNAL_LIBS})
        set_target_properties(${name}_static PROPERTIES OUTPUT_NAME ${name})
    endif()
endfunction()

file(GLOB_RECURSE STORE_SRC RELATIVE ${CMAKE_SOURCE_DIR} "src/Store/*.cpp")
add_tf_library(Store ${STORE_SRC})

file(GLOB_RECURSE LOGGING_SRC RELATIVE ${CMAKE_SOURCE_DIR} "src/Logging/*.cpp")
add_tf_library(Logging ${LOGGING_SRC})

file(GLOB_RECURSE DATAMODELBASE_SRC RELATIVE ${CMAKE_SOURCE_DIR} "src/DataModelBase/*.cpp")
add_tf_library(DataModelBase ${DATAMODELBASE_SRC})

file(GLOB_RECURSE TEMPDATAMODEL_SRC RELATIVE ${CMAKE_SOURCE_DIR} "DataModel/*.cpp")
add_tf_library(TempDataModel ${TEMPDATAMODEL_SRC})

file(GLOB_RECURSE TEMPTOOLS_SRC RELATIVE ${CMAKE_SOURCE_DIR} "UserTools/*.cpp")
add_tf_library(TempTools ${TEMPTOOLS_SRC})

file(GLOB_RECURSE TOOLCHAIN_SRC RELATIVE ${CMAKE_SOURCE_DIR} "src/ToolChain/*.cpp")
add_tf_library(ToolChain ${TOOLCHAIN_SRC})

set(DEP_ToolChain Store Logging DataModelBase)
set(DEP_TempDataModel DataModelBase)
set(DEP_TempTools Store Logging DataModelBase ToolChain)

foreach(_lib ToolChain TempDataModel TempTools)
    foreach(_variant "" "_static")
        if(TARGET ${_lib}${_variant})
            set(_deps "")
            foreach(_d ${DEP_${_lib}})
                list(APPEND _deps ${_d}${_variant})
            endforeach()
            target_link_libraries(${_lib}${_variant} PUBLIC ${_deps})
        endif()
    endforeach()
endforeach()

if(TF_BUILD_SHARED)
    set(_tf_suffix "")
else()
    set(_tf_suffix "_static")
endif()

set(TF_CORE_LIBS
    Store${_tf_suffix}
    Logging${_tf_suffix}
    DataModelBase${_tf_suffix}
    ToolChain${_tf_suffix}
    TempTools${_tf_suffix}
    TempDataModel${_tf_suffix})

add_executable(main ${PROJECT_SOURCE_DIR}/src/main.cpp)

target_link_options(main PRIVATE -Wl,--no-as-needed -Wl,--allow-shlib-undefined)

target_link_libraries(main
    tf_project_options
    ${TF_CORE_LIBS}
    pthread
    ${EXTERNAL_LIBS}
    ${DATAMODEL_LIBS}
    ${MYTOOLS_LIBS})

