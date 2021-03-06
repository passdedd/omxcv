#omxcv cmake configuration file

cmake_minimum_required (VERSION 2.8)
project (omxcv)

include (FindPkgConfig)

#Versioning
execute_process (COMMAND git rev-parse --short HEAD
				 OUTPUT_STRIP_TRAILING_WHITESPACE
				 OUTPUT_VARIABLE OMXCV_VERSION
)
execute_process (COMMAND git  show -s --format=%ci HEAD
				 OUTPUT_STRIP_TRAILING_WHITESPACE
				 OUTPUT_VARIABLE OMXCV_DATE
)

#Required libraries
find_package (OpenCV 3.4.3 REQUIRED)
include_directories (${OpenCV_INCLUDE_DIR})
find_package (Threads REQUIRED) # For pthreads

pkg_check_modules (LIBAVFORMAT libavformat REQUIRED)
pkg_check_modules (LIBAVCODEC libavcodec REQUIRED)
pkg_check_modules (LIBAVUTIL libavutil REQUIRED)

include_directories (${LIBAVFORMAT_INCLUDE_DIRS})
link_directories (${LIBAVFORMAT_LIBRARY_DIRS})
include_directories (${LIBAVCODEC_INCLUDE_DIRS})
link_directories (${LIBAVCODEC_LIBRARY_DIRS})
include_directories (${LIBAVUTIL_INCLUDE_DIRS})
link_directories (${LIBAVUTIL_LIBRARY_DIRS})

#Optional libraries
CHECK_LIBRARY_EXISTS(tcmalloc malloc "" HAVE_LIBTCMALLOC)
CHECK_LIBRARY_EXISTS(profiler malloc "" HAVE_LIBPROFILER)

#Raspberry Pi specific includes.
#Requires the installation of the userland package.
include_directories (/sdc/raspberrypi/firmware-master/opt/vc/include /sdc/raspberrypi/firmware-master/opt/vc/src/hello_pi/libs/ilclient)
include_directories (/sdc/raspberrypi/firmware-master/opt/vc/include/interface/vcos/pthreads)
include_directories (/sdc/raspberrypi/firmware-master/opt/vc/include/IL /sdc/raspberrypi/firmware-master/opt/vc/include/interface/vmcs_host/linux)
link_directories (/sdc/raspberrypi/firmware-master/opt/vc/src/hello_pi/libs/ilclient)

#Compile options
if (CMAKE_COMPILER_IS_GNUCXX)
	set (CMAKE_CXX_FLAGS "-Wall -Wno-variadic-macros -std=c++11 -pedantic -g")
endif (CMAKE_COMPILER_IS_GNUCXX)

#Source files
set (HEADERS
    omxcv-config.h
    omxcv.h
    image_gpu.h
)

set (SOURCE
    image_gpu.cpp
    omxcv.cpp
    omxcv_jpeg.cpp
)

#Enable NEON extensions
if (ENABLE_NEON)
    set (SOURCE ${SOURCE} bgr2rgb.s)
    set_property (SOURCE bgr2rgb.s PROPERTY LANGUAGE C)
    set_source_files_properties (bgr2rgb.s PROPERTIES COMPILE_FLAGS "-mfpu=neon -mcpu=cortex-a7 -g")
endif (ENABLE_NEON)

#Generate config header
configure_file (omxcv-config.h.in omxcv-config.h)

#Library creation
add_library (omxcv ${HEADERS} ${SOURCE})
target_link_libraries (omxcv LINK_PRIVATE EGL GLESv2)
target_link_libraries (omxcv LINK_PRIVATE ${LIBAVFORMAT_LIBRARIES} ${LIBAVCODEC_LIBRARIES} ${LIBAVUTIL_LIBRARIES} ${OpenCV_LIBS})
target_link_libraries (omxcv LINK_PRIVATE  ilclient bcm_host openmaxil bcm_host vcos vchiq_arm)
target_link_libraries (omxcv LINK_PRIVATE ${CMAKE_THREAD_LIBS_INIT})

if (HAVE_LIBTCMALLOC)
	target_link_libraries (omxcv LINK_PUBLIC tcmalloc)
endif (HAVE_LIBTCMALLOC)

if (HAVE_LIBPROFILER)
	target_link_libraries (omxcv LINK_PUBLIC profiler)
endif (HAVE_LIBPROFILER)

#Test executable
add_executable (omxcv-test omxcv-test.cpp)
add_executable (glsl-test glsl-test.cpp)
target_link_libraries (omxcv-test LINK_PUBLIC omxcv)
target_link_libraries (glsl-test LINK_PUBLIC omxcv)
