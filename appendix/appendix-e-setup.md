# 附录 E：环境准备——从零到一台可动手的 NixOS

> **本附录导读**：全书机制讲了 46 章，唯独没讲「怎么获得一个能动手的环境」——本附录补上这个断点。按投入程度给三条路径：A. 在现有 Linux/macOS 上只装包管理器试水（10 分钟）；B. 虚拟机里体验完整 NixOS（半小时）；C. 裸机安装 NixOS 当日常系统（一小时起步）。路径 C 之后有「装完第一小时」清单：连网、SSH、桌面与显卡、日常升级与回滚。安装器界面与选项随版本变化较快，本附录以 NixOS 26.05 为基准，细节以官方手册 Installation 章节为准（链接见延伸阅读）。

## E.1 三条路径怎么选

| | 路径 A：只装 Nix | 路径 B：虚拟机 | 路径 C：裸机安装 |
| --- | --- | --- | --- |
| 你得到什么 | nix / nix-shell / flakes | 一台完整 NixOS | 日常使用的 NixOS |
| 能否练 configuration.nix | 否（无系统模块） | 完全能，且坏了可回滚快照 | 完全能 |
| 风险 | 极低（`$HOME` 外只多一个 `/nix`） | 零 | 需分区，建议先备份 |
| 对应全书内容 | 第 13–21 章 | 第 1–46 章全部 | 全部 + 第 24/26/30 章实战 |
| 适合谁 | 先验证兴趣 | 学习期主力方案 | 已决定迁移 |

学习期最推荐 **B**：虚拟机快照 + NixOS 自带的 generation 回滚（第 18 章）双保险，怎么折腾都救得回来。C 路径没有任何不可逆步骤——但分区动磁盘，老规矩：先备份。

## E.2 路径 A：现有 Linux/macOS 上只装 Nix

适合「书看到第二部分，想跟着敲第 6–21 章的命令」的读者。两种安装器：

```console
# 推荐：Determinate Systems 安装器（flake 支持开箱即用，卸载干净）
$ curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install

# 官方脚本：https://nixos.org/download 页面给出的对应平台命令
# 多用户安装需要 sudo；macOS 会创建合成卷承载 /nix
```

装完验证（新开一个 shell 让 profile 生效）：

```console
$ nix --version
nix (Nix) 2.35.x
$ nix shell nixpkgs#hello -c hello
Hello, world!
```

需要知道的边界：

- 这条路径上**没有 configuration.nix**：装包用 `nix profile install`（第 18 章）或直接 `nix shell`/`nix run` 临时使用，环境用 flakes 的 devShells（第 21 章）——这些足够支撑第 6–21 章的全部练习；
- 卸载：Determinate 安装器带卸载命令；官方脚本的多用户安装按官方文档的 uninstall 小节执行（删 `/nix`、`/etc/nix`、profile 与 systemd 服务）；
- macOS 注意：Nix 包管理器与 NixOS 模块系统是两回事，本书第四部分（NixOS）的内容在 macOS 上无从练起——那是路径 B/C 的事。

## E.3 路径 B：虚拟机里体验 NixOS

1. **下载 ISO**：nixos.org/download 选 「NixOS 26.05: GNOME 或 minimal ISO」，x86_64-linux，约 1 GB 量级。GNOME 版自带桌面（练路径 C 前先看两眼），minimal 版只有命令行（更贴近本书语境，资源占用小）；
2. **建虚拟机**：GNOME Boxes / Virt-Manager / VMware / VirtualBox 均可，给 2 核 / 4 GB 内存 / 20 GB 磁盘起；UEFI 固件（OVMF）更贴近现代裸机，模拟路径 C 的实际体验；
3. **进系统**：ISO 启动后自动以 `nixos` 用户登录（minimal 版是 tty，GNOME 版自动进桌面开终端），此后与路径 C 完全一致（E.4 第 3 步起）；
4. **动手前先拍快照**：装完系统拍一张，之后任何实验搞坏了，回快照重来——虚拟机快照（整机粗粒度）与 NixOS generation（细粒度、秒回）配合使用（第 18 章讲过两者粒度差异）。

> 另一条零下载路径：已有 NixOS 的机器上 `nixos-rebuild build-vm --flake .#myhost`（第 41/43 章用过）可以把你自己的配置变成一台 QEMU 虚拟机试跑，改配置先在 VM 里验证是 deploy 前的标准动作（第 31 章）。

## E.4 路径 C：裸机安装 NixOS

以下为 UEFI + systemd-boot + ext4 的最小手动流程（NixOS 没有图形化「下一步下一步」安装器——这本身是哲学的一部分：安装就是写一份配置，而这份配置从此跟你走，第 23/24 章）。磁盘操作会清掉目标盘全部数据，先确认设备名再动手。

**第 1 步：制作安装介质**

```console
# 从 nixos.org/download 下载 ISO 并核对页面给出的 SHA256 校验和后：
$ sudo dd if=nixos-26.05-xxx.iso of=/dev/sdX bs=4M status=progress oflag=sync   # Linux
# Windows 用 Rufus（DD 模式）；macOS 与 Linux 相同（目标为 /dev/rdiskX）
```

BIOS 里关 Secure Boot（NixOS 默认引导链不带微软签名；开启 Secure Boot 的方案见延伸阅读 lanzaboote）并从 U 盘的 UEFI 项启动。

**第 2 步：分区并挂载**

```console
$ sudo -i
# 分区（目标盘 /dev/sda，按实际替换）：512M EFI + 剩余根分区
# parted /dev/sda -- mklabel gpt
# parted /dev/sda -- mkpart ESP fat32 1MB 513MB
# parted /dev/sda -- set 1 esp on
# parted /dev/sda -- mkpart primary 513MB 100%
# 格式化
# mkfs.fat -F 32 /dev/sda1
# mkfs.ext4 -L nixos /dev/sda2
# 挂载
# mount /dev/disk/by-label/nixos /mnt
# mkdir -p /mnt/boot && mount /dev/sda1 /mnt/boot
# 可选：加 swap 分区或 swapfile；内存 ≥8G 且用 zram（E.5）可以不加
```

加密盘（LUKS）、btrfs、双系统共存等布局，官方手册 Installation 章有逐条命令；disko 方案（用 Nix 声明分区表，第 31 章）适合装完后再重构。

**第 3 步：连网并生成配置**

```console
# minimal ISO 自带 NetworkManager：有线通常已自动连上，Wi-Fi：
# nmtui    # 或 nmcli device wifi connect "SSID" password "密码"
# 生成骨架配置
# nixos-generate-config --root /mnt
#    → /mnt/etc/nixos/configuration.nix
#    → /mnt/etc/nixos/hardware-configuration.nix（自动识别刚挂的文件系统，勿手改）
```

**第 4 步：编辑 configuration.nix**

在 `/mnt/etc/nixos/configuration.nix` 里确认/修改以下核心项（完整逐行精讲见第 24 章）：

```nix
{ config, pkgs, ... }: {
  # 引导（UEFI 标准两行）
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "mynixos";
  networking.networkmanager.enable = true;   # 桌面/笔记本连网管理

  time.timeZone = "Asia/Shanghai";
  i18n.defaultLocale = "zh_CN.UTF-8";

  # 你的用户（务必设：安装器默认只有 root 的临时密码）
  users.users.alice = {
    isNormalUser = true;
    extraGroups = [ "wheel" "networkmanager" ];   # wheel = sudo 权限
    initialPassword = "changeme";                 # 首次登录后立即 passwd 改掉
  };

  services.openssh.enable = true;   # 远程/救援通道，建议开

  environment.systemPackages = with pkgs; [
    git vim wget curl
  ];

  # 「这套配置生成时的 NixOS 版本」——首次安装写什么就永远保持什么，别手动改它：
  # 它标记的是配置格式基线而非当前版本，升级流程不会也不应动它（第 30 章有专论）
  system.stateVersion = "26.05";
}
```

**第 5 步：安装、重启**

```console
# nixos-install            # 求值+构建+安装引导，结束时提示设置 root 密码
# reboot                   # 拔 U 盘，从硬盘引导进你自己的系统
```

重启后用 `alice` / `changeme` 登录（GNOME ISO 用户则自动进桌面），第一件事 `passwd` 改密码。

## E.5 装完第一小时

按顺序过一遍，这台机器就从「能开机」变成「能日用」：

**1. 配置入库。** NixOS 的一切改动都该经 git（第 24 章的灾难恢复论证）：

```console
$ cd /etc/nixos && sudo git init && sudo git add -A
$ sudo git commit -m "baseline"
```

**2. 桌面与显卡（按需）。** 桌面环境与 GPU 是手册单列的两大主题，最小可用配置：

```nix
# 桌面二选一：GNOME（开箱即用）或最小化窗口管理器
services.xserver.enable = true;
services.displayManager.gdm.enable = true;
services.desktopManager.gnome.enable = true;

# 显卡：Intel/AMD 核显开箱即用（默认 modesetting 驱动 + 硬件加速默认开启）；
# NVIDIA 闭源驱动需要三件套：
nixpkgs.config.allowUnfree = true;               # 闭源驱动属 unfree
services.xserver.videoDrivers = [ "nvidia" ];    # 选定驱动，模块随之加载
hardware.graphics.enable = true;                 # OpenGL/Vulkan 用户态（24.05 前叫 hardware.opengl）
# 笔记本混合显卡再考虑 offload/prime 相关选项，以官方手册 GPU acceleration 小节为准
```

装好后 `nixos-rebuild test` 先试（第 24 章讲过 test 与 switch 的差异：test 不生成引导项，坏了下次重启自动回到老配置），确认桌面能进再 `switch`。

**3. 日常改配置的固定节奏。** 从此这台机器的所有变更都是「改 configuration.nix → `sudo nixos-rebuild switch` → 出问题 `sudo nixos-rebuild switch --rollback`（或启动菜单选上一代，第 27/28 章）」。第一次亲手走一遍 rollback：改个错配置故意让它失败，看看回滚有多省心——这是建立安全感的最好练习。

**4. 更新与升级。** 两条命令别混：

```console
# channel 世界：更新 channel + 重建
$ sudo nixos-rebuild switch --upgrade
# flake 世界：更新 flake.lock 再重建（推荐，第 21 章）
$ nix flake update && sudo nixos-rebuild switch --flake /etc/nixos
```

跨大版本升级（26.05 → 26.11）= 换 channel 指针或 flake 输入的分支名，步骤见官方手册 Upgrading NixOS 小节；升级前先把 `/nix` 清一清（第 19 章）。

**5. 迁移存量软件习惯。** 把手动装的软件逐个转为声明（第 18 章的自检清单），用户级配置交给 home-manager（第 45 章）。

## E.6 安装期常见问题速查

- **Wi-Fi 列表为空/搜不到卡**：多为部分无线固件属 unfree，在 configuration.nix 加 `nixpkgs.config.allowUnfree = true;` 后重建；个别老卡需要额外固件包，以官方手册 Wireless 小节为准；
- **安装器里浏览器卡顿/编译报内存不足**：机器内存小又没有 swap，先加 swapfile 或 zram（`zramSwap.enable = true;`），或改用 minimal ISO；
- **装完和 Windows 双系统时间差 8 小时**：Windows 默认硬件时钟为本地时间而 NixOS 为 UTC，NixOS 侧加 `time.hardwareClockInLocalTime = true;`；
- **minimal 版启动后没有图形界面**：这是特性不是故障——minimal ISO 本就不带 X，按 E.5 第 2 步声明桌面即可；
- **其他卡壳**：第 46 章排错手册 + 官方手册 Troubleshooting 章（Boot Problems、Maintenance Mode、Rolling Back 各小节）。

## E.7 本附录小结

- 三条路径按投入递进：现有系统装 Nix（够练语言与 flakes）→ 虚拟机装 NixOS（学习主力，快照+双回滚兜底）→ 裸机日用；
- NixOS 的「安装器」就是 configuration.nix：nixos-generate-config 生成骨架，你补上引导、用户、时区、包清单，nixos-install 收尾——从第一分钟起就在用全书讲的方式管理这台机器；
- `system.stateVersion` 记录配置基线、永不手改；改动走 test → switch → rollback 的固定节奏，配置文件永远在 git 里；
- 桌面用 GNOME 起步，显卡区分核显（开箱即用）与 NVIDIA（allowUnfree + videoDrivers + hardware.graphics）两条线；
- 更新（`nix flake update` / `--upgrade`）与回滚（`--rollback` / 启动菜单选代）是日常两大动作，装完第一小时就该各演练一遍。

## 延伸阅读

- 官方手册 Installation 章节（本附录路径 C 的权威来源，含 LUKS/btrfs/双系统等布局）—— https://nixos.org/manual/nixos/stable/#ch-installation
- 官方手册 Getting started（装完后的第一段路）—— https://nixos.org/manual/nixos/stable/#ch-getting-started
- nixos-anywhere：SSH 直装远程机器（本书第 31 章有部署语境的讲解）—— https://github.com/nix-community/nixos-anywhere
- lanzaboote：NixOS 的 Secure Boot 支持方案 —— https://github.com/nix-community/lanzaboote


---

[← 上一章：附录 D：学习资源索引](appendix-d-resources.md) · [↑ 返回目录](../README.md)
