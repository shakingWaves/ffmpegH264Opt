cmake_minimum_required (VERSION 2.8)

# Configure build type (Release when no one is selected)
if (NOT DEFINED CMAKE_BUILD_TYPE)
  set (CMAKE_BUILD_TYPE Release CACHE STRING "Build type")
endif ()

project (ffmpegH264)
SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/MyCMakeScripts)

message("DEBUG: CMAKE_SYSTEM_PROCESSOR = ${CMAKE_SYSTEM_PROCESSOR}")
message("DEBUG: CMAKE_SYSTEM_NAME      = ${CMAKE_SYSTEM_NAME}")

macro (my_check_function_exists arg result)
  check_function_exists(${arg} ${result})
  if(${result} STREQUAL "")
    set(${result} 0)
  endif()
endmacro (my_check_function_exists)

macro (my_check_include_files arg result)
  check_include_files(${arg} ${result})
  if(${result} STREQUAL "")
    set(${result} 0)
  endif()
endmacro (my_check_include_files)

include(CheckTypeSize)
check_type_size("void*" SIZEOF_VOID_P BUILTIN_TYPES_ONLY)
message("DEBUG: SIZEOF_VOID_P      = ${SIZEOF_VOID_P}")
include(CheckFunctionExists)
include(CheckIncludeFiles)
include(OptimizeForArchitecture)

OptimizeForArchitecture()

my_check_function_exists(GetProcessAffinityMask GETPROCESSAFFINITYMASK_FOUND)
my_check_function_exists(gettimeofday           GETTIMEOFDAY_FOUND)
my_check_function_exists(sched_getaffinity      SCHED_GETAFFINITY_FOUND)
my_check_function_exists(strerror_r             STRERROR_R_FOUND)
my_check_function_exists(sysconf                SYSCONF_FOUND)
my_check_function_exists(usleep                 USLEEP_FOUND)
my_check_function_exists(localtime_r            LOCALTIME_R_FOUND)
my_check_function_exists(gmtime_r               GMTIME_R_FOUND)

my_check_include_files(fcntl.h                  FCNTL_H_FOUND)
my_check_include_files(pthread.h                PTHREADS_FOUND)
my_check_include_files(unistd.h                 UNISTD_H_FOUND)
my_check_include_files(windows.h                WINDOWS_H_FOUND)

#find asm compiler
option (USE_YASM "Use YASM. If YASM is not enabled the assembly implementation will be disabled." ON)
if (USE_YASM)
  find_package (Yasm)
endif ()

if("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7l")
	enable_language(ASM)
	add_definitions(
		-DEXTERN_ASM=
	)
endif()

if(WIN32)
    add_definitions(
        -Dsnprintf=avpriv_snprintf
        -Dvsnprintf=avpriv_vsnprintf
        -Dinline=__inline
        -Drestrict=__restrict
    )
endif()

#define asm sources
if(NOT ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7l") )
if(YASM_FOUND)
set(YASM_NAMES
libavcodec/x86/fpel.asm
libavcodec/x86/h264_chromamc.asm
libavcodec/x86/h264_chromamc_10bit.asm
libavcodec/x86/h264_deblock.asm
libavcodec/x86/h264_deblock_10bit.asm
libavcodec/x86/h264_idct.asm
libavcodec/x86/h264_idct_10bit.asm
libavcodec/x86/h264_intrapred.asm
libavcodec/x86/h264_intrapred_10bit.asm
libavcodec/x86/h264_qpel_10bit.asm
libavcodec/x86/h264_qpel_8bit.asm
libavcodec/x86/h264_weight.asm
libavcodec/x86/h264_weight_10bit.asm
libavcodec/x86/idctdsp.asm
libavcodec/x86/me_cmp.asm
libavcodec/x86/pixblockdsp.asm
libavcodec/x86/qpel.asm
libavcodec/x86/simple_idct10.asm
libavcodec/x86/videodsp.asm
libswresample/x86/audio_convert.asm
libswresample/x86/rematrix.asm
libswresample/x86/resample.asm
libswscale/x86/input.asm
libswscale/x86/output.asm
libswscale/x86/scale.asm
libavutil/x86/cpuid.asm
libavutil/x86/fixed_dsp.asm
libavutil/x86/float_dsp.asm
libavutil/x86/lls.asm
)
endif(YASM_FOUND)
endif(NOT ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7l") )

if(NOT ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7l") )
set(COMMON_YASM_ARGS
    -I./
    -I "${CMAKE_CURRENT_SOURCE_DIR}"
    -P "${CMAKE_CURRENT_SOURCE_DIR}/config.asm"
    -I "${CMAKE_CURRENT_SOURCE_DIR}/libavfilter/x86/"
    -I "${CMAKE_CURRENT_SOURCE_DIR}/libavutil/x86/"
    -DPIC
)
endif(NOT ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7l") )

if(YASM_FOUND)
if(APPLE)
    set(YASM_ARGS -f macho64 -m amd64 -DPREFIX ${COMMON_YASM_ARGS})
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64")
    set(YASM_ARGS -f elf     -m amd64 ${COMMON_YASM_ARGS})
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "i686")
    set(YASM_ARGS -f elf     -DX86_32 ${COMMON_YASM_ARGS})
else()
    set(YASM_ARGS -f win32   -m amd64 ${COMMON_YASM_ARGS})
endif()

#compile all asm files
foreach(_asm_file ${YASM_NAMES})
    set(YASM_SRC "${CMAKE_CURRENT_SOURCE_DIR}/${_asm_file}")
    get_filename_component(BASENAME ${YASM_SRC} NAME_WE)
    set(YASM_OBJ "${CMAKE_CURRENT_BINARY_DIR}/${BASENAME}.o")
    add_custom_command(
        OUTPUT ${YASM_OBJ}
        COMMAND "${YASM_EXECUTABLE}"
        ARGS ${YASM_ARGS} -o ${YASM_OBJ} ${YASM_SRC}
        DEPENDS ${YASM_SRC}
    )
    set(YASM_OBJECTS ${YASM_OBJECTS} ${YASM_OBJ})
endforeach()
endif(YASM_FOUND)

if("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64")
    if(MINGW)
        AddCompilerFlag("-arch x86_64 -m64" C_FLAGS Vc_ARCHITECTURE_FLAGS)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wl,-Bsymbolic")
    elseif(NOT APPLE)
        string(REGEX MATCH "clang*" CLANG_COMPILER "${CMAKE_C_COMPILER}")
        if ("${CLANG_COMPILER}" STREQUAL "clang")
            set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-Bsymbolic")
        else()
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wl,-Bsymbolic")
        endif()
    endif()
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "i686")
    #add_definitions("-DX86_32")
endif()

add_definitions("-DPIC")
add_definitions("-DUSE_SDL")
AddCompilerFlag("-fpic" C_FLAGS Vc_ARCHITECTURE_FLAGS)
AddCompilerFlag("-fno-tree-vectorize" C_FLAGS Vc_ARCHITECTURE_FLAGS)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${Vc_ARCHITECTURE_FLAGS}")

find_package(SDL)
find_package(SDL2)

set(libfilenames
gpac/modules/openhevc_dec/openHevcWrapper.c

libavdevice/alldevices.c
libavdevice/avdevice.c
libavdevice/utils.c
libavfilter/allfilters.c
libavfilter/audio.c
libavfilter/avfilter.c
libavfilter/avfiltergraph.c
libavfilter/buffersink.c
libavfilter/buffersrc.c
libavfilter/drawutils.c
libavfilter/fifo.c
libavfilter/formats.c
libavfilter/framepool.c
libavfilter/graphdump.c
libavfilter/graphparser.c
libavfilter/opencl_allkernels.c
libavfilter/pthread.c
libavfilter/transform.c
libavfilter/video.c
libavformat/allformats.c
libavformat/avio.c
libavformat/aviobuf.c
libavformat/cutils.c
libavformat/dump.c
libavformat/file.c
libavformat/format.c
libavformat/h264dec.c
libavformat/id3v1.c
libavformat/id3v2.c
libavformat/metadata.c
libavformat/mux.c
libavformat/options.c
libavformat/os_support.c
libavformat/protocols.c
libavformat/qtpalette.c
libavformat/rawdec.c
libavformat/riff.c
libavformat/sdp.c
libavformat/url.c
libavformat/utils.c
libavcodec/allcodecs.c
libavcodec/audioconvert.c
libavcodec/avdct.c
libavcodec/avpacket.c
libavcodec/avpicture.c
libavcodec/bitstream.c
libavcodec/bitstream_filter.c
libavcodec/bitstream_filters.c
libavcodec/bsf.c
libavcodec/cabac.c
libavcodec/codec_desc.c
libavcodec/d3d11va.c
libavcodec/dirac.c
libavcodec/dv_profile.c
libavcodec/error_resilience.c
libavcodec/faandct.c
libavcodec/faanidct.c
libavcodec/fdctdsp.c
libavcodec/golomb.c
libavcodec/h2645_parse.c
libavcodec/h264_cabac.c
libavcodec/h264_cavlc.c
libavcodec/h264_direct.c
libavcodec/h264_loopfilter.c
libavcodec/h264_mb.c
libavcodec/h264_parse.c
libavcodec/h264_parser.c
libavcodec/h264_picture.c
libavcodec/h264_ps.c
libavcodec/h264_refs.c
libavcodec/h264_sei.c
libavcodec/h264_slice.c
libavcodec/h264chroma.c
libavcodec/h264data.c
libavcodec/h264dec.c
libavcodec/h264dsp.c
libavcodec/h264idct.c
libavcodec/h264pred.c
libavcodec/h264qpel.c
libavcodec/idctdsp.c
libavcodec/imgconvert.c
libavcodec/jfdctfst.c
libavcodec/jfdctint.c
libavcodec/jni.c
libavcodec/jrevdct.c
libavcodec/mathtables.c
libavcodec/me_cmp.c
libavcodec/mediacodec.c
libavcodec/options.c
libavcodec/parser.c
libavcodec/pixblockdsp.c
libavcodec/profiles.c
libavcodec/pthread.c
libavcodec/pthread_frame.c
libavcodec/pthread_slice.c
libavcodec/qsv_api.c
libavcodec/raw.c
libavcodec/resample.c
libavcodec/resample2.c
libavcodec/simple_idct.c
libavcodec/startcode.c
libavcodec/utils.c
libavcodec/videodsp.c
libavcodec/vorbis_parser.c
libavcodec/x86/constants.c
libavcodec/x86/fdct.c
libavcodec/x86/fdctdsp_init.c
libavcodec/x86/h264_intrapred_init.c
libavcodec/x86/h264_qpel.c
libavcodec/x86/h264chroma_init.c
libavcodec/x86/h264dsp_init.c
libavcodec/x86/idctdsp_init.c
libavcodec/x86/me_cmp_init.c
libavcodec/x86/pixblockdsp_init.c
libavcodec/x86/simple_idct.c
libavcodec/x86/videodsp_init.c
libavcodec/xiph.c
libswresample/audioconvert.c
libswresample/dither.c
libswresample/options.c
libswresample/rematrix.c
libswresample/resample.c
libswresample/resample_dsp.c
libswresample/swresample.c
libswresample/swresample_frame.c
libswresample/x86/audio_convert_init.c
libswresample/x86/rematrix_init.c
libswresample/x86/resample_init.c
libswscale/alphablend.c
libswscale/gamma.c
libswscale/hscale.c
libswscale/hscale_fast_bilinear.c
libswscale/input.c
libswscale/options.c
libswscale/output.c
libswscale/rgb2rgb.c
libswscale/slice.c
libswscale/swscale.c
libswscale/swscale_unscaled.c
libswscale/utils.c
libswscale/vscale.c
libswscale/x86/hscale_fast_bilinear_simd.c
libswscale/x86/rgb2rgb.c
libswscale/x86/swscale.c
libswscale/x86/yuv2rgb.c
libswscale/yuv2rgb.c
libavutil/adler32.c
libavutil/aes.c
libavutil/aes_ctr.c
libavutil/audio_fifo.c
libavutil/avstring.c
libavutil/base64.c
libavutil/blowfish.c
libavutil/bprint.c
libavutil/buffer.c
libavutil/camellia.c
libavutil/cast5.c
libavutil/channel_layout.c
libavutil/color_utils.c
libavutil/cpu.c
libavutil/crc.c
libavutil/des.c
libavutil/dict.c
libavutil/display.c
libavutil/downmix_info.c
libavutil/error.c
libavutil/eval.c
libavutil/fifo.c
libavutil/file.c
libavutil/file_open.c
libavutil/fixed_dsp.c
libavutil/float_dsp.c
libavutil/frame.c
libavutil/hash.c
libavutil/hmac.c
libavutil/hwcontext.c
libavutil/imgutils.c
libavutil/integer.c
libavutil/intmath.c
libavutil/lfg.c
libavutil/lls.c
libavutil/log.c
libavutil/log2_tab.c
libavutil/mastering_display_metadata.c
libavutil/mathematics.c
libavutil/md5.c
libavutil/mem.c
libavutil/murmur3.c
libavutil/opt.c
libavutil/parseutils.c
libavutil/pixdesc.c
libavutil/pixelutils.c
libavutil/random_seed.c
libavutil/rational.c
libavutil/rc4.c
libavutil/reverse.c
libavutil/ripemd.c
libavutil/samplefmt.c
libavutil/sha.c
libavutil/sha512.c
libavutil/stereo3d.c
libavutil/tea.c
libavutil/threadmessage.c
libavutil/time.c
libavutil/timecode.c
libavutil/tree.c
libavutil/twofish.c
libavutil/utils.c
libavutil/x86/cpu.c
libavutil/x86/fixed_dsp_init.c
libavutil/x86/float_dsp_init.c
libavutil/x86/lls_init.c
libavutil/xga_font_data.c
libavutil/xtea.c
)

if(WIN32)
list(APPEND libfilenames
    compat/strtod.c
    compat/msvcrt/snprintf.c
)
endif()

if(WIN32 OR ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7l"))
    option (BUILD_SHARED_LIBS "enabled static library instead of shared" OFF)
else()
    option (BUILD_SHARED_LIBS "enabled static library instead of shared" OFF)
endif()
add_library (LibffmpegH264Wrapper ${libfilenames} ${YASM_OBJECTS})


include_directories(. gpac/modules/openhevc_dec/ platform/x86/)

option(ENABLE_EXECUTABLE "Generate the test application" ON)

if(ENABLE_EXECUTABLE)

    set(LINK_LIBRARIES_LIST LibffmpegH264Wrapper)
    set(HEVC_SOURCES_FILES
        main_hm/getopt.c
        main_hm/main.c
#ffmpeg.c
#cmdutils.c
#ffmpeg_opt.c
#ffmpeg_filter.c
    )
    if(MINGW)
        list(APPEND LINK_LIBRARIES_LIST -lwinmm)
    endif()

    if(SDL_FOUND)
        # Build executable
        add_executable(hevc ${HEVC_SOURCES_FILES} main_hm/sdl.c main_hm/SDL_framerate.c)
        # Link executable
        target_link_libraries(hevc ${LINK_LIBRARIES_LIST} ${SDL_LIBRARY})
        # Set include directory specific for this file. Avoid conflicts when including SDL.h
        # if both SDL and SDL2 are installed
        set_source_files_properties(main_hm/sdl.c main_hm/SDL_framerate.c PROPERTIES COMPILE_FLAGS -I"${SDL_INCLUDE_DIR}")
    endif()

    if(SDL2_FOUND)
        # Build executable
        add_executable(h264_sdl2 ${HEVC_SOURCES_FILES} main_hm/sdl2.c main_hm/SDL_framerate.c)
        # Link executable
        target_link_libraries(h264_sdl2 ${LINK_LIBRARIES_LIST} ${SDL2_LIBRARY})
        # Set include directory specific for this file. Avoid conflicts when including SDL.h
        # if both SDL and SDL2 are installed
        set_source_files_properties(main_hm/sdl2.c main_hm/SDL_framerate.c PROPERTIES COMPILE_FLAGS -I"${SDL2_INCLUDE_DIR}")
    endif()

endif()

install(FILES
    gpac/modules/openhevc_dec/openHevcWrapper.h
    libavcodec/hevcdsp.h
    libavcodec/hevcpred.h
    DESTINATION include
)

install(
    TARGETS LibffmpegH264Wrapper
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/uninstall_target.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/uninstall_target.cmake"
    IMMEDIATE @ONLY)

# uninstall target
add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/uninstall_target.cmake)
