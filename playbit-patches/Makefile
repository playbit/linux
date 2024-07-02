#!DEPENDS build:openssl build:libelf
include ../playbit.defs.mk

LINUX_VERSION    := 6.8.9
LINUX_VERSION_Ma := $(word 1,$(subst ., ,$(LINUX_VERSION)))
LINUX_TAR_SHA256 := f905f1238ea7a8e85314bacf283302e8097006010d25fcea726d0de0ea5bc9b6
LINUX_TAR_NAME   := linux-$(LINUX_VERSION).tar.xz
LINUX_TAR        := $(DOWNLOAD)/$(LINUX_TAR_NAME)
LINUX_TAR_URL    := \
	https://cdn.kernel.org/pub/linux/kernel/v$(LINUX_VERSION_Ma).x/$(LINUX_TAR_NAME)

DESTDIR := $(DISTROOT)-$(ARCH)
BUILDDIR := $(BUILD_DIR)/linux-$(LINUX_VERSION)-$(ARCH)
CONFIG := $(CURDIR)/$(ARCH)-virt.conf
EXE := $(OUTPUTDIR)/playbit-kernel
PATCHES := $(addprefix $(CURDIR)/,$(sort $(wildcard patches/*.patch)))
KARCH := $(if $(filter $(ARCH),aarch64),arm64,x86)
VMLINUX:=$(BUILDDIR)/arch/$(KARCH)/boot/$(if $(filter $(ARCH),aarch64),Image.gz,bzImage)

all: $(EXE)

include headers.mk
# treat this one header specially so that in quiet mode we get some output from QLOG
$(DESTDIR)/usr/include/linux/version.h: include/linux/version.h
	$(QLOG) "INSTALL_HEADERS"
	$(Q)install -D -m0644 $< $@
$(DESTDIR)/usr/include/%.h: include-$(ARCH)/%.h
	$(Q)install -D -m0644 $< $@
$(DESTDIR)/usr/%.h: %.h
	$(Q)install -D -m0644 $< $@
DEST_HEADERS := $(addprefix $(DESTDIR)/usr/include/,$(HEADERS))
headers: $(DEST_HEADERS)
# allow speeding up common case
$(BUILDDIR)/headers.install: include/linux/version.h $(DESTDIR)/usr/include/linux/version.h
	$(MAKE) headers ARCH=$(ARCH)
	$(Q)mkdir -p $(@D)
	$(Q)touch $@

clean:
	$(Q)rm -rf $(DEST_HEADERS)
	$(Q)if [ -d $(BUILDDIR) ]; then \
		make -C $(BUILDDIR) LLVM=1 ARCH=$(KARCH) mrproper || rm -rf $(BUILDDIR); \
	fi

config: $(BUILDDIR)/.config

update-config:
	$(Q)cp $(CONFIG) $(BUILDDIR)/.config
	$(Q)make -C $(BUILDDIR) LLVM=1 ARCH=$(KARCH) oldconfig
	$(Q)cp $(BUILDDIR)/.config $(CONFIG).new
	$(Q)touch $(CONFIG)
	@echo "Updated config saved as: $(CONFIG).new"

print-vars:
	@echo "LINUX_VERSION=$(LINUX_VERSION)"
	@echo "LINUX_TAR_NAME=$(LINUX_TAR_NAME)"
	@echo "LINUX_TAR_SHA256=$(LINUX_TAR_SHA256)"
	@echo "LINUX_TAR_URL=$(LINUX_TAR_URL)"
	@echo "BUILDDIR=$(BUILDDIR)"
	@echo "DESTDIR=$(DESTDIR)"
	@echo "CONFIG=$(CONFIG)"
	@echo "EXE=$(EXE)"
	@echo "PATCHES=$(PATCHES)"
	@echo "ARCH=$(ARCH)"
	@echo "KARCH=$(KARCH)"
	@echo "VMLINUX=$(VMLINUX)"

$(EXE): $(VMLINUX)
	$(QLOG) "COPY $< -> $@"
	$(Q)mkdir -p $(@D)
	$(Q)cp $< $@

$(VMLINUX): $(BUILDDIR)/headers.install $(BUILDDIR)/.config
	make -C $(BUILDDIR) \
		LLVM=1 \
		READELF=readelf \
		PKG_CONFIG="PKG_CONFIG_LIBDIR=$(DESTDIR)/lib/pkgconfig pkg-config" \
		HOSTPKG_CONFIG="PKG_CONFIG_LIBDIR=$(DISTROOT)-$(HOST_ARCH)/lib/pkgconfig pkg-config" \
		ARCH=$(KARCH) \
		-j$(NCPU) $(@F)
	$(Q)touch -c $@

$(BUILDDIR)/.config: $(CONFIG) $(BUILDDIR)/Makefile
	$(QLOG) "COPY $< -> $@"
	$(Q)cp $< $@
	@echo "Checking for config changes"
	make -C $(BUILDDIR) LLVM=1 ARCH=$(KARCH) -j oldconfig

$(BUILDDIR)/Makefile: $(PATCHES)
	$(QLOG) "FETCH $(LINUX_TAR_URL)"
	$(Q)download -o "$(LINUX_TAR)" --sha256 $(LINUX_TAR_SHA256) "$(LINUX_TAR_URL)"
	$(Q)rm -rf "$(@D)"
	$(Q)mkdir -p "$(@D)"
	$(QLOG) "UNTAR $(LINUX_TAR_NAME) -> $(@D)"
	$(Q)tar -C "$(@D)" --strip-components=1 -xoJf "$(LINUX_TAR)"
	$(Q)cd $(@D) && for f in $(PATCHES); do \
		echo "PATCH $$f"; \
		patch -p1 < "$$f"; \
	done
	$(Q)touch -c "$@"

# install-headers:
# 	$(QLOG) "MAKE headers_install in $(BUILDDIR)"
# 	$(Q)make -C $(BUILDDIR) \
# 		LLVM=1 ARCH=$(KARCH) INSTALL_HDR_PATH="$(DESTDIR)/usr" headers_install \
# 		$(if $(filter 1,$(V)),,>/dev/null)

# update-headers builds headers and replaces the source "include" directory.
# Requires a host distroot with libc and libunwind building helper programs.
update-headers:
	$(MAKE) config ARCH=aarch64
	$(MAKE) config ARCH=x86_64
	rm -rf "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers"
	mkdir -p "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers"
	make -C $(BUILD_DIR)/linux-$(LINUX_VERSION)-aarch64 \
		LLVM=1 ARCH=arm64 \
		INSTALL_HDR_PATH="$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/aarch64" \
		headers_install
	make -C $(BUILD_DIR)/linux-$(LINUX_VERSION)-x86_64 \
		LLVM=1 ARCH=x86 \
		INSTALL_HDR_PATH="$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/x86_64" \
		headers_install
	$(MAKE) -C ../build-tools dirdedup
	cd "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers" && \
		$(CURDIR)/../build-tools/dirdedup . > dups.txt && \
		while IFS= read -r line; do \
			src1=$${line%%:*}; \
			dst=generic/$${src1#*/}; \
			dstdir=$${dst%/*}; \
			[ -d $$dstdir ] || mkdir -vp $$dstdir; \
  		mv -v $$src1 $$dst; \
  		rm -v $${line##*:}; \
		done < dups.txt && \
		rm dups.txt && \
		find . -type d -empty -delete
	@# Workaround for case-sensitive clashing files in netfiler.
	@# I.e. ipt_ttl.h and ipt_TTL.h are two different files.
	cd "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/generic/include/linux" && \
		find netfilter*/ -type f -regex '.*[A-Z].*' \
		| while IFS= read -r f; do \
			dst=$${f%/*}/\$$$${f##*/}; \
			mv -v $$f $$dst; \
		done
	cd "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/generic/include" && \
		echo "# This file was generated by make $@" > ../../headers.mk && \
		echo "HEADERS := \\" >> ../../headers.mk && \
		find . -type f | sort \
		| sed -E 's/^\.\/(.+)$$/  \1 \\/' >> ../../headers.mk
	cd "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/aarch64/include" && \
		echo "" >> ../../headers.mk && \
		echo 'ifeq ($$(ARCH),aarch64)' >> ../../headers.mk && \
		echo "HEADERS += \\" >> ../../headers.mk && \
		find . -type f | sort \
		| sed -E 's/^\.\/(.+)$$/  \1 \\/' >> ../../headers.mk && \
		echo "" >> ../../headers.mk && \
		echo "endif" >> ../../headers.mk
	cd "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/x86_64/include" && \
		echo "" >> ../../headers.mk && \
		echo 'ifeq ($$(ARCH),x86_64)' >> ../../headers.mk && \
		echo "HEADERS += \\" >> ../../headers.mk && \
		find . -type f | sort \
		| sed -E 's/^\.\/(.+)$$/  \1 \\/' >> ../../headers.mk && \
		echo "" >> ../../headers.mk && \
		echo "endif" >> ../../headers.mk
	cd "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers" && \
		cp headers.mk headers.mk.tmp && \
		sed -i -E 's@/\$$@/@' headers.mk && \
		echo "" >> headers.mk && \
		grep -F '/$$' headers.mk.tmp | while IFS= read -r f; do \
			i=$${f% *}; \
			i=$${i:2}; \
			o=$${i/\/\$$/\/}; \
			i=$${i/\/\$$/\/\$$\$$}; \
			echo "\$$(DESTDIR)/usr/include/$$o: include/$$i" >> headers.mk; \
			printf "\t\$$(Q)install -D -m0644 '\$$<' \$$@\n" >> headers.mk; \
		done
	cp "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/headers.mk" headers.mk
	rm -rf include include-aarch64 include-x86_64
	cp -R "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/generic/include" include
	cp -R "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/aarch64/include" include-aarch64
	cp -R "$(BUILD_DIR)/linux-$(LINUX_VERSION)-headers/x86_64/include" include-x86_64

.PHONY: all clean print-vars headers update-headers config update-config
