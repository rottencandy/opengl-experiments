cmake_minimum_required(VERSION 3.0)

project(learnopengl)

# Include contains glad and shader.h
include_directories(include)
add_executable(out include/glad.c src/main.cpp)

# Getting glfw(static) lib location and flags
# using pkg-config module inside CMake because glfw3.x loader is still buggy
include(FindPkgConfig)
pkg_check_modules(GLFW3
    REQUIRED
    glfw3)

# Copy assets to build location
file(COPY "${CMAKE_SOURCE_DIR}/src/shaders" DESTINATION "${CMAKE_BINARY_DIR}")
file(COPY "${CMAKE_SOURCE_DIR}/src/textures" DESTINATION "${CMAKE_BINARY_DIR}")

# glfw flags
target_compile_options(out PUBLIC ${GLFW3_STATIC_LDFLAGS})

# static glfw lirary files
target_link_libraries(out ${GLFW3_LINK_LIBRARIES} ${GLFW3_STATIC_LIBRARIES})
