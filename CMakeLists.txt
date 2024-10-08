cmake_minimum_required(VERSION 3.14...3.26)

# Read the version from fast_matrix_market.hpp
set(VERSION_HEADER_FILE include/fast_matrix_market/fast_matrix_market.hpp)
file(STRINGS ${VERSION_HEADER_FILE} VERSION_MAJOR_STR REGEX "define .*_VERSION_MAJOR")
file(STRINGS ${VERSION_HEADER_FILE} VERSION_MINOR_STR REGEX "define .*_VERSION_MINOR")
file(STRINGS ${VERSION_HEADER_FILE} VERSION_PATCH_STR REGEX "define .*_VERSION_PATCH")
string(REGEX MATCH "[0-9]+" FMM_MAJOR ${VERSION_MAJOR_STR})
string(REGEX MATCH "[0-9]+" FMM_MINOR ${VERSION_MINOR_STR})
string(REGEX MATCH "[0-9]+" FMM_PATCH ${VERSION_PATCH_STR})

project(fast_matrix_market VERSION "${FMM_MAJOR}.${FMM_MINOR}.${FMM_PATCH}" LANGUAGES CXX)
message("fast_matrix_market version ${PROJECT_VERSION}")

# Main
add_library(fast_matrix_market INTERFACE)
# add alias so the project can be used with add_subdirectory
add_library(fast_matrix_market::fast_matrix_market ALIAS fast_matrix_market)

target_include_directories(
        fast_matrix_market INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

target_compile_features(fast_matrix_market INTERFACE cxx_std_17)

# Some older compilers (eg. Clang, GCC <9) require extra link flags for C++11 threads
find_package(Threads)
target_link_libraries(fast_matrix_market INTERFACE Threads::Threads)

# temporarly disable portability reasons
# CMake options to enable/disable dependencies
option(FMM_USE_FAST_FLOAT "Enable fast_float float/double parser" OFF)
option(FMM_USE_DRAGONBOX "Enable dragonbox float/double formatter for shortest representation" OFF)
option(FMM_USE_RYU "Enable Ryu float/double formatter with precision support" OFF)

############################################
# Test for available versions of std::from_chars.
include(cmake/from_chars_tests.cmake)
include(cmake/to_chars_tests.cmake)

if (from_chars_int_supported)
    message("std::from_chars<int> detected.")
    target_compile_definitions(fast_matrix_market INTERFACE FMM_FROM_CHARS_INT_SUPPORTED)
else()
    message("std::from_chars<int> not detected.")
endif()

if (from_chars_double_supported)
    target_compile_definitions(fast_matrix_market INTERFACE FMM_FROM_CHARS_DOUBLE_SUPPORTED)
else()
    message("std::from_chars<double> not detected.")
endif()

if (from_chars_long_double_supported)
    target_compile_definitions(fast_matrix_market INTERFACE FMM_FROM_CHARS_LONG_DOUBLE_SUPPORTED)
else()
    message("std::from_chars<long double> not detected.")
endif()

###############################################
# Test for available versions of std::to_chars.

if (to_chars_int_supported)
    message("std::to_chars<int> detected.")
    target_compile_definitions(fast_matrix_market INTERFACE FMM_TO_CHARS_INT_SUPPORTED)
else()
    message("std::to_chars<int> not detected.")
endif()

if (to_chars_double_supported)
    message("std::to_chars<double> detected.")
    target_compile_definitions(fast_matrix_market INTERFACE FMM_TO_CHARS_DOUBLE_SUPPORTED)
else()
    message("std::to_chars<double> not detected.")
endif()

if (to_chars_long_double_supported)
    message("std::to_chars<long double> detected.")
    target_compile_definitions(fast_matrix_market INTERFACE FMM_TO_CHARS_LONG_DOUBLE_SUPPORTED)
else()
    message("std::to_chars<long double> not detected.")
endif()

###############################################
# Setup dependencies

# Setup fast_float.
# float and double parser.
if (FMM_USE_FAST_FLOAT)
    message("Using fast_float")
    set(FASTFLOAT_INSTALL OFF)
    add_subdirectory(dependencies/fast_float)

    target_compile_definitions(fast_matrix_market INTERFACE FMM_USE_FAST_FLOAT)
    target_link_libraries(fast_matrix_market INTERFACE fast_float)
endif()

# Setup Dragonbox.
# float and double formatter for shortest representation. No precision support.
if (FMM_USE_DRAGONBOX)
    message("Using Dragonbox")
    add_subdirectory(dependencies/dragonbox)

    target_compile_definitions(fast_matrix_market INTERFACE FMM_USE_DRAGONBOX)
    target_link_libraries(fast_matrix_market INTERFACE dragonbox::dragonbox_to_chars)
endif()

# Setup Ryu.
# float, double formatter with precision support. Slower than Dragonbox for shortest representation.
if (FMM_USE_RYU)
    message("Using ryu")
    add_subdirectory(dependencies/ryu)

    target_link_libraries(fast_matrix_market INTERFACE ryu::ryu)
    target_compile_definitions(fast_matrix_market INTERFACE FMM_USE_RYU)
endif()

###############################################

# Tests
option(FAST_MATRIX_MARKET_TEST "Enable tests" OFF)
if(FAST_MATRIX_MARKET_TEST)
    enable_testing()
    add_subdirectory(tests)
endif(FAST_MATRIX_MARKET_TEST)

# Benchmarks
option(FAST_MATRIX_MARKET_BENCH "Enable benchmarks" OFF)
if(FAST_MATRIX_MARKET_BENCH)
    add_subdirectory(benchmark)
endif(FAST_MATRIX_MARKET_BENCH)


# load cmake modules
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
# wrap function to install package, don't change it starting from here. 
function(configure_and_install_package PACKAGE_NAME VERSION)
    set(INCLUDE_INSTALL_DIR ${CMAKE_INSTALL_INCLUDEDIR})
    set(LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR})

    install(TARGETS ${PACKAGE_NAME}
            EXPORT ${PACKAGE_NAME}Targets
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

    install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp")

    install(EXPORT ${PACKAGE_NAME}Targets
            FILE ${PACKAGE_NAME}Targets.cmake
            NAMESPACE ${PACKAGE_NAME}::
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PACKAGE_NAME})

    configure_package_config_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/TemplateConfig.cmake.in
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
        PATH_VARS INCLUDE_INSTALL_DIR LIB_INSTALL_DIR
    )

    write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}ConfigVersion.cmake"
        VERSION ${VERSION}
        COMPATIBILITY AnyNewerVersion)

    install(FILES
            "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}Config.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}ConfigVersion.cmake"
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PACKAGE_NAME})
endfunction()
option(INSTALLPROJECT "Install the project" OFF)
if(INSTALLPROJECT)
    configure_and_install_package(${PROJECT_NAME} ${PROJECT_VERSION})
endif()