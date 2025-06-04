cmake_minimum_required(VERSION 3.22)

# ==============================================================================
# 1) CROSS‐COMPILER / TOOLCHAIN
# ==============================================================================
# Tell CMake this is NOT a host build; it’s a “Generic ARM” build.
set(CMAKE_SYSTEM_NAME    Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# Replace the path below with wherever CubeIDE’s "…/tools/bin" lives on your machine:
set(STM32CUBE_TOOLCHAIN_DIR
    "/Applications/STM32CubeIDE.app/Contents/Eclipse/plugins/com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.13.3.rel1.macos64_1.0.0.202411102158/tools/bin"
)

# All ARM executables share this prefix:
set(ARM_PREFIX "${STM32CUBE_TOOLCHAIN_DIR}/arm-none-eabi-")

# Force CMake to call exactly those cross‐tools:
set(CMAKE_C_COMPILER   "${ARM_PREFIX}gcc")
set(CMAKE_CXX_COMPILER "${ARM_PREFIX}g++")
set(CMAKE_ASM_COMPILER "${ARM_PREFIX}gcc")
set(CMAKE_LINKER       "${ARM_PREFIX}gcc")
set(CMAKE_OBJCOPY      "${ARM_PREFIX}objcopy")
set(CMAKE_SIZE         "${ARM_PREFIX}size")

# Make sure CMake produces a .elf, not a host binary:
set(CMAKE_EXECUTABLE_SUFFIX .elf)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)


# ==============================================================================
# 2) PROJECT & GLOBAL COMPILER FLAGS
# ==============================================================================
project(scion_firmware LANGUAGES C ASM)

# Keep exactly the same flags CubeIDE uses, including stack‐boundary & unaligned‐access:
set(CMAKE_C_FLAGS "-mcpu=cortex-m7 -std=gnu11 -g3 -DDEBUG -DUSE_HAL_DRIVER -DSTM32H735xx -O2 -ffunction-sections -fdata-sections -mno-unaligned-access -Wall -fstack-usage -mfpu=fpv5-d16 -mfloat-abi=hard -mthumb")
#set(CMAKE_C_FLAGS "-mcpu=cortex-m7 -std=gnu11 -g3 -DDEBUG -DUSE_HAL_DRIVER -DSTM32H735xx -O2 -ffunction-sections -fdata-sections -Wall -fstack-usage -mfpu=fpv5-d16 -mfloat-abi=hard -mthumb")

set(CMAKE_C_FLAGS_DEBUG   "")              # no extra debug tweaks
set(CMAKE_C_FLAGS_RELEASE "-Os -g0")       # match CubeIDE’s “release” style

# Assembler (startup, etc.) flags:
set(CMAKE_ASM_FLAGS
    "-mcpu=cortex-m7 -g3 -DDEBUG -x assembler-with-cpp -mfpu=fpv5-d16 -mfloat-abi=hard -mthumb"
)

# Linker flags: same as CubeIDE but **without** `--align-data=8`
set(CMAKE_EXE_LINKER_FLAGS
    "-mcpu=cortex-m7 -T${CMAKE_SOURCE_DIR}/STM32H735IGKX_FLASH.ld --specs=nosys.specs -Wl,-Map,${CMAKE_PROJECT_NAME}.map -Wl,--gc-sections -static --specs=nano.specs -mfpu=fpv5-d16 -mfloat-abi=hard -mthumb -Wl,--start-group -lc -lm -Wl,--end-group"
)


# ==============================================================================
# 3) STM32CubeMX‐GENERATED CODE (OBJECT LIBRARIES)
# ==============================================================================
# (This is exactly your “section 3” from the original, unchanged.)


# 3a) “Interface” library that only carries include‐paths and #defines:
set(MX_Defines_Syms
    USE_PWR_DIRECT_SMPS_SUPPLY
    USE_HAL_DRIVER
    STM32H735xx
    $<$<CONFIG:Debug>:DEBUG>
)
set(MX_Include_Dirs
    ${CMAKE_SOURCE_DIR}/Core/Inc
    ${CMAKE_SOURCE_DIR}/LWIP/App
    ${CMAKE_SOURCE_DIR}/LWIP/Target
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/system
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Inc
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Inc/Legacy
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/include
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/CMSIS_RTOS_V2
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/portable/GCC/ARM_CM4F
    ${CMAKE_SOURCE_DIR}/Drivers/BSP/Components/lan8742
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/netif/ppp
    ${CMAKE_SOURCE_DIR}/Drivers/CMSIS/Device/ST/STM32H7xx/Include
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/lwip
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/lwip/apps
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/lwip/priv
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/lwip/prot
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/netif
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/compat/posix
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/compat/posix/arpa
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/compat/posix/net
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/compat/posix/sys
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/include/compat/stdc
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/system/arch
    ${CMAKE_SOURCE_DIR}/Drivers/CMSIS/Include
)
add_library(stm32cubemx INTERFACE)
target_include_directories(stm32cubemx INTERFACE ${MX_Include_Dirs})
target_compile_definitions(stm32cubemx INTERFACE ${MX_Defines_Syms})


# 3b) STM32 HAL/LL drivers as an OBJECT‐library:
set(STM32_Drivers_Src
    ${CMAKE_SOURCE_DIR}/Core/Src/system_stm32h7xx.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_tim.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_tim_ex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_eth.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_eth_ex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_rcc.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_rcc_ex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_flash.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_flash_ex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_gpio.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_hsem.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_dma.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_dma_ex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_mdma.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_pwr.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_pwr_ex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_cortex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_i2c.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_i2c_ex.c
    ${CMAKE_SOURCE_DIR}/Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_exti.c
    ${CMAKE_SOURCE_DIR}/Drivers/BSP/Components/lan8742/lan8742.c
)
add_library(STM32_Drivers OBJECT)
target_sources(STM32_Drivers PRIVATE ${STM32_Drivers_Src})
target_link_libraries(STM32_Drivers PUBLIC stm32cubemx)


# 3c) FreeRTOS as an OBJECT library:
set(FreeRTOS_Src
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/croutine.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/event_groups.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/list.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/queue.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/stream_buffer.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/tasks.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/timers.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/CMSIS_RTOS_V2/cmsis_os2.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/portable/MemMang/heap_4.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/FreeRTOS/Source/portable/GCC/ARM_CM4F/port.c
)
add_library(FreeRTOS OBJECT)
target_sources(FreeRTOS PRIVATE ${FreeRTOS_Src})
target_link_libraries(FreeRTOS PUBLIC stm32cubemx)


# 3d) LwIP as an OBJECT library:
set(LwIP_Src
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/auth.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/ccp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/chap_ms.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/chap-md5.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/chap-new.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/demand.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/eap.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/eui64.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/fsm.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/ipcp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/ipv6cp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/lcp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/magic.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/mppe.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/multilink.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/ppp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/pppapi.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/pppcrypt.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/pppoe.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/pppol2tp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/pppos.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/upap.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/utils.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/vj.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/bridgeif.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/bridgeif_fdb.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ethernet.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/lowpan6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/lowpan6_ble.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/lowpan6_common.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/slipif.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/zepif.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/netif/ppp/ecp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/sockets.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/if_api.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/netdb.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/tcpip.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/err.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/netifapi.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/api_lib.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/api_msg.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/api/netbuf.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/memp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/sys.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/inet_chksum.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/raw.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/tcp_out.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ip.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/dns.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/init.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/mem.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/altcp_alloc.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/def.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/udp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/stats.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/tcp_in.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/netif.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/altcp_tcp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/tcp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/pbuf.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/timeouts.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/altcp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/autoip.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/dhcp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/igmp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/etharp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/icmp.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/ip4.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/ip4_frag.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv4/ip4_addr.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/ip6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/icmp6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/mld6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/dhcp6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/ethip6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/nd6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/inet6.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/ip6_addr.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/core/ipv6/ip6_frag.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/system/OS/sys_arch.c
    ${CMAKE_SOURCE_DIR}/Middlewares/Third_Party/LwIP/src/apps/mqtt/mqtt.c
)
add_library(LwIP OBJECT)
target_sources(LwIP PRIVATE ${LwIP_Src})
target_link_libraries(LwIP PUBLIC stm32cubemx)


# ==============================================================================
# 4) FINAL FIRMWARE TARGET (“scion_firmware”)
# ==============================================================================
set(MX_Application_Src
    ${CMAKE_SOURCE_DIR}/LWIP/Target/ethernetif.c
    ${CMAKE_SOURCE_DIR}/LWIP/App/lwip.c
    ${CMAKE_SOURCE_DIR}/Core/Src/main.c
    ${CMAKE_SOURCE_DIR}/Core/Src/lwiperf.c
    ${CMAKE_SOURCE_DIR}/Core/Src/freertos.c
    ${CMAKE_SOURCE_DIR}/Core/Src/stm32h7xx_it.c
    ${CMAKE_SOURCE_DIR}/Core/Src/stm32h7xx_hal_msp.c
    ${CMAKE_SOURCE_DIR}/Core/Src/stm32h7xx_hal_timebase_tim.c
    ${CMAKE_SOURCE_DIR}/Core/Src/sysmem.c
    ${CMAKE_SOURCE_DIR}/Core/Src/syscalls.c
    ${CMAKE_SOURCE_DIR}/Core/Startup/startup_stm32h735igkx.s
)

add_executable(${CMAKE_PROJECT_NAME}
    ${MX_Application_Src}
    $<TARGET_OBJECTS:STM32_Drivers>
    $<TARGET_OBJECTS:FreeRTOS>
    $<TARGET_OBJECTS:LwIP>
)

# Inherit all include‐dirs / defines from “stm32cubemx”
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE stm32cubemx)

# Explicitly force ".elf" suffix again (just in case):
set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES SUFFIX ".elf")

# Turn *.elf → *.bin after build
add_custom_command(
  TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_OBJCOPY} -O binary
          $<TARGET_FILE:${CMAKE_PROJECT_NAME}>
          $<TARGET_FILE_DIR:${CMAKE_PROJECT_NAME}>/$<TARGET_FILE_BASE_NAME:${CMAKE_PROJECT_NAME}>.bin
  COMMENT "Generating ${CMAKE_PROJECT_NAME}.bin from ELF"
)

# Clean up the .map file when “make clean” runs
set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
    ADDITIONAL_CLEAN_FILES ${CMAKE_PROJECT_NAME}.map
)
