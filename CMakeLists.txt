cmake_minimum_required(VERSION 3.12)
project(area VERSION 0.3.0 LANGUAGES CXX)

# ==============================================================================
# 1. Options & Build Configuration
# ==============================================================================
option(USE_CLIPPER_FOR_CPP "Use clipper library (ON) or kbool (OFF) for C++ portion of libarea" ON)
option(USE_CLIPPER_FOR_PYTHON "Use clipper library (ON) or kbool (OFF) for Python portion of libarea" OFF)

# Modern C++ standard requirement
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Enable Position Independent Code globally for libraries/modules
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# ==============================================================================
# 2. Dependencies (Modern Find Modules)
# ==============================================================================
# Find Python 3 Development artifacts natively
find_package(Python3 COMPONENTS Development REQUIRED)

# Find Boost and its Python component matching the discovered Python version
find_package(Boost COMPONENTS python3 REQUIRED)

# Include GNU Standard Installation Directories (provides CMAKE_INSTALL_LIBDIR, etc.)
include(GNUInstallDirs)

# ==============================================================================
# 3. Source Files Definitions
# ==============================================================================
set(AREA_SRC_COMMON
    Arc.cpp
    Area.cpp
    AreaDxf.cpp
    AreaOrderer.cpp
    AreaPocket.cpp
    Circle.cpp
    Curve.cpp
    dxf.cpp
    kurve/Construction.cpp
    kurve/Finite.cpp
    kurve/kurve.cpp
    kurve/Matrix.cpp
    kurve/offset.cpp
)

set(AREA_SRC_KBOOL
    AreaBoolean.cpp
    kbool/src/booleng.cpp
    kbool/src/record.cpp
    kbool/src/scanbeam.cpp
    kbool/src/graph.cpp
    kbool/src/graphlst.cpp
    kbool/src/instonly.cpp
    kbool/src/line.cpp
    kbool/src/link.cpp
    kbool/src/lpoint.cpp
    kbool/src/node.cpp
)

set(AREA_SRC_CLIPPER
    AreaClipper.cpp
    clipper.cpp
)

set(PYAREA_SRC
    PythonStuff.cpp
)

# Determine backend components based on user options
set(CPP_BACKEND_SRC ${AREA_SRC_KBOOL})
if(USE_CLIPPER_FOR_CPP)
    set(CPP_BACKEND_SRC ${AREA_SRC_CLIPPER})
endif()

set(PY_BACKEND_SRC ${AREA_SRC_KBOOL})
if(USE_CLIPPER_FOR_PYTHON)
    set(PY_BACKEND_SRC ${AREA_SRC_CLIPPER})
endif()

# ==============================================================================
# 4. Target: libarea (C++ Shared Library)
# ==============================================================================
add_library(libarea SHARED ${AREA_SRC_COMMON} ${CPP_BACKEND_SRC})

target_compile_options(libarea PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wno-deprecated>
)

target_include_directories(libarea
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/kurve
        ${CMAKE_CURRENT_SOURCE_DIR}/kbool/src
)

set_target_properties(libarea PROPERTIES
    PREFIX ""
    SOVERSION ${PROJECT_VERSION_MAJOR}
    VERSION ${PROJECT_VERSION}
)

# ==============================================================================
# 5. Target: area (Python Module Wrapper)
# ==============================================================================
add_library(area MODULE ${AREA_SRC_COMMON} ${PY_BACKEND_SRC} ${PYAREA_SRC})

target_compile_options(area PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wno-deprecated>
)

target_include_directories(area
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}/kurve
        ${CMAKE_CURRENT_SOURCE_DIR}/kbool/src
)

# Modern target-linking resolves include directories automatically
target_link_libraries(area PRIVATE Boost::python3 Python3::Python)

set_target_properties(area PROPERTIES
    PREFIX ""
    SUFFIX "${Python3_MODULE_EXTENSION}"
)

# ==============================================================================
# 6. Installation Rules & CMake Export Configuration
# ==============================================================================

# 6a. Fallback protection for the python site-packages path
if(NOT Python3_SITELIB OR "${Python3_SITELIB}" STREQUAL "")
    set(Python3_SITELIB "lib/python3/dist-packages")
    message(STATUS "Python3_SITELIB was empty. Using fallback: ${Python3_SITELIB}")
endif()

# 6b. Install C++ Headers
file(GLOB kurve_headers "${CMAKE_CURRENT_SOURCE_DIR}/kurve/*.h")
install(FILES ${kurve_headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/area/kurve COMPONENT headers)

file(GLOB root_headers "${CMAKE_CURRENT_SOURCE_DIR}/*.h")
install(FILES ${root_headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/area COMPONENT headers)

# 6c. Install C++ Shared Library & EXPORT the target info for FreeCAD
install(TARGETS libarea
    EXPORT libareaTargets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

# 6d. Install Python Module
install(TARGETS area
    EXPORT libareaTargets
    LIBRARY DESTINATION "${Python3_SITELIB}"
    ARCHIVE DESTINATION "${Python3_SITELIB}"
    RUNTIME DESTINATION "${Python3_SITELIB}"
)

# 6e. Generate and Install the libareaConfig.cmake files required by find_package()
include(CMakePackageConfigHelpers)

# Generate a matching Version file (FreeCAD asks for 0.3.0, let's match its expected project version)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/libareaConfigVersion.cmake"
    VERSION 0.3.0
    COMPATIBILITY AnyNewerVersion
)

# Install the explicit targets file
install(EXPORT libareaTargets
    FILE libareaTargets.cmake
    NAMESPACE libarea::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libarea
)

# Write a basic configuration file directly that includes the exported targets
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/libareaConfig.cmake" "
include(CMakeFindDependencyMacro)
find_dependency(Boost COMPONENTS python3)
find_dependency(Python3 COMPONENTS Development)
include(\"\${CMAKE_CURRENT_LIST_DIR}/libareaTargets.cmake\")
")

# Install the configuration files to the standard system search layout
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/libareaConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/libareaConfigVersion.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libarea
)

# ==============================================================================
# 7. Debian Packaging Config (CPack)
# ==============================================================================
set(CPACK_PACKAGE_NAME "libarea")
set(CPACK_PACKAGE_VENDOR "Heeks")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "CAM library and Python wrapper for profile/pocketing operations")
set(CPACK_PACKAGE_VERSION "0.3.0") # Match version to satisfy FreeCAD's 0.3.0 requirement
set(CPACK_PACKAGE_CONTACT "Your Name <your.email@example.com>")

set(CPACK_GENERATOR "DEB")
set(CPACK_PACKAGING_INSTALL_PREFIX "/usr")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libboost-python1.90.0 (>= 1.90.0), python3")

include(CPack)
