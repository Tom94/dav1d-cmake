# Script to build dav1d with CMake
# I, Miku AuahDark grant you permission to use this script under
# BSD-2 Clause license
# This CMake also depends on TargetArch.cmake which is shipped along with this
# repository, modified to detect AArch64. To get the unmodified version, visit
# https://github.com/axr/solar-cmake/blob/73cfea0/TargetArch.cmake

cmake_minimum_required(VERSION 3.10.0)

set(CMAKE_C_STANDARD 99)

project(dav1d VERSION "0.7.1" LANGUAGES C)
set(PROJECT_VERSION_REVISION ${PROJECT_VERSION_PATCH})

if(POLICY CMP0025)
    cmake_policy(SET CMP0025 NEW)
endif()

find_file(DAV1D_VERSION_H
  NAMES dav1d/version.h
  PATHS dav1d/include
  NO_DEFAULT_PATH
)

if(NOT DAV1D_VERSION_H)
  message(FATAL_ERROR "Could not find dav1d/version.h in ${dav1d_inc_dirs}")
endif()

file(READ "${DAV1D_VERSION_H}" _dav1d_version_h)

string(REGEX MATCH "DAV1D_API_VERSION_MAJOR[ \t]+([0-9]+)"
  _match "${_dav1d_version_h}")
set(DAV1D_API_VERSION_MAJOR "${CMAKE_MATCH_1}")

string(REGEX MATCH "DAV1D_API_VERSION_MINOR[ \t]+([0-9]+)"
  _match "${_dav1d_version_h}")
set(DAV1D_API_VERSION_MINOR "${CMAKE_MATCH_1}")

string(REGEX MATCH "DAV1D_API_VERSION_PATCH[ \t]+([0-9]+)"
  _match "${_dav1d_version_h}")
set(DAV1D_API_VERSION_REVISION "${CMAKE_MATCH_1}")

set(API_VERSION_MAJOR DAV1D_API_VERSION_MAJOR)
set(API_VERSION_MINOR DAV1D_API_VERSION_MINOR)
set(API_VERSION_REVISION DAV1D_API_VERSION_REVISION)

###########
# Options #
###########

# bitdepths
option(bitdepth_8 "Enable specified bitdepths" ON)
option(bitdepth_16 "Enable specified bitdepths" ON)
set(bitdepth_16 ON CACHE BOOL "Enable specified bitdepths" FORCE)

# enable_asm
option(enable_asm "Build asm files, if available" ON)

# enable_avx512
option(enable_avx512 "Build AVX-512 asm files, requires nasm 2.14" ON)

# enable_tools
option(enable_tools "Build dav1d cli tools" OFF)

# enable_examples
option(enable_examples "Build dav1d examples" OFF)

# TODO enable_tests

# logging
option(logging "Print error log messages using the provided callback function" ON)

# TODO testdata_tests
# TODO fuzzing_engine
# TODO fuzzer_ldflags

# stack_alignment
set(stack_alignment "0" CACHE STRING "")

##############
# End Option #
##############

# compatibility
set(bitdepths "")

if(bitdepth_8)
    list(APPEND bitdepths "8")
endif()

if(bitdepth_16)
    list(APPEND bitdepths "16")
endif()

foreach(BITS IN LISTS bitdepths)
    set(CONFIG_${BITS}BPC 1)
endforeach()

function(target_architecture output_var)
    if(APPLE AND CMAKE_OSX_ARCHITECTURES)
        # Explicit arch list wins (universal or forced single arch)
        set(archs "")
        foreach(a ${CMAKE_OSX_ARCHITECTURES})
            if(a STREQUAL "arm64")
                list(APPEND archs "arm64")
            elseif(a STREQUAL "x86_64")
                list(APPEND archs "x86_64")
            elseif(a STREQUAL "i386")
                list(APPEND archs "i386")
            else()
                list(APPEND archs "${a}")
            endif()
        endforeach()
        set(${output_var} "${archs}" PARENT_SCOPE)
        return()
    endif()

    set(archdetect [[
#if defined(__aarch64__) || defined(__arm64__) || defined(__arm64) || defined(_M_ARM64)
    #error cmake_ARCH aarch64
#elif defined(__ARM_ARCH) || defined(__arm__) || defined(__thumb__) || defined(_M_ARM)
    #error cmake_ARCH arm
#elif defined(__x86_64__) || defined(__x86_64) || defined(__amd64) || defined(_M_X64) || defined(_M_AMD64)
    #error cmake_ARCH x86_64
#elif defined(__i386__) || defined(__i386) || defined(_M_IX86)
    #error cmake_ARCH i386
#elif defined(__ppc64__) || defined(__powerpc64__)
    #error cmake_ARCH ppc64
#elif defined(__ppc__) || defined(__powerpc__)
    #error cmake_ARCH ppc
#elif defined(__riscv) && __riscv_xlen == 64
    #error cmake_ARCH riscv64
#else
    #error cmake_ARCH unknown
#endif
]])

    set(probe "${CMAKE_CURRENT_BINARY_DIR}/arch_probe.c")
    file(WRITE "${probe}" "${archdetect}")

    enable_language(C)
    try_run(run_result compile_result
        "${CMAKE_CURRENT_BINARY_DIR}"
        "${probe}"
        COMPILE_OUTPUT_VARIABLE compile_out)

    string(REGEX MATCH "cmake_ARCH ([a-zA-Z0-9_]+)" _ "${compile_out}")
    set(arch "${CMAKE_MATCH_1}")

    if(NOT arch)
        set(arch "unknown")
    endif()

    set(${output_var} "${arch}" PARENT_SCOPE)
endfunction()

target_architecture(PROCESSOR)

if(${PROCESSOR} STREQUAL "i386" OR ${PROCESSOR} STREQUAL "x86_64")
    set(ARCH_X86 1)
endif()

if(${PROCESSOR} STREQUAL "riscv64" OR ${PROCESSOR} STREQUAL "riscv32")
    set(ARCH_RISCV 1)
endif()

if(enable_asm)
    # NOTE: MSVC does not support .S assembly files, so disable asm on ARM-based Windows. X86 is fine via NASM
    if(ARCH_X86 OR ((${PROCESSOR} STREQUAL "aarch64" OR ${PROCESSOR} MATCHES "^arm") AND NOT WIN32))
        set(HAVE_ASM 1)
    endif()
endif()

set(CONFIG_LOG ${logging})

set(TEMP_COMPILE_DEFS "")
set(TEMP_COMPILE_FLAGS "")
set(TEMP_LINK_LIBRARIES "")
set(TEMP_COMPAT_FILES "")
set(RT_DEPENDENCY "")

include(CheckSymbolExists)

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    list(APPEND TEMP_COMPILE_DEFS -D_GNU_SOURCE)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND TEMP_COMPILE_DEFS -D_DARWIN_C_SOURCE)
else()
    list(APPEND TEMP_COMPILE_DEFS -D_POSIX_C_SOURCE=200112L)
endif()

set(CMAKE_REQUIRED_DEFINITIONS ${TEMP_COMPILE_DEFS})

if(WIN32)
    set(_TARGETTING_WINDOWS 1)

    check_symbol_exists(fseeko "stdio.h" HAS_FSEEKO)

    if(HAS_FSEEKO)
        set(_FILE_OFFSET_BITS 64)
    else()
        set(_FSEEKO_FUNCTION _fseeki64)
        set(_FTELLO_FUNCTION _ftelli64)
    endif()

    if (${PROCESSOR} STREQUAL "x86_64" AND ${CMAKE_C_COMPILER_ID} STREQUAL "GNU")
        list(APPEND TEMP_COMPILE_FLAGS -mcmodel=small)
    endif()

    # On Windows, we use a compatibility layer to emulate pthread
    add_library(thread_compat_dep STATIC dav1d/src/win32/thread.c)
    target_include_directories(thread_compat_dep PRIVATE
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}/dav1d
        ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/include
    )
    set(THREAD_DEPENDENCY thread_compat_dep)
else()
    set(THREAD_DEPENDENCY pthread)

    check_include_file("pthread_np.h" HAVE_PTHREAD_NP_H)
    check_symbol_exists(pthread_getaffinity_np "pthread_np.h" HAVE_PTHREAD_GETAFFINITY_NP)
    check_symbol_exists(pthread_setaffinity_np "pthread_np.h" HAVE_PTHREAD_SETAFFINITY_NP)
    check_symbol_exists(pthread_setname_np "pthread_np.h" HAVE_PTHREAD_SETNAME_NP)
    check_symbol_exists(pthread_set_name_np "pthread_np.h" HAVE_PTHREAD_SET_NAME_NP)

    check_symbol_exists(clock_gettime "time.h" HAVE_CLOCK_GETTIME)

    if(NOT HAVE_CLOCK_GETTIME)
        message(FATAL_ERROR "clock_gettime not found")
    endif()
endif()

# Compile+link check of an asm fragment. Writes to a file to avoid
# check_c_source_compiles mangling backslash/quote escapes. The caller
# passes only the __asm__ argument list; we wrap it in a main().
function(check_c_compiles out_var c_code)
    set(_f "${CMAKE_BINARY_DIR}/ccheck_${out_var}.c")
    file(WRITE "${_f}" "${c_code}")
    try_compile(_res
        "${CMAKE_BINARY_DIR}/asmcheck_${out_var}_dir"
        SOURCES "${_f}")
    set(${out_var} ${_res} PARENT_SCOPE)
endfunction()

check_c_compiles(HAVE_C11_GENERIC "int main(void){ int x = _Generic(0, default: 0); return 0; }\n")
message(STATUS "HAVE_C11_GENERIC: ${HAVE_C11_GENERIC}")

# check for fseeko on android. It is not always available if _FILE_OFFSET_BITS is defined to 64
if(${CMAKE_SYSTEM_NAME} STREQUAL "Android")
    check_symbol_exists(fseeko "stdio.h" HAS_FSEEKO)

    if(NOT HAS_FSEEKO)
        set(CMAKE_REQUIRED_FLAGS "-U_FILE_OFFSET_BITS")
        check_symbol_exists(fseeko "stdio.h" HAS_FSEEKO)
        unset(CMAKE_REQUIRED_FLAGS)

        if(HAS_FSEEKO)
            message(WARNING "Files larger than 2 gigabytes might not be supported in the dav1d CLI tool.")
            list(APPEND TEMP_COMPILE_FLAGS -U_FILE_OFFSET_BITS)
        elseif(enable_tools)
            message(FATAL_ERROR "dav1d CLI tool needs fseeko()")
        else()
            unset(HAS_FSEEKO)
        endif()
    endif()
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_DL_LIBS})
    check_symbol_exists(dlsym "dlfcn.h" HAVE_DLSYM)
    unset(CMAKE_REQUIRED_LIBRARIES)

    if(HAVE_DLSYM)
        list(APPEND TEMP_LINK_LIBRARIES ${CMAKE_DL_LIBS})
    endif()
endif()

#
# Header checks
#
include(CheckIncludeFile)

check_include_file(stdatomic.h HAS_STDATOMIC)

if(NOT HAS_STDATOMIC)
    if(MSVC)
        # we have a custom replacement for MSVC
        add_library(stdatomic_dependency INTERFACE)
        target_include_directories(stdatomic_dependency INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/include/compat/msvc)
    else()
        try_compile(_GCC_STYLE_ATOMICS ${CMAKE_CURRENT_BINARY_DIR} stdatomic_gcc_test.c
            COMPILE_DEFINITIONS ${TEMP_COMPILE_DEFS}
        )

        if(_GCC_STYLE_ATOMICS)
            add_library(stdatomic_dependency INTERFACE)
            target_include_directories(stdatomic_dependency INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/include/compat/gcc)
        else()
            message(FATAL_ERROR "Atomics not supported")
        endif()
    endif()
endif()

check_include_file("sys/types.h" HAVE_SYS_TYPES_H)
check_include_file("unistd.h" HAVE_UNISTD_H)
check_include_file("io.h" HAVE_IO_H)

#
# Function checks
#
set(CMAKE_REQUIRED_DEFINITIONS ${TEMP_COMPILE_DEFS})
check_symbol_exists(getopt_long "getopt.h" HAS_GETOPT_LONG)

if(NOT HAS_GETOPT_LONG)
    add_library(getopt_dependency STATIC dav1d/tools/compat/getopt.c)
    set_property(TARGET getopt_dependency PROPERTY POSITION_INDEPENDENT_CODE ON)
    target_include_directories(getopt_dependency PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/include/compat)
endif()

check_symbol_exists(sigaction "signal.h" HAVE_SIGACTION)
check_symbol_exists(aligned_alloc "stdlib.h" HAVE_ALIGNED_ALLOC)
check_symbol_exists(posix_memalign "stdlib.h" HAVE_POSIX_MEMALIGN)
check_symbol_exists(memalign "malloc.h" HAVE_MEMALIGN)

if(
    ${PROCESSOR} STREQUAL "aarch64" OR
    ${PROCESSOR} MATCHES "^arm" OR
    ${PROCESSOR} STREQUAL "ppc64"
)
    check_symbol_exists(getauxval "sys/auxv.h" HAVE_GETAUXVAL)
    check_symbol_exists(elf_aux_info "sys/auxv.h" HAVE_ELF_AUX_INFO)
endif()

#
# Compiler flag tests
#
include(CheckCCompilerFlag)

if(NOT MSVC)
    check_c_compiler_flag("-fvisibility=hidden" HAS_VISIBILITY_HIDDEN)
    if(HAS_VISIBILITY_HIDDEN)
        list(APPEND TEMP_COMPILE_FLAGS -fvisibility=hidden)
    else()
        message(WARNING "Compiler does not support -fvisibility=hidden, all symbols will be public!")
    endif()
endif()

# Compiler flags that should be set
# But when the compiler does not supports them
# it is not an error and silently tolerated
set(OPTIONAL_FLAGS "")

if(MSVC)
    list(APPEND OPTIONAL_FLAGS -wd4028) # parameter different from declaration
    list(APPEND OPTIONAL_FLAGS -wd4996) # use of POSIX functions
else()
    list(APPEND OPTIONAL_FLAGS
      -Wundef
      -Werror=vla
      -Wno-maybe-uninitialized
      -Wno-missing-field-initializers
      -Wno-unused-parameter
      -Werror=missing-prototypes
      -Wshorten-64-to-32
    )

    if(${CMAKE_SYSTEM_NAME} MATCHES "^i{d}86$")
        list(APPEND OPTIONAL_FLAGS
            -msse2
            -mfpmath=sse
        )
    endif()
endif()

if(NOT ${CMAKE_BUILD_TYPE} STREQUAL "Debug")
    list(APPEND OPTIONAL_FLAGS
        -fomit-frame-pointer
        -ffast-math
    )
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin" AND "${XCODE_VERSION}" MATCHES "^11")
    # Workaround for Xcode 11 -fstack-check bug, see #301
    list(APPEND OPTIONAL_FLAGS -fno-stack-check)
endif()

foreach(FLAG IN LISTS OPTIONAL_FLAGS)
    check_c_compiler_flag(${FLAG} _COMPILE_FLAG_${FLAG})
    if(_COMPILE_FLAG_${FLAG})
        list(APPEND TEMP_COMPILE_FLAGS ${FLAG})
    endif()
endforeach()

# TODO libFuzzer related things

#
# Stack alignments flags
#
include(TestBigEndian)
test_big_endian(ENDIANNESS_BIG)

set(TRIM_DSP_FUNCTIONS 0)

if(ARCH_X86)
    if(${stack_alignment} GREATER 0)
        set(STACK_ALIGNMENT ${stack_alignment})
    elseif(${PROCESSOR} STREQUAL "x86_64")
        set(STACK_ALIGNMENT 16)
    elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux" OR ${CMAKE_SYSTEM_NAME} MATCHES "Darwin" OR APPLE)
        set(STACK_ALIGNMENT 16)
    else()
        set(STACK_ALIGNMENT 4)
    endif()
endif()

if(${PROCESSOR} STREQUAL "aarch64" OR ${PROCESSOR} STREQUAL "arm64")
    set(ARCH_AARCH64 1)
elseif(${PROCESSOR} MATCHES "^arm")
    set(ARCH_ARM 1)
endif()

if(HAVE_ASM AND (ARCH_ARM OR ARCH_AARCH64))
    enable_language(ASM)

    set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

    # Compile+link check of an asm fragment. Writes to a file to avoid
    # check_c_source_compiles mangling backslash/quote escapes. The caller
    # passes only the __asm__ argument list; we wrap it in a main().
    function(check_asm_compiles out_var asm_args)
        set(_f "${CMAKE_BINARY_DIR}/asmcheck_${out_var}.c")
        file(WRITE "${_f}" "int main(void){ ${asm_args} return 0; }\n")
        try_compile(_res
            "${CMAKE_BINARY_DIR}/asmcheck_${out_var}_dir"
            SOURCES "${_f}")
        set(${out_var} ${_res} PARENT_SCOPE)
    endfunction()

    check_asm_compiles(HAVE_AS_FUNC "__asm__ (
        \".func meson_test\"
        \".endfunc\"
    );")

    check_asm_compiles(HAVE_AS_ARCH_DIRECTIVE "__asm__ (\".arch armv8-a\\n\");")

    message(STATUS "HAVE_AS_FUNC: ${HAVE_AS_FUNC}")
    message(STATUS "HAVE_AS_ARCH_DIRECTIVE: ${HAVE_AS_ARCH_DIRECTIVE}")

    if (HAVE_AS_ARCH_DIRECTIVE)
        set(as_arch_level "armv8-a")

        # Check what .arch levels are supported. In principle, we only
        # want to detect up to armv8.2-a here (binutils requires that
        # in order to enable i8mm). However, older Clang versions
        # (before Clang 17, and Xcode versions up to and including 15.0)
        # didn't support controlling dotprod/i8mm extensions via
        # .arch_extension, therefore try to enable a high enough .arch
        # level as well, to implicitly make them available via that.
        foreach(arch "armv8.2-a" "armv8.4-a" "armv8.6-a")
            check_asm_compiles(as_arch_supports_${arch} "__asm__ (\".arch ${arch}\\n\");")
            if(as_arch_supports_${arch})
                set(as_arch_level "${arch}")
            endif()
        endforeach()

        # Clang versions before 17 also had a bug
        # (https://github.com/llvm/llvm-project/issues/32220)
        # causing a plain ".arch <level>" to not have any effect unless it
        # had an extra "+<feature>" included - but it was activated on the
        # next ".arch_extension" directive instead. Check if we can include
        # "+crc" as dummy feature to make the .arch directive behave as
        # expected and take effect right away.
        check_asm_compiles(as_arch_level_supports_crc "__asm__ (\".arch ${as_arch_level}+crc\\n\");")
        if(as_arch_level_supports_crc)
            set(as_arch_level "${as_arch_level}+crc")
        endif()

        set(AS_ARCH_LEVEL "${as_arch_level}")
        set(as_arch_str "\".arch ${as_arch_level}\\n\"")

        message(STATUS "AS_ARCH_LEVEL: ${AS_ARCH_LEVEL}")
    endif()

    # aarch64 extension test instructions.
    set(aarch64_ext_names    "dotprod" "i8mm" "sve" "sve2")
    set(aarch64_ext_dotprod  "udot v0.4s, v0.16b, v0.16b")
    set(aarch64_ext_i8mm     "usdot v0.4s, v0.16b, v0.16b")
    set(aarch64_ext_sve      "whilelt p0.s, x0, x1")
    set(aarch64_ext_sve2     "sqrdmulh z0.s, z0.s, z0.s")

    set(supported_aarch64_archexts "")
    set(supported_aarch64_instructions "")

    foreach(name ${aarch64_ext_names})
        set(instr "${aarch64_ext_${name}}")

        # Test for support for the various extensions. First test if
        # the assembler supports the .arch_extension directive for
        # enabling/disabling the extension, then separately check whether
        # the instructions themselves are supported. Even if .arch_extension
        # isn't supported, we may be able to assemble the instructions
        # if the .arch level includes support for them.
        string(TOUPPER "${name}" name_upper)

        check_asm_compiles(as_supports_archext_${name} "__asm__ (${as_arch_str}\".arch_extension ${name}\\n\");")

        set(instr_code "__asm__ (${as_arch_str}")
        if(as_supports_archext_${name})
            list(APPEND supported_aarch64_archexts "${name}")
            string(APPEND instr_code "\".arch_extension ${name}\\n\"")
        endif()
        string(APPEND instr_code "\"${instr}\\n\");")

        check_asm_compiles(as_supports_instr_${name} "${instr_code}")
        if(as_supports_instr_${name})
            list(APPEND supported_aarch64_instructions "${name}")
        endif()
    endforeach()

    message(STATUS "Supported AArch64 .arch_extension: ${supported_aarch64_archexts}")
    message(STATUS "Supported AArch64 instructions: ${supported_aarch64_instructions}")

    foreach(name ${aarch64_ext_names})
        string(TOUPPER "${name}" name_upper)

        if("${name}" IN_LIST supported_aarch64_archexts)
            set(HAVE_AS_ARCHEXT_${name_upper}_DIRECTIVE 1)
        else()
            set(HAVE_AS_ARCHEXT_${name_upper}_DIRECTIVE 0)
        endif()

        if("${name}" IN_LIST supported_aarch64_instructions)
            set(HAVE_${name_upper} 1)
        else()
            set(HAVE_${name_upper} 0)
        endif()

        message(STATUS "HAVE_AS_ARCHEXT_${name_upper}_DIRECTIVE: ${HAVE_AS_ARCHEXT_${name_upper}_DIRECTIVE}")
        message(STATUS "HAVE_${name_upper}: ${HAVE_${name_upper}}")
    endforeach()

    # fedora package build infrastructure uses a gcc specs file to enable
    # '-fPIE' by default. The chosen way only adds '-fPIE' to the C compiler
    # with integrated preprocessor. It is not added to the standalone
    # preprocessor or the preprocessing stage of '.S' files. So we have to
    # compile code to check if we have to define PIC for the arm asm to
    # avoid absolute relocations when building for example checkasm.
    try_compile(_HAS_PIC ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/check_pic_code.c)
    unset(CMAKE_TRY_COMPILE_TARGET_TYPE)

    if(_HAS_PIC)
        set(PIC 3)
    endif()
endif()

if(ARCH_X86)
    set(private_prefix "dav1d")

    check_c_compiles(FORCE_VEX_ENCODING "
        #ifndef __AVX__
        #error no avx
        #endif
        int main(void) { return 0; }
    ")

    # Convert SSE asm into (128-bit) AVX when compiler flags are set to use AVX instructions
    if(${PROCESSOR} STREQUAL "x86_64")
        set(ARCH_X86_64 1)
    else()
        set(ARCH_X86_32 1)
    endif()

    set(PIC 1)
endif()

if(${PROCESSOR} STREQUAL "ppc64")
    set(ARCH_PPC64LE 1)
endif()

if (${PROCESSOR} STREQUAL "loongarch64")
    set(ARCH_LOONGARCH64 1)
    set(ARCH_LOONGARCH 1)
elseif (${PROCESSOR} STREQUAL "loongarch32")
    set(ARCH_LOONGARCH32 1)
    set(ARCH_LOONGARCH 1)
endif()

# https://mesonbuild.com/Release-notes-for-0-37-0.html#new-compiler-function-symbols_have_underscore_prefix
# For example, Windows 32-bit prefixes underscore, but 64-bit does not.
# Linux does not prefix an underscore but OS X does.
if((WIN32 AND CMAKE_SIZEOF_VOID_P EQUAL 4 AND ARCH_X86) OR ${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(UNDERSCORE_PREFIXED 1)
endif()

#
# ASM specific stuff
#
if(HAVE_ASM AND ARCH_X86)
    # NASM compiler support
    enable_language(ASM_NASM)

    # check NASM version
    execute_process(COMMAND "${CMAKE_ASM_NASM_COMPILER}" "--version" OUTPUT_VARIABLE NASM_VERSION_FULL)

    if("${NASM_VERSION_FULL}" MATCHES "^NASM version")
        string(REPLACE " " ";" NASM_VERSION_FULL "${NASM_VERSION_FULL}")
        list(GET NASM_VERSION_FULL 2 NASM_VERSION)

        if(enable_avx512 AND ${NASM_VERSION} VERSION_LESS "2.14")
            message(FATAL_ERROR "nasm 2.14 or later is required for AVX-512 asm. AVX-512 asm can be disabled with '-Denable_avx512=false'")
        elseif(${NASM_VERSION} VERSION_LESS "2.13.02")
            message(FATAL_ERROR "nasm 2.13.02 or later is required, found nasm ${NASM_VERSION}")
        endif()

        set(HAVE_AVX512ICL ${enable_avx512})
    else()
        message(FATAL_ERROR "unexpected nasm version string: ${NASM_VERSION_FULL}")
    endif()

    # Generate config.asm
    configure_file(config.asm.in ${CMAKE_CURRENT_BINARY_DIR}/config.asm.in2)
    file(READ ${CMAKE_CURRENT_BINARY_DIR}/config.asm.in2 CONFIG_ASM_CONTENTS_TEMP)
    string(REPLACE "#" "%" CONFIG_ASM_CONTENTS_TEMP2 ${CONFIG_ASM_CONTENTS_TEMP})
    string(REPLACE "/*" ";" CONFIG_ASM_CONTENTS ${CONFIG_ASM_CONTENTS_TEMP2})
    file(GENERATE OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/config.asm CONTENT "${CONFIG_ASM_CONTENTS}")
endif()

configure_file(config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

# Revision file (vcs_version.h) generation
set(DAV1D_GIT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/dav1d/.git")

find_package(Git)

if(Git_FOUND)
    execute_process(COMMAND ${GIT_EXECUTABLE} --git-dir "${DAV1D_GIT_DIR}" describe --tags --long --match ?.*.* --always
        OUTPUT_VARIABLE VCS_TAG
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
else()
    set(VCS_TAG "unknown")
endif()

message(STATUS "Git revision: ${VCS_TAG}")
configure_file(dav1d/include/vcs_version.h.in ${CMAKE_CURRENT_BINARY_DIR}/vcs_version.h)
unset(VCS_TAG)

# configure_file(dav1d/include/dav1d/version.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/dav1d/version.h)

# libdav1d source files
set(LIBDAV1D_SOURCES
    dav1d/src/cdf.c
    dav1d/src/cpu.c
    dav1d/src/ctx.c
    dav1d/src/data.c
    dav1d/src/decode.c
    dav1d/src/dequant_tables.c
    dav1d/src/getbits.c
    dav1d/src/intra_edge.c
    dav1d/src/itx_1d.c
    dav1d/src/lf_mask.c
    dav1d/src/log.c
    dav1d/src/mem.c
    dav1d/src/msac.c
    dav1d/src/obu.c
    dav1d/src/pal.c
    dav1d/src/picture.c
    dav1d/src/qm.c
    dav1d/src/ref.c
    dav1d/src/refmvs.c
    dav1d/src/scan.c
    dav1d/src/tables.c
    dav1d/src/warpmv.c
    dav1d/src/wedge.c
)

# libdav1d bitdepth source files
# These files are compiled for each bitdepth with
# `BITDEPTH` defined to the currently built bitdepth.
set(LIBDAV1D_TMPL_SOURCES
    dav1d/src/cdef_apply_tmpl.c
    dav1d/src/cdef_tmpl.c
    dav1d/src/fg_apply_tmpl.c
    dav1d/src/filmgrain_tmpl.c
    dav1d/src/ipred_prepare_tmpl.c
    dav1d/src/ipred_tmpl.c
    dav1d/src/itx_tmpl.c
    dav1d/src/lf_apply_tmpl.c
    dav1d/src/loopfilter_tmpl.c
    dav1d/src/looprestoration_tmpl.c
    dav1d/src/lr_apply_tmpl.c
    dav1d/src/mc_tmpl.c
    dav1d/src/recon_tmpl.c
)

set(LIBDAV1D_ARCH_TMPL_SOURCES "")
set(LIBDAV1D_BITDEPTH_OBJS "")

# libdav1d entrypoint source files
# These source files contain library entry points and are
# built with the stack-realign flag set, where necessary.
set(LIBDAV1D_ENTRYPOINT_SOURCES
    dav1d/src/lib.c
    dav1d/src/thread_task.c
)

# ASM specific sources
set(LIBDAV1D_SOURCES_ASM "")
# Arch-specific flags
set(ARCH_SPECIFIC_FLAGS "")

if(HAVE_ASM)
    if(ARCH_ARM OR ARCH_AARCH64)
        list(APPEND LIBDAV1D_SOURCES dav1d/src/arm/cpu.c)

        if(ARCH_AARCH64)
            list(APPEND LIBDAV1D_SOURCES
                # itx.S is used for both 8 and 16 bpc.
                dav1d/src/arm/64/itx.S
                dav1d/src/arm/64/looprestoration_common.S
                dav1d/src/arm/64/msac.S
                dav1d/src/arm/64/refmvs.S
            )

            if(CONFIG_8BPC)
                list(APPEND LIBDAV1D_SOURCES
                    dav1d/src/arm/64/cdef.S
                    dav1d/src/arm/64/filmgrain.S
                    dav1d/src/arm/64/ipred.S
                    dav1d/src/arm/64/loopfilter.S
                    dav1d/src/arm/64/looprestoration.S
                    dav1d/src/arm/64/mc.S
                    dav1d/src/arm/64/mc_dotprod.S
                )
            endif()

            if(CONFIG_16BPC)
                list(APPEND LIBDAV1D_SOURCES
                    dav1d/src/arm/64/cdef16.S
                    dav1d/src/arm/64/filmgrain16.S
                    dav1d/src/arm/64/ipred16.S
                    dav1d/src/arm/64/itx16.S
                    dav1d/src/arm/64/loopfilter16.S
                    dav1d/src/arm/64/looprestoration16.S
                    dav1d/src/arm/64/mc16.S
                    dav1d/src/arm/64/mc16_sve.S
                )
            endif()
        else()
            list(APPEND LIBDAV1D_SOURCES
                dav1d/src/arm/32/itx.S
                dav1d/src/arm/32/looprestoration_common.S
                dav1d/src/arm/32/msac.S
                dav1d/src/arm/32/refmvs.S
            )

            if(CONFIG_8BPC)
                list(APPEND LIBDAV1D_SOURCES
                    dav1d/src/arm/32/cdef.S
                    dav1d/src/arm/32/filmgrain.S
                    dav1d/src/arm/32/ipred.S
                    dav1d/src/arm/32/loopfilter.S
                    dav1d/src/arm/32/looprestoration.S
                    dav1d/src/arm/32/mc.S
                )
            endif()

            if(CONFIG_16BPC)
                list(APPEND LIBDAV1D_SOURCES
                    dav1d/src/arm/32/cdef16.S
                    dav1d/src/arm/32/filmgrain16.S
                    dav1d/src/arm/32/ipred16.S
                    dav1d/src/arm/32/itx16.S
                    dav1d/src/arm/32/loopfilter16.S
                    dav1d/src/arm/32/looprestoration16.S
                    dav1d/src/arm/32/mc16.S
                )
            endif()
        endif()
    elseif(ARCH_X86)
        list(APPEND LIBDAV1D_SOURCES
            dav1d/src/x86/cpu.c
        )

        # NASM source files
        list(APPEND LIBDAV1D_SOURCES_ASM
            dav1d/src/x86/cpuid.asm
            dav1d/src/x86/msac.asm
            dav1d/src/x86/pal.asm
            dav1d/src/x86/refmvs.asm
            dav1d/src/x86/itx_avx512.asm
            dav1d/src/x86/cdef_avx2.asm
            dav1d/src/x86/itx_avx2.asm
            dav1d/src/x86/cdef_sse.asm
            dav1d/src/x86/itx_sse.asm
        )

        if(CONFIG_8BPC)
            list(APPEND LIBDAV1D_SOURCES_ASM
                dav1d/src/x86/cdef_avx512.asm
                dav1d/src/x86/filmgrain_avx512.asm
                dav1d/src/x86/ipred_avx512.asm
                dav1d/src/x86/loopfilter_avx512.asm
                dav1d/src/x86/looprestoration_avx512.asm
                dav1d/src/x86/mc_avx512.asm
                dav1d/src/x86/filmgrain_avx2.asm
                dav1d/src/x86/ipred_avx2.asm
                dav1d/src/x86/loopfilter_avx2.asm
                dav1d/src/x86/looprestoration_avx2.asm
                dav1d/src/x86/mc_avx2.asm
                dav1d/src/x86/cdef_sse.asm
                dav1d/src/x86/filmgrain_sse.asm
                dav1d/src/x86/ipred_sse.asm
                dav1d/src/x86/loopfilter_sse.asm
                dav1d/src/x86/looprestoration_sse.asm
                dav1d/src/x86/mc_sse.asm
            )
        endif()

        if(CONFIG_16BPC)
            list(APPEND LIBDAV1D_SOURCES_ASM
                dav1d/src/x86/cdef16_avx512.asm
                dav1d/src/x86/filmgrain16_avx512.asm
                dav1d/src/x86/ipred16_avx512.asm
                dav1d/src/x86/itx16_avx512.asm
                dav1d/src/x86/loopfilter16_avx512.asm
                dav1d/src/x86/looprestoration16_avx512.asm
                dav1d/src/x86/mc16_avx512.asm
                dav1d/src/x86/cdef16_avx2.asm
                dav1d/src/x86/filmgrain16_avx2.asm
                dav1d/src/x86/ipred16_avx2.asm
                dav1d/src/x86/itx16_avx2.asm
                dav1d/src/x86/loopfilter16_avx2.asm
                dav1d/src/x86/looprestoration16_avx2.asm
                dav1d/src/x86/mc16_avx2.asm
                dav1d/src/x86/cdef16_sse.asm
                dav1d/src/x86/filmgrain16_sse.asm
                dav1d/src/x86/ipred16_sse.asm
                dav1d/src/x86/itx16_sse.asm
                dav1d/src/x86/loopfilter16_sse.asm
                dav1d/src/x86/looprestoration16_sse.asm
                dav1d/src/x86/mc16_sse.asm
            )
        endif()
    endif()
    # TODO: add powerpc, loongarch, riscv. Not needed in tev, so not implemented yet
endif()

#
# Windows .rc file and API export flags
#
set(API_EXPORT_DEFS "")

if(WIN32 AND BUILD_SHARED_LIBS)
    set(COPYRIGHT_YEARS 2019)
    configure_file(dav1d/src/dav1d.rc.in ${CMAKE_CURRENT_BINARY_DIR}/dav1d.rc)
    list(APPEND LIBDAV1D_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/dav1d.rc)
    list(APPEND API_EXPORT_DEFS DAV1D_BUILDING_DLL)
endif()

#
# Library definitions
#

set(LIBDAV1D_INCLUDE_DIRS_PRIV
    ${CMAKE_CURRENT_SOURCE_DIR}/dav1d
    ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/include/dav1d
    ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/src
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}/include/dav1d
)
set(LIBDAV1D_INCLUDE_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/include
    ${CMAKE_CURRENT_BINARY_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/include/dav1d
    ${CMAKE_CURRENT_BINARY_DIR}/include/dav1d
)

# Helper library for each bitdepth (and architecture-specific flags)
set(DAV1D_BITDEPTH_OBJECTS "")
foreach(BITS IN LISTS bitdepths)
    add_library(dav1d_bitdepth_${BITS} OBJECT ${LIBDAV1D_TMPL_SOURCES})
    set_property(TARGET dav1d_bitdepth_${BITS} PROPERTY POSITION_INDEPENDENT_CODE ON)
    target_include_directories(dav1d_bitdepth_${BITS} PRIVATE ${LIBDAV1D_INCLUDE_DIRS_PRIV})
    target_include_directories(dav1d_bitdepth_${BITS} PUBLIC ${LIBDAV1D_INCLUDE_DIRS})
    target_compile_options(dav1d_bitdepth_${BITS} PRIVATE ${TEMP_COMPILE_FLAGS})
    target_compile_definitions(dav1d_bitdepth_${BITS} PRIVATE ${TEMP_COMPILE_DEFS} BITDEPTH=${BITS})

    if(NOT HAS_STDATOMIC)
        target_link_libraries(dav1d_bitdepth_${BITS} stdatomic_dependency)
    endif()

    list(APPEND DAV1D_BITDEPTH_OBJECTS $<TARGET_OBJECTS:dav1d_bitdepth_${BITS}>)

    if(LIBDAV1D_ARCH_TMPL_SOURCES)
        add_library(dav1d_arch_bitdepth_${BITS} OBJECT ${LIBDAV1D_ARCH_TMPL_SOURCES})
        set_property(TARGET dav1d_arch_bitdepth_${BITS} PROPERTY POSITION_INDEPENDENT_CODE ON)
        target_include_directories(dav1d_arch_bitdepth_${BITS} PRIVATE ${LIBDAV1D_INCLUDE_DIRS_PRIV})
        target_include_directories(dav1d_arch_bitdepth_${BITS} PUBLIC ${LIBDAV1D_INCLUDE_DIRS})
        target_compile_options(dav1d_arch_bitdepth_${BITS} PRIVATE ${TEMP_COMPILE_FLAGS} ${ARCH_SPECIFIC_FLAGS})
        target_compile_definitions(dav1d_arch_bitdepth_${BITS} PRIVATE ${TEMP_COMPILE_DEFS} BITDEPTH=${BITS})
        if(NOT HAS_STDATOMIC)
            target_link_libraries(dav1d_arch_bitdepth_${BITS} stdatomic_dependency)
        endif()

        list(APPEND DAV1D_BITDEPTH_OBJECTS $<TARGET_OBJECTS:dav1d_arch_bitdepth_${BITS}>)
    endif()
endforeach()

# The final dav1d library
add_library(dav1d
    ${TEMP_COMPAT_FILES}
    ${LIBDAV1D_SOURCES}
    ${LIBDAV1D_SOURCES_ASM}
    ${LIBDAV1D_ENTRYPOINT_SOURCES}
    ${DAV1D_BITDEPTH_OBJECTS}
)

target_include_directories(dav1d PRIVATE ${LIBDAV1D_INCLUDE_DIRS_PRIV})
target_include_directories(dav1d PUBLIC ${LIBDAV1D_INCLUDE_DIRS})
target_compile_definitions(dav1d PRIVATE ${TEMP_COMPILE_DEFS} ${API_EXPORT_DEFS})

add_library(dav1d::dav1d ALIAS dav1d)

if(NOT WIN32)
    set_target_properties(dav1d PROPERTIES SOVERSION "${DAV1D_API_VERSION_MAJOR}")
else()
    set_target_properties(dav1d PROPERTIES PREFIX "lib")
endif()

if (NOT ("x${CMAKE_DL_LIBS}" STREQUAL "x"))
    target_link_libraries(dav1d ${CMAKE_DL_LIBS})
endif()

if(NOT HAS_STDATOMIC)
    target_link_libraries(dav1d stdatomic_dependency)
endif()

if(TEMP_LINK_LIBRARIES)
    target_link_directories(dav1d PUBLIC ${TEMP_LINK_LIBRARIES})
endif()

if(THREAD_DEPENDENCY)
    target_link_libraries(dav1d ${THREAD_DEPENDENCY})
endif()

if(enable_tools)
    file(GENERATE OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/cli_config.h CONTENT " ")

    add_executable(dav1d_exe
        dav1d/tools/input/input.c
        dav1d/tools/input/annexb.c
        dav1d/tools/input/ivf.c
        dav1d/tools/input/section5.c
        dav1d/tools/output/md5.c
        dav1d/tools/output/null.c
        dav1d/tools/output/output.c
        dav1d/tools/output/y4m2.c
        dav1d/tools/output/yuv.c
        dav1d/tools/dav1d.c
        dav1d/tools/dav1d_cli_parse.c
    )
    target_include_directories(dav1d_exe PRIVATE
        ${LIBDAV1D_INCLUDE_DIRS_PRIV}
        ${LIBDAV1D_INCLUDE_DIRS}
        ${CMAKE_CURRENT_SOURCE_DIR}/dav1d/tools
    )
    target_link_libraries(dav1d_exe dav1d)
    set_target_properties(dav1d_exe PROPERTIES OUTPUT_NAME "dav1d")

    if(NOT HAS_GETOPT_LONG)
        target_link_libraries(dav1d_exe getopt_dependency)
    endif()

    if(THREAD_DEPENDENCY)
        target_link_libraries(dav1d ${THREAD_DEPENDENCY})
    endif()
endif()

# TODO tests
