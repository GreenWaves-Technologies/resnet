# Copyright (c) 2022 GreenWaves Technologies SAS
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# 1. Redistributions of source code must retain the above copyright notice,
#    this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
# 3. Neither the name of GreenWaves Technologies SAS nor the names of its
#    contributors may be used to endorse or promote products derived from
#    this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required(VERSION 3.16)

###############################################################################
# Application related definitions
###############################################################################

set(TARGET_NAME         "ResNet")
set(MAIN                main.c)

###############################################################################
# CMake pre initialization
###############################################################################

include($ENV{GAP_SDK_HOME}/utils/cmake/setup.cmake)

set(APP_MSG_PREFIX "[${BoldRed}${TARGET_NAME}${ColourReset}]")

###############################################################################
# Application's kconfig options
###############################################################################

set(IMAGE images/goldfish.ppm)

###############################################################################
# NNTool Variables 
###############################################################################
set(CONFIG_NNTOOL_SCRIPT_PATH model/nntool_script)
if(DEFINED CONFIG_RESNET50)
    if (DEFINED MODEL_INT8)
        set(CONFIG_NNTOOL_MODEL_PATH model/resnet50-v1-7-int8.onnx)
        set(CONFIG_NNTOOL_SCRIPT_PATH model/nntool_script_quant)
    else()
        set(CONFIG_NNTOOL_MODEL_PATH model/resnet50-v1-7.onnx)
    endif()
    list(APPEND NNTOOLSCRIPT_FLAGS -DSTATS_DICT=${CMAKE_CURRENT_SOURCE_DIR}/model/resnet50_astats.pickle)
else()
    set(CONFIG_NNTOOL_MODEL_PATH model/resnet18-v1-7.onnx)
    list(APPEND NNTOOLSCRIPT_FLAGS -DSTATS_DICT=${CMAKE_CURRENT_SOURCE_DIR}/model/resnet18_astats.pickle)
endif()

if (CONFIG_USE_PRIVILEGED_MRAM)
    message(STATUS "Selected MRAM Privileged flash")
    list(APPEND NNTOOLSCRIPT_FLAGS -DMODEL_SEC_L3_FLASH=AT_MEM_L3_MRAMFLASH)
else()
    list(APPEND NNTOOLSCRIPT_FLAGS -DMODEL_SEC_L3_FLASH=)
endif()

autotiler_define_gen_names(${TARGET_NAME})
nntool_setup(${TARGET_NAME})
autotiler_setup(${TARGET_NAME})

###############################################################################
# Application dependencies
###############################################################################

list(APPEND TARGET_SRCS ${MAIN}
                        ${CNN_LIB}
                        ${COMMON_MODEL_BUILDDIR}/${KERNEL_C}
)

list(APPEND TARGET_INCS -I${TILER_INC}
                        -I${TILER_EMU_INC}
                        -I${GAP_SDK_HOME}/libs/gap_lib/include
                          ${CNN_LIB_INCLUDE}
                        -I${COMMON_MODEL_BUILDDIR}
                        -I${CMAKE_CURRENT_SOURCE_DIR}
                        -I${GAP_SDK_HOME}/utils/power_meas_utils)

list(APPEND TARGET_CFLAGS   -gdwarf-2
                            -gstrict-dwarf
                            -O3
                            -mno-memcpy
                            -fno-tree-loop-distribute-patterns)

math(EXPR FREQ_CL_MHZ "${CONFIG_FREQUENCY_CLUSTER}/1000/1000")
math(EXPR FREQ_FC_MHZ "${CONFIG_FREQUENCY_FC}/1000/1000")
math(EXPR FREQ_PE_MHZ "${CONFIG_FREQUENCY_PERIPH}/1000/1000")
list(APPEND TARGET_PREPROCESSOR -DPERF
                                -DAT_MODEL_PREFIX=${MODEL_NAME}
                                -DSTACK_SIZE=${CONFIG_CL_MASTER_CORE_STACK_SIZE}
                                -DSLAVE_STACK_SIZE=${CONFIG_CL_SLAVE_CORE_STACK_SIZE}
                                -DAT_IMAGE=${CMAKE_CURRENT_SOURCE_DIR}/${IMAGE}
                                -DAT_INPUT_HEIGHT=${AT_INPUT_WIDTH}
                                -DAT_INPUT_WIDTH=${AT_INPUT_HEIGHT}
                                -DAT_INPUT_COLORS=${AT_INPUT_COLORS}
                                -DFREQ_CL=${FREQ_CL_MHZ}
                                -DFREQ_FC=${FREQ_FC_MHZ}
                                -DFREQ_PE=${FREQ_PE_MHZ}
                                -DAT_CONSTRUCT=${AT_CONSTRUCT}
                                -DAT_DESTRUCT=${AT_DESTRUCT}
                                -DAT_CNN=${AT_CNN}
                                -DGROUND_TRUTH=644)
if (DEFINED PERF_CI)
list(APPEND TARGET_PREPROCESSOR -DPERF_CI=${PERF_CI})
endif()

project(${TARGET_NAME} C ASM)
add_executable(${TARGET_NAME} ${TARGET_SRCS})

target_compile_options(${TARGET_NAME} PUBLIC    ${TARGET_CFLAGS} 
                                                ${TARGET_INCS}
                                                ${TARGET_PREPROCESSOR})


###############################################################################
# CMake post initialization
###############################################################################
setupos(${TARGET_NAME})

