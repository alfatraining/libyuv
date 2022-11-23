# CMakeLists for libyuv
# Originally created for "roxlu build system" to compile libyuv on windows
# Run with -DTEST=ON to build unit tests

CMAKE_MINIMUM_REQUIRED(VERSION 3.15)
project(libyuv VERSION 1850)
option(BUILD_TEST "Built unit tests" OFF)
option(BUILD_CONVERT "Build convert executable" OFF)
option(USE_LIBJPEG "Use libjpeg" ON)

SET ( ly_base_dir	${PROJECT_SOURCE_DIR} )
SET ( ly_src_dir	${ly_base_dir}/source )
SET ( ly_inc_dir	${ly_base_dir}/include )
SET ( ly_tst_dir	${ly_base_dir}/unit_test )

FILE ( GLOB_RECURSE	ly_source_files ${ly_src_dir}/*.cc )
LIST ( SORT			ly_source_files )

FILE ( GLOB_RECURSE	ly_header_files ${ly_src_dir}/*.h )
LIST ( SORT			ly_header_files )

FILE ( GLOB_RECURSE	ly_unittest_sources ${ly_tst_dir}/*.cc )
LIST ( SORT			ly_unittest_sources )

INCLUDE_DIRECTORIES( BEFORE ${ly_inc_dir} )

if(MSVC)
  ADD_DEFINITIONS ( -D_CRT_SECURE_NO_WARNINGS )
endif()

# this creates the static library (.a)
add_library(libyuv STATIC ${ly_source_files} ${ly_header_files})
set_target_properties(libyuv PROPERTIES
    COMPILE_DEFINITIONS "_CRT_SECURE_NO_WARNINGS"
    INCLUDE_DIRECTORIES "${ly_inc_dir}"
    INTERFACE_INCLUDE_DIRECTORIES "$<INSTALL_INTERFACE:include>;$<BUILD_INTERFACE:${ly_inc_dir}>"
    SOVERSION ${PROJECT_VERSION_MAJOR}
    IMPORT_PREFIX ""
    IMPORT_SUFFIX _imp${CMAKE_IMPORT_LIBRARY_SUFFIX}
    PREFIX "")

# this creates the shared library (.so)
ADD_LIBRARY				( ${ly_lib_shared} SHARED ${ly_source_files} )
SET_TARGET_PROPERTIES	( ${ly_lib_shared} PROPERTIES OUTPUT_NAME "${ly_lib_name}" )
SET_TARGET_PROPERTIES	( ${ly_lib_shared} PROPERTIES PREFIX "lib" )
if(WIN32)
  SET_TARGET_PROPERTIES	( ${ly_lib_shared} PROPERTIES IMPORT_PREFIX "lib" )
endif()

if(BUILD_CONVERT)
    add_executable(convert ${ly_base_dir}/util/convert.cc)
    target_link_libraries(convert libyuv)

	add_executable(yuvconstants ${ly_base_dir}/util/yuvconstants.c )
	target_link_libraries(yuvconstants libyuv} )
endif()

if(USE_LIBJPEG)
    include(FindJPEG)
    if (JPEG_FOUND)
        include_directories(${JPEG_INCLUDE_DIR})
        if (BUILD_CONVERT)
            target_link_libraries(convert ${JPEG_LIBRARY})
        endif()
        add_definitions(-DHAVE_JPEG)
    endif()
endif()

if(BUILD_TEST)
  find_library(GTEST_LIBRARY gtest)
  if(GTEST_LIBRARY STREQUAL "GTEST_LIBRARY-NOTFOUND")
    set(GTEST_SRC_DIR /usr/src/gtest CACHE STRING "Location of gtest sources")
    if(EXISTS ${GTEST_SRC_DIR}/src/gtest-all.cc)
      message(STATUS "building gtest from sources in ${GTEST_SRC_DIR}")
      set(gtest_sources ${GTEST_SRC_DIR}/src/gtest-all.cc)
      add_library(gtest STATIC ${gtest_sources})
      include_directories(${GTEST_SRC_DIR})
      include_directories(${GTEST_SRC_DIR}/include)
      set(GTEST_LIBRARY gtest)
    else()
      message(FATAL_ERROR "TEST is set but unable to find gtest library")
    endif()
  endif()

  add_executable(libyuv_unittest ${ly_unittest_sources})
  target_link_libraries(libyuv_unittest libyuv ${GTEST_LIBRARY})
  find_library(PTHREAD_LIBRARY pthread)
  if(NOT PTHREAD_LIBRARY STREQUAL "PTHREAD_LIBRARY-NOTFOUND")
    target_link_libraries(libyuv_unittest pthread)
  endif()
  if (USE_LIBJPEG AND JPEG_FOUND)
    target_link_libraries(libyuv_unittest ${JPEG_LIBRARY})
  endif()

  if(NACL AND NACL_LIBC STREQUAL "newlib")
    target_link_libraries(libyuv_unittest glibc-compat)
  endif()

  find_library(GFLAGS_LIBRARY gflags)
  if(NOT GFLAGS_LIBRARY STREQUAL "GFLAGS_LIBRARY-NOTFOUND")
    target_link_libraries(libyuv_unittest gflags)
    add_definitions(-DLIBYUV_USE_GFLAGS)
  endif()
endif()


# install the conversion tool, .so, .a, and all the header files
if (BUILD_CONVERT)
	install ( PROGRAMS ${CMAKE_BINARY_DIR}/yuvconvert DESTINATION bin )
endif()

install(TARGETS libyuv
    EXPORT ${CMAKE_PROJECT_NAME}Config
    LIBRARY DESTINATION lib COMPONENT dev
    ARCHIVE DESTINATION lib COMPONENT dev
    RUNTIME DESTINATION bin COMPONENT bin)

install ( DIRECTORY ${PROJECT_SOURCE_DIR}/include/ DESTINATION include )

install(EXPORT ${CMAKE_PROJECT_NAME}Config
    DESTINATION cmake)

# create the .deb and .rpm packages using cpack
INCLUDE ( CM_linux_packages.cmake )

