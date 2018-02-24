cmake_minimum_required(VERSION 3.0)
project(imgui)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)


set(IMGUI_HDRS
    imgui.h
    imconfig.h
    examples/opengl3_example/imgui_impl_glfw_gl3.h
)
set(IMGUI_SRCS
    imgui.cpp
    examples/opengl3_example/imgui_impl_glfw_gl3.cpp

)

find_package(glfw3 REQUIRED CONFIG)

add_library(imgui STATIC ${IMGUI_SRCS} ${IMGUI_HDRS})
add_library(::imgui ALIAS imgui)

target_link_libraries(imgui PUBLIC glfw)

target_include_directories(${PROJECT_NAME} 
    PUBLIC 
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include/imgui>)

set_target_properties(${PROJECT_NAME} PROPERTIES DEBUG_POSTFIX _d)
set_target_properties(${PROJECT_NAME} PROPERTIES FOLDER libs)
if (MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE "/MP")
endif()


find_package(glew QUIET CONFIG)
if (NOT glew_FOUND)
    # hope gles2 is available
    find_library(GLESV2_LIBRARY NAMES GLESv2 PATHS /usr/lib/)
    find_path(GL2_INCLUDE NAMES GLES2/gl2.h PATHS /usr/include)
 
    if (GLESV2_LIB AND GL2_INCLUDE)
        add_library(::gles2 SHARED IMPORTED)
        set_property(TARGET ::gles2 PROPERTY IMPORTED_IMPLIB_RELEASE ${GLESV2_LIB})
        set_property(TARGET ::gles2 PROPERTY IMPORTED_LOCATION_RELEASE ${GLESV2_LIB})
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
	 
if (glew_FOUND)
    target_link_libraries(${PROJECT_NAME} PUBLIC ::glew)
    target_compile_definitions(${PROJECT_NAME} PUBLIC -DHAVE_GLEW=1)
    set(gles2_FOUND FALSE)
elseif(gles2_FOUND)
    target_link_libraries(${PROJECT_NAME} PUBLIC ::gles2)
else()
    message(FATAL_ERROR "Could not find glew nor gles2")
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
