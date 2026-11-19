cmake_minimum_required(VERSION 3.24)
project(RunnerMetal LANGUAGES C CXX OBJCXX)

set(CMAKE_OSX_ARCHITECTURES "arm64")
if(NOT CMAKE_OSX_DEPLOYMENT_TARGET)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "14.0")
endif()

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(ENABLE_MOLTENVK_ORACLE "Build temporary RunnerOracle target backed by DXVK + MoltenVK" OFF)

# Homebrew prefix (Apple Silicon default)
set(HOMEBREW_PREFIX "/opt/homebrew" CACHE PATH "Homebrew prefix")

find_package(SDL3 QUIET CONFIG PATHS "${HOMEBREW_PREFIX}/opt/sdl3/lib/cmake" "${HOMEBREW_PREFIX}/lib/cmake")
if(NOT SDL3_FOUND)
    find_path(SDL3_INCLUDE_DIR NAMES SDL3/SDL.h PATHS "${HOMEBREW_PREFIX}/opt/sdl3/include" "${HOMEBREW_PREFIX}/include" REQUIRED)
    find_library(SDL3_LIBRARY NAMES SDL3 PATHS "${HOMEBREW_PREFIX}/opt/sdl3/lib" "${HOMEBREW_PREFIX}/lib" REQUIRED)
    add_library(SDL3::SDL3 SHARED IMPORTED)
    set_target_properties(SDL3::SDL3 PROPERTIES
        IMPORTED_LOCATION "${SDL3_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${SDL3_INCLUDE_DIR}"
    )
endif()

# ---- Lifted Game + DLL Modules (src/lifted/) ----
set(LIFTED_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src/lifted")
set(LIFTED_TABLE "${LIFTED_DIR}/rt_modules_gen.c")

set(LIFTED_CMAKE "${LIFTED_DIR}/lifted_sources.cmake")
if(NOT EXISTS "${LIFTED_TABLE}" OR NOT EXISTS "${LIFTED_CMAKE}")
    message(FATAL_ERROR "Lifted sources not found at ${LIFTED_TABLE}. Run tools/setup_lifted.py first.")
endif()

include("${LIFTED_CMAKE}")

add_library(me_lifted OBJECT ${LIFTED_SOURCES})
target_include_directories(me_lifted PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/src/host_macos/runtime")
target_compile_options(me_lifted PRIVATE
    -O2 -g0 -fwrapv -ffp-contract=off -fno-strict-aliasing
    -Wno-unused-label -Wno-unused-variable -Wno-unused-but-set-variable -Wno-unused-function
)

# PhysXCore x87 register-local optimization (validated by RunnerRecompiled)
if(PHYSXCORE_LIFTED)
    set_source_files_properties(${PHYSXCORE_LIFTED} PROPERTIES
        COMPILE_DEFINITIONS "X86_RESTRICT=restrict;X86_FLOAT_PLAIN=1;X86_X87_LOCAL=1;X86_X87_SLOTS=1"
    )
endif()

# ---- Shared Host Runtime + Shatter PhysX (src/host_macos/) ----
set(RUNTIME_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src/host_macos/runtime")
set(HOST_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src/host_macos/host")
set(SHATTER_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src/host_macos/shatter")

set(RUNTIME_SOURCES
    "${RUNTIME_DIR}/rt_core.c"
    "${RUNTIME_DIR}/rt_kernel32.c"
    "${RUNTIME_DIR}/rt_sync.c"
    "${RUNTIME_DIR}/rt_files.c"
    "${RUNTIME_DIR}/rt_user32.c"
    "${RUNTIME_DIR}/rt_d3d9.c"
    "${RUNTIME_DIR}/rt_d3d9_gen.c"
    "${RUNTIME_DIR}/rt_physx_support.c"
    "${RUNTIME_DIR}/rt_dinput.c"
    "${RUNTIME_DIR}/rt_ue3.c"
    "${RUNTIME_DIR}/rt_openal.c"
    "${RUNTIME_DIR}/rt_stubs.c"
    "${RUNTIME_DIR}/rt_main.c"
    "${HOST_DIR}/rt_host_macos.c"
)

set_source_files_properties("${RUNTIME_DIR}/rt_main.c" PROPERTIES COMPILE_DEFINITIONS "main=me_pc_main")

set(SHATTER_SOURCES
    "${SHATTER_DIR}/solver/fluid/fluid.cpp"
    "${SHATTER_DIR}/solver/cloth/cloth.cpp"
    "${SHATTER_DIR}/solver/cloth/cloth_attach.cpp"
    "${SHATTER_DIR}/solver/cloth/cloth_edgeshape.cpp"
    "${SHATTER_DIR}/solver/cloth/cloth_mesh.cpp"
    "${SHATTER_DIR}/solver/cloth/cloth_self.cpp"
    "${SHATTER_DIR}/solver/cloth/cloth_selftri.cpp"
    "${SHATTER_DIR}/solver/cloth/cloth_state.cpp"
    "${SHATTER_DIR}/shim/fluid_ours.cpp"
    "${SHATTER_DIR}/shim/cloth_ours.cpp"
    "${SHATTER_DIR}/shim/scene_read.cpp"
    "${SHATTER_DIR}/shim/coverage.cpp"
    "${SHATTER_DIR}/port/bridge.cpp"
    "${SHATTER_DIR}/port/game_hooks_port.cpp"
)

add_library(me_shatter OBJECT ${SHATTER_SOURCES})
target_include_directories(me_shatter PRIVATE
    "${RUNTIME_DIR}"
    "${SHATTER_DIR}/port"
    "${SHATTER_DIR}/port/neon"
    "${SHATTER_DIR}/shim"
)
target_compile_options(me_shatter PRIVATE -O2 -g0 -fwrapv -ffp-contract=off -fno-strict-aliasing)

# ---- Standalone MIT SM3 -> IR -> SPIR-V Shader Compiler Library (Zero Vulkan Runtime) ----
set(DXBC_SPV_DIR "${CMAKE_CURRENT_SOURCE_DIR}/third_party/dxbc-spirv")
set(DXBC_SPV_SOURCES
    "${DXBC_SPV_DIR}/sm3/sm3_types.cpp"
    "${DXBC_SPV_DIR}/sm3/sm3_parser.cpp"
    "${DXBC_SPV_DIR}/sm3/sm3_disasm.cpp"
    "${DXBC_SPV_DIR}/sm3/sm3_converter.cpp"
    "${DXBC_SPV_DIR}/sm3/sm3_io_map.cpp"
    "${DXBC_SPV_DIR}/sm3/sm3_registers.cpp"
    "${DXBC_SPV_DIR}/sm3/sm3_resources.cpp"
    "${DXBC_SPV_DIR}/spirv/spirv_builder.cpp"
    "${DXBC_SPV_DIR}/spirv/spirv_mapping.cpp"
    "${DXBC_SPV_DIR}/ir/ir.cpp"
    "${DXBC_SPV_DIR}/ir/ir_builder.cpp"
    "${DXBC_SPV_DIR}/ir/ir_disasm.cpp"
    "${DXBC_SPV_DIR}/ir/ir_divergence.cpp"
    "${DXBC_SPV_DIR}/ir/ir_dominance.cpp"
    "${DXBC_SPV_DIR}/ir/ir_legalize.cpp"
    "${DXBC_SPV_DIR}/ir/ir_serialize.cpp"
    "${DXBC_SPV_DIR}/ir/ir_utils.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_arithmetic.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_buffer_kind.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_cfg_cleanup.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_cfg_convert.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_cse.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_derivative.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_descriptor_indexing.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_function.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_lower_consume.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_lower_io.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_lower_min16.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_propagate_resource_types.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_propagate_types.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_remove_unused.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_scalarize.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_scratch.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_ssa.cpp"
    "${DXBC_SPV_DIR}/ir/passes/ir_pass_sync.cpp"
    "${DXBC_SPV_DIR}/util/util_float16.cpp"
    "${DXBC_SPV_DIR}/util/util_log.cpp"
    "${DXBC_SPV_DIR}/util/util_md5.cpp"
    "${DXBC_SPV_DIR}/util/util_swizzle.cpp"
)

add_library(dxbc_spv_sm3 STATIC ${DXBC_SPV_SOURCES})
target_include_directories(dxbc_spv_sm3 PUBLIC "${DXBC_SPV_DIR}")
target_compile_definitions(dxbc_spv_sm3 PUBLIC DXBC_SPV_ENABLE_SM3 DXBC_SPV_ENABLE_SPIRV)
target_compile_options(dxbc_spv_sm3 PRIVATE -O2 -g0 -Wno-unused-parameter -Wno-missing-field-initializers)

set(SPIRV_CROSS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/third_party/spirv-cross")
set(SPIRV_CROSS_INCLUDE_DIR "${SPIRV_CROSS_DIR}/include")
add_library(spirv_cross STATIC
    "${SPIRV_CROSS_DIR}/src/spirv_cross.cpp"
    "${SPIRV_CROSS_DIR}/src/spirv_parser.cpp"
    "${SPIRV_CROSS_DIR}/src/spirv_cross_parsed_ir.cpp"
    "${SPIRV_CROSS_DIR}/src/spirv_cfg.cpp"
    "${SPIRV_CROSS_DIR}/src/spirv_glsl.cpp"
    "${SPIRV_CROSS_DIR}/src/spirv_msl.cpp"
)
target_include_directories(spirv_cross
    PUBLIC "${SPIRV_CROSS_INCLUDE_DIR}"
    PRIVATE "${SPIRV_CROSS_INCLUDE_DIR}/spirv_cross"
)
target_compile_options(spirv_cross PRIVATE -O2 -g0 -Wno-unused-parameter)
set(SPIRV_CROSS_LIBS spirv_cross)

# Standalone shader translation verification tool
add_executable(shader_test
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/shaders/mtl_shader_compiler.mm"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/shaders/shader_test_main.mm"
)
target_include_directories(shader_test PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/include"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/shaders"
    "${SPIRV_CROSS_INCLUDE_DIR}"
)
target_compile_options(shader_test PRIVATE -O2 -fobjc-arc)
target_link_libraries(shader_test PRIVATE
    dxbc_spv_sm3
    ${SPIRV_CROSS_LIBS}
    "-framework Metal"
    "-framework Foundation"
)

# ---- Primary Target: RunnerMetal (100% Native Metal, Zero Vulkan/MoltenVK) ----
set(D3D9_METAL_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/core/mtl_d3d9_entry.mm"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/core/mtl_d3d9_device.mm"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/resources/mtl_format.mm"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/resources/mtl_resources.mm"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/pipeline/mtl_pipeline_cache.mm"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/shaders/mtl_shader_compiler.mm"
)

add_executable(RunnerMetal MACOSX_BUNDLE
    ${RUNTIME_SOURCES}
    ${D3D9_METAL_SOURCES}
    $<TARGET_OBJECTS:me_shatter>
    $<TARGET_OBJECTS:me_lifted>
)
set_target_properties(RunnerMetal PROPERTIES
    MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/src/host_macos/packaging/Info.plist.in"
    OUTPUT_NAME "RunnerMetal"
)
# Ad-hoc sign .app bundle (required for macOS Game Mode / gamepolicyd) and keep build/RunnerMetal symlink
add_custom_command(TARGET RunnerMetal POST_BUILD
    COMMAND codesign --force --deep --sign - RunnerMetal.app
    COMMAND ${CMAKE_COMMAND} -E rm -f RunnerMetal
    COMMAND ${CMAKE_COMMAND} -E create_symlink "RunnerMetal.app/Contents/MacOS/RunnerMetal" RunnerMetal
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
)
target_include_directories(RunnerMetal PRIVATE
    "${RUNTIME_DIR}"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/include"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/core"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/resources"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/pipeline"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/d3d9_metal/shaders"
    "${SPIRV_CROSS_INCLUDE_DIR}"
)
target_compile_definitions(RunnerMetal PRIVATE RT_SHATTER=1)
target_compile_options(RunnerMetal PRIVATE
    $<$<COMPILE_LANGUAGE:C>:-O2 -g0 -fwrapv -ffp-contract=off -fno-strict-aliasing>
    $<$<COMPILE_LANGUAGE:CXX,OBJCXX>:-O2 -g0 -fobjc-arc>
)
target_link_libraries(RunnerMetal PRIVATE
    SDL3::SDL3
    dxbc_spv_sm3
    ${SPIRV_CROSS_LIBS}
    "-framework Metal"
    "-framework QuartzCore"
    "-framework Cocoa"
    "-framework CoreAudio"
    "-framework Foundation"
    m
    pthread
)
target_link_options(RunnerMetal PRIVATE
    "-Wl,-stack_size,0x4000000"
    "-Wl,-rpath,@executable_path/../Frameworks"
    "-Wl,-rpath,${HOMEBREW_PREFIX}/opt/sdl3/lib"
    "-Wl,-rpath,${HOMEBREW_PREFIX}/lib"
)

# ---- Optional Visual Oracle Target: RunnerOracle (DXVK + MoltenVK) ----
if(ENABLE_MOLTENVK_ORACLE)
    add_executable(RunnerOracle
        ${RUNTIME_SOURCES}
        $<TARGET_OBJECTS:me_shatter>
        $<TARGET_OBJECTS:me_lifted>
    )
    target_include_directories(RunnerOracle PRIVATE "${RUNTIME_DIR}")
    target_compile_definitions(RunnerOracle PRIVATE RT_SHATTER=1 ENABLE_MOLTENVK_ORACLE=1)
    target_compile_options(RunnerOracle PRIVATE
        $<$<COMPILE_LANGUAGE:C>:-O2 -g0 -fwrapv -ffp-contract=off -fno-strict-aliasing>
    )
    target_link_libraries(RunnerOracle PRIVATE SDL3::SDL3 m pthread)
    target_link_options(RunnerOracle PRIVATE
        "-Wl,-stack_size,0x4000000"
        "-Wl,-rpath,${HOMEBREW_PREFIX}/opt/sdl3/lib"
        "-Wl,-rpath,${HOMEBREW_PREFIX}/lib"
    )
endif()
