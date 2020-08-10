cmake_minimum_required(VERSION 3.17)
project(jdk11u_dev_src)

set(CMAKE_CXX_STANDARD 14)

# C/C++ 编译设置
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-stringop-overflow -Wno-stringop-truncation -Wno-return-type -Wno-format-truncation -Wno-cast-function-type -Wno-maybe-uninitialized -fcommon")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-stringop-overflow -Wno-write-strings -Wno-type-limits -Wno-class-memaccess")

set(SUPPORT_PALTFORMS aix linux posix solaris unix)

include_directories("/usr/include")

# java.base 目录下的
set(SRC_DIR "${PROJECT_SOURCE_DIR}/src")

# hotspot 相关处理
set(HOTSPOT "${SRC_DIR}/hotspot")
set(HOTSPOT_SHARE_DIR "${HOTSPOT}/share")

include_directories(
        "${HOTSPOT}/share"
        "${HOTSPOT}/share/precompiled"
        "${HOTSPOT}/share/include"
)
foreach(PLATFORM ${SUPPORT_PALTFORMS})
   # message("${HOTSPOT}/os/${PLATFORM}")
   include_directories("${HOTSPOT}/os/${PLATFORM}/include")
#    include_directories("${}")
endforeach()


# java.xxxx 目录处理
file(GLOB JAVA_LIB_DIRS LIST_DIRECTORIES true "${SRC_DIR}/java.*" "${SRC_DIR}/jdk.*")
foreach(JAVA_LIB_DIR ${JAVA_LIB_DIRS})
    # 添加include 目录
    include_directories("${JAVA_LIB_DIR}/share/native/include")

    file(GLOB JAVA_BASE_LIB_DIR LIST_DIRECTORIES true "${JAVA_LIB_DIR}/share/native/lib*")
    foreach(LIB_DIR ${JAVA_BASE_LIB_DIR})
        include_directories("${LIB_DIR}")
        # 获取当前目录的名字
        STRING(REGEX REPLACE ".*/(.*)" "\\1" LIB_NAME ${LIB_DIR} )
        file(GLOB_RECURSE LIBJAVA_FILES "${LIB_DIR}/*.[ch]*")
        # 扫描 相关平台下的代码，并添加到lib编译中
        foreach(PLATFORM ${SUPPORT_PALTFORMS})
            include_directories("${JAVA_LIB_DIR}/${PLATFORM}/native/include")

            set(EXT_INCLUDE_DIR "${JAVA_LIB_DIR}/${PLATFORM}/native/${LIB_NAME}")
            if (EXISTS "${EXT_INCLUDE_DIR}")
                include_directories("${EXT_INCLUDE_DIR}")
                # message("include directory: ${EXT_INCLUDE_DIR}")
            endif ()

            set(EXT_LIB_DIR "${JAVA_LIB_DIR}/${PLATFORM}/native/${LIB_NAME}")
            if (EXISTS "${EXT_LIB_DIR}")
                file(GLOB_RECURSE LIBJAVA_EXT_FILES "${EXT_LIB_DIR}/*.[ch]*")
                list(APPEND LIBJAVA_FILES ${LIBJAVA_EXT_FILES})
            endif ()
        endforeach()

        add_library(${LIB_NAME} SHARED "${LIBJAVA_FILES}")
    endforeach()
endforeach()

file(GLOB JAVA_BASE_LAUNCHER "${SRC_DIR}/java.base/share/native/launcher/*.c")
add_executable(java "${JAVA_BASE_LAUNCHER}")

file(GLOB_RECURSE HOTSPOT_FILES "${HOTSPOT_SHARE_DIR}/*.[ch]pp")
add_executable(hotspot "${HOTSPOT_FILES}")

target_link_libraries(java libjli libjava)

######################################################################
# 编译生成的
set(BUILD_OUT_DIR "${PROJECT_SOURCE_DIR}/build/linux-x86_64-normal-server-fastdebug")
if (NOT EXISTS ${BUILD_OUT_DIR})
    message("need build first with configuration and make all")
    return(1)
endif()

set(SUPPORT_INCLUDES headers modules_include)
foreach(SUPPORT_INCLUDE ${SUPPORT_INCLUDES})
    file(GLOB LIB_DIRS LIST_DIRECTORIES true "${BUILD_OUT_DIR}/support/${SUPPORT_INCLUDE}/java.*")
    foreach(LIB_DIR ${LIB_DIRS})
        include_directories("${LIB_DIR}")
         message("${LIB_DIR}")
    endforeach()
endforeach()
# /support/modules_include/java.base

add_compile_definitions(VERSION_STRING "jdk_11.0")

