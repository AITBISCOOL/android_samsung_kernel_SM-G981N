#
# arch/arm64/Makefile
#
# This file is included by the global makefile so that you can add your own
# architecture-specific flags and dependencies.
#
# This file is subject to the terms and conditions of the GNU General Public
# License.  See the file "COPYING" in the main directory of this archive
# for more details.
#
# Copyright (C) 1995-2001 by Russell King

LDFLAGS_vmlinux	:=--no-undefined -X
CPPFLAGS_vmlinux.lds = -DTEXT_OFFSET=$(TEXT_OFFSET)
GZFLAGS		:=-9

ifeq ($(CONFIG_RELOCATABLE), y)
# Pass --no-apply-dynamic-relocs to restore pre-binutils-2.27 behaviour
# for relative relocs, since this leads to better Image compression
# with the relocation offsets always being zero.
LDFLAGS_vmlinux		+= -shared -Bsymbolic -z notext \
			$(call ld-option, --no-apply-dynamic-relocs)
endif

ifeq ($(CONFIG_ARM64_ERRATUM_843419),y)
  ifeq ($(call ld-option, --fix-cortex-a53-843419),)
$(warning ld does not support --fix-cortex-a53-843419; kernel may be susceptible to erratum)
  else
LDFLAGS_vmlinux	+= --fix-cortex-a53-843419
  endif
endif

KBUILD_DEFCONFIG := defconfig

# Check for binutils support for specific extensions
lseinstr := $(call as-instr,.arch_extension lse,-DCONFIG_AS_LSE=1)

ifeq ($(CONFIG_ARM64_LSE_ATOMICS), y)
  ifeq ($(lseinstr),)
$(warning LSE atomics not supported by binutils)
  endif
endif

ifeq ($(CONFIG_ARM64), y)
brokengasinst := $(call as-instr,1:\n.inst 0\n.rept . - 1b\n\nnop\n.endr\n,,-DCONFIG_BROKEN_GAS_INST=1)

  ifneq ($(brokengasinst),)
$(warning Detected assembler with broken .inst; disassembly will be unreliable)
  endif
endif

ifeq ($(cc-name),clang)
KBUILD_CFLAGS	+= -mno-implicit-float
else
KBUILD_CFLAGS	+= -mgeneral-regs-only
endif
KBUILD_CFLAGS	+= $(lseinstr) $(brokengasinst)
KBUILD_CFLAGS	+= -fno-asynchronous-unwind-tables
KBUILD_CFLAGS	+= $(call cc-disable-warning, psabi)
KBUILD_AFLAGS	+= $(lseinstr) $(brokengasinst)

KBUILD_CFLAGS	+= $(call cc-option,-mabi=lp64)
KBUILD_AFLAGS	+= $(call cc-option,-mabi=lp64)

ifeq ($(CONFIG_SHADOW_CALL_STACK), y)
KBUILD_CFLAGS	+= -ffixed-x18
endif

ifeq ($(CONFIG_CPU_BIG_ENDIAN), y)
KBUILD_CPPFLAGS	+= -mbig-endian
CHECKFLAGS	+= -D__AARCH64EB__
AS		+= -EB
# Prefer the baremetal ELF build target, but not all toolchains include
# it so fall back to the standard linux version if needed.
KBUILD_LDFLAGS	+= -EB $(call ld-option, -maarch64elfb, -maarch64linuxb)
UTS_MACHINE	:= aarch64_be
else
KBUILD_CPPFLAGS	+= -mlittle-endian
CHECKFLAGS	+= -D__AARCH64EL__
AS		+= -EL
# Same as above, prefer ELF but fall back to linux target if needed.
KBUILD_LDFLAGS	+= -EL $(call ld-option, -maarch64elf, -maarch64linux)
UTS_MACHINE	:= aarch64
endif

CHECKFLAGS	+= -D__aarch64__

ifeq ($(CONFIG_ARM64_MODULE_PLTS),y)
KBUILD_LDFLAGS_MODULE	+= -T $(srctree)/arch/arm64/kernel/module.lds
endif

# Default value
head-y		:= arch/arm64/kernel/head.o

# The byte offset of the kernel image in RAM from the start of RAM.
ifeq ($(CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET), y)
TEXT_OFFSET := $(shell awk "BEGIN {srand(); printf \"0x%06x\n\", \
		 int(2 * 1024 * 1024 / (2 ^ $(CONFIG_ARM64_PAGE_SHIFT)) * \
		 rand()) * (2 ^ $(CONFIG_ARM64_PAGE_SHIFT))}")
else
TEXT_OFFSET := 0x00080000
endif

ifeq ($(CONFIG_KASAN_SW_TAGS), y)
KASAN_SHADOW_SCALE_SHIFT := 4
else
KASAN_SHADOW_SCALE_SHIFT := 3
endif

KBUILD_CFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)
KBUILD_CPPFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)
KBUILD_AFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)

# KASAN_SHADOW_OFFSET = VA_START + (1 << (VA_BITS - KASAN_SHADOW_SCALE_SHIFT))
#				 - (1 << (64 - KASAN_SHADOW_SCALE_SHIFT))
# in 32-bit arithmetic
KASAN_SHADOW_OFFSET := $(shell printf "0x%08x00000000\n" $$(( \
	(0xffffffff & (-1 << ($(CONFIG_ARM64_VA_BITS) - 32))) \
	+ (1 << ($(CONFIG_ARM64_VA_BITS) - 32 - $(KASAN_SHADOW_SCALE_SHIFT))) \
	- (1 << (64 - 32 - $(KASAN_SHADOW_SCALE_SHIFT))) )) )

export	TEXT_OFFSET GZFLAGS

core-y		+= arch/arm64/kernel/ arch/arm64/mm/
core-$(CONFIG_NET) += arch/arm64/net/
core-$(CONFIG_KVM) += arch/arm64/kvm/
core-$(CONFIG_XEN) += arch/arm64/xen/
core-$(CONFIG_CRYPTO) += arch/arm64/crypto/
libs-y		:= arch/arm64/lib/ $(libs-y)
core-$(CONFIG_EFI_STUB) += $(objtree)/drivers/firmware/efi/libstub/lib.a

ifeq ($(CONFIG_BUILD_ARM64_KERNEL_COMPRESSION_GZIP),y)
KBUILD_IMAGE   := Image.gz
else
KBUILD_IMAGE   := Image
endif

# Default target when executing plain make
boot		:= arch/arm64/boot

ifeq ($(CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE),y)
KBUILD_TARGET  := $(addsuffix -dtb,$(KBUILD_IMAGE))
KBUILD_IMAGE	:= $(boot)/$(addsuffix -dtb,$(KBUILD_IMAGE))
else
KBUILD_TARGET := $(KBUILD_IMAGE)
KBUILD_IMAGE  := $(boot)/$(KBUILD_IMAGE)
endif

KBUILD_DTBS	:= dtbs

ifeq ($(CONFIG_BUILD_ARM64_DT_OVERLAY),y)
export DTC_FLAGS := -@
endif

all:	$(KBUILD_DTBS) $(KBUILD_TARGET)

Image: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

Image.%: Image
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

zinstall install:
	$(Q)$(MAKE) $(build)=$(boot) $@

%.dtb: scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts $(boot)/dts/$@

PHONY += dtbs dtbs_install

dtbs: prepare scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts

dtbs_install:
	$(Q)$(MAKE) $(dtbinst)=$(boot)/dts

Image-dtb: vmlinux scripts dtbs
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

Image.gz-dtb: vmlinux scripts dtbs Image.gz
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

PHONY += vdso_install
vdso_install:
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso $@

# We use MRPROPER_FILES and CLEAN_FILES now
archclean:
	$(Q)$(MAKE) $(clean)=$(boot)
	$(Q)$(MAKE) $(clean)=$(boot)/dts

ifeq ($(KBUILD_EXTMOD),)
# We need to generate vdso-offsets.h before compiling certain files in kernel/.
# In order to do that, we should use the archprepare target, but we can't since
# asm-offsets.h is included in some files used to generate vdso-offsets.h, and
# asm-offsets.h is built in prepare0, for which archprepare is a dependency.
# Therefore we need to generate the header after prepare0 has been made, hence
# this hack.
prepare: vdso_prepare
vdso_prepare: prepare0
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso include/generated/vdso-offsets.h
endif

define archhelp
  echo  '* Image.gz      - Compressed kernel image (arch/$(ARCH)/boot/Image.gz)'
  echo  '  Image         - Uncompressed kernel image (arch/$(ARCH)/boot/Image)'
  echo  '* dtbs          - Build device tree blobs for enabled boards'
  echo  '  dtbs_install  - Install dtbs to $(INSTALL_DTBS_PATH)'
  echo  '  install       - Install uncompressed kernel'
  echo  '  zinstall      - Install compressed kernel'
  echo  '                  Install using (your) ~/bin/installkernel or'
  echo  '                  (distribution) /sbin/installkernel or'
  echo  '                  install to $$(INSTALL_PATH) and run lilo'
endef
