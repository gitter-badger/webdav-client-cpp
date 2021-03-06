cmake_minimum_required(VERSION 2.8)
project(webdavclient)

add_definitions("-std=c++11")

option(BUILD_SHARED_LIBS "Build shared instead of static library" ON)
option(STATIC_RUNTIME_LIBRARY "Static runtime library" OFF)
option(UNICODE "Character set is UNICODE" ON)
option(BUILD_TESTS_FOR_${PROJECT_NAME} "Build tests" OFF)
option(USE_CPM "Use CPM" OFF)

if(STATIC_RUNTIME_LIBRARY)
    set(CompilerFlags
                CMAKE_CXX_FLAGS
                CMAKE_CXX_FLAGS_DEBUG
                CMAKE_CXX_FLAGS_RELEASE
                CMAKE_C_FLAGS
                CMAKE_C_FLAGS_DEBUG
                CMAKE_C_FLAGS_RELEASE)
    foreach(CompilerFlag ${CompilerFlags})
        string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
    endforeach()
endif()

if(USE_CPM)
include(cmake/FindCPM.cmake)
CPM_AddModule("Pugixml"
  GIT_REPOSITORY "https://github.com/designerror/cpm-pugixml")
include_directories(${CPM_INCLUDE_DIRS})
else()
find_package(pugixml REQUIRED)
include_directories(${pugixml_INCLUDE_DIRS})
endif()

if(UNICODE)
    add_definitions(-D_UNICODE -DUNICODE)
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include/)
file(GLOB ${PROJECT_NAME}_sources "${CMAKE_CURRENT_SOURCE_DIR}/source/*.cpp")

set(${PROJECT_NAME}_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include/)

if(WIN32 AND MSVC AND NOT BUILD_SHARED_LIBS)
    SET(CMAKE_MODULE_PATH "${CMAKE_ROOT}/Modules")
    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
    include(FindStaticOpenSSL)
else()
    find_package(OpenSSL)
endif()
include_directories(${OPENSSL_INCLUDE_DIR})

find_package(CURL REQUIRED)
include_directories(${CURL_INCLUDE_DIR})

if(BUILD_SHARED_LIBS)
    add_library(${PROJECT_NAME} SHARED ${${PROJECT_NAME}_sources} ${${PROJECT_NAME}_headers})
else()
    add_definitions(-DCURL_STATICLIB)
    add_library(${PROJECT_NAME} STATIC ${${PROJECT_NAME}_sources} ${${PROJECT_NAME}_headers})
endif()

if(USE_CPM)
if(UNIX)
    target_link_libraries(${PROJECT_NAME} pthread ${CPM_LIBRARIES} ${OPENSSL_LIBRARIES} ${CURL_LIBRARY})
else()
    target_link_libraries(${PROJECT_NAME} ws2_32 ${CPM_LIBRARIES} ${OPENSSL_LIBRARIES} ${CURL_LIBRARY})
endif()
else()
if(UNIX)
    target_link_libraries(${PROJECT_NAME} pthread pugixml ${OPENSSL_LIBRARIES} ${CURL_LIBRARY})
else()
    target_link_libraries(${PROJECT_NAME} ws2_32 pugixml ${OPENSSL_LIBRARIES} ${CURL_LIBRARY})
endif()
endif()

install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Config
    ARCHIVE DESTINATION lib/static
    LIBRARY DESTINATION lib
)
install(DIRECTORY ${${PROJECT_NAME}_INCLUDE_DIRS} DESTINATION include)

install(EXPORT ${PROJECT_NAME}Config DESTINATION cmake)

configure_file(${PROJECT_SOURCE_DIR}/cmake/ProjectConfig-include.cmake.in
               ${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}Config-include.cmake)

install(FILES ${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}Config-include.cmake 
        DESTINATION cmake)

if(BUILD_TESTS_FOR_${PROJECT_NAME})
    add_subdirectory(tests)
endif()
