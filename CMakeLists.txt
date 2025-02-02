#####################################
# Define Project-Wide Settings
#####################################
cmake_minimum_required(VERSION 3.15.0 FATAL_ERROR)

# Define the Project Name and Description
project (crow_all LANGUAGES CXX)

# Define the module path
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")

# Set required C++ Standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

if (NOT CMAKE_BUILD_TYPE)
	message(STATUS "No build type selected, default to Release")
	set(CMAKE_BUILD_TYPE "Release")
endif()

#####################################
# Define Options
#####################################
option(BUILD_EXAMPLES "Builds the examples in the project" ON)
option(BUILD_TESTING  "Builds the tests in the project"    ON)

#####################################
# Define CMake Module Imports
#####################################
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/dependencies.cmake)
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/compiler_options.cmake)

#####################################
# Define project-wide imports
#####################################
# this can be alternatively (and as recommended way) done with target_include_directories()
if(BUILD_EXAMPLES OR BUILD_TESTING)
	set(PROJECT_INCLUDE_DIR
		${CMAKE_CURRENT_SOURCE_DIR}/include
	)

	include_directories("${PROJECT_INCLUDE_DIR}")
	include_directories("${CMAKE_CURRENT_SOURCE_DIR}")
	include_directories("${CMAKE_CURRENT_BINARY_DIR}")  # To include crow_all.h
endif()

#####################################
# Define Targets
#####################################
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/crow_all.h
	COMMAND python3 ${CMAKE_CURRENT_SOURCE_DIR}/scripts/merge_all.py
	${CMAKE_CURRENT_SOURCE_DIR}/include
	${CMAKE_CURRENT_BINARY_DIR}/crow_all.h
	WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
	DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include/*.h ${CMAKE_CURRENT_SOURCE_DIR}/include/crow/*.h ${CMAKE_CURRENT_SOURCE_DIR}/include/crow/middlewares/*.h
)

# Amalgamation
add_custom_target(amalgamation ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/crow_all.h)

add_library(crow INTERFACE)
add_dependencies(crow amalgamation)
target_include_directories(crow INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<INSTALL_INTERFACE:include>)

# Examples
if(BUILD_EXAMPLES)
	add_subdirectory(examples)
endif()

# Tests
if (NOT MSVC AND BUILD_TESTING)
	add_subdirectory(tests)
	enable_testing()
	add_test(NAME crow_test COMMAND ${CMAKE_CURRENT_BINARY_DIR}/tests/unittest)
	add_test(NAME template_test COMMAND ${CMAKE_CURRENT_BINARY_DIR}/tests/template/test.py WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/tests/template)
endif()

#####################################
# Install Files
#####################################
# Install the library (fake, it doesn't have any library files--just used to create the export)
install(TARGETS crow EXPORT CrowConfigTargets)

# Install the header file
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/crow_all.h" DESTINATION include)

# Install the export file
install(EXPORT CrowConfigTargets FILE CrowConfigTargets.cmake NAMESPACE Crow:: DESTINATION lib/cmake/Crow)

# Write and install Config*.cmake files
include(CMakePackageConfigHelpers)
configure_package_config_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/CrowConfig.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/CrowConfig.cmake" INSTALL_DESTINATION lib/cmake/Crow)
write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/CrowConfigVersion.cmake" VERSION 0.2 COMPATIBILITY AnyNewerVersion)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/CrowConfig.cmake" "${CMAKE_CURRENT_BINARY_DIR}/CrowConfigVersion.cmake" DESTINATION lib/cmake/Crow)

set(CPACK_GENERATOR "DEB")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "CrowCpp")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)

include(CPack)
