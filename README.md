# jczhl-package

HiWiFi HC5861（极路由3）的 Padavan 板级配置，用于 [immortalwrt/padavan](https://github.com/immortalwrt/padavan) master。

## 文件对应关系

| 本仓库 | 源码树 |
| --- | --- |
| `HC5861/board.h` | `trunk/configs/boards/HC5861/board.h` |
| `HC5861/board.mk` | `trunk/configs/boards/HC5861/board.mk` |
| `HC5861/kernel-3.4.x.config` | `trunk/configs/boards/HC5861/kernel-3.4.x.config` |
| `HC5861/HC5861.config` | `trunk/configs/templates/HC5861.config` |
| `default-settings/`、`luci-theme-netgear/` | 旧 OpenWrt 时期遗留，Padavan 构建不用 |

## 编译

```sh
git clone --depth 1 https://github.com/immortalwrt/padavan.git
cd padavan
mkdir -p trunk/configs/boards/HC5861
cp /path/to/jczhl-package/HC5861/{board.h,board.mk,kernel-3.4.x.config} trunk/configs/boards/HC5861/
cp /path/to/jczhl-package/HC5861/HC5861.config trunk/configs/templates/
# 预编译工具链
cd toolchain-mipsel && ./dl_toolchain.sh && cd ..
cd trunk && ./build_firmware_modify HC5861
```

产物：`trunk/images/HC5861_3.4.3.9-099.trx`（精简配置约 5.1MB，16MB 闪存余量充足）。

### Dropbear + hanwckf 工具链

该工具链没有 `libssp`，Dropbear 的 configure 自动加 `-fstack-protector-strong` 会链接失败：

```
ld: cannot find -lssp_nonshared
ld: cannot find -lssp
```

给 `trunk/user/dropbear/Makefile` 的 configure 参数加 `--disable-harden`，见 `patches/dropbear-disable-harden.patch`。

## 当前配置取舍

- 保留：闭源无线（2.4G 2.7 / 5G 3.0）、IPv6、中文界面、外置 PA（`CONFIG_EXTERNAL_PA_EXTERNAL_LNA=y`，解决啸叫）
- SSH：Dropbear，关闭 OpenSSH / SFTP / OpenVPN / OpenSSL 附加项
- 关闭：USB 文件系统（ufsd/FAT/exFAT/EXT/FUSE）、SMB/FTP/打印共享、UVC/音频/串口、tcpdump/hdparm/parted、ipset、xUPNPd、QoS/IMQ/IFB、swap
- WAN：`CONFIG_RAETH_HAS_PORT5=y` + `CONFIG_RAETH_ESW_PORT_WAN=5`，即外置千兆 PHY 的 port 5 作 WAN，LAN1~4 对应交换口 0~3

## ccache

在源码树外做编译器 shim（`ccache-shim/`），把 `mipsel-linux-uclibc-gcc/g++` 指向 ccache，其余工具链命令仍走真身；`vendors/config/mips/config.arch` 里用 `CCACHE_SHIM` 环境变量选择前缀。这样内核和用户态都能命中缓存，不改各软件包 Makefile。
