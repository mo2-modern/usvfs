cmake_minimum_required(VERSION 3.16)

find_package(asmjit CONFIG REQUIRED)

# find_library() is configuration-agnostic: it resolves once at configure time
# and that single path is used for every configuration. vcpkg puts its debug
# prefix FIRST on CMAKE_PREFIX_PATH whenever CMAKE_BUILD_TYPE is undefined,
# which is always the case for a multi-config generator, so a bare
# find_library() binds the debug libudis86 into Release as well -- pulling
# MSVCRTD into the usvfs DLL that gets injected into the target process.
# Resolve both explicitly and let CMake select per configuration.
include(SelectLibraryConfigurations)
set(LIBUDIS86_ROOT "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}")
find_library(LIBUDIS86_LIBRARY_RELEASE
    NAMES libudis86 udis86
    HINTS "${LIBUDIS86_ROOT}/lib"
    NO_DEFAULT_PATH)
find_library(LIBUDIS86_LIBRARY_DEBUG
    NAMES libudis86 udis86
    HINTS "${LIBUDIS86_ROOT}/debug/lib"
    NO_DEFAULT_PATH)
unset(LIBUDIS86_ROOT)
select_library_configurations(LIBUDIS86)

find_package(spdlog CONFIG REQUIRED)

file(GLOB sources CONFIGURE_DEPENDS "*.cpp" "*.h")
source_group("" FILES ${sources})

add_library(thooklib STATIC ${sources})
# asmjit is part of thooklib's PUBLIC interface: ttrampolinepool.h includes
# <asmjit/asmjit.h> and its member signatures take asmjit::X86Assembler&. Linking
# it PRIVATE withheld asmjit's ASMJIT_STATIC usage requirement from consumers, so
# usvfs_dll compiled those headers with ASMJIT_API = __declspec(dllimport) and
# imported symbols that live in a static lib (LNK4217/LNK4286 at every link).
target_link_libraries(thooklib
    PUBLIC asmjit::asmjit
    PRIVATE shared ${LIBUDIS86_LIBRARY} spdlog::spdlog_header_only)
target_include_directories(thooklib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
set_target_properties(thooklib PROPERTIES FOLDER injection)
target_precompile_headers(shared PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../shared/pch.h)
