cmake_minimum_required(VERSION 3.10)
project(UAIBOT_CPP_BINDS)

# Windows-specific setup
if(WIN32)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)  # Export all symbols for DLL
    # add_compile_definitions(PYBIND11_EXPORT)
endif()
# Add python 3.11
find_package(Python3 3.10 REQUIRED COMPONENTS Interpreter Development.Module)
# include_directories(${Python3_INCLUDE_DIR})
# link_directories(${Python3_LIBRARY_DIR})

# Add pybind11 
execute_process(
    COMMAND "${Python3_EXECUTABLE}" -c "import pybind11; print(pybind11.get_include())"
    OUTPUT_VARIABLE PYBIND11_INCLUDE_DIR
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

# Pybind11 discovery with cross-platform support
execute_process(
    COMMAND "${Python3_EXECUTABLE}" -c 
        "import pybind11, os; print(os.path.dirname(pybind11.__file__))"
    OUTPUT_VARIABLE PYBIND11_ROOT_DIR
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

message(STATUS "Pybind11 root directory: ${PYBIND11_ROOT_DIR}")

# Set pybind11 paths
set(PYBIND11_INCLUDE_DIR "${PYBIND11_ROOT_DIR}/include")
set(pybind11_DIR "${PYBIND11_ROOT_DIR}/share/cmake/pybind11")
list(APPEND CMAKE_PREFIX_PATH ${pybind11_DIR})
include_directories(${PYBIND11_INCLUDE_DIR})
# Find pybind11 package
find_package(pybind11 CONFIG REQUIRED)

# # CHECK ALL FILES INSIDE PYBIND11_INCLUDE_DIR
# execute_process(
#     COMMAND "${Python3_EXECUTABLE}" -c "import os, pybind11; print(os.listdir(os.path.join(os.path.dirname(pybind11.__file__), 'share', 'cmake', 'pybind11')))"
#     OUTPUT_VARIABLE PYBIND11_FILES
#     OUTPUT_STRIP_TRAILING_WHITESPACE
# )
message(STATUS "Python includes: ${Python3_INCLUDE_DIRS}")
message(STATUS "Python library: ${Python3_LIBRARIES}")
message(STATUS "Python executable: ${Python3_EXECUTABLE}")
message(STATUS "Python lib directory: ${Python3_LIBRARY_DIR}")
# message(STATUS "Pybind11 include directory: ${PYBIND11_INCLUDE_DIR}")
# message(STATUS "Pybind11 files: ${PYBIND11_FILES}")

# string(REGEX REPLACE "/include$" "" PYBIND11_DIR "${PYBIND11_INCLUDE_DIR}")
# set(pybind11_DIR ${PYBIND11_DIR})
# set(pybind11_INCLUDE_DIR ${PYBIND11_INCLUDE_DIR})
# set(pybind11_DIR ${PYBIND11_DIR}/share/cmake/pybind11) 
# set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${pybind11_DIR}) 
# include_directories(${PYBIND11_INCLUDE_DIR})
# find_package(pybind11 REQUIRED CONFIG)


#Add eigen
set(THIRD_PARTY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/third_party)
set(EIGEN3_INCLUDE_DIR ${THIRD_PARTY_DIR})
include_directories(${EIGEN3_INCLUDE_DIR})


# Set C++ standard and optimization flags
set(CMAKE_CXX_STANDARD 14)
if(MSVC)
    # Enable math constants (M_PI etc.) and strict standards compliance
    add_compile_definitions(_USE_MATH_DEFINES _HAS_STD_BYTE=0)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /O2 /DEIGEN_MPL2_ONLY /wd4996 /permissive- /D_USE_MATH_DEFINES")
    # /wd4244  # Disable conversion warnings
    # /wd4267  # Disable size_t conversion warnings
    # /wd4996  # Disable deprecated function warnings
    # Copied some flags from mplcairo (https://github.com/matplotlib/mplcairo/blob/93c97b00f07e24bb86e8a53dd49bde9bfe45e6ad/setup.py)
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -DEIGEN_MPL2_ONLY -Wno-deprecated-declarations")
endif()

# Define source files 
set(SOURCES
    pybind_main.cpp   
    manipulator.cpp  
    utils.cpp  
    declarations.h  
    gjk.c         
    gjk.h
)

# Define an INTERFACE library for nanoflann (header-only)
add_library(nanoflann INTERFACE)
target_include_directories(nanoflann INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})

# Ensure pybind11 and Eigen are properly included
pybind11_add_module(uaibot_cpp_bind ${SOURCES})

target_link_libraries(uaibot_cpp_bind PRIVATE
    pybind11::pybind11      # Python headers + core
    pybind11::windows_extras  # Windows-specific optimizations
)
# EXPLICITLY LINK PYTHON 3.11 LIBRARY
# target_include_directories(uaibot_cpp_bind PRIVATE ${EIGEN3_INCLUDE_DIR} ${pybind11_INCLUDE_DIR} ${Python3_INCLUDE_DIR})
target_include_directories(uaibot_cpp_bind PRIVATE ${EIGEN3_INCLUDE_DIR})
target_link_libraries(uaibot_cpp_bind PRIVATE nanoflann)
# Windows-specific linking
if(WIN32)
    target_link_libraries(uaibot_cpp_bind PRIVATE ${Python3_LIBRARIES})
endif()

# SET RUNTIME SEARCH PATH (Ensures the `.so` file finds dependencies)
set_target_properties(uaibot_cpp_bind PROPERTIES INSTALL_RPATH_USE_LINK_PATH TRUE)
set_target_properties(uaibot_cpp_bind PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)
# set_target_properties(uaibot_cpp_bind PROPERTIES INSTALL_RPATH ${Python3_LIBRARY_DIR})
set_target_properties(uaibot_cpp_bind PROPERTIES INSTALL_RPATH "$ORIGIN")

# Ensure `.so` is output to the correct directory
# set_target_properties(uaibot_cpp_bind PROPERTIES
#     LIBRARY_OUTPUT_DIRECTORY ../../
# )

execute_process(
    COMMAND ${Python3_EXECUTABLE} -c "import sysconfig; print(sysconfig.get_path('platlib'))"
    OUTPUT_VARIABLE PYTHON_SITE_PACKAGES
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
set_target_properties(uaibot_cpp_bind PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${PYTHON_SITE_PACKAGES}/uaibot
)

# Ensure gjk.c is compiled as C
set_source_files_properties(gjk.c PROPERTIES LANGUAGE C)
