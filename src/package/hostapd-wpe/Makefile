#
# Copyright (C) 2006-2014 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=hostapd-wpe
PKG_VERSION:=2014-06-03.1
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.bz2
PKG_SOURCE_SUBDIR:=hostapd-$(PKG_VERSION)

PKG_MAINTAINER:=Oscar Mallo - Tarlogic Security <oscar.mallo@tarlogic.com>, Original hostapd package: Felix Fietkau <nbd@openwrt.org>

PKG_BUILD_PARALLEL:=1

PKG_BUILD_DEPENDS:= \
	PACKAGE_kmod-madwifi:madwifi \

PKG_CONFIG_DEPENDS:= \
	CONFIG_WPA_SUPPLICANT_NO_TIMESTAMP_CHECK \
	CONFIG_PACKAGE_kmod-ath9k \
	CONFIG_PACKAGE_kmod-mac80211 \
	CONFIG_PACKAGE_kmod-madwifi \
	CONFIG_PACKAGE_hostapd \
	CONFIG_PACKAGE_hostapd-mini \
	CONFIG_PACKAGE_kmod-hostap \
	CONFIG_WPA_RFKILL_SUPPORT \
	CONFIG_DRIVER_WEXT_SUPPORT \
	CONFIG_DRIVER_11N_SUPPORT

LOCAL_TYPE=$(strip \
		$(if $(findstring wpad,$(BUILD_VARIANT)),wpad, \
		$(if $(findstring supplicant,$(BUILD_VARIANT)),supplicant, \
		hostapd \
	)))
LOCAL_VARIANT=$(patsubst wpad-%,%,$(patsubst supplicant-%,%,$(BUILD_VARIANT)))

ifeq ($(LOCAL_TYPE),supplicant)
  ifeq ($(LOCAL_VARIANT),full)
    PKG_CONFIG_DEPENDS += \
		CONFIG_WPA_SUPPLICANT_INTERNAL \
		CONFIG_WPA_SUPPLICANT_OPENSSL
  endif
endif

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(BUILD_VARIANT)/$(PKG_NAME)-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

STAMP_CONFIGURED:=$(STAMP_CONFIGURED)_$(CONFIG_WPA_MSG_MIN_PRIORITY)

ifneq ($(CONFIG_DRIVER_11N_SUPPORT),)
  HOSTAPD_IEEE80211N:=y
endif

DRIVER_MAKEOPTS= \
	CONFIG_DRIVER_NL80211=$(CONFIG_PACKAGE_kmod-mac80211) \
	CONFIG_DRIVER_MADWIFI=$(CONFIG_PACKAGE_kmod-madwifi) \
	CONFIG_DRIVER_HOSTAP=$(CONFIG_PACKAGE_kmod-hostap) \
	CONFIG_IEEE80211N=$(HOSTAPD_IEEE80211N) \
	CONFIG_IEEE80211W=$(CONFIG_PACKAGE_kmod-ath9k) \
	CONFIG_DRIVER_WEXT=$(CONFIG_DRIVER_WEXT_SUPPORT) \
	$(if $(CONFIG_WPA_RFKILL_SUPPORT),NEED_RFKILL=y)

ifneq ($(LOCAL_TYPE),hostapd)
  ifdef CONFIG_WPA_SUPPLICANT_OPENSSL
    ifeq ($(LOCAL_VARIANT),full)
      DRIVER_MAKEOPTS += CONFIG_TLS=openssl
      TARGET_LDFLAGS += -lcrypto -lssl
    endif
  endif
  ifdef CONFIG_WPA_SUPPLICANT_NO_TIMESTAMP_CHECK
    TARGET_CFLAGS += -DNO_TIMESTAMP_CHECK
  endif
  DRIVER_MAKEOPTS += \
	CONFIG_DRIVER_ROBOSWITCH=$(CONFIG_PACKAGE_kmod-switch)
endif

ifdef CONFIG_USE_EGLIBC
  TARGET_LDFLAGS += -lrt
  TARGET_LDFLAGS_C += -lrt
endif

DRV_DEPENDS:=+PACKAGE_kmod-mac80211:libnl-tiny @(!TARGET_avr32||BROKEN)

define Package/hostapd-wpe/Default
  SECTION:=net
  CATEGORY:=Network
  TITLE:=IEEE 802.1x Authenticator with WPE support
  DEPENDS:=$(DRV_DEPENDS) +libubus
endef

define Package/hostapd-wpe
$(call Package/hostapd-wpe/Default)
  TITLE+= (full)
  VARIANT:=full
endef

define Package/hostapd-wpe/description
 This package contains a full featured IEEE 802.1x/WPA/EAP/RADIUS Authenticator with WPE patch support.
endef

ifneq ($(wildcard $(PKG_BUILD_DIR)/.config_*),$(subst .configured_,.config_,$(STAMP_CONFIGURED)))
  define Build/Configure/rebuild
	$(FIND) $(PKG_BUILD_DIR) -name \*.o -or -name \*.a | $(XARGS) rm -f
	rm -f $(PKG_BUILD_DIR)/hostapd/hostapd
	rm -f $(PKG_BUILD_DIR)/wpa_supplicant/wpa_supplicant
	rm -f $(PKG_BUILD_DIR)/.config_*
	touch $(subst .configured_,.config_,$(STAMP_CONFIGURED))
  endef
endif

define Build/Configure
	$(Build/Configure/rebuild)
	$(if $(wildcard ./files/hostapd-$(LOCAL_VARIANT).config), \
		$(CP) ./files/hostapd-$(LOCAL_VARIANT).config $(PKG_BUILD_DIR)/hostapd/.config \
	)
	$(CP) ./files/wpa_supplicant-$(LOCAL_VARIANT).config $(PKG_BUILD_DIR)/wpa_supplicant/.config
endef

TARGET_CPPFLAGS := \
	-I$(STAGING_DIR)/usr/include/libnl-tiny \
	-I$(PKG_BUILD_DIR)/src/crypto \
	$(TARGET_CPPFLAGS) \
	-I$(CURDIR)/madwifi \
	-DCONFIG_LIBNL20 \
	-D_GNU_SOURCE \
	$(if $(CONFIG_WPA_MSG_MIN_PRIORITY),-DCONFIG_MSG_MIN_PRIORITY=$(CONFIG_WPA_MSG_MIN_PRIORITY))

TARGET_CFLAGS += -ffunction-sections -fdata-sections
TARGET_LDFLAGS += -Wl,--gc-sections
ifeq ($(findstring supplicant,$(BUILD_VARIANT)),)
  TARGET_LDFLAGS += -lubox -lubus
endif

ifdef CONFIG_PACKAGE_kmod-mac80211
  TARGET_LDFLAGS += -lm -lnl-tiny
endif

define Build/RunMake
	CFLAGS="$(TARGET_CPPFLAGS) $(TARGET_CFLAGS)" \
	$(MAKE) $(PKG_JOBS) -C $(PKG_BUILD_DIR)/$(1) \
		$(TARGET_CONFIGURE_OPTS) \
		$(DRIVER_MAKEOPTS) \
		LIBS="$(TARGET_LDFLAGS)" \
		LIBS_c="$(TARGET_LDFLAGS_C)" \
		BCHECK= \
		$(2)
endef

define Build/Compile/hostapd
	$(call Build/RunMake,hostapd, \
		hostapd hostapd_cli \
	)
endef

define Build/Compile
	echo ">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> LOCAL TYPE = $(LOCAL_TYPE)"
	echo ">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> BUILD VARIANT = $(BUILD_VARIANT)"
	$(Build/Compile/hostapd)
	$(Build/Compile/full)
endef

config_files=hostapd-wpe-bgn.conf hostapd-wpe-an.conf hostapd-wpe.eap_user
cert_config_files=ca.der ca.key ca.pem dh server.key server.pem server_no_pass.key

define Install/hostapd-wpe
	$(INSTALL_DIR) $(1)/usr/sbin
	$(INSTALL_DIR) $(1)/usr/local/etc/hostapd-wpe
	$(INSTALL_DIR) $(1)/usr/local/etc/hostapd-wpe/certs
endef

define Package/hostapd-wpe/install
	$(call Install/hostapd-wpe,$(1))
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/hostapd/hostapd-wpe $(1)/usr/sbin/
	$(INSTALL_DATA) $(addprefix $(PKG_BUILD_DIR)/hostapd/etc/,$(config_files)) $(1)/usr/local/etc/hostapd-wpe/
	$(INSTALL_DATA) $(addprefix $(PKG_BUILD_DIR)/hostapd/etc/certs/,$(cert_config_files)) $(1)/usr/local/etc/hostapd-wpe/certs/
endef

$(eval $(call BuildPackage,hostapd-wpe))
