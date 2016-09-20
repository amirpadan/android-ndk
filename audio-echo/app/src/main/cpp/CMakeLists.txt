cmake_minimum_required(VERSION 3.4.1)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Werror")
set(echo_SRCS debug_utils.cpp
              audio_common.cpp
              audio_player.cpp
              audio_recorder.cpp
              audio_main.cpp)
add_library(echo SHARED ${echo_SRCS})

# include libraries needed for hello-jni lib
target_link_libraries(echo android log OpenSLES atomic)

# Android Studio CMake does not pack stl shared libraries, so app needs to pack
# the right shared lib into APK. The following code find right stl type and copy
# the needed shared lib into app's app/src/main/jniLibs, android studio assembles
# it into the final APK
# Helper function to retrieve shared stl path and name in NDK
# stl_path: the path to the NDK's shared lib path; empty if not using shared stl
function(get_stl_info stl_path stl_name)
   # assume app not uses shared stl lib
   set(${stl_path} "" PARENT_SCOPE)
   if(NOT ${ANDROID_STL} MATCHES "_shared")
       return()
   endif()

   # using shared lib, config lib name and path
   if("${ANDROID_STL}" MATCHES "c\\\+\\\+_")
       # app uses c++_shared for stl type
       set(stlPath "llvm-libc++/libs/${ANDROID_ABI}")
       set(stlName "libc++_shared.so")
   elseif(${ANDROID_STL} MATCHES "gnustl_")
       set(stlPath "gnu-libstdc++/4.9/libs/${ANDROID_ABI}")
       set(stlName "libgnustl_shared.so")
   else()
       # this sample not supporting other stl types
       message(FATAL_ERROR "Not Suppored STL type: ${ANDROID_STL}")
       return()
   endif()

   set(${stl_path} ${ANDROID_NDK}/sources/cxx-stl/${stlPath} PARENT_SCOPE)
   set(${stl_name} ${stlName} PARENT_SCOPE)
endfunction()

# force copying needed shared stl lib into ${project}/app/src/main/jniLibs
# so it will be packed into APK
get_stl_info(ndk_stl_path  ndk_stl_name)
if(NOT ${ndk_stl_path} STREQUAL "")
    set(echo_jniLibs_dir "${CMAKE_CURRENT_SOURCE_DIR}/../jniLibs")
    add_custom_command(TARGET echo PRE_BUILD
                   COMMAND "${CMAKE_COMMAND}" -E
                   copy ${ndk_stl_path}/${ndk_stl_name}
                   "${echo_jniLibs_dir}/${ANDROID_ABI}/${ndk_stl_name}"
                   COMMENT "Copying Shared library to the packing directory")
endif()

