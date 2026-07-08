.PHONY: clean

CC := gcc
LD := ld
CFLAGS := -std=gnu11 -m32 -ggdb3 -MMD -MP -Wall -O2 -I./include -I../include -I../libs -I../libs/nemu-ref/include
LDFLAGS := -m32 -Wl,-m,elf_i386 -L/usr/lib/i386-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/15/32 -lreadline -lSDL

CFILES := $(shell find src/ -name "*.c")
OBJS := $(CFILES:.c=.o)
IFILES := $(CFILES:.c=.i)

nemu: $(OBJS) ../include/config.h 
	$(CC) -o nemu $(OBJS) ../libs/nemu-ref/lib-nemu-ref.a $(LDFLAGS)

%.i: %.c
	$(CC) $(CFLAGS) -E -o $@ $<

clean:
	-rm -f nemu $(OBJS) $(IFILES) $(OBJS:.o=.d)
