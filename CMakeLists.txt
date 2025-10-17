cmake_minimum_required(VERSION 3.28)
project(Hello-Vulkan)
set(CMAKE_CXX_STANDARD 20)

find_package(Vulkan REQUIRED)

add_subdirectory(lib/glm)
add_subdirectory(lib/glfw)

set(RpsRootSolutionFolder "rps")
add_subdirectory(lib/rps)

set(RpsBuildExamples OFF)
set(RpsBuildTests OFF)
set(RpsEnableImGui OFF)

add_executable(program main.cpp)
target_compile_definitions(program PRIVATE RPS_VK_RUNTIME)
target_link_libraries(program PRIVATE glfw glm ${Vulkan_LIBRARIES})
target_include_directories(program PRIVATE ${Vulkan_INCLUDE_DIRS})

target_include_directories(program PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/lib/rps/include)
target_link_libraries(program PRIVATE rps_runtime_vk)

if (WIN32)
    set(RPS_HLSLC "${CMAKE_CURRENT_SOURCE_DIR}/lib/rps/tools/rps_hlslc/win64/rps-hlslc.exe")
else()
    set(RPS_HLSLC "${CMAKE_CURRENT_SOURCE_DIR}/lib/rps/tools/rps_hlslc/linux64/rps-hlslc")
endif()

set(RPSL_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/rpsl)
file(GLOB RPSL_FILES CONFIGURE_DEPENDS "${RPSL_SOURCE_DIR}/*.rpsl")

set(RPSL_GEN_DIR ${CMAKE_CURRENT_BINARY_DIR}/rpsl_gen)
file(MAKE_DIRECTORY ${RPSL_GEN_DIR})

set(RPSL_GENERATED_CSRC)
foreach(SRC IN LISTS RPSL_FILES)
    get_filename_component(BASE "${SRC}" NAME_WE)          # e.g. mygraph
    # RPS generates a .c + .h for each module
    set(OUT_C "${RPSL_GEN_DIR}/${BASE}.c")
    set(OUT_H "${RPSL_GEN_DIR}/${BASE}.h")

    add_custom_command(
        OUTPUT ${OUT_C} ${OUT_H}
        COMMAND ${CMAKE_COMMAND} -E make_directory ${RPSL_GEN_DIR}
        # -od = output dir, -m = module name (optional, defaults to source name)
        COMMAND "${RPS_HLSLC}" "${SRC}" -od "${RPSL_GEN_DIR}" -m "${BASE}"
        DEPENDS "${SRC}"
        VERBATIM
        COMMENT "RPSL compile ${BASE}.rpsl -> C"
    )

    list(APPEND RPSL_GENERATED_CSRC ${OUT_C})
endforeach()

# Add generated C to your app and include dir for the generated headers:
target_sources(program PRIVATE ${RPSL_GENERATED_CSRC})
target_include_directories(program PRIVATE ${RPSL_GEN_DIR})


# ---------- Shaders ----------
set(SHADER_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/shaders)
set(SHADER_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/shaders)

# Find glslc (use CMake's FindVulkan hint if available)
if (DEFINED Vulkan_GLSLC_EXECUTABLE)
    set(GLSLC ${Vulkan_GLSLC_EXECUTABLE})
else()
    find_program(GLSLC glslc
            HINTS "$ENV{VULKAN_SDK}/Bin" "$ENV{VULKAN_SDK}/bin"
            REQUIRED)
endif()

# Correct GLOB syntax: variable FIRST, then options
file(GLOB SHADER_FILES CONFIGURE_DEPENDS
        "${SHADER_SOURCE_DIR}/*.vert"
        "${SHADER_SOURCE_DIR}/*.frag"
        "${SHADER_SOURCE_DIR}/*.comp"
        "${SHADER_SOURCE_DIR}/*.geom"
        "${SHADER_SOURCE_DIR}/*.tesc"
        "${SHADER_SOURCE_DIR}/*.tese"
        "${SHADER_SOURCE_DIR}/*.mesh"
        "${SHADER_SOURCE_DIR}/*.task"
        "${SHADER_SOURCE_DIR}/*.rgen"
        "${SHADER_SOURCE_DIR}/*.rchit"
        "${SHADER_SOURCE_DIR}/*.rmiss"
)

set(SPV_SHADERS)
foreach(SRC IN LISTS SHADER_FILES)
    get_filename_component(BASE ${SRC} NAME)   # e.g. foo.vert
    set(SPV "${SHADER_BINARY_DIR}/${BASE}.spv") # e.g. foo.vert.spv

    add_custom_command(
            OUTPUT ${SPV}
            COMMAND ${CMAKE_COMMAND} -E make_directory ${SHADER_BINARY_DIR}
            COMMAND ${GLSLC} -o ${SPV} ${SRC}
            MAIN_DEPENDENCY ${SRC}
            VERBATIM
            COMMENT "Compiling ${BASE}"
    )

    list(APPEND SPV_SHADERS ${SPV})
endforeach()

# Compile RPSL files into C and add them to a target.
# Usage:
#   compile_rpsl(
#     TARGET <cmake-target>
#     SOURCES <file1.rpsl> [file2.rpsl ...]    # OR GLOB <pattern>
#     [OUT_DIR <binary-output-dir>]            # default: ${CMAKE_CURRENT_BINARY_DIR}/rpsl_gen
#     [HLSLC <path-to-rps-hlslc>]              # default: lib/rps/tools/... per platform
#     [GROUP <ide-group-name>]                 # default: "Generated RPSL"
#   )
function(compile_rpsl)
    cmake_parse_arguments(RPSL
            ""
            "TARGET;OUT_DIR;HLSLC;GROUP"
            "SOURCES;GLOB"
            ${ARGN}
    )

    if(NOT RPSL_TARGET)
        message(FATAL_ERROR "compile_rpsl: TARGET is required")
    endif()
    if(NOT TARGET ${RPSL_TARGET})
        message(FATAL_ERROR "compile_rpsl: TARGET '${RPSL_TARGET}' does not exist")
    endif()

    # Resolve input files
    set(_inputs)
    if(RPSL_SOURCES)
        list(APPEND _inputs ${RPSL_SOURCES})
    endif()
    if(RPSL_GLOB)
        foreach(_pat IN LISTS RPSL_GLOB)
            file(GLOB _globbed CONFIGURE_DEPENDS "${_pat}")
            list(APPEND _inputs ${_globbed})
        endforeach()
    endif()
    if(NOT _inputs)
        message(FATAL_ERROR "compile_rpsl: provide SOURCES or GLOB")
    endif()

    # Defaults
    if(NOT RPSL_OUT_DIR)
        set(RPSL_OUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/rpsl_gen")
    endif()
    if(NOT RPSL_HLSLC)
        if(WIN32)
            set(RPSL_HLSLC "${CMAKE_CURRENT_SOURCE_DIR}/lib/rps/tools/rps_hlslc/win-x64/rps-hlslc.exe")
        else()
            set(RPSL_HLSLC "${CMAKE_CURRENT_SOURCE_DIR}/lib/rps/tools/rps_hlslc/linux-x64/rps-hlslc")
        endif()
    endif()
    if(NOT EXISTS "${RPSL_HLSLC}")
        message(FATAL_ERROR "compile_rpsl: rps-hlslc not found at '${RPSL_HLSLC}'. Override with HLSLC <path>.")
    endif()
    if(NOT RPSL_GROUP)
        set(RPSL_GROUP "Generated RPSL")
    endif()

    file(MAKE_DIRECTORY "${RPSL_OUT_DIR}")

    set(_generated_cs)
    foreach(_src IN LISTS _inputs)
        if(NOT EXISTS "${_src}")
            message(FATAL_ERROR "compile_rpsl: input file not found: ${_src}")
        endif()

        get_filename_component(_base "${_src}" NAME_WE)   # hello from shaders/hello.rpsl

        set(_out_c "${RPSL_OUT_DIR}/${_base}.rpsl.g.c")
        set(_tmp_ll "${RPSL_OUT_DIR}/${_base}.tmp.tps.ll")

        add_custom_command(
                OUTPUT "${_out_c}"                                 # main output
                BYPRODUCTS "${_tmp_ll}"                            # temp file for correct build graph
                COMMAND ${CMAKE_COMMAND} -E make_directory "${RPSL_OUT_DIR}"
                COMMAND "${RPSL_HLSLC}" "${_src}" -od "${RPSL_OUT_DIR}" -m "${_base}"
                DEPENDS "${_src}"
                VERBATIM
                COMMENT "RPSL compile ${_base}.rpsl -> ${_base}.rpsl.g.c"
        )

        list(APPEND _generated_cs "${_out_c}")
        source_group("${RPSL_GROUP}" FILES "${_out_c}")
    endforeach()

    # Add generated C sources to target
    target_sources(${RPSL_TARGET} PRIVATE ${_generated_cs})

    # Make sure the compiler treats these as C (in case your target is C++)
    set_source_files_properties(${_generated_cs} PROPERTIES LANGUAGE C)
endfunction()

compile_rpsl(
        TARGET program
        SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/shaders/hello.rpsl"
)

# Use a name that does NOT collide with the 'shaders' directory
add_custom_target(compile_shaders ALL DEPENDS ${SPV_SHADERS})
add_dependencies(program compile_shaders)
