cmake_minimum_required(VERSION 3.9)
set(CMAKE_GENERATOR_TOOLSET "host=x86")
set(CMAKE_GENERATOR_PLATFORM "Win32")
project(coolq-cpp-sdk)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_FLAGS "/utf-8 ${CMAKE_CXX_FLAGS}")  # use UTF-8 source files
set(CMAKE_CXX_FLAGS "/MP ${CMAKE_CXX_FLAGS}")  # build with object level parallelism
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")

include(cotire)
include(FindVcpkgIncludeDir)
include(FixDebugLibraryLookup)

include_directories(${VCPKG_INCLUDE_DIR})
include_directories(src)

add_compile_definitions(BOOST_CONFIG_SUPPRESS_OUTDATED_MESSAGE
                        _SILENCE_ALL_CXX17_DEPRECATION_WARNINGS
                        _WIN32_WINNT=0x0501
                        WIN32_LEAN_AND_MEAN
                        NOMINMAX)

# read app id from app_id.txt
file(READ "app_id.txt" APP_ID)
string(STRIP "${APP_ID}" APP_ID)
set(APP_ID "\"${APP_ID}\"")
add_compile_definitions(APP_ID=${APP_ID})

find_package(unofficial-iconv CONFIG REQUIRED)
find_package(SQLiteCpp CONFIG REQUIRED)
find_package(sqlite3 CONFIG REQUIRED)
find_package(cpprestsdk CONFIG REQUIRED)
find_package(unofficial-brotli CONFIG REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(ZLIB REQUIRED)

file(GLOB_RECURSE SOURCE_FILES src/*.cpp)
set(LIB_NAME "app")
add_library(${LIB_NAME} SHARED ${SOURCE_FILES})

target_link_libraries(${LIB_NAME} PRIVATE ZLIB::ZLIB)
target_link_libraries(${LIB_NAME} PRIVATE OpenSSL::SSL OpenSSL::Crypto)
target_link_libraries(${LIB_NAME} PRIVATE unofficial::brotli::brotlidec-static unofficial::brotli::brotlienc-static unofficial::brotli::brotlicommon-static)
target_link_libraries(${LIB_NAME} PRIVATE cpprestsdk::cpprest cpprestsdk::cpprestsdk_boost_internal)
target_link_libraries(${LIB_NAME} PRIVATE unofficial::iconv::libiconv unofficial::iconv::libcharset)
target_link_libraries(${LIB_NAME} PRIVATE sqlite3)
target_link_libraries(${LIB_NAME} PRIVATE SQLiteCpp)

cotire(${LIB_NAME})

add_custom_command(TARGET ${LIB_NAME}
                   POST_BUILD
                   COMMAND
                   powershell -ExecutionPolicy Bypass -NoProfile -File "${PROJECT_SOURCE_DIR}/scripts/post_build.ps1" ${APP_ID} ${LIB_NAME} "$<TARGET_FILE_DIR:${LIB_NAME}>")