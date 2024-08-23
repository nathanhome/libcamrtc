cmake_minimum_required(VERSION 3.10)
project(PiCamRTC)

# Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find necessary packages for the main application
find_package(libcamera REQUIRED)

# Main application
add_executable(camera_app src/main.cpp src/CameraOps.cpp)
target_include_directories(camera_app PRIVATE ${libcamera_INCLUDE_DIRS})
target_link_libraries(camera_app PRIVATE ${libcamera_LIBRARIES})

# Option to enable building and running of tests
option(ENABLE_UNIT_TESTS "Enable unit tests" ON)

# Check if testing is enabled and configure accordingly
if(ENABLE_UNIT_TESTS)
    # Enable CMake's testing capabilities
    enable_testing()

    # Find GoogleTest package, required for building the tests
    find_package(GTest REQUIRED)

    # Include Google Test's functionality
    include(GoogleTest)

    # Add test executable
    add_executable(camera_tests tests/CameraTest.cpp src/CameraOps.cpp)
    target_include_directories(camera_tests PRIVATE ${GTEST_INCLUDE_DIRS} ${libcamera_INCLUDE_DIRS})
    target_link_libraries(camera_tests PRIVATE ${GTEST_BOTH_LIBRARIES} ${libcamera_LIBRARIES} pthread)

    # Add a test to be run by CTest
    add_test(NAME CameraUnitTests COMMAND camera_tests)

    # Optionally use GoogleTest's discovery to register tests
    gtest_discover_tests(camera_tests)
endif()
