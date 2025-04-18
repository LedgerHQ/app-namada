#*******************************************************************************
#  Ledger App
#  (c) 2018 - 2024 Zondax AG
#  (c) 2017 Ledger
#
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.
#*******************************************************************************

ifeq ($(BOLOS_SDK),)
$(error BOLOS_SDK is not set)
endif

MY_DIR := $(dir $(lastword $(MAKEFILE_LIST)))

# Set the default value for PRODUCTION_BUILD to 1 if not already defined
PRODUCTION_BUILD ?= 1

# Display whether this is a production build or for internal use
ifeq ($(PRODUCTION_BUILD), 1)
    $(info ************ PRODUCTION_BUILD  = [PRODUCTION BUILD])
else
    $(info ************ PRODUCTION_BUILD  = [INTERNAL USE])
endif

# Add the PRODUCTION_BUILD definition to the compiler flags
DEFINES += PRODUCTION_BUILD=$(PRODUCTION_BUILD)

include $(CURDIR)/../deps/ledger-zxlib/makefiles/Makefile.app_testing

ifndef COIN
COIN=NAM
endif

VARIANT_PARAM=COIN
VARIANT_VALUES=$(COIN)

include $(CURDIR)/Makefile.version

$(info COIN  = [$(COIN)])

ifeq ($(COIN),NAM)
# Main app configuration
APPNAME = "Namada"
APPPATH = "44'/877'" --path "44'/1'"

else
define error_message

COIN value not supported: [$(COIN)]

endef
$(error "$(error_message)")
endif

APP_LOAD_PARAMS =  --curve ed25519 $(COMMON_LOAD_PARAMS) --path $(APPPATH)

# Enable QR code display for all devices
ENABLE_NBGL_QRCODE ?= 1

RUST_TARGET:=thumbv6m-none-eabi

$(info ************ RUST_TARGET  = [$(RUST_TARGET)])

include $(BOLOS_SDK)/Makefile.target

include $(CURDIR)/../deps/ledger-zxlib/makefiles/Makefile.devices

include $(CURDIR)/../deps/ledger-zxlib/makefiles/Makefile.platform

# Compile MASP mode for all devices excetpt Nano S, 
ifneq ($(TARGET_NAME),TARGET_NANOS)
    DEFINES += COMPILE_MASP
endif

# Add SDK BLAKE2b
DEFINES += HAVE_HASH HAVE_BLAKE2
INCLUDES_PATH += $(BOLOS_SDK)/lib_cxng/src

# Building Rust
LDFLAGS += -z muldefs
LDLIBS += -L$(MY_DIR)rust/target/$(RUST_TARGET)/release -lrslib
APP_SOURCE_PATH += $(CURDIR)/rust/include

include $(CURDIR)/../deps/ledger-zxlib/makefiles/Makefile.installer_script

$(info TARGET_NAME  = [$(TARGET_NAME)])
$(info ICONNAME  = [$(ICONNAME)])

ifndef ICONNAME
$(error ICONNAME is not set)
endif

# make rust a prerequisite for all object files
$(OBJECT_FILES): | rust

.PHONY: rust
rust:
	cd rust && RUSTC_BOOTSTRAP=1 CARGO_HOME="$(CURDIR)/rust/.cargo" cargo build --target $(RUST_TARGET) --release

.PHONY: rust_clean
rust_clean:
	cd rust && CARGO_HOME="$(CURDIR)/rust/.cargo" cargo clean

clean: rust_clean

#add dependency on custom makefile filename
dep/%.d: %.c Makefile

.PHONY: version
version:
	@echo "v$(APPVERSION)" > app.version
