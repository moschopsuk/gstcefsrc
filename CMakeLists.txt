cmake_minimum_required(VERSION 2.8.12.1)

set(CMAKE_CONFIGURATION_TYPES Debug Release)

project(gstcef)

set_property(GLOBAL PROPERTY OS_FOLDERS ON)

set(CEF_VERSION "3.3497.1836.gb472a8d")

# Determine the platform.
if("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  set(CEF_PLATFORM "macosx64")
  set(PROJECT_ARCH "x86_64")
  set(CMAKE_BUILD_TYPE "Release")
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  if(CMAKE_SIZEOF_VOID_P MATCHES 8)
    set(CEF_PLATFORM "linux64")
  else()
    set(CEF_PLATFORM "linux32")
  endif()
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Windows")
  if(CMAKE_SIZEOF_VOID_P MATCHES 8)
    set(CEF_PLATFORM "windows64")
  else()
    set(CEF_PLATFORM "windows32")
  endif()
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(DownloadCEF)
DownloadCEF("${CEF_PLATFORM}" "${CEF_VERSION}" "${CMAKE_SOURCE_DIR}/third_party/cef")
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CEF_ROOT}/cmake")

find_package(CEF REQUIRED)
add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)

set(GSTCEF_SRCS
  gstcef.cc
)

set(GSTCEFSUBPROCESS_SRCS
  gstcefsubprocess.cc
)

find_package(PkgConfig)

pkg_check_modules(GST REQUIRED gstreamer-1.0
                  gstreamer-video-1.0
		  gio-2.0)

ADD_LOGICAL_TARGET("libcef_lib" "${CEF_LIB_DEBUG}" "${CEF_LIB_RELEASE}")

SET_CEF_TARGET_OUT_DIR()

if(OS_LINUX)
  link_directories(${GST_LIBRARY_DIRS})

  add_executable("gstcefsubprocess" ${GSTCEFSUBPROCESS_SRCS})
  SET_EXECUTABLE_TARGET_PROPERTIES("gstcefsubprocess")
  target_link_libraries("gstcefsubprocess" libcef_lib libcef_dll_wrapper ${CEF_STANDARD_LIBS})
  set_target_properties("gstcefsubprocess" PROPERTIES INSTALL_RPATH "$ORIGIN")
  set_target_properties("gstcefsubprocess" PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)
  set_target_properties("gstcefsubprocess" PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CEF_TARGET_OUT_DIR})

  add_library("gstcef" SHARED ${GSTCEF_SRCS})
  SET_LIBRARY_TARGET_PROPERTIES("gstcef")
  add_dependencies("gstcef" libcef_dll_wrapper)
  target_link_libraries("gstcef" libcef_lib libcef_dll_wrapper ${CEF_STANDARD_LIBS} ${GST_LIBRARIES})
  target_include_directories("gstcef" PUBLIC ${GST_INCLUDE_DIRS})
  target_compile_definitions("gstcef" PUBLIC "-DCEF_SUBPROCESS_PATH=\"$<TARGET_FILE:gstcefsubprocess>\"")
  set_target_properties("gstcef" PROPERTIES INSTALL_RPATH "$ORIGIN")
  set_target_properties("gstcef" PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)
  set_target_properties("gstcef" PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CEF_TARGET_OUT_DIR})


  COPY_FILES("gstcef" "${CEF_BINARY_FILES}" "${CEF_BINARY_DIR}" "${CEF_TARGET_OUT_DIR}")
  COPY_FILES("gstcef" "${CEF_RESOURCE_FILES}" "${CEF_RESOURCE_DIR}" "${CEF_TARGET_OUT_DIR}")
endif()


if(OS_MACOSX)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Werror -stdlib=libc++ -std=c++11 -Wno-inconsistent-missing-override")

  link_directories(${GST_LIBRARY_DIRS})

  add_executable("gstcefsubprocess" ${GSTCEFSUBPROCESS_SRCS})
  SET_EXECUTABLE_TARGET_PROPERTIES("gstcefsubprocess")
  target_link_libraries("gstcefsubprocess" libcef_lib libcef_dll_wrapper ${CEF_STANDARD_LIBS})
  set_target_properties("gstcefsubprocess" PROPERTIES INSTALL_RPATH "$ORIGIN")
  set_target_properties("gstcefsubprocess" PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)
  set_target_properties("gstcefsubprocess" PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CEF_TARGET_OUT_DIR})

  add_library("gstcef" SHARED ${GSTCEF_SRCS})
  SET_LIBRARY_TARGET_PROPERTIES("gstcef")
  add_dependencies("gstcef" libcef_dll_wrapper)
  target_link_libraries("gstcef" libcef_lib libcef_dll_wrapper ${CEF_STANDARD_LIBS} ${GST_LIBRARIES})
  target_include_directories("gstcef" PUBLIC ${GST_INCLUDE_DIRS})
  target_compile_definitions("gstcef" PUBLIC "-DCEF_SUBPROCESS_PATH=\"$<TARGET_FILE:gstcefsubprocess>\"")
  set_target_properties("gstcef" PROPERTIES INSTALL_RPATH "$ORIGIN")
  set_target_properties("gstcef" PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)
  set_target_properties("gstcef" PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CEF_TARGET_OUT_DIR})

  COPY_FILES("gstcef" "${CEF_BINARY_FILES}" "${CEF_BINARY_DIR}" "${CEF_TARGET_OUT_DIR}")
  COPY_FILES("gstcef" "${CEF_RESOURCE_FILES}" "${CEF_RESOURCE_DIR}" "${CEF_TARGET_OUT_DIR}")

  add_custom_command(
    TARGET gstcef
    POST_BUILD
    COMMAND install_name_tool -change
    "@rpath/Frameworks/Chromium Embedded Framework.framework/Chromium Embedded Framework"
    "/usr/local/Frameworks/Chromium Embedded Framework.framework/Chromium Embedded Framework"
    "$<TARGET_FILE:gstcef>"
  )

  add_custom_command(
    TARGET gstcefsubprocess
    POST_BUILD
    COMMAND install_name_tool -change
    "@rpath/Frameworks/Chromium Embedded Framework.framework/Chromium Embedded Framework"
    "/usr/local/Frameworks/Chromium Embedded Framework.framework/Chromium Embedded Framework"
    "$<TARGET_FILE:gstcefsubprocess>"
  )

  install(
    TARGETS gstcef
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_GST_PLUGINS_DIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
  
  install(
    TARGETS gstcefsubprocess
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_GST_PLUGINS_DIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
endif()