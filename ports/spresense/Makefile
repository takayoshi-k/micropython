include ../../py/mkenv.mk

# qstr definitions (must come before including py.mk)
QSTR_DEFS = qstrdefsport.h

# include py core make definitions
include $(TOP)/py/py.mk

CROSS_COMPILE = arm-none-eabi-

SPRESENSE_PORT = $(TOP)/ports/spresense
SPR_BASE = $(TOP)/ports/spresense/export/sdk-export
SPR_SDKBSP_INC = $(SPR_BASE)/sdk/bsp/include
SPR_SDKMOD_INC = $(SPR_BASE)/sdk/modules/include
SPR_NUTTX_INC  = $(SPR_BASE)/nuttx/include
SPR_SDKLIBS    = $(SPR_BASE)/sdk/libs
SPR_TOOLTOP    = $(SPR_BASE)/sdk/tools/linux
SPR_MKSPK      = $(SPR_TOOLTOP)/mkspk
TMP_GCCLIB = "/usr/bin/../lib/gcc/arm-none-eabi/6.2.1/thumb/v7e-m/fpv4-sp/hard/libgcc.a"

SPR_LDFILE = $(SPR_BASE)/nuttx/build/ramconfig.ld

INC += -I.
INC += -I$(TOP)
INC += -I$(BUILD)
INC += -I$(SPR_SDKBSP_INC)
INC += -I$(SPR_SDKMOD_INC)
INC += -I$(SPR_NUTTX_INC)

DFU = $(TOP)/tools/dfu.py
PYDFU = $(TOP)/tools/pydfu.py
# CFLAGS_CORTEX_M4 = -mthumb -mtune=cortex-m4 -mabi=aapcs-linux -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard -fsingle-precision-constant -Wdouble-promotion
CFLAGS_CORTEX_M4 = -fno-builtin	      \
									 -mabi=aapcs        \
									 -Wall              \
									 -Wstrict-prototypes\
									 -Wshadow						\
									 -g									\
									 -Os								\
									 -fno-strict-aliasing	\
									 -fno-strength-reduce	\
									 -fomit-frame-pointer	\
									 -mcpu=cortex-m4			\
									 -mthumb							\
									 -mfpu=fpv4-sp-d16		\
									 -mfloat-abi=hard 		\
									 -isystem $(SPR_NUTTX_INC)	\
									 -pipe								\
									 -gdwarf-3						\
									 -isystem $(SPR_SDKBSP_INC)	\
									 -isystem $(SPR_SDKMOD_INC)

CFLAGS = $(INC) -Wall -Werror -std=c99 -nostdlib $(CFLAGS_CORTEX_M4) $(COPT)
# LDFLAGS = -nostdlib -T stm32f405.ld -Map=$@.map --cref --gc-sections
LDFLAGS = \
	--entry=__start \
	-nostartfiles		\
	-nodefaultlibs	\
	-g							\
	-Map=$(SPRESENSE_PORT)/mpython.map\
	--cref														\
	--defsym __stack=_vectors+1572864	\
	-T$(SPR_LDFILE)										\
	-L$(SPR_SDKLIBS)									\
  --gc-sections                     \
	--start-group											\
	-lsdk															\
	-lapps														\
	$(TMP_GCCLIB)											\
	--end-group


# Tune for Debugging or Optimization
ifeq ($(DEBUG), 1)
CFLAGS += -O0 -ggdb
else
CFLAGS += -Os -DNDEBUG
CFLAGS += -fdata-sections -ffunction-sections
endif

LIBS = \
	-lsdk															\
	-lapps														\
	$(TMP_GCCLIB)											\

# lib/utils/printf.c \
# lib/utils/stdout_helpers.c \
# lib/mp-readline/readline.c \
# lib/libc/string0.c \

SRC_C = \
	main.c \
  uart_core.c \
	lib/utils/pyexec.c \
	$(BUILD)/_frozen_mpy.c \

OBJ = $(PY_CORE_O) $(addprefix $(BUILD)/, $(SRC_C:.c=.o))

# all: $(BUILD)/mpython.elf
all: mpython.spk

$(BUILD)/_frozen_mpy.c: frozentest.mpy $(BUILD)/genhdr/qstrdefs.generated.h
	$(ECHO) "MISC freezing bytecode"
	$(Q)$(TOP)/tools/mpy-tool.py -f -q $(BUILD)/genhdr/qstrdefs.preprocessed.h -mlongint-impl=none $< > $@

$(BUILD)/mpython.elf: $(OBJ)
	$(ECHO) "LINK $@"
	$(Q)$(LD) $(LDFLAGS) -o $@ $^ $(LIBS)
	$(Q)$(SIZE) $@

mpython.spk: $(BUILD)/mpython.elf
	$(ECHO) "SPK $@"
	$(SPR_MKSPK) -c 2 $^ nuttx $@ >/dev/null

$(BUILD)/firmware.bin: $(BUILD)/firmware.elf
	$(Q)$(OBJCOPY) -O binary -j .isr_vector -j .text -j .data $^ $(BUILD)/firmware.bin

$(BUILD)/firmware.dfu: $(BUILD)/firmware.bin
	$(ECHO) "Create $@"
	$(Q)$(PYTHON) $(DFU) -b 0x08000000:$(BUILD)/firmware.bin $@

deploy: $(BUILD)/firmware.dfu
	$(ECHO) "Writing $< to the board"
	$(Q)$(PYTHON) $(PYDFU) -u $<

# Run emulation build on a POSIX system with suitable terminal settings
run:
	stty raw opost -echo
	build/firmware.elf
	@echo Resetting terminal...
# This sleep is useful to spot segfaults
	sleep 1
	reset

test: $(BUILD)/firmware.elf
	$(Q)/bin/echo -e "print('hello world!', list(x+1 for x in range(10)), end='eol\\\\n')\\r\\n\\x04" | $(BUILD)/firmware.elf | tail -n2 | grep "^hello world! \\[1, 2, 3, 4, 5, 6, 7, 8, 9, 10\\]eol"

include $(TOP)/py/mkrules.mk
