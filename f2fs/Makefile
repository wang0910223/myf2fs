# # SPDX-License-Identifier: GPL-2.0
# obj-$(CONFIG_F2FS_FS) += f2fs.o

# f2fs-y		:= dir.o file.o inode.o namei.o hash.o super.o inline.o
# f2fs-y		+= checkpoint.o gc.o data.o node.o segment.o recovery.o
# f2fs-y		+= shrinker.o extent_cache.o sysfs.o
# f2fs-$(CONFIG_F2FS_STAT_FS) += debug.o
# f2fs-$(CONFIG_F2FS_FS_XATTR) += xattr.o
# f2fs-$(CONFIG_F2FS_FS_POSIX_ACL) += acl.o
# f2fs-$(CONFIG_FS_VERITY) += verity.o
# f2fs-$(CONFIG_F2FS_FS_COMPRESSION) += compress.o
# f2fs-$(CONFIG_F2FS_IOSTAT) += iostat.o

# /* CXL DAX */
# SPDX-License-Identifier: GPL-2.0

# 1. 強制宣告這是一個獨立載入的模組 (輸出為 myf2fs.ko)
obj-m += myf2fs.o

# 2. 將原本的 f2fs-y 替換為 myf2fs-y，並列出所有核心檔案
myf2fs-y := dir.o file.o inode.o namei.o hash.o super.o inline.o
myf2fs-y += checkpoint.o gc.o data.o node.o segment.o recovery.o
myf2fs-y += shrinker.o extent_cache.o sysfs.o

# 3. 繼承你目前伺服器的 Kernel Config，自動決定是否加入擴充功能
myf2fs-$(CONFIG_F2FS_STAT_FS) += debug.o
myf2fs-$(CONFIG_F2FS_FS_XATTR) += xattr.o
myf2fs-$(CONFIG_F2FS_FS_POSIX_ACL) += acl.o
myf2fs-$(CONFIG_FS_VERITY) += verity.o
myf2fs-$(CONFIG_F2FS_FS_COMPRESSION) += compress.o
myf2fs-$(CONFIG_F2FS_IOSTAT) += iostat.o

# ===================================================================
# 4. Out-of-Tree 專用編譯指令區塊
# ===================================================================
KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

# 注意：下方的 $(MAKE) 前面必須是「Tab」鍵，不能是空白建！
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean