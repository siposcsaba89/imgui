cmake_minimum_required(VERSION 3.0)
project(imgui)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

option(BUILD_IMGUI_VULKAN "Building vulkan backend!" ON)
option(BUILD_IMGUI_OPENGL "Building vulkan backend!" ON)
option(BUILD_DEMO_APP "Build imgui glfw3 opengl demo app" OFF)

set(IMGUI_HDRS
    imgui.h
    imconfig.h
    examples/imgui_impl_glfw.h
)
set(IMGUI_SRCS
    imgui.cpp
    imgui_draw.cpp
    imgui_internal.h
    imgui_widgets.cpp
    imstb_rectpack.h
    imstb_textedit.h
    imstb_truetype.h
    examples/imgui_impl_glfw.cpp
)
#this variable needed in cmake.config.in file, set default value to false
set(gles2_FOUND FALSE)
set(glew_FOUND FALSE)
if (BUILD_IMGUI_OPENGL)
    find_package(glew QUIET CONFIG)
    if (NOT glew_FOUND)
        # hope gles2 is available
        find_library(GLESV2_LIBRARY NAMES GLESv2 PATHS /usr/lib/ /usr/lib/x86_64-linux-gnu/)
        find_path(GL2_INCLUDE NAMES GLES2/gl3.h PATHS /usr/include)
        if (GLESV2_LIBRARY AND GL2_INCLUDE)
            add_library(::gles2 SHARED IMPORTED)
            set_property(TARGET ::gles2 PROPERTY IMPORTED_IMPLIB_RELEASE ${GLESV2_LIBRARY})
            set_property(TARGET ::gles2 PROPERTY IMPORTED_LOCATION_RELEASE ${GLESV2_LIBRARY})
            set_property(TARGET ::gles2 PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
            set_property(TARGET ::gles2 PROPERTY INTERFACE_INCLUDE_DIRECTORIES
                $<INSTALL_INTERFACE:${GL2_INCLUDE}>  # <prefix>/include/mylib
                $<BUILD_INTERFACE:${GL2_INCLUDE}>  # <prefix>/include/mylib
            )
            set_property(TARGET ::gles2 PROPERTY INTERFACE_COMPILE_DEFINITIONS HAVE_GLES2 GLFW_INCLUDE_ES2)
            set(gles2_FOUND TRUE)
        else()
            set(gles2_FOUND FALSE)
        endif()
    endif()
    list(APPEND IMGUI_SRCS examples/imgui_impl_opengl3.cpp)
    list(APPEND IMGUI_HDRS examples/imgui_impl_opengl3.h)
endif()
set(Vulkan_FOUND FALSE)
if (BUILD_IMGUI_VULKAN)
    find_package(Vulkan REQUIRED)
    list(APPEND IMGUI_SRCS examples/imgui_impl_vulkan.cpp)
    list(APPEND IMGUI_HDRS examples/imgui_impl_vulkan.h)
endif()
find_package(glfw3 REQUIRED CONFIG)

add_library(imgui STATIC ${IMGUI_SRCS} ${IMGUI_HDRS})
add_library(::imgui ALIAS imgui)

target_link_libraries(imgui PUBLIC glfw)

target_include_directories(${PROJECT_NAME} 
    PUBLIC 
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include/imgui>)

set_target_properties(${PROJECT_NAME} PROPERTIES DEBUG_POSTFIX _d)
set_target_properties(${PROJECT_NAME} PROPERTIES RELWITHDEBINFO_POSTFIX _rd)
set_target_properties(${PROJECT_NAME} PROPERTIES FOLDER libs)
if (MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE "/MP")
endif()

if (BUILD_IMGUI_OPENGL)
    if (glew_FOUND)
        target_link_libraries(${PROJECT_NAME} PUBLIC ::glew)
        target_compile_definitions(${PROJECT_NAME} PUBLIC -DHAVE_GLEW=1 -DIMGUI_IMPL_OPENGL_LOADER_GLEW=1)
        set(gles2_FOUND FALSE)
    elseif(gles2_FOUND)
        #danger find gles3 instead of gles2
        target_link_libraries(${PROJECT_NAME} PUBLIC ::gles2 )
        target_compile_definitions(${PROJECT_NAME} PUBLIC -DUSE_GL_ES3=1)
    else()
        message(FATAL_ERROR "Could not find glew nor gles2")
    endif()
endif()

if (BUILD_IMGUI_VULKAN)
    target_link_libraries(${PROJECT_NAME} PUBLIC Vulkan::Vulkan)
endif()

if(BUILD_DEMO_APP)
    if (BUILD_IMGUI_OPENGL)
        add_executable(imgui_demo examples/example_glfw_opengl3/main.cpp imgui_demo.cpp)
        target_link_libraries(imgui_demo PRIVATE ::imgui)
        target_include_directories(imgui_demo PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/examples)
    endif()
    if (BUILD_IMGUI_VULKAN)
        add_executable(imgui_demo_vulkan examples/example_glfw_vulkan/main.cpp imgui_demo.cpp)
        target_link_libraries(imgui_demo_vulkan PRIVATE ::imgui)
        target_include_directories(imgui_demo_vulkan PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/examples)
    endif()
endif()


include(GenerateExportHeader)
generate_export_header(${PROJECT_NAME} EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/gen/imgui/${PROJECT_NAME}_export.h)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/gen/${PROJECT_NAME}-config-version.cmake"
  VERSION 0.0.1
  COMPATIBILITY AnyNewerVersion
)

configure_file(cmake/${PROJECT_NAME}-config.cmake.in ${PROJECT_NAME}-config.cmake @ONLY)
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}-targets DESTINATION 
    ARCHIVE DESTINATION lib LIBRARY DESTINATION lib RUNTIME DESTINATION bin)
install(FILES 
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake 
        ${CMAKE_CURRENT_BINARY_DIR}/gen/${PROJECT_NAME}-config-version.cmake
    DESTINATION 
        lib/cmake/${PROJECT_NAME})
    
install(EXPORT ${PROJECT_NAME}-targets NAMESPACE :: DESTINATION lib/cmake/${PROJECT_NAME})

install(FILES
    ${CMAKE_BINARY_DIR}/gen/imgui/${PROJECT_NAME}_export.h
	${IMGUI_HDRS}
    DESTINATION include/${PROJECT_NAME})
