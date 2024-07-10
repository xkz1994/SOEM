cmake_minimum_required(VERSION 3.9)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules")
project(SOEM
        DESCRIPTION "Simple Open EtherCAT Master"
        VERSION 1.4.0
        LANGUAGES C)

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    # Default to installing in SOEM source directory
    set(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_LIST_DIR}/install)
endif ()

set(SOEM_INCLUDE_INSTALL_DIR include/soem)
set(SOEM_LIB_INSTALL_DIR lib)

if (CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(BUILD_TESTS TRUE)
else ()
    message(STATUS "SOEM: not building tests when built as dependency")
    set(BUILD_TESTS FALSE)
endif ()

if (WIN32)
    set(OS "win32")
    include_directories(oshw/win32/wpcap/Include) # include_directories 是 CMake 中用来指定头文件搜索路径的命令。它告诉编译器在哪里可以找到包含的头文件。这对于组织和管理项目中头文件的位置非常有用。
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
        link_directories(${CMAKE_CURRENT_LIST_DIR}/oshw/win32/wpcap/Lib/x64) # link_directories 是 CMake 中用于指定库文件搜索路径的命令。它告诉编译器在哪里可以找到需要链接的库文件。这在组织和管理项目中外部库的位置非常有用
    elseif (CMAKE_SIZEOF_VOID_P EQUAL 4)
        link_directories(${CMAKE_CURRENT_LIST_DIR}/oshw/win32/wpcap/Lib)
    endif ()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /D _CRT_SECURE_NO_WARNINGS")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}  /WX")
    set(OS_LIBS wpcap.lib Packet.lib Ws2_32.lib Winmm.lib)
elseif (UNIX AND NOT APPLE)
    set(OS "linux")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
    set(OS_LIBS pthread rt)
elseif (APPLE)
    # This must come *before* linux or MacOSX will identify as Unix.
    set(OS "macosx")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
    set(OS_LIBS pthread pcap)
elseif (${CMAKE_SYSTEM_NAME} MATCHES "rt-kernel")
    set(OS "rtk")
    message(STATUS "ARCH is ${ARCH}")
    message(STATUS "BSP is ${BSP}")
    include_directories(oshw/${OS}/${ARCH})
    file(GLOB OSHW_EXTRA_SOURCES oshw/${OS}/${ARCH}/*.c)
    set(OSHW_SOURCES "${OS_HW_SOURCES} ${OSHW_ARCHSOURCES}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-but-set-variable")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-function")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format")
    set(OS_LIBS "-Wl,--start-group -l${BSP} -l${ARCH} -lkern -ldev -lsio -lblock -lfs -lusb -llwip -leth -li2c -lrtc -lcan -lnand -lspi -lnor -lpwm -ladc -ltrace -lc -lm -Wl,--end-group")
elseif (${CMAKE_SYSTEM_NAME} MATCHES "rtems")
    message(STATUS "Building for RTEMS")
    set(OS "rtems")
    set(SOEM_LIB_INSTALL_DIR ${LIB_DIR})
    set(BUILD_TESTS FALSE)
endif ()

message(STATUS "OS is ${OS}")

file(GLOB SOEM_SOURCES soem/*.c) # CMake 中用于查找匹配文件模式的命令。它会在指定目录中查找符合模式的文件，并将它们存储在变量中
file(GLOB OSAL_SOURCES osal/${OS}/*.c)
file(GLOB OSHW_SOURCES oshw/${OS}/*.c)

file(GLOB SOEM_HEADERS soem/*.h)
file(GLOB OSAL_HEADERS osal/osal.h osal/${OS}/*.h)
file(GLOB OSHW_HEADERS oshw/${OS}/*.h)

add_library(soem STATIC
        ${SOEM_SOURCES}
        ${OSAL_SOURCES}
        ${OSHW_SOURCES}
        ${OSHW_EXTRA_SOURCES}) # 创建静态库
target_link_libraries(soem ${OS_LIBS}) # CMake 中用于链接库到目标（如可执行文件或另一个库）的命令 # 链接库到可执行文件

target_include_directories(soem PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/soem>
        $<INSTALL_INTERFACE:include/soem>) #  CMake 中用于设置目标的包含目录的命令

target_include_directories(soem PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/osal>
        $<INSTALL_INTERFACE:include/soem>)

target_include_directories(soem PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/osal/${OS}>
        $<INSTALL_INTERFACE:include/soem>)

target_include_directories(soem
        PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/oshw/${OS}>
        $<INSTALL_INTERFACE:include/soem>
)

message(STATUS "LIB_DIR: ${SOEM_LIB_INSTALL_DIR}")

install(TARGETS soem EXPORT soemConfig DESTINATION ${SOEM_LIB_INSTALL_DIR}) # 用于安装构建的目标文件（可执行文件、库文件、头文件等）到指定的目标位置

install(EXPORT soemConfig DESTINATION share/soem/cmake)

install(FILES
        ${SOEM_HEADERS}
        ${OSAL_HEADERS}
        ${OSHW_HEADERS}
        DESTINATION ${SOEM_INCLUDE_INSTALL_DIR})

if (BUILD_TESTS)
    add_subdirectory(test/simple_ng) # 命令用于向当前项目添加一个子目录，并在该子目录中执行另一个 CMakeLists.txt 文件
    add_subdirectory(test/linux/slaveinfo)
    add_subdirectory(test/linux/eepromtool)
    add_subdirectory(test/linux/simple_test)
endif ()
