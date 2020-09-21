cmake_minimum_required(VERSION 3.0.0)

# needed to omit the linker (avoid to build executeable on test-program)
set(CMAKE_TRY_COMPILE_TARGET_TYPE "STATIC_LIBRARY")
project(TemplateProject C CXX ASM)


################################################################################
# Set default values for input parameters
################################################################################

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif()

set(BASE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

MESSAGE(${BASE_DIR})

################################################################################
# Processor, microprocessor settings
################################################################################
set(CPU "cortex-m3")
set(DEFINITIONS "${DEFINITIONS} -DSTM32 -DSTM32F1 -DSTM32F103C8Tx -DDEBUG -DSTM32F10X_MD -DARM_MATH_CM3")
set(DEFINITIONS "${DEFINITIONS} -DUSE_STDPERIPH_DRIVER")

# Fill in the linker script to use here
set(LINKER_SCRIPT "${BASE_DIR}/scripts/STM32F103C8Tx_FLASH.ld")
set(OPENOCD_SCRIPT "${BASE_DIR}/scripts/openocd.cfg")

################################################################################
# Compiler/compile settings
################################################################################
set(COMPILER_PRE_NAME "arm-none-eabi-")

set(CMAKE_C_COMPILER    "${COMPILER_PRE_NAME}gcc")
set(CMAKE_CXX_COMPILER  "${COMPILER_PRE_NAME}g++")
set(CMAKE_ASM_COMPILER  "${COMPILER_PRE_NAME}as")

set(CMAKE_C_STANDARD 99)
set(C_STANDARD_REQUIRED ON)


set(WARNING_FLAGS           "-Wall -Wextra -Wpedantic")
set(COMMON_COMPILE_FLAGS    "-mcpu=${CPU} -mthumb")

set(CMAKE_C_FLAGS           "${CMAKE_C_FLAGS} ${COMMON_COMPILE_FLAGS} ${WARNING_FLAGS} ${DEFINITIONS}")
set(CMAKE_CXX_FLAGS         "${CMAKE_CXX_FLAGS} ${COMMON_COMPILE_FLAGS} ${WARNING_FLAGS} ${DEFINITIONS}")
set(CMAKE_ASM_FLAGS         "${CMAKE_ASM_FLAGS} ${COMMON_COMPILE_FLAGS}")
set(CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} -T ${LINKER_SCRIPT} -specs=nosys.specs \
                              -Wl,-Map=${PROJECT_NAME}.map,--cref -Wl,--gc-sections")

set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "" )
set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "" )

################################################################################
# Source Files
################################################################################
set(STARTUP_SOURCES     "${BASE_DIR}/Drivers/CMSIS/startup/startup_stm32.s"
                        "${BASE_DIR}/Drivers/CMSIS/startup/system_stm32f10x.c"
)
set(APP_SOURCES         "${BASE_DIR}/App/src/main.c")

set(APP_LIBRARIES       "arm_cortexM3l_math.a")
set(APP_LIBRARY_PATHS   "${BASE_DIR}/Drivers/CMSIS/Lib/GCC")

AUX_SOURCE_DIRECTORY(   "${BASE_DIR}/Drivers/StdPeriph_Driver/src" STD_PERIPH_SOURCES)

################################################################################
# Header Files
################################################################################
include_directories(
    "${CMAKE_CURRENT_SOURCE_DIR}/Drivers/StdPeriph_Driver/inc"
    "${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/core"
    "${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/device"
    "${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/DSP/Include"
)

link_directories(${APP_LIBRARY_PATHS})

################################################################################
# Targets
################################################################################

# Compile the sources to an .elf
add_executable(${PROJECT_NAME}.elf ${STARTUP_SOURCES} ${APP_SOURCES} ${STD_PERIPH_SOURCES} ${HEADERS})
target_link_libraries(${PROJECT_NAME}.elf ${APP_LIBRARIES})

# Create a binary and print size information
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND echo "# generating binary and print size information in bytes"
    COMMAND ${COMPILER_PRE_NAME}objcopy -Oihex ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex
    COMMAND ${COMPILER_PRE_NAME}size ${PROJECT_NAME}.elf
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

# Flash the previously built firmware to the target (openocd needs to run in background)
add_custom_target(flash
    COMMAND openocd -f ${OPENOCD_SCRIPT} -c "program ${PROJECT_NAME}.elf verify reset exit"
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    DEPENDS ${PROJECT_NAME}.elf
)

