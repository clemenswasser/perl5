cmake_minimum_required(VERSION 3.20)

project(perl)

add_library(perldefaults INTERFACE)
target_include_directories(perldefaults INTERFACE
    include
    ${CMAKE_CURRENT_SOURCE_DIR}
    ..
    ${CMAKE_CURRENT_BINARY_DIR})
target_compile_definitions(perldefaults INTERFACE
    $<$<CONFIG:Debug>:DEBUGGING>
    _CONSOLE
    _CRT_NONSTDC_NO_DEPRECATE
    _CRT_SECURE_NO_DEPRECATE
    _WINSOCK_DEPRECATED_NO_WARNINGS
    NO_STRICT
    PERL_CORE
    NO_STRICT
    PERLDLL
    PERL_CORE
    WIN64)

include(CMakeDependentOption)

cmake_dependent_option(PERL_USE_64_BIT_INT "" ON
    "${CMAKE_SIZEOF_VOID_P} EQUAL 8" OFF)
option(PERL_USE_CPLUSPLUS "" OFF)

add_executable(perlglob perlglob.c)

function(perl_generator_append_mini_config_h OUT)
    list(TRANSFORM ARGN PREPEND "\"${CMAKE_COMMAND}\" -E echo \"")
    list(TRANSFORM ARGN APPEND "\" $<ANGLE-R>$<ANGLE-R> mini/config.h")
    string(JOIN " && " RES ${ARGN})
    set(${OUT} ${RES} PARENT_SCOPE)
endfunction()

set(PERL_INT64 __int64)

perl_generator_append_mini_config_h(PERL_CONFIG_BASE
    ""
    "#ifndef _config_h_footer_"
    "#define _config_h_footer_"
    "#undef PTRSIZE"
    "#undef SSize_t"
    "#undef HAS_ATOLL"
    "#undef HAS_STRTOLL"
    "#undef HAS_STRTOULL"
    "#undef Size_t_size"
    "#undef IVTYPE"
    "#undef UVTYPE"
    "#undef IVSIZE"
    "#undef UVSIZE"
    "#undef NV_PRESERVES_UV"
    "#undef NV_PRESERVES_UV_BITS"
    "#undef IVdf"
    "#undef UVuf"
    "#undef UVof"
    "#undef UVxf"
    "#undef UVXf"
    "#undef USE_64_BIT_INT"
    "#undef USE_LONG_DOUBLE"
    "#undef USE_CPLUSPLUS")

perl_generator_append_mini_config_h(PERL_CONFIG_64
    "#define PTRSIZE 8"
    "#define SSize_t ${PERL_INT64}"
    "#define HAS_ATOLL"
    "#define HAS_STRTOLL"
    "#define HAS_STRTOULL"
    "#define Size_t_size 8")

perl_generator_append_mini_config_h(PERL_CONFIG_32
    "#define PTRSIZE 4"
    "#define SSize_t int"
    "#undef HAS_ATOLL"
    "#undef HAS_STRTOLL"
    "#undef HAS_STRTOULL"
    "#define Size_t_size 4")

perl_generator_append_mini_config_h(PERL_CONFIG_USE_64_BIT_INT
    "#define IVTYPE ${PERL_INT64}"
    "#define UVTYPE unsigned ${PERL_INT64}"
    "#define IVSIZE 8"
    "#define UVSIZE 8"
    "#undef NV_PRESERVES_UV"
    "#define NV_PRESERVES_UV_BITS 53"
    "#define IVdf \\\"I64d\\\""
    "#define UVuf \\\"I64u\\\""
    "#define UVof \\\"I64o\\\""
    "#define UVxf \\\"I64x\\\""
    "#define UVXf \\\"I64X\\\""
    "#define USE_64_BIT_INT")

perl_generator_append_mini_config_h(PERL_CONFIG_USE_NORMAL_INT
    "#define IVTYPE long"
    "#define UVTYPE unsigned long"
    "#define IVSIZE 4"
    "#define UVSIZE 4"
    "#define NV_PRESERVES_UV"
    "#define NV_PRESERVES_UV_BITS 32"
    "#define IVdf \\\"ld\\\""
    "#define UVuf \\\"lu\\\""
    "#define UVof \\\"lo\\\""
    "#define UVxf \\\"lx\\\""
    "#define UVXf \\\"lX\\\""
    "#undef USE_64_BIT_INT")

perl_generator_append_mini_config_h(PERL_CONFIG_USE_CPLUSPLUS "#define USE_CPLUSPLUS" "#endif")
perl_generator_append_mini_config_h(PERL_CONFIG_NOT_USE_CPLUSPLUS "#undef USE_CPLUSPLUS" "#endif")

perl_generator_append_mini_config_h(PERL_CONFIG_NOT_MSVC_120
    "#undef FILE_ptr"
    "#undef FILE_cnt"
    "#undef FILE_base"
    "#undef FILE_bufsiz"
    "#define FILE_ptr(fp) PERLIO_FILE_ptr(fp)"
    "#define FILE_cnt(fp) PERLIO_FILE_cnt(fp)"
    "#define FILE_base(fp) PERLIO_FILE_base(fp)"
    "#define FILE_bufsiz(fp) (PERLIO_FILE_cnt(fp) + PERLIO_FILE_ptr(fp) - PERLIO_FILE_base(fp))"
    "#define I_STDBOOL")

add_custom_command(OUTPUT mini/config.h
    COMMAND
    ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/config_H.vc mini/config.h &&
    ${PERL_CONFIG_BASE} &&
    "$<$<NOT:$<EQUAL:${MSVC_TOOLSET_VERSION},120>>:${PERL_CONFIG_NOT_MSVC_120}>" &&
    "$<IF:$<EQUAL:${CMAKE_SIZEOF_VOID_P},8>,${PERL_CONFIG_64},${PERL_CONFIG_32}>" &&
    "$<IF:$<BOOL:${PERL_USE_64_BIT_INT}>,${PERL_CONFIG_USE_64_BIT_INT},${PERL_CONFIG_USE_NORMAL_INT}>" &&
    "$<IF:$<BOOL:${PERL_USE_CPLUSPLUS}>,${PERL_CONFIG_USE_CPLUSPLUS},${PERL_CONFIG_NOT_USE_CPLUSPLUS}>"
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/config_H.vc)

add_executable(genuudmap ../generate_uudmap.c)
target_include_directories(genuudmap PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

add_custom_command(OUTPUT uudmap.h bitcount.h mg_data.h
    COMMAND $<TARGET_FILE:genuudmap> uudmap.h bitcount.h mg_data.h)

add_executable(miniperl
    ../av.c
    ../caretx.c
    ../builtin.c
    ../deb.c
    ../doio.c
    ../doop.c
    ../dquote.c
    ../dump.c
    ../globals.c
    ../gv.c
    ../mro_core.c
    ../hv.c
    ../locale.c
    ../keywords.c
    ../mathoms.c
    ../mg.c
    ../numeric.c
    ../op.c
    ../pad.c
    ../peep.c
    ../perl.c
    ../perly.c
    ../pp.c
    ../pp_ctl.c
    ../pp_hot.c
    ../pp_pack.c
    ../pp_sort.c
    ../pp_sys.c
    ../reentr.c
    ../regcomp.c
    ../regexec.c
    ../run.c
    ../scope.c
    ../sv.c
    ../taint.c
    ../time64.c
    ../toke.c
    ../universal.c
    ../utf8.c
    ../util.c
    ../miniperlmain.c
    ../perlio.c
    win32.c
    win32sck.c
    win32thread.c
    fcrypt.c
    mini/config.h
    uudmap.h
    bitcount.h
    mg_data.h)

target_include_directories(miniperl PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/mini)
target_compile_definitions(miniperl PRIVATE PERL_EXTERNAL_GLOB PERL_IS_MINIPERL)
target_link_libraries(miniperl PRIVATE perldefaults Ws2_32.lib Comctl32.lib)

set(PERL_INVOKE_MINIPERL
    $<TARGET_FILE:miniperl>
    -I${CMAKE_CURRENT_SOURCE_DIR}/../lib
    -I${CMAKE_CURRENT_SOURCE_DIR})

add_custom_command(OUTPUT git_version.h COMMAND
    ${PERL_INVOKE_MINIPERL} ${CMAKE_CURRENT_SOURCE_DIR}/../make_patchnum.pl
    DEPENDS ../make_patchnum.pl)

add_custom_command(OUTPUT config.sh COMMAND
    ${PERL_INVOKE_MINIPERL}
    config_sh.PL
    INST_TOP=c:\\perl
    INST_VER=
    INST_ARCH=
    archname=MSWin32-${CMAKE_SYSTEM_PROCESSOR}-multi-thread
    cc=cl
    ld=link
    ccflags=
    usecplusplus=
    cf_email=
    d_mymalloc=undef
    libs=
    incpath=
    libperl=$<TARGET_PROPERTY:perllib,OUTPUT_NAME>
    libpth=
    libc=
    make=
    static_ext=
    usethreads=define
    useithreads=define
    usemultiplicity=define
    use64bitint=define
    uselongdouble=undef
    usequadmath=undef
    usesitecustomize=undef
    default_inc_excludes_dot=define
    LINK_FLAGS=
    optimize=
    WIN64=define
    config.vc
    > ../config.sh
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS config_sh.PL config.vc FindExt.pm)

add_custom_command(OUTPUT config.pm COMMAND
    ${PERL_INVOKE_MINIPERL} ../configpm --chdir=..
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS config.sh ../configpm config_h.PL git_version.h)

add_custom_command(OUTPUT config.h COMMAND
    ${PERL_INVOKE_MINIPERL}
    -I${CMAKE_CURRENT_SOURCE_DIR}/../dist/Exporter/lib
    -I${CMAKE_CURRENT_SOURCE_DIR}/../dist/Pathtools/lib
    -I${CMAKE_CURRENT_SOURCE_DIR}/../dist/Pathtools
    -I${CMAKE_CURRENT_SOURCE_DIR}/../dist/constant/lib
    config_h.PL
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS config.pm ../configpm config_h.PL)

add_custom_command(OUTPUT perllibst.h COMMAND
    ${PERL_INVOKE_MINIPERL} ${CMAKE_CURRENT_SOURCE_DIR}/create_perllibst_h.pl
    DEPENDS config.pm create_perllibst_h.pl)

add_library(perllib SHARED
    ../av.c
    ../caretx.c
    ../builtin.c
    ../deb.c
    ../doio.c
    ../doop.c
    ../dquote.c
    ../dump.c
    ../globals.c
    ../gv.c
    ../mro_core.c
    ../hv.c
    ../locale.c
    ../keywords.c
    ../mathoms.c
    ../mg.c
    ../numeric.c
    ../op.c
    ../pad.c
    ../peep.c
    ../perl.c
    ../perly.c
    ../pp.c
    ../pp_ctl.c
    ../pp_hot.c
    ../pp_pack.c
    ../pp_sort.c
    ../pp_sys.c
    ../reentr.c
    ../regcomp.c
    ../regexec.c
    ../run.c
    ../scope.c
    ../sv.c
    ../taint.c
    ../time64.c
    ../toke.c
    ../universal.c
    ../utf8.c
    ../util.c
    ../perlio.c
    win32.c
    win32sck.c
    win32thread.c
    fcrypt.c
    perllib.c
    git_version.h
    perllibst.h
    config.h)

target_include_directories(perllib PUBLIC ../lib/CORE)
target_link_libraries(perllib
    PUBLIC perldefaults
    PRIVATE Ws2_32.lib Comctl32.lib)
target_compile_definitions(perllib PUBLIC
    MULTIPLICITY
    PERL_IMPLICIT_SYS
    PERL_TEXTMODE_SCRIPTS)
set_target_properties(perllib PROPERTIES OUTPUT_NAME perl537)
set_source_files_properties(perllib.c PROPERTIES LANGUAGE CXX)

add_executable(perl runperl.c perlexe.rc perlexe.manifest perlexe.ico)
target_link_libraries(perl PRIVATE perllib)
