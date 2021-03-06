cmake_minimum_required(VERSION 2.8.11)
set(SAMPLES_DATA_DIR ${SAMPLES_DATA_DIR} "${PROJECT_SOURCE_DIR}/API-Samples/data")
set(SHADER_FILES ${SHADER_FILES} "")
include_directories( ${SAMPLES_DATA_DIR} ${GLSLANG_SPIRV_INCLUDE_DIR} ${GLMINC_PREFIX})

# Additional includes for spirv-tools
include_directories(${SPIRV_TOOLS_INCLUDE_DIR})
                    
# function to build a single-source-file sample that uses glsl shaders;
#   - assumes SAMPLE_GLSL_VERT_SHADERS and SAMPLE_GLSL_FRAG_SHADERS is set to
#     the list of shader files of that type, with .vert and .frag omitted
#   - uses glslangValidator to convert glsl to spirv
function(sampleExtGLSLShaders SNAME)
    set(SAMPLE_NAME ${SNAME})
    set(SHADER_DIR ${SAMPLES_DATA_DIR}/${VULKAN_VERSION})
    foreach(SFILE ${SAMPLE_GLSL_VERT_SHADERS})
        add_custom_command(OUTPUT ${SHADER_DIR}/${SFILE}-vert.spv
            COMMAND ${GLSLANG_VALIDATOR} -s -V ${SHADER_DIR}/${SFILE}.vert
            COMMAND ${MOVE_CMD} vert.spv ${SHADER_DIR}/${SFILE}-vert.spv
            DEPENDS ${SHADER_DIR}/${SFILE}.vert ${GLSLANG_VALIDATOR}
        )
        set(SHADER_FILES ${SHADER_FILES} ${SHADER_DIR}/${SFILE}-vert.spv)
    endforeach(SFILE)
    foreach(SFILE ${SAMPLE_GLSL_FRAG_SHADERS})
        add_custom_command(OUTPUT ${SHADER_DIR}/${SFILE}-frag.spv
            COMMAND ${GLSLANG_VALIDATOR} -s -V ${SHADER_DIR}/${SFILE}.frag
            COMMAND ${MOVE_CMD} frag.spv ${SHADER_DIR}/${SFILE}-frag.spv
            DEPENDS ${SHADER_DIR}/${SFILE}.frag ${GLSLANG_VALIDATOR}
        )
        set(SHADER_FILES ${SHADER_FILES} ${SHADER_DIR}/${SFILE}-frag.spv)
    endforeach(SFILE)
    add_executable(${SAMPLE_NAME} ${SAMPLE_NAME}/${SAMPLE_NAME}.cpp ${SHADER_FILES})
    target_link_libraries(${SAMPLE_NAME} ${VULKAN_LOADER} ${UTILS_NAME} ${GLSLANG_LIBRARIES} ${PTHREAD})
endfunction(sampleExtGLSLShaders)

# function to build a single-source-file sample that uses spirv shaders
#   - assumes SAMPLE_SPIRV_SHADERS is set to list to list of spirv shsders
#     used by the sample, with the .spv omitted
function(sampleExtSPIRVShaders SNAME)
    set(SAMPLE_NAME ${SNAME})
    set(SHADER_DIR ${SAMPLES_DATA_DIR}/${VULKAN_VERSION})
    foreach(SFILE ${SAMPLE_SPIRV_SHADERS})
        set(SHADER_FILES ${SHADER_FILES} ${SHADER_DIR}/${SFILE}.spv)
    endforeach(SFILE)
    add_executable(${SAMPLE_NAME} ${SAMPLE_NAME}/${SAMPLE_NAME}.cpp ${SHADER_FILES})
    target_link_libraries(${SAMPLE_NAME} ${VULKAN_LOADER} ${UTILS_NAME} ${GLSLANG_LIBRARIES} ${PTHREAD})
endfunction(sampleExtSPIRVShaders)

# function to build a simple single-source-file sample
#   - assumes S_TARGETS set to list of build targets
function(sampleWithSingleFile)
    # Projectwide settings.
    # Product flavor (ABI).
    # mips & mips64 does not work with -finline_functions flag.
    set (SUPPORTED_ABI armeabi-v7a arm64-v8a x86 x86_64)
    if (";${SUPPORTED_ABI};" MATCHES ";${ANDROID_ABI};")
        set (FLAVOR "${ANDROID_ABI}")
        set (ABI_FILTER "ndk.abiFilters.add(\"${ANDROID_ABI}\")")
    else()
        set (FLAVOR "all")
        foreach(ABI ${SUPPORTED_ABI})
            set (ABI_FILTER
              "${ABI_FILTER}\n            ndk.abiFilters.add(\"${ABI}\")")
        endforeach(ABI)
    endif()

    foreach(TARG ${S_TARGETS})
        set(SAMPLE_NAME ${TARG})
        if(ANDROID)
            # Generate AndroidStudio project files.
            MESSAGE("Generating AndroidStudio project:" ${SAMPLE_NAME})
            # Replace "-" to "_", which is invalid in a package name.
            STRING(REPLACE "-" "_" PACKAGE_NAME ${SAMPLE_NAME})
            set (SAMPLE_DESCRIPTION "Vulkan sample: ${SAMPLE_NAME}")
            file(COPY ./android/project_template/README.md DESTINATION ./android/${SAMPLE_NAME}/)
            file(COPY ./android/project_template/build.gradle DESTINATION ./android/${SAMPLE_NAME}/)
            file(COPY ./android/project_template/src DESTINATION ./android/${SAMPLE_NAME}/)

            # Clear flags.
            set (OTHER_FLAGS "")
            set (ASSET_DIR "")
            set (EXTRA_PERMISSIONS "")
            set (SHADERC_LINKAGE "")
            set (VALIDATIONLAYER_PACKING "")

            # Add extra setting for some samples.
            # shaderc usage.
            set (SHADERC_LINKAGE "library \"shaderc\"         linkage \"static\"")

            # Include resource files.
            set (SAMPLE_WITH_RESOURCES
             drawtexturedcube inittexture immutable_sampler multiple_sets pipeline_cache
             pipeline_derivative secondarycmd separate_image_sampler spirv_assembly
             spirv_specialization template)
            if (";${SAMPLE_WITH_RESOURCES};" MATCHES ";${SAMPLE_NAME};")
                set (ASSET_DIR "sourceSets.main.assets.srcDirs '../../data'")
            endif()

            # Add external storage access permission.
            set (SAMPLE_WITH_EXTERNALSTORAGEACCESS pipeline_cache)
            if (";${SAMPLE_WITH_EXTERNALSTORAGEACCESS};" MATCHES ";${SAMPLE_NAME};")
                set (EXTRA_PERMISSIONS "<uses-permission android:name=\"android.permission.WRITE_EXTERNAL_STORAGE\"/>")
            endif()

            # Add validation layer settings.
            set (SAMPLE_WITH_TOOLS dbgcreatemsgcallback enable_validation_with_callback)
            if (";${SAMPLE_WITH_TOOLS};" MATCHES ";${SAMPLE_NAME};")
                set (VALIDATIONLAYER_PACKING "sourceSets.main.jniLibs.srcDirs file(ndkDir).absolutePath +\n\t\t\t\t'/sources/third_party/vulkan/src/build-android/jniLibs'")
            endif()

            # Cook template files.
            set (TEMPLATE_FILES
                android/${SAMPLE_NAME}/src/main/res/values/strings.xml
                android/${SAMPLE_NAME}/src/main/AndroidManifest.xml
                android/${SAMPLE_NAME}/build.gradle
                android/${SAMPLE_NAME}/README.md
            )
            foreach(TEMPLATE_FILE ${TEMPLATE_FILES})
                configure_file(${TEMPLATE_FILE} ${TEMPLATE_FILE} @ONLY)
            endforeach(TEMPLATE_FILE)
            SET(SETTINGS_GRADLE "${SETTINGS_GRADLE}include 'LunarGSamples/API-Samples/android/:${SAMPLE_NAME}'\n")
        elseif(UNIX)
            add_executable(${SAMPLE_NAME} ${SAMPLE_NAME}/${SAMPLE_NAME}.cpp)
            target_link_libraries(${SAMPLE_NAME} ${UTILS_NAME} ${GLSLANG_LIBRARIES} ${XCB_LIBRARIES} ${VULKAN_LOADER} ${PTHREAD} ${SPIRV_TOOLS_LIBRARIES})
        else()
            add_executable(${SAMPLE_NAME} WIN32 ${SAMPLE_NAME}/${SAMPLE_NAME}.cpp)
            target_link_libraries(${SAMPLE_NAME} ${UTILS_NAME} ${GLSLANG_LIBRARIES} ${VULKAN_LOADER} ${WINLIBS} ${SPIRV_TOOLS_LIBRARIES})
        endif()
    endforeach(TARG)

    if(ANDROID)
        #generate Settings.gradle
        configure_file(android/project_template/settings.gradle   ${PROJECT_SOURCE_DIR}/../settings.gradle @ONLY)
        file(COPY ./android/project_template/CMakeLists.txt DESTINATION ./android/)
    endif()
endfunction(sampleWithSingleFile)

# function to build a simple single-source-file sample
#   - assumes that code is standalone and doesn't depend on utilitites
#   - assumes S_TARGETS set to list of build targets
function(sampleInlineGLSLShaders)
    foreach(TARG ${S_TARGETS})
        set(SAMPLE_NAME ${TARG})
        if(UNIX)
            add_executable(${SAMPLE_NAME} ${SAMPLE_NAME}/${SAMPLE_NAME}.cpp)
            target_link_libraries(${SAMPLE_NAME} ${GLSLANG_LIBRARIES} ${XCB_LIBRARIES} ${VULKAN_LOADER} ${PTHREAD})
        else()
            add_executable(${SAMPLE_NAME} WIN32 ${SAMPLE_NAME}/${SAMPLE_NAME}.cpp)
            target_link_libraries(${SAMPLE_NAME} ${GLSLANG_LIBRARIES} ${VULKAN_LOADER} ${WINLIBS})
        endif()
    endforeach(TARG)
endfunction(sampleInlineGLSLShaders)

if(WIN32)
    set (MOVE_CMD "move")
    set (VULKAN_LOADER_NAME "vulkan-${MAJOR}")
else()
    set (MOVE_CMD "mv")
    set (PTHREAD "pthread")
    set (VULKAN_LOADER_NAME "vulkan")
endif()

if(EXISTS "${CMAKE_SOURCE_DIR}/layers")
    set (VULKAN_LOADER ${VULKAN_LOADER_NAME})
else()
    find_library(VULKAN_LOADER NAMES ${VULKAN_LOADER_NAME}
                 HINTS "${PROJECT_SOURCE_DIR}/../${BINDATA_DIR}" )
endif()

set (LIBGLM_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/libs)

if(NOT WIN32 AND NOT ANDROID)
    include_directories(
        ${XCB_INCLUDE_DIRS}
    )
    link_libraries(${XCB_LIBRARIES} ${VULKAN_LOADER} m )
endif()
if(WIN32)
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_CRT_SECURE_NO_WARNINGS -D_USE_MATH_DEFINES")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_SECURE_NO_WARNINGS -D_USE_MATH_DEFINES")

    # If MSVC, disable some signed/unsigned mismatch warnings.
    if (MSVC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4267")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4267")
    endif()

else()
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-sign-compare")
endif()

if (NOT WIN32)
    # extra setup for out-of-tree builds
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        add_custom_target(samples-binary-dir-symlinks ALL
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_all_samples.sh
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/get-short-descripts.sh
            VERBATIM
            )
    endif()
else()
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lunarg.ppm LUNARG)
        FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/run_all_samples.ps1 RUN_ALL)
        add_custom_target(samples-binary-dir-symlinks ALL
            COMMAND ${CMAKE_COMMAND} -E copy_if_different ${RUN_ALL} run_all_samples.ps1
            VERBATIM
            )
    endif()
endif()


#
# START BUILDING SAMPLES HERE
#

# Generate Android project.
option(ANDROID OFF)

# simple one file sample targets, no additional files
set (S_TARGETS instance device enumerate enumerate-adv
    depthbuffer initswapchain inittexture uniformbuffer
    descriptor_pipeline_layouts initcommandbuffer
    initrenderpass initframebuffers vertexbuffer
    allocdescriptorsets initshaders initpipeline multithreadcmdbuf
    drawcube drawtexturedcube dynamicuniform texelbuffer
    instance_layer_properties instance_extension_properties
    instance_layer_extension_properties dbgcreatemsgcallback
    enable_validation_with_callback multiple_sets
    copyblitimage template separate_image_sampler
    occlusion_query pipeline_cache pipeline_derivative
    immutable_sampler push_constants drawsubpasses secondarycmd
    spirv_assembly spirv_specialization)
sampleWithSingleFile()

if(NOT ANDROID)
    add_subdirectory(utils)
endif()

# targets that use spv shaders

# usespirvshader
# set(SAMPLE_SPIRV_SHADERS spirvshader-vert spirvshader-frag)
# sampleExtSPIRVShaders()

# targets that require glsl to spv shader conversion

# useglslshader
# set(SAMPLE_GLSL_FRAG_SHADERS glslshader)
# set(SAMPLE_GLSL_VERT_SHADERS glslshader)
# sampleExtGLSLShaders()

