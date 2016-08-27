include ../py/mkenv.mk

CROSS = 0

# qstr definitions (must come before including py.mk)
QSTR_DEFS = qstrdefsport.h

# include py core make definitions
include ../py/py.mk

ifeq ($(CROSS), 1)
CROSS_COMPILE = arm-none-eabi-
endif

INC += -I.
INC += -I..
INC += -I../lib/mp-readline
# INC += -I../stmhal not an ST part
INC += -I$(BUILD)

ifeq ($(CROSS), 1)
# DFU = ../tools/dfu.py
# PYDFU = ../tools/pydfu.py
# CFLAGS_CORTEX_M4 = -mthumb -mtune=cortex-m4 -mabi=aapcs-linux -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard -fsingle-precision-constant -Wdouble-promotion
# CFLAGS = $(INC) -Wall -Werror -ansi -std=gnu99 -nostdlib $(CFLAGS_CORTEX_M4) $(COPT)
CFLAGS = $(INC) -Wall -O2 -nostdlib -nostartfiles -ffreestanding
else
CFLAGS = -m32 $(INC) -Wall -Werror -ansi -std=gnu99 $(COPT)
endif

#Debugging/Optimization
ifeq ($(DEBUG), 1)
CFLAGS += -O0 -ggdb
else
CFLAGS += -Os -DNDEBUG
endif

ifeq ($(CROSS), 1)
# LDFLAGS = -nostdlib -T stm32f405.ld -Map=$@.map --cref
else
LD = gcc
LDFLAGS = -m32 -Wl,-Map=$@.map,--cref
endif
LIBS =

SRC_C = \
	main.c \
	uart_core.c \
	lib/utils/stdout_helpers.c \
	lib/utils/printf.c \
	lib/utils/pyexec.c \
	lib/libc/string0.c \
	lib/mp-readline/readline.c \
	$(BUILD)/_frozen_mpy.c \

OBJ = $(PY_O) $(addprefix $(BUILD)/, $(SRC_C:.c=.o))

ifeq ($(CROSS), 1)
# all: $(BUILD)/firmware.dfu
all : kernel.img
else
all: $(BUILD)/firmware.elf
endif

$(BUILD)/_frozen_mpy.c: frozentest.mpy $(BUILD)/genhdr/qstrdefs.generated.h
	$(ECHO) "MISC freezing bytecode"
	$(Q)../tools/mpy-tool.py -f -q $(BUILD)/genhdr/qstrdefs.preprocessed.h -mlongint-impl=none $< > $@

$(BUILD)/firmware.elf: $(OBJ)
	$(ECHO) "LINK $@"
	$(Q)$(LD) $(LDFLAGS) -o $@ $^ $(LIBS)
	$(Q)$(SIZE) $@

# $(BUILD)/firmware.dfu: $(BUILD)/firmware.elf
# 	$(ECHO) "Create $@"
# 	$(Q)$(OBJCOPY) -O binary -j .isr_vector -j .text -j .data $^ $(BUILD)/firmware.bin
# 	$(Q)$(PYTHON) $(DFU) -b 0x08000000:$(BUILD)/firmware.bin $@
vectors.o : vectors.s
	$(CROSS_COMPILE)as vectors.s -o vectors.o

bootloader07.o : bootloader07.c
	$(CROSS_COMPILE)gcc $(CFLAGS) -c bootloader07.c -o bootloader07.o

periph.o : periph.c 
	$(CROSS_COMPILE)gcc $(CFLAGS) -c periph.c -o periph.o

kernel.img : loader vectors.o periph.o bootloader07.o 
	$(CROSS_COMPILE)ld vectors.o periph.o bootloader07.o -T loader -o bootloader07.elf
	$(CROSS_COMPILE)objdump -D bootloader07.elf > bootloader07.list
	$(CROSS_COMPILE)objcopy bootloader07.elf -O ihex bootloader07.hex
	$(CROSS_COMPILE)objcopy bootloader07.elf -O binary kernel.img

# deploy: $(BUILD)/firmware.dfu
# 	$(ECHO) "Writing $< to the board"
# 	$(Q)$(PYTHON) $(PYDFU) -u $<

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

include ../py/mkrules.mk
