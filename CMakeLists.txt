CMAKE_MINIMUM_REQUIRED(VERSION 3.7)

PROJECT(
    carma
    VERSION 0.3.0
    LANGUAGES CXX
)

# ##############################################################################
#                                   SETTINGS                                   #
# ##############################################################################
SET(CMAKE_CXX_STANDARD 14)
SET(DEFAULT_BUILD_TYPE "RELEASE")
IF(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  MESSAGE(STATUS "Setting build type to '${DEFAULT_BUILD_TYPE}' as none was specified.")
  SET(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
ENDIF()

OPTION(BUILD_TESTS "Build tests for carma" OFF)
OPTION(BUILD_EXAMPLES "Build examples for carma" OFF)
# ##############################################################################
#                                 DEFINITIONS                                  #
# ##############################################################################
#ADD_DEFINITIONS(-DARMA_DONT_USE_WRAPPER)

# Don't copy arrays if the data is not owned by Numpy
# ADD_DEFINITIONS(-DCARMA_DONT_REQUIRE_OWNDATA)

# Don't copy c-style arrays definition
# ADD_DEFINITIONS(-DCARMA_DONT_REQUIRE_F_CONTIGUOUS)

# ##############################################################################
#                                   TARGET                                     #
# ##############################################################################
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" 
        OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang" 
        OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    SET(PROJECT_RELEASE_FLAGS "-march=native" "-mtune=native")
elseif (CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    # workaround like https://github.com/nlohmann/json/issues/1408
    # to avoid error like: carma\third_party\armadillo-code\include\armadillo_bits/arma_str.hpp(194): error C2039: '_snprintf': is not a member of 'std' (compiling source file carma\tests\src\bindings.cpp) 
    ADD_DEFINITIONS(-DHAVE_SNPRINTF)
endif ()
SET(PROJECT_RELEASE_DEFINITIONS ARMA_NO_DEBUG)
# see linkg for a more expensive approach https://github.com/VcDevel/Vc/blob/master/cmake/OptimizeForArchitecture.cmake

if (ENABLE_COVERAGE)
# --coverage option is used to compile and link code instrumented for coverage analysis.
# The option is a synonym for -fprofile-arcs -ftest-coverage (when compiling) and -lgcov (when linking).
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} --coverage")
endif()

ADD_LIBRARY(carma INTERFACE)
TARGET_LINK_LIBRARIES(carma INTERFACE pybind11 armadillo)
TARGET_INCLUDE_DIRECTORIES(carma
        INTERFACE
        # path of the headers after installation
        $<INSTALL_INTERFACE:include> # <prefix>/include/carma
        # path of the headers before installation
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
        )
INSTALL(DIRECTORY "${PROJECT_SOURCE_DIR}/include" 
        DESTINATION .
        FILES_MATCHING PATTERN "*.h")

if (VALGRIND_TEST_WRAPPER)
    if (WIN32)
        MESSAGE(FATAL_ERROR "VALGRIND_TEST_WRAPPER is not supported in WIN32 environment")
    endif()
    find_program(VALGRIND_EXECUTABLE valgrind)
    if (NOT VALGRIND_EXECUTABLE)
        MESSAGE(FATAL_ERROR "VALGRIND_TEST_WRAPPER is requested but valgrind executable cannot be found")
    endif()
    MESSAGE(STATUS "Valgrind test mode enabled")
endif()

IF (BUILD_TESTS OR BUILD_EXAMPLES)
    # ##############################################################################
    #                             THIRD PARTY LIBS                                 #
    # ##############################################################################
    # Use a header only version of armadillo; default CMake project is not header only
    # That's why we use a custom 'armadillo' target with only include directory
    if(NOT TARGET armadillo)
        if (ARMADILLO_ROOT_DIR)
            #add_subdirectory(${ARMADILLO_ROOT_DIR} armadillo-code)
            SET(ARMADILLO_INCLUDE_DIR ${ARMADILLO_ROOT_DIR}/include)
        else()
            #add_subdirectory(third_party/armadillo-code)
            SET(ARMADILLO_INCLUDE_DIR third_party/armadillo-code/include)
        endif()
        ADD_LIBRARY(armadillo INTERFACE)
        target_include_directories(armadillo INTERFACE ${ARMADILLO_INCLUDE_DIR})
    endif()

    if(NOT TARGET pybind11)
        if (PYTHON_PREFIX_PATH)
            set(CMAKE_PREFIX_PATH_SAVED ${CMAKE_PREFIX_PATH})
            list(APPEND CMAKE_PREFIX_PATH ${PYTHON_PREFIX_PATH})
        endif()
        if (PYBIND11_ROOT_DIR)
            add_subdirectory(${PYBIND11_ROOT_DIR} pybind11)
        else()
            add_subdirectory(third_party/pybind11)
        endif()
        if (CMAKE_PREFIX_PATH_SAVED)
            set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH_SAVED})
            unset(CMAKE_PREFIX_PATH_SAVED)
        endif()
    endif()
    
    ENABLE_TESTING()
ENDIF()

IF(BUILD_TESTS)
    add_subdirectory(tests)
    ADD_SUBDIRECTORY(third_party/Catch2)
ENDIF()
IF(BUILD_EXAMPLES)
    add_subdirectory(examples)
ENDIF()

#------------------------------------------------------

## search for clang-format and add target
if(NOT DEFINED CARMA_DEV_TARGET OR CARMA_DEV_TARGET)
    find_program(CLANG_FORMAT clang-format)
    if (CLANG_FORMAT)
        exec_program(${CLANG_FORMAT} ARGS -version
                OUTPUT_VARIABLE CLANG_FORMAT_RAW_VERSION)
        string(REGEX MATCH "[1-9][0-9]*\\.[0-9]+\\.[0-9]+"
                CLANG_FORMAT_VERSION ${CLANG_FORMAT_RAW_VERSION})
        if (CLANG_FORMAT_VERSION VERSION_GREATER_EQUAL "6.0.0") 
            add_custom_target(clang-format
                    COMMAND echo "running ${CLANG_FORMAT} ..."
                    COMMAND ${CMAKE_COMMAND}
                    -DPROJECT_SOURCE_DIR="${PROJECT_SOURCE_DIR}"
                    -DCLANG_FORMAT="${CLANG_FORMAT}"
                    -P ${PROJECT_SOURCE_DIR}/cmake/ClangFormatProcess.cmake)
            message(STATUS "clang-format target for updating code format is available")
        else()
            message(WARNING "incompatible clang-format found (<6.0.0); clang-format target is not available.")
            add_custom_target(clang-format
                    COMMAND ${CMAKE_COMMAND} -E echo ""
                    COMMAND ${CMAKE_COMMAND} -E echo "*** code formatting not available since clang-format version is incompatible ***"
                    COMMAND ${CMAKE_COMMAND} -E echo ""
                    COMMENT "Inform about not available code format."
                    )
        endif()
    else ()
        message(WARNING "clang-format no found; clang-format target is not available.")
        add_custom_target(clang-format
                COMMAND ${CMAKE_COMMAND} -E echo ""
                COMMAND ${CMAKE_COMMAND} -E echo "*** code formatting not available since clang-format has not been found ***"
                COMMAND ${CMAKE_COMMAND} -E echo ""
                COMMENT "Inform about not available code format."
                )
    endif ()

    #------------------------------------------------------
    # unit tests coverage

    if (ENABLE_COVERAGE)
        find_program(LCOV lcov)
        if (NOT LCOV)
            message(FATAL_ERROR "lcov not found, cannot perform coverage.")
        endif ()
    
        # coveralls.io does not support striped paths
        #find_program (SED NAMES sed)
        #if (NOT SED)
        #    message(FATAL_ERROR "Unable to find sed")
        #else()
        #    # message(STATUS "sed found at ${SED}")
        #endif (NOT SED)
    
        # Don't forget '' around each pattern
        set(LCOV_EXCLUDE_PATTERN "'${PROJECT_SOURCE_DIR}/third_party/*'")
    
        add_custom_target(coverage
                # Cleanup previously generated profiling data
                COMMAND ${LCOV} --base-directory ${PROJECT_SOURCE_DIR} --directory ${PROJECT_BINARY_DIR} --zerocounters
                # Initialize profiling data with zero coverage for every instrumented line of the project
                # This way the percentage of total lines covered will always be correct, even when not all source code files were loaded during the test(s)
                COMMAND ${LCOV} --base-directory ${PROJECT_SOURCE_DIR} --directory ${PROJECT_BINARY_DIR} --capture --initial --output-file coverage_base.info
                # Run tests
                COMMAND ${CMAKE_CTEST_COMMAND} -j ${PROCESSOR_COUNT}
                # Collect data from executions
                COMMAND ${LCOV} --base-directory ${PROJECT_SOURCE_DIR} --directory ${PROJECT_BINARY_DIR} --capture --output-file coverage_ctest.info
                # Combine base and ctest results
                COMMAND ${LCOV} --add-tracefile coverage_base.info --add-tracefile coverage_ctest.info --output-file coverage_full.info
                # Extract only project data (--no-capture or --remove options may be used to select collected data)
                COMMAND ${LCOV} --remove coverage_full.info ${LCOV_EXCLUDE_PATTERN} --output-file coverage_filtered.info
                COMMAND ${LCOV} --extract coverage_filtered.info '${PROJECT_SOURCE_DIR}/*' --output-file coverage.info
                # coveralls.io does not support striped paths
                #COMMAND ${SED} -i.bak 's|SF:${PROJECT_SOURCE_DIR}/|SF:|g' coverage.info
                DEPENDS tests
                COMMENT "Running test coverage."
                WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
                )
        message(STATUS "coverage target for code coverage is available")
        
        find_program(GENHTML genhtml)
        if (NOT GENHTML)
            message(WARNING "genhtml not found, cannot perform report-coverage.")
        else ()
            add_custom_target(coverage-report
                    COMMAND ${CMAKE_COMMAND} -E remove_directory "${PROJECT_BINARY_DIR}/coverage"
                    COMMAND ${CMAKE_COMMAND} -E make_directory "${PROJECT_BINARY_DIR}/coverage"
                    COMMAND ${GENHTML} -o coverage -t "${CMAKE_PROJECT_NAME} test coverage" --ignore-errors source --legend --num-spaces 4 coverage.info
                    COMMAND ${LCOV} --list coverage.info
                    DEPENDS coverage
                    COMMENT "Building coverage html report."
                    WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
                    )
        endif ()
    else ()
        add_custom_target(coverage
                COMMAND ${CMAKE_COMMAND} -E echo ""
                COMMAND ${CMAKE_COMMAND} -E echo "*** Use CMAKE_BUILD_TYPE=Coverage option in cmake configuration to enable code coverage ***"
                COMMAND ${CMAKE_COMMAND} -E echo ""
                COMMENT "Inform about not available code coverage."
                )
        add_custom_target(coverage-report DEPENDS coverage)
    endif ()
endif()
