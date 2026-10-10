cmake_minimum_required(VERSION 3.15)
project(orb_slam3 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(Python 3 REQUIRED COMPONENTS Interpreter Development.Module NumPy)
include_directories(${Python_INCLUDE_DIRS})
include_directories(${Python_NumPy_INCLUDE_DIRS})

find_package(pybind11 CONFIG REQUIRED)
include_directories(${pybind11_INCLUDE_DIRS})

find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})

if(WIN32)
    # OpenCV 5's Windows static package lists pthread in INTERFACE_LINK_LIBRARIES
    # (x64/vc*/staticlib/OpenCVModules.cmake). MSVC has no pthread.lib, so drop
    # it for the consumer link. Linux/macOS keep it as-is.
    foreach(_ocv_tgt IN ITEMS zlib libclapack opencv_core opencv_flann opencv_geometry opencv_imgproc opencv_features opencv_calib opencv_video)
        if(TARGET ${_ocv_tgt})
            get_target_property(_ocv_libs ${_ocv_tgt} INTERFACE_LINK_LIBRARIES)
            if(_ocv_libs)
                list(FILTER _ocv_libs EXCLUDE REGEX "pthread")
                set_target_properties(${_ocv_tgt} PROPERTIES INTERFACE_LINK_LIBRARIES "${_ocv_libs}")
            endif()
        endif()
    endforeach()
endif()

file(GLOB_RECURSE ORB_SLAM3_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.h
)
pybind11_add_module(orb_slam3 MODULE ${ORB_SLAM3_SRC})

target_link_libraries(orb_slam3 PRIVATE ${OpenCV_LIBS})
target_include_directories(orb_slam3 PRIVATE ${OpenCV_INCLUDE_DIRS})

if(NOT MSVC AND NOT ${CMAKE_BUILD_TYPE} MATCHES Debug|RelWithDebInfo)
    pybind11_strip(orb_slam3)
endif()

if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    set_property(TARGET orb_slam3 PROPERTY 
        MSVC_RUNTIME_LIBRARY ${CMAKE_MSVC_RUNTIME_LIBRARY})
endif()
