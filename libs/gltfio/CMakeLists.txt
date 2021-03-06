cmake_minimum_required(VERSION 3.1)
project(gltfio C ASM)

set(TARGET gltfio)
set(PUBLIC_HDR_DIR include)

# ==================================================================================================
# Sources and headers
# ==================================================================================================
set(PUBLIC_HDRS
        include/gltfio/Animator.h
        include/gltfio/AssetLoader.h
        include/gltfio/MaterialProvider.h
        include/gltfio/AssetPipeline.h
        include/gltfio/ResourceLoader.h
        include/gltfio/SimpleCamera.h
        include/gltfio/SimpleViewer.h
        include/gltfio/FilamentAsset.h
)

set(SRCS
        src/Animator.cpp
        src/AssetLoader.cpp
        src/FFilamentAsset.h
        src/FilamentAsset.cpp
        src/GltfEnums.h
        src/MaterialProvider.cpp
        src/ResourceLoader.cpp
        src/UbershaderLoader.cpp
        src/Wireframe.cpp
        src/Wireframe.h
        src/math.h
        src/upcast.h
)

# ==================================================================================================
# Build materials
# ==================================================================================================

set(RESOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR})

if (CMAKE_CROSSCOMPILING)
    include(${IMPORT_EXECUTABLES})
endif()

if (ANDROID OR WEBGL OR IOS)
    set(MATC_TARGET mobile)
else()
    set(MATC_TARGET desktop)
endif()

set(MATC_FLAGS -a all)
if (NOT CMAKE_BUILD_TYPE MATCHES Release)
    set(MATC_FLAGS -g ${MATC_FLAGS})
endif()

function(generate_mat SHADINGMODEL BLENDING)
    set(GENERATED_MAT "${RESOURCE_DIR}/${SHADINGMODEL}_${BLENDING}.mat")
    configure_file(materials/ubershader.mat.in ${GENERATED_MAT})
    set(MATERIAL_SRCS ${MATERIAL_SRCS} ${GENERATED_MAT} PARENT_SCOPE)
endfunction()

set(MATERIAL_SRCS)
generate_mat(lit transparent)
generate_mat(lit opaque)
generate_mat(lit masked)
generate_mat(specularGlossiness transparent)
generate_mat(specularGlossiness opaque)
generate_mat(specularGlossiness masked)
generate_mat(unlit transparent)
generate_mat(unlit opaque)
generate_mat(unlit masked)

set(RESOURCE_BINS)
foreach (input_path ${MATERIAL_SRCS})
    get_filename_component(basename "${input_path}" NAME_WE)
    set(output_path "${RESOURCE_DIR}/${basename}.filamat")
    add_custom_command(
            OUTPUT ${output_path}
            COMMAND matc ${MATC_FLAGS} -p ${MATC_TARGET} -m material
                    -o ${output_path} ${input_path}
            MAIN_DEPENDENCY ${input_path}
            DEPENDS matc
            COMMENT "Compiling material ${mat_src} to ${output_path}"
    )
    list(APPEND RESOURCE_BINS ${output_path})
endforeach()

# ==================================================================================================
# Build resources library
# ==================================================================================================

get_resgen_vars(${RESOURCE_DIR} gltfresources)

add_custom_command(
        OUTPUT ${RESGEN_OUTPUTS}
        COMMAND resgen ${RESGEN_FLAGS} ${RESOURCE_BINS}
        DEPENDS resgen ${RESOURCE_BINS}
)

if (DEFINED RESGEN_SOURCE_FLAGS)
    set_source_files_properties(${RESGEN_SOURCE} PROPERTIES COMPILE_FLAGS ${RESGEN_SOURCE_FLAGS})
endif()

set(DUMMY_SRC "${RESOURCE_DIR}/dummy.c")
add_custom_command(OUTPUT ${DUMMY_SRC} COMMAND echo "//" > ${DUMMY_SRC})

add_library(gltfio_resources ${DUMMY_SRC} ${RESGEN_SOURCE})

# ==================================================================================================
# Check if embree is available.
# This is an optional dependency and can be installed with homebrew, apt-get, etc.
# ==================================================================================================

find_package(embree 3.0 QUIET PATHS /usr/lib64/cmake)
if (embree_FOUND)
    message("Found embree in ${embree_DIR}")
    add_definitions(-DFILAMENT_HAS_EMBREE)
else()
    message("Embree not found, pipeline features are disabled.")
endif()

# ==================================================================================================
# Build the "core" library (no filamat or pipeline)
# ==================================================================================================

include_directories(${PUBLIC_HDR_DIR} ${RESOURCE_DIR} ${EMBREE_INCLUDE_DIRS})
link_libraries(math utils filament cgltf stb geometry gltfio_resources)

add_library(gltfio_core STATIC ${PUBLIC_HDRS} ${SRCS})

target_include_directories(gltfio_core PUBLIC ${PUBLIC_HDR_DIR})

if (NOT WEBGL AND NOT ANDROID AND NOT IOS)

    # ==================================================================================================
    # Link the core library with additional dependencies to create the "full" library
    # ==================================================================================================
    add_library(${TARGET} STATIC ${PUBLIC_HDRS}
        src/MaterialGenerator.cpp
        src/AssetPipeline.cpp
        src/PathTracer.cpp
        src/PathTracer.h)
    target_link_libraries(${TARGET} PUBLIC
        filamat
        xatlas
        meshoptimizer
        gltfio_core
        ${EMBREE_LIBRARY})
    target_include_directories(${TARGET} PUBLIC ${PUBLIC_HDR_DIR})

    # ==================================================================================================
    # Compiler flags
    # ==================================================================================================
    target_compile_options(${TARGET} PRIVATE -Wno-deprecated-register)

    if (MSVC OR CLANG_CL)
        target_compile_options(${TARGET} PRIVATE $<$<CONFIG:Release>:/fp:fast>)
    else()
        target_compile_options(${TARGET} PRIVATE $<$<CONFIG:Release>:-ffast-math>)
    endif()

    # ==================================================================================================
    # Installation
    # ==================================================================================================
    install(TARGETS ${TARGET} gltfio_core ARCHIVE DESTINATION lib/${DIST_DIR})
    install(DIRECTORY ${PUBLIC_HDR_DIR}/gltfio DESTINATION include)

else()

    install(TARGETS gltfio_core ARCHIVE DESTINATION lib/${DIST_DIR})
    install(DIRECTORY ${PUBLIC_HDR_DIR}/gltfio DESTINATION include)

endif()
