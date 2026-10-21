# J6412 四核工控机移植 Xenomai 实时补丁操作手册（I210，暂不含 EtherCAT）

本文面向 Intel J6412 四核 x86_64 工控机，网卡为 Intel I210，以 Debian 12 amd64 原生编译为参考环境，建立 Xenomai 3.3.1 Cobalt 实时环境。I210 在本阶段使用 Linux 原生 igb 驱动；暂不安装 EtherCAT 主站或 RTnet。其他发行版的软件包名、内核安装钩子和引导流程需要重新核对，不能直接视为已验证。

本文参考博客《xenomai3+linux构建linux实时操作系统-基于X86_64和arm》：
https://www.cnblogs.com/wsg1100/p/12864199.html

博客的可迁移原则是：

1. 本手册选用 Linux 6.1 系列的 Dovetail + Cobalt 路径；博客的旧 I-pipe 示例不适用于这里。
2. Xenomai 内核部分由 Cobalt 组成，用户空间部分由 libcobalt 和测试工具组成。
3. 内核、Dovetail、Xenomai 必须匹配，不能把不同内核版本的补丁强行套用。
4. 编译完成后先验证 Cobalt 启动，再做 latency 和 xeno-test 压力测试。

博客中的 4.19/I-pipe、5.13/Dovetail 版本只是示例，作者也声明不保证直接用于生产。本手册不把这些示例版本当作 J6412 的可用版本。

本文保留博客的“中断流水线 → Cobalt 内核 → 匹配的用户库 → latency/xeno-test”顺序；将旧版本示例、make-kpkg 和混用路径替换成 6.1 Dovetail 源码、Deb 包构建及独立构建目录。博客中其他 CPU 的测试数字不能作为 J6412 的验收结果。

本文是待目标机执行的操作规程，尚未在该工控机上完成内核构建或性能测试。后文的“通过条件”用于现场判定，验收表初始状态均为待验收。

**已有源码包的取用入口是第 4.2 节；新手仍应从阶段 0 开始。** 工作区中的 [Xenomai_Source_Packages](Xenomai_Source_Packages/源码包说明.md) 已提供 Linux/Dovetail 和 Xenomai 两个 tar.gz 快照及 SHA256SUMS。目标 Debian 机不能直接读取 Windows 工作区；先用 U 盘、局域网或其他受控介质把整个目录复制到目标机，再按第 4.2 节校验。压缩包是 Git 快照，不带 `.git` 目录，因此从压缩包解压后不要执行 `git status`、`git log` 或 `git rev-parse`；版本依据使用 `SHA256SUMS`、`SOURCE_MANIFEST.json` 和保存的提交号文件。第 4.2 节完成后直接继续第 5 节。

操作约定：所有 bash 代码块都在目标 Debian 的 Bash 中执行，以普通用户编译，只在安装/系统配置时使用 sudo；不是 Windows PowerShell 命令。按顺序逐块执行并检查退出码，失败即停。多终端或重启后需重新进入相应目录并恢复变量；网卡名与对端 IP 必须使用现场值。先做阶段 0-9 基线，再做阶段 10 调优，最后执行 12.4 的长时间验收。

## 1. 目标版本和兼容性原则

推荐先采用下面的版本策略：

| 项目 | 建议 |
|---|---|
| CPU/架构 | Intel J6412，x86_64，4 个物理核 |
| 实时核 | Xenomai 3.3.1 Cobalt |
| Linux 接口层 | Dovetail，选择与 6.1 内核精确匹配的分支或补丁 |
| Linux 内核 | 官方 linux-dovetail 仓库的固定标签 v6.1.170-cip55-dovetail1；下载后校验提交号 |
| 网卡 | Linux 原生 igb，I210 对应 CONFIG_IGB；不使用 EtherCAT 专用驱动 |
| 构建方式 | Debian 12 amd64 原生编译，内核并行度先固定为 -j4 |
| 测试目标 | 先验证 1 ms，再验证 125 us；125 us 周期是否满足应用要求必须由实际负载和最坏延迟数据决定 |

Xenomai 3.3.1 官方安装文档：

- https://source.denx.de/Xenomai/xenomai/-/blob/v3.3.1/doc/asciidoc/README.INSTALL.adoc
- https://source.denx.de/Xenomai/xenomai/-/blob/v3.3.1/scripts/prepare-kernel.sh
- Dovetail 内核仓库：https://source.denx.de/Xenomai/linux-dovetail
- Xenomai 3.3.1 镜像（官方仓库的 CI 镜像，仅供官方仓库无法访问时比对提交）：https://github.com/xenomai-ci/xenomai/tree/v3.3.1

Xenomai 3.3.1 的源码标签可以用 Git 仓库校验：

~~~bash
git ls-remote --tags https://source.denx.de/Xenomai/xenomai.git \
  'refs/tags/v3.3.1' 'refs/tags/v3.3.1^{}'
~~~

2026-10-20 已通过官方 Git 查询核对以下源码入口；现场下载后仍须校验，任何不一致都先停止并调查：

| 官方资源 | 固定标签 | 实际源码提交 SHA |
|---|---|---|
| https://source.denx.de/Xenomai/linux-dovetail.git | v6.1.170-cip55-dovetail1 | 6062891c48355c4295db5e33d47341a3770dde38 |
| https://source.denx.de/Xenomai/xenomai.git | v3.3.1 | 2f15f8c5970b23428a00ccd17249de26a240dc33 |

Xenomai 的 v3.3.1 是附注标签：ls-remote 中标签对象为 48c5d6641e54f198542252612eea5805a1133e28，带 ^{} 的记录才是上表的源码提交。核对 HEAD 时使用源码提交，不能误用标签对象。Git SHA 用于固定版本，不能代替目标硬件兼容性验证。

Xenomai 3.3.1 官方安装文档将 Cobalt 描述为与 Linux 集成的双内核实时扩展，并要求通过 Dovetail 接口层准备内核；文档还说明 Cobalt 需要 5.10 或更高版本内核。因此本手册不采用博客中的 4.19/I-pipe 示例作为 J6412 6.1 构建方案。

### 1.1 不要混用以下路径

以下组合不能直接混用：

- Linux 6.1 + 4.19 的 I-pipe 补丁；
- Linux 6.1 + 5.13 的 Dovetail 补丁；
- Xenomai 3.3.1 用户空间库 + 另一套 Cobalt 内核；
- PREEMPT_RT 内核和 Cobalt 双内核作为同一个实时核心；
- I210 的 igb 驱动和 EtherCAT 专用 ec_igb 驱动；
- v6.1.y-dovetail、v6.1.y-cip-dovetail、厂商 BSP 内核之间未经验证的任意拼接。

本手册已确认 6.1.170-cip55 的官方 Dovetail 源码标签入口，后续将接入同一份 Xenomai 3.3.1 源码中的 Cobalt。此前提到的完整名称 Linux 6.1.170-cip55-xenomai-3.3.1-j6412-rtipc 仍不能据此认定为官方发布的成品包；其中 j6412/rtipc 可能是项目自定义标识。本文使用自己的本地版本后缀，不把文件名当作兼容性证明。

## 2. 阶段 0：确定工控机和恢复条件

### 2.1 进入本地控制台

第一次编译和安装不要只依赖 SSH。准备以下条件：

- 本地显示器和键盘，或串口/带外控制台；
- 能进入 GRUB 高级选项；
- 能从旧内核启动；
- 能备份 /boot、/etc/default/grub、当前内核配置；
- 有一份可启动的系统盘或镜像。

### 2.2 记录当前状态

~~~bash
mkdir -p ~/xenomai-migration-record
date -Is | tee ~/xenomai-migration-record/start-time.txt
uname -a | tee ~/xenomai-migration-record/uname-before.txt
cat /etc/os-release | tee ~/xenomai-migration-record/os-release.txt
lscpu | tee ~/xenomai-migration-record/lscpu.txt
lspci -nnk | tee ~/xenomai-migration-record/lspci-before.txt
ip -br link | tee ~/xenomai-migration-record/ip-link-before.txt
sudo cp -a /boot/config-$(uname -r) ~/xenomai-migration-record/
sudo cp -a /etc/default/grub ~/xenomai-migration-record/grub-before
~~~

检查：

~~~bash
test "$(uname -m)" = x86_64 && echo PASS || echo FAIL
lscpu | grep -E 'CPU\(s\)|Model name|Core|Thread'
lspci -nnk | grep -A3 -i ethernet
~~~

通过条件：

- uname -m 为 x86_64；
- lscpu 显示 4 个在线 CPU（0-3）；若数量不符，先修正 BIOS/启动配置，再进行本文的 CPU3 测试；
- I210 的 PCI 设备存在；
- lspci -nnk 显示当前驱动为 igb，或者已记录待安装的驱动状态。

### 2.3 备份和回滚验收

记录当前 GRUB 内核：

~~~bash
grep -E '^menuentry |^submenu ' /boot/grub/grub.cfg | head -30
ls -lh /boot/vmlinuz-* /boot/initrd.img-* 2>/dev/null
sha256sum /boot/vmlinuz-$(uname -r) /boot/config-$(uname -r) 2>/dev/null \
  | tee ~/xenomai-migration-record/boot-hash-before.txt
sudo tar -C / -czf ~/xenomai-migration-record/boot-grub-before.tar.gz \
  boot etc/default/grub
tar -tzf ~/xenomai-migration-record/boot-grub-before.tar.gz | head
~~~

阶段 0 通过条件：

- 至少有一个旧内核和对应 initrd；
- 已确认 GRUB 可以选择旧内核；
- boot-hash-before.txt 已保存；
- 在没有本地恢复通道时，不得继续安装新内核。

## 3. 阶段 1：准备编译环境

以下命令按 Debian 12 编写。先确认系统为 Debian 12 amd64；不符合时先核对包名、GRUB 布局和内核安装钩子。

~~~bash
sudo apt update
sudo apt install -y \
  build-essential bc bison flex libssl-dev libelf-dev \
  libncurses-dev pahole fakeroot dpkg-dev \
  debhelper rsync cpio kmod grub2-common initramfs-tools \
  git wget curl ca-certificates pkg-config \
  autoconf automake libtool libltdl-dev \
  python3 python3-pip ethtool pciutils stress-ng iperf3 \
  iproute2 numactl rt-tests lm-sensors mokutil dmidecode
~~~

验证工具：

~~~bash
for x in gcc make git bc bison flex dpkg-deb ethtool stress-ng; do
    command -v "$x" >/dev/null && echo "PASS $x" || echo "FAIL $x"
done
gcc --version | head -1
make --version | head -1
~~~

阶段 1 通过条件：

- 所有必需命令均输出 PASS；
- 编译主机有至少 20 GB 可用空间，建议 40 GB；
- /boot 有足够空间存放新内核、模块和 initrd：

~~~bash
df -h / /boot
sudo dmidecode -t bios | tee ~/xenomai-migration-record/bios-before.txt
free -h
~~~

建议内存至少 4 GB、优先 8 GB；完整发行版配置含调试信息时可能需要更多构建空间，实际以 df 和构建日志为准。首次部署前在 BIOS 中确认全部 4 核启用、记录固件版本和电源策略，不提前修改所有性能选项。

确认 Secure Boot 状态：mokutil --sb-state。若为 enabled，未签名的自编译内核可能无法由固件加载；应按现场的签名/MOK 流程处理，或在获得设备所有者许可后从固件设置中关闭 Secure Boot。不要通过删除原有引导项来绕过此检查。把状态和处理方式写入验收记录。

## 4. 阶段 2：取得内核、Dovetail 和 Xenomai 源码

### 4.1 建立独立工作目录

~~~bash
mkdir -p ~/src/xenomai-j6412
cd ~/src/xenomai-j6412
~~~

不要在发行版正在使用的内核源码目录上直接修改。每次尝试使用新的目录，并保存版本记录。

在 Debian 本地 ext4 等 Linux 文件系统上解压和编译，保留可执行权限和符号链接。源码及构建目录的绝对路径不要包含空格或中文；本版本的接入脚本有未引用的路径操作，可使用 `/home/operator/src/xenomai-j6412` 这样的全英文路径（operator 换成实际用户名）。Windows 和 U 盘只用于传输压缩包。

### 4.2 首选路径：使用已下载的源码包

在目标 Debian 机上先找到复制过来的 `Xenomai_Source_Packages` 目录。下面的 `PACKAGE_DIR` 只是示例，必须替换成现场的绝对路径；路径中含空格时保留双引号。不要把 Windows 路径（例如 `D:\\ProjectCx`）直接填入 Bash。

如果通过 SSH 从 Windows 复制，可在 Windows PowerShell 中执行下面的示例；把用户名、目标地址和目标目录换成现场值。也可以把整个目录复制到 U 盘后挂载到 Debian，复制完成后再从 Debian 终端执行校验。不要在 Windows 上运行后面的 `sha256sum`、`tar` 或内核编译命令。

~~~powershell
scp -r .\Xenomai_Source_Packages\ user@192.0.2.10:/home/user/
~~~

若目录复制到了当前用户主目录，可在 Debian 上用下面命令确认文件名和大小；挂载在 U 盘或其他位置时改成该实际路径：

~~~bash
ls -lh "$HOME/Xenomai_Source_Packages"
~~~

~~~bash
set -o pipefail
read -r -p '请输入源码包目录的绝对路径: ' PACKAGE_DIR
test -d "$PACKAGE_DIR" || exit 1
mkdir -p "$HOME/xenomai-migration-record"
printf '%s\n' "$PACKAGE_DIR" > "$HOME/xenomai-migration-record/package-dir.txt"
cd "$PACKAGE_DIR" || exit 1
test -r SHA256SUMS || exit 1
test -r SOURCE_MANIFEST.json || exit 1
test -r linux-dovetail-v6.1.170-cip55-dovetail1.tar.gz || exit 1
test -r xenomai-v3.3.1.tar.gz || exit 1
sha256sum -c SHA256SUMS || exit 1
~~~

必须看到两个 `OK`。任一校验失败都要停止，不要解压或编译。把校验结果保存到迁移记录：

~~~bash
PACKAGE_DIR=$(cat "$HOME/xenomai-migration-record/package-dir.txt")
cd "$PACKAGE_DIR" || exit 1
sha256sum -c SHA256SUMS | tee "$HOME/xenomai-migration-record/source-sha256-check.txt" || exit 1
cp -a SHA256SUMS SOURCE_MANIFEST.json "$HOME/xenomai-migration-record/" || exit 1
~~~

确认工作目录为空后解压。`test ! -e` 是为了避免把一次失败的尝试和新版本混在一起；需要重做时先保留日志，再由人工删除整个旧目录并重新执行。

~~~bash
PACKAGE_DIR=$(cat "$HOME/xenomai-migration-record/package-dir.txt")
cd "$PACKAGE_DIR" || exit 1
mkdir -p "$HOME/src/xenomai-j6412"
test ! -e "$HOME/src/xenomai-j6412/linux-dovetail" || { echo 'linux-dovetail 已存在，换新目录或先人工清理'; exit 1; }
test ! -e "$HOME/src/xenomai-j6412/xenomai-3.3.1" || { echo 'xenomai-3.3.1 已存在，换新目录或先人工清理'; exit 1; }
test ! -e "$HOME/src/xenomai-j6412/linux-dovetail-v6.1.170-cip55-dovetail1" || exit 1
test ! -e "$HOME/src/xenomai-j6412/xenomai-v3.3.1" || exit 1
tar -xzf linux-dovetail-v6.1.170-cip55-dovetail1.tar.gz -C "$HOME/src/xenomai-j6412" || exit 1
tar -xzf xenomai-v3.3.1.tar.gz -C "$HOME/src/xenomai-j6412" || exit 1
mv "$HOME/src/xenomai-j6412/linux-dovetail-v6.1.170-cip55-dovetail1" "$HOME/src/xenomai-j6412/linux-dovetail" || exit 1
mv "$HOME/src/xenomai-j6412/xenomai-v3.3.1" "$HOME/src/xenomai-j6412/xenomai-3.3.1" || exit 1
cd "$HOME/src/xenomai-j6412" || exit 1
printf '%s\n' '6062891c48355c4295db5e33d47341a3770dde38' > linux-dovetail-commit.txt
printf '%s\n' '2f15f8c5970b23428a00ccd17249de26a240dc33' > xenomai-commit.txt
~~~

压缩包没有 Git 元数据，使用下面的文件和源码特征完成离线验收：

~~~bash
cd "$HOME/src/xenomai-j6412"
test ! -e linux-dovetail/.git || exit 1
test ! -e xenomai-3.3.1/.git || exit 1
test -r linux-dovetail/include/linux/dovetail.h || exit 1
test -r linux-dovetail/arch/x86/include/asm/dovetail.h || exit 1
test -x xenomai-3.3.1/scripts/prepare-kernel.sh || exit 1
make -s -C linux-dovetail kernelversion | tee linux-kernelversion.txt || exit 1
test "$(cat linux-kernelversion.txt)" = 6.1.170 || exit 1
test "$(cat xenomai-3.3.1/config/version-label)" = 3.3.1 || exit 1
xenomai-3.3.1/scripts/prepare-kernel.sh --help
grep -E '^(VERSION|PATCHLEVEL|SUBLEVEL|EXTRAVERSION) =' linux-dovetail/Makefile
cat linux-dovetail-commit.txt xenomai-commit.txt
grep -F '"commit": "6062891c48355c4295db5e33d47341a3770dde38"' "$HOME/xenomai-migration-record/SOURCE_MANIFEST.json" || exit 1
grep -F '"commit": "2f15f8c5970b23428a00ccd17249de26a240dc33"' "$HOME/xenomai-migration-record/SOURCE_MANIFEST.json" || exit 1
~~~

通过条件：SHA256SUMS 两项均为 `OK`；两个目录存在且不包含 `.git`；Dovetail 头文件、`prepare-kernel.sh` 均存在；`kernelversion` 为 6.1.170（后缀以现场输出为准）；两个提交号已写入记录。`SOURCE_MANIFEST.json` 中的提交号和 SHA256 必须与记录一致。归档快照没有可供 `git status` 比较的基线，不能把“没有 Git 状态”误判为“没有修改”；接入 Cobalt 前后应保留源码目录副本或用 `diff -qr` 记录变化。

保存来源记录的校验值；第 5.1 节会统一采集接入前文件清单：

~~~bash
cd "$HOME/src/xenomai-j6412"
sha256sum linux-dovetail-commit.txt xenomai-commit.txt linux-kernelversion.txt \
  | tee source-record-sha256.txt
~~~

不要在 `linux-dovetail` 目录内手工复制 Xenomai 子目录，也不要把 PREEMPT_RT 或 4.19 I-pipe 补丁叠加到这棵已含 Dovetail 的树上。完成本节后直接继续第 5 节；第 5.2 节的 Git 检查在归档路径按“文件清单/差异”方式执行。

### 4.3 联网替代路径：固定官方 Dovetail 标签

已执行 4.2 的读者跳过 4.3—4.5，直接进入第 5 节。仅当不用源码包、需要保留 Git 历史时，才在空工作目录中执行本节和 4.5。两条路径二选一，不能在已解压目录中重复 clone。

直接使用官方已经集成 Dovetail 的 Linux 树，避免额外拼接 Linux 与流水线补丁。先查询本次指定的标签：

~~~bash
git ls-remote --tags https://source.denx.de/Xenomai/linux-dovetail.git \
  'refs/tags/v6.1.170-cip55-dovetail1' \
  'refs/tags/v6.1.170-cip55-dovetail1^{}'
~~~

应得到上表所列源码提交。随后只下载这一固定标签：

~~~bash
git clone --branch v6.1.170-cip55-dovetail1 --single-branch --depth 1 \
  https://source.denx.de/Xenomai/linux-dovetail.git linux-dovetail
cd linux-dovetail
git rev-parse HEAD | tee ../linux-dovetail-commit.txt
test "$(git rev-parse HEAD)" = 6062891c48355c4295db5e33d47341a3770dde38 || exit 1
git status --porcelain | tee ../linux-dovetail-status-before.txt
test ! -s ../linux-dovetail-status-before.txt || exit 1
test -r include/linux/dovetail.h || exit 1
test -r arch/x86/include/asm/dovetail.h || exit 1
git log --oneline --decorate -10
make -s kernelversion | tee ../linux-kernelversion.txt
grep -E '^(VERSION|PATCHLEVEL|SUBLEVEL|EXTRAVERSION) =' Makefile
git ls-files '*localversion*'
for f in localversion*; do
  test -f "$f" && printf '%s: %s\n' "$f" "$(cat "$f")"
done
cd ..
~~~

验收：两个 Dovetail 头文件存在、工作树状态为空、HEAD 与指定 SHA 一致；Makefile 的 VERSION/PATCHLEVEL/SUBLEVEL 分别为 6/1/170。已核对该标签 Makefile 的 EXTRAVERSION 为空；CIP/Dovetail 后缀还可能由 localversion 文件参与生成，因此以阶段 4 的实际 kernelrelease 为准。保存全部结果。若仓库访问失败，先恢复官方仓库访问；不要在没有来源证明的镜像上继续。

v6.1.y-cip-dovetail 是会更新的维护分支，不作为本手册的部署基线。v6.1.170-cip55-dovetail1-rebase 是另一个标签，也不要在同一批设备中混用。确认标签和源码来源，仅表示具备可复建的输入；仍须完成接入、编译、启动和实时验收。

### 4.4 必须使用独立 CIP/BSP 树时

先取得该内核的精确提交和匹配的 Dovetail 补丁，记录两者来源及 SHA256；在干净树中执行 git apply --check 或 patch --dry-run -p1。只凭文件名含有 6.1/cip55 不足以证明匹配。检查失败就停止，不使用强制三方合并。本手册后续可执行命令均以 4.2 的预集成树为准；采用独立树时需要单独形成并验证版本清单。

### 4.5 取得 Xenomai 3.3.1（联网可选）

~~~bash
cd ~/src/xenomai-j6412
git clone --branch v3.3.1 --depth 1 \
  https://source.denx.de/Xenomai/xenomai.git xenomai-3.3.1
cd xenomai-3.3.1
git rev-parse HEAD | tee ../xenomai-commit.txt
test "$(git rev-parse HEAD)" = 2f15f8c5970b23428a00ccd17249de26a240dc33 || exit 1
git status --short
cd ..
~~~

若官方 Git 服务不可达，可从上文 CI 镜像取得同一标签；须验证提交号与上述值一致，并记录镜像 URL。若两处均无法核对，停止。

验证 Xenomai 脚本存在：

~~~bash
test -x xenomai-3.3.1/scripts/prepare-kernel.sh \
  && echo "PASS prepare-kernel.sh" || echo "FAIL prepare-kernel.sh"
xenomai-3.3.1/scripts/prepare-kernel.sh --help | head -40
~~~

阶段 2 通过条件（源码包和 Git 两条路径都适用）：

- 源码包路径的 SHA256 两项均为 OK，SHA256SUMS 和 SOURCE_MANIFEST.json 已归档；
- Linux 提交号、内核版本、Dovetail 分支/补丁、Xenomai 提交号均已保存；
- 目标内核源码树没有未记录的修改；
- prepare-kernel.sh --help 能正常执行；
- 版本不一致时已经停止，而不是进入编译阶段。

## 5. 阶段 3：接入 Dovetail 和 Cobalt

### 5.1 将 Cobalt 接入预集成 Dovetail 树

先确认内核树已含 Dovetail，再运行官方脚本。脚本默认在内核树中建立指向 Xenomai 源码的链接；后续编译期间必须保留两棵源码树及其目录位置。

~~~bash
cd ~/src/xenomai-j6412 || exit 1
set -o pipefail
find linux-dovetail -path '*/.git' -prune -o \( -type f -o -type l \) -print \
  | sort > linux-files-before-prepare.txt || exit 1
( cd linux-dovetail && sha256sum init/Kconfig arch/x86/Makefile drivers/Makefile kernel/Makefile ) \
  > kernel-integration-files-before.sha256 || exit 1
cd xenomai-3.3.1 || exit 1
test -r ../linux-dovetail/include/linux/dovetail.h || exit 1
set -o pipefail
./scripts/prepare-kernel.sh \
  --linux=../linux-dovetail \
  --arch=x86 --verbose 2>&1 | tee ../prepare-kernel.log || exit 1
~~~

执行前先运行 set -o pipefail，使 tee 不掩盖脚本退出码。若脚本报告缺少 IRQ pipeline，停止并修正源码。独立 CIP 树需要经干净树预检查的 --dovetail 补丁，属于另一条版本链。

### 5.2 接入后检查

~~~bash
cd ~/src/xenomai-j6412/linux-dovetail
test -r kernel/xenomai/Kconfig || exit 1
test -r arch/x86/xenomai/Kconfig || exit 1
test -r drivers/xenomai/Kconfig || exit 1
grep -n 'menuconfig XENOMAI' init/Kconfig || exit 1
grep -n 'CONFIG_XENOMAI' kernel/Makefile arch/x86/Makefile
sha256sum init/Kconfig arch/x86/Makefile drivers/Makefile kernel/Makefile \
  > ../kernel-integration-files-after.sha256
find kernel/xenomai arch/x86/xenomai drivers/xenomai include/xenomai \
  -xtype l -print > ../broken-xenomai-links.txt
test ! -s ../broken-xenomai-links.txt || exit 1
cd .. || exit 1
find linux-dovetail -path '*/.git' -prune -o \( -type f -o -type l \) -print \
  | sort > linux-files-after-prepare.txt
diff -u linux-files-before-prepare.txt linux-files-after-prepare.txt > linux-files-prepare-diff.txt
DIFF_RC=$?
test "$DIFF_RC" -le 1 || exit 1
sed -n '1,80p' linux-files-prepare-diff.txt
~~~

归档路径没有 `.git`，因此不能运行 `git status`。`diff` 的退出码为 1 表示文件清单有差异，这是预期结果；退出码为 2 则应停止排查。文件清单仅显示文件名变化，四个接入入口的内容校验值也已分别保存。Git 路径可额外在 `linux-dovetail` 内运行 `git status --short`，并确认只出现预期的 Xenomai 链接和配置文件。

如需可分发补丁，可另在相同干净内核树上使用 `prepare-kernel.sh --outpatch=绝对路径` 生成，再验证重放。本手册使用默认的源码链接方式；编译期间不要移动 `linux-dovetail` 或 `xenomai-3.3.1` 目录。

阶段 3 通过条件：

- kernel/xenomai 等目标文件可读，init/Kconfig 出现 Xenomai 菜单；
- 归档路径的接入前后文件清单差异已保存，Git 路径的 `git status` 只包含预期的 Xenomai/Dovetail 修改；
- 没有 patch failed、Hunk FAILED、未解决冲突；
- make olddefconfig 尚未运行前，先保存当前源码提交号和修改清单。


### 5.3 选做：导出可归档的 Cobalt 补丁文件

默认链接方式已足够支持本机编译。如果交付物还需要一份真正的 `.patch` 文件，可在另一棵干净的、同版本 Dovetail 源码副本上导出并验证。这个补丁只包含 Xenomai Cobalt 的接入，适用基线严格限定为本手册的 `v6.1.170-cip55-dovetail1`，其中的 Dovetail 已包含在原始 Linux 包中。

下面使用已校验的 Linux 压缩包重新展开副本，需要额外的源码空间。不要对已运行过默认接入脚本的主源码树导出补丁；否则新增内容会被当作已存在，补丁可能不完整。

~~~bash
PACKAGE_DIR=$(cat "$HOME/xenomai-migration-record/package-dir.txt")
PATCH_WORK="$HOME/src/xenomai-cobalt-export"
test ! -e "$PATCH_WORK" || { echo '导出目录已存在，请先归档或换新目录'; exit 1; }
mkdir -p "$PATCH_WORK" || exit 1
cd "$PACKAGE_DIR" || exit 1
sha256sum -c SHA256SUMS || exit 1
tar -xzf linux-dovetail-v6.1.170-cip55-dovetail1.tar.gz -C "$PATCH_WORK" || exit 1
PATCH_TREE="$PATCH_WORK/linux-dovetail-v6.1.170-cip55-dovetail1"
PATCH_FILE="$PATCH_WORK/xenomai-3.3.1-cobalt-for-v6.1.170-cip55-dovetail1-x86.patch"
set -o pipefail
"$HOME/src/xenomai-j6412/xenomai-3.3.1/scripts/prepare-kernel.sh" \
  --linux="$PATCH_TREE" --arch=x86 --outpatch="$PATCH_FILE" --verbose \
  2>&1 | tee "$PATCH_WORK/export.log" || exit 1
test -s "$PATCH_FILE" || exit 1
cd "$PATCH_TREE" || exit 1
patch --dry-run -p1 < "$PATCH_FILE" || exit 1
patch -p1 < "$PATCH_FILE" || exit 1
test -r kernel/xenomai/Kconfig || exit 1
test -r arch/x86/xenomai/Kconfig || exit 1
test -r drivers/xenomai/Kconfig || exit 1
test ! -L kernel/xenomai/Kconfig || exit 1
grep -n 'menuconfig XENOMAI' init/Kconfig || exit 1
sha256sum "$PATCH_FILE" | tee "$PATCH_WORK/cobalt-patch.sha256"
~~~

通过条件：导出退出码为 0，补丁非空，在全新副本上预检查和实际应用均成功，Cobalt 文件可读且已实体化，补丁的 SHA256 已保存。这个阶段验证补丁能够接入指定源码，不等于通过内核编译与启动验收。

本节产物位于 `~/src/xenomai-cobalt-export`，主流程仍在 `~/src/xenomai-j6412` 中继续。如果以后选用实体化的副本进行可搬迁构建，应从第 6 节开始为它单独使用构建目录和版本记录，完整重做构建、启动和性能验收。

## 6. 阶段 4：配置 J6412 四核内核

### 6.1 复制发行版配置

使用当前可启动内核配置作为基础。.config 只保存在独立构建目录：

~~~bash
cd ~/src/xenomai-j6412/linux-dovetail
mkdir -p ../build-j6412
cp /boot/config-$(uname -r) ../build-j6412/.config
./scripts/config --file ../build-j6412/.config \
  --enable XENOMAI --enable SMP --enable HIGH_RES_TIMERS \
  --enable PROC_FS --enable XENO_OPT_VFILE --enable XENO_OPT_STATS \
  --enable XENO_DRIVERS_SWITCHTEST --enable XENO_DRIVERS_HEAPCHECK \
  --enable XENO_DRIVERS_TIMERBENCH \
  --enable PCI --enable DEVTMPFS --enable DEVTMPFS_MOUNT \
  --module IGB --disable PREEMPT_RT --disable LOCALVERSION_AUTO \
  --disable DEBUG_INFO_BTF \
  --set-str SYSTEM_TRUSTED_KEYS "" \
  --set-str SYSTEM_REVOCATION_KEYS "" \
  --set-str LOCALVERSION "-j6412-xeno331"
make O=../build-j6412 olddefconfig
~~~

发行版配置中若引用不存在的证书文件，构建会因缺少 PEM 失败；上述两个证书字段清空的是证书文件路径，不会自动解决 Secure Boot 的内核签名。暂时关闭 BTF 可避开构建主机缺失 vmlinux/BTF 前置文件，首次编译通过后可单独评估恢复。保留 CONFIG_FUSE_FS（若用户空间启用 registry）、磁盘控制器和根文件系统驱动。

进入配置界面：

~~~bash
make O=../build-j6412 menuconfig
~~~

### 6.2 menuconfig 中的关键检查

在 menuconfig 中确认：

1. Xenomai 菜单存在并启用 Cobalt；CONFIG_XENOMAI=y 会选择 CONFIG_DOVETAIL=y。
2. CONFIG_SMP=y、CONFIG_HIGH_RES_TIMERS=y、CONFIG_XENO_OPT_VFILE=y、CONFIG_XENO_OPT_STATS=y。
3. CONFIG_IGB=m 或 y；x86_64、PCI、定时器、当前磁盘控制器和根文件系统驱动保留。
4. CONFIG_PREEMPT_RT 不启用；CONFIG_LOCALVERSION 仅在 .config 中设置一次。
5. 不启用 EtherCAT 专用网卡驱动，不删除 igb。

保存退出后检查关键符号：

~~~bash
cd ~/src/xenomai-j6412/linux-dovetail
make O=../build-j6412 olddefconfig
grep -E 'CONFIG_(DOVETAIL|XENOMAI|XENO|SMP|HIGH_RES_TIMERS|PREEMPT|IGB|PCI|LOCALVERSION)' \
  ../build-j6412/.config | tee ../config-key-options.txt
for symbol in XENOMAI DOVETAIL SMP HIGH_RES_TIMERS PROC_FS \
  XENO_OPT_VFILE XENO_OPT_STATS PCI DEVTMPFS DEVTMPFS_MOUNT \
  XENO_DRIVERS_SWITCHTEST XENO_DRIVERS_HEAPCHECK XENO_DRIVERS_TIMERBENCH; do
  grep -qx "CONFIG_$symbol=y" ../build-j6412/.config || exit 1
done
grep -Eq '^CONFIG_IGB=(y|m)$' ../build-j6412/.config
! grep -qx 'CONFIG_PREEMPT_RT=y' ../build-j6412/.config
make -s O=../build-j6412 kernelrelease | tee ../planned-kernelrelease.txt
~~~

各项检查退出码必须为 0；若失败，检查该提交的 Kconfig 依赖并重新配置。kernelrelease 应含 -j6412-xeno331 且仅出现一次；保留并记录 CIP 树原有的 EXTRAVERSION。

### 6.3 J6412 的初始实时配置原则

第一次测试不要一开始就做激进裁剪：

- 保留 ACPI 和温度保护；
- 先记录 BIOS 中 Burst、深度 C-state、Package C-state、主动散热策略；J6412 的选项名称由整机 BIOS 决定，不能假定有 Turbo/超线程菜单；
- 关闭 irqbalance 只应在固定 IRQ 亲和性测试期间进行；
- 不要使用 idle=poll 作为生产默认值，它会显著增加功耗和温升；
- 不要一开始加入 isolcpus、nohz_full、rcu_nocbs，先建立未隔离基线；
- 后续每次只改变一个变量，并保存 latency 结果。

阶段 4 通过条件：

- config-key-options.txt 已保存；
- CONFIG_IGB 未被删除；
- Xenomai/Cobalt 和 Dovetail 选项已经进入最终 .config；
- 内核配置与目标硬件存储、PCI、串口/显示、网卡驱动一致。

## 7. 阶段 5：编译和生成可回滚内核包

J6412 是四核，首次构建固定使用 4 个并发任务，避免把 nproc 误用于另一台高核数编译机：

~~~bash
cd ~/src/xenomai-j6412/linux-dovetail
set -o pipefail
make O=../build-j6412 -j4 bindeb-pkg 2>&1 | tee ../kernel-build.log
~~~

退出码应为 0，末尾正常生成 Deb 包。若缺少证书/BTF 报错，回到阶段 4；内存不足时改 -j2 并重新构建，不忽略编译错误。

使用精确的 kernelrelease 选择包，避免把旧版本或调试包一起安装。下面命令在同一个 Bash 会话中依次运行：

~~~bash
cd ~/src/xenomai-j6412
KREL=$(cat planned-kernelrelease.txt)
mapfile -t IMAGES < <(find . -maxdepth 1 -type f -name "linux-image-${KREL}_*.deb")
mapfile -t HEADERS < <(find . -maxdepth 1 -type f -name "linux-headers-${KREL}_*.deb")
test "${#IMAGES[@]}" -eq 1 || exit 1
test "${#HEADERS[@]}" -eq 1 || exit 1
IMAGE=${IMAGES[0]}
HEADER=${HEADERS[0]}
dpkg-deb -f "$IMAGE" Package Version Architecture
dpkg-deb -f "$HEADER" Package Version Architecture
sha256sum "$IMAGE" "$HEADER" | tee kernel-deb.sha256
~~~

核对两个包的 Architecture 均为 amd64，Package 名称和 KREL 对应。安装前确认预期镜像和模块路径：

本文按在目标 J6412 上原生构建并安装编写。默认接入方式创建的 Xenomai 头文件链接，可能被原样带入 headers 包；保留本机两棵源码树及原始路径。若需要把 headers 包交付到另一台机器并用于编译外部模块，建议用第 5.3 节的实体化源码另行构建，并检查所有头文件链接和实际模块构建；安装包的独立搬迁能力需要另行验收。

~~~bash
dpkg-deb --contents "$IMAGE" | grep -E 'vmlinuz|/lib/modules/' | head
grep '^CONFIG_LOCALVERSION=' build-j6412/.config
~~~

先固定**旧内核为默认启动项**，再安装新包。Debian 的内核安装钩子通常会把最新内核放到菜单首项，所以只写“不要默认启动新内核”并不够。

~~~bash
OLDKREL=$(uname -r)
printf '%s\n' "$OLDKREL" > ~/xenomai-migration-record/old-kernelrelease.txt
sudo install -d /etc/default/grub.d
sudo tee /etc/default/grub.d/99-xenomai-trial.cfg >/dev/null <<'EOF'
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=false
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
EOF
sudo update-grub
OLD_ENTRY=$(awk -F"'" -v r="$OLDKREL" '
  /^submenu / { sid=$4 }
  /^[[:space:]]*menuentry / {
    if (index($4, "gnulinux-" r "-advanced-")==1) {
      print sid ">" $4; exit
    }
  }' /boot/grub/grub.cfg)
test -n "$OLD_ENTRY" || exit 1
printf '旧内核菜单 ID：%s\n' "$OLD_ENTRY"
sudo grub-set-default "$OLD_ENTRY"
sudo grub-editenv list
~~~

检查 saved_entry 对应旧内核；若本机自定义了 GRUB 菜单导致上面的提取结果为空，人工查看 grub.cfg 的 submenu/menuentry ID 后配置，未确认前停止。已有同名试装配置文件时先备份。安装精确匹配的两个包：

~~~bash
sudo dpkg -i "$HEADER" "$IMAGE"
sudo update-grub
test -e "/boot/vmlinuz-$KREL" || exit 1
test -d "/lib/modules/$KREL" || exit 1
test -e "/boot/initrd.img-$KREL" || exit 1
grep -F "$KREL" /boot/grub/grub.cfg | head
sudo grub-editenv list
~~~

若 dpkg 报依赖错误，先检查 apt 模拟的处理清单，确保不会移除旧内核，再修复依赖：

~~~bash
sudo apt-get -s -f install
sudo apt-get -f install
sudo update-grub
~~~

上面的模拟命令若计划移除旧内核，停止，不执行真正的修复。若 initrd 缺失，运行 sudo update-initramfs -c -k "$KREL" 生成后重新检查。阶段 5 通过条件：

- 新内核、模块和 initrd 均已生成；
- /boot/grub/grub.cfg 中出现新内核；
- 旧内核仍存在；
- kernel-deb.sha256 已保存；
- 安装过程中没有覆盖当前正在运行的内核文件。

## 8. 阶段 6：首次启动和 Cobalt 验收

### 8.1 首次启动

重启后在 GRUB 高级选项中**手动**选择含 j6412-xeno331 的新内核。若没有手动选择，阶段 5 设置的 saved_entry 应启动旧内核。首次成功进入新内核前保留本地控制台和旧内核菜单。新内核若启动失败，等待或按电源键重启，再从旧内核进入；不要连续盲目重启。

启动后执行：

~~~bash
uname -a
uname -r
test "$(uname -r)" = "$(cat ~/src/xenomai-j6412/planned-kernelrelease.txt)"
sudo dmesg | grep -i -E 'xenomai|cobalt|dovetail|pipeline'
test -r /proc/xenomai/version && cat /proc/xenomai/version
test -r /proc/xenomai/sched/stat && head -30 /proc/xenomai/sched/stat
grep -E '^CONFIG_(XENOMAI|DOVETAIL|IGB)=' /boot/config-$(uname -r)
~~~

官方预期日志形态类似：

~~~text
[Xenomai] Cobalt vX.Y.Z
~~~

实际版本字符串可能包含发行版本地版本。必须确认：

- uname -r 是新内核；
- dmesg 中出现 Xenomai/Cobalt 成功初始化，而不是 error、failed 或 disabled；
- /proc/xenomai/version 可读；
- /proc/xenomai/sched/stat 可读；
- 系统磁盘、串口、显示和 I210 网络仍可用。

### 8.2 I210 驱动验收

~~~bash
lspci -Dnnk | grep -A4 -i ethernet
ip -br link
# 根据 lspci 的 PCI 地址和 ip -br link，输入真实 I210 接口名
read -r -p '请输入已确认的 I210 接口名：' IF
test -n "$IF" && test -e "/sys/class/net/$IF/device" || exit 1
readlink -f "/sys/class/net/$IF/device"
ethtool -i "$IF"
ethtool "$IF" | sed -n '1,25p'
printf '%s\n' "$IF" > ~/xenomai-migration-record/i210-interface.txt
~~~

期望 ethtool -i 中：

- driver: igb
- 版本与当前内核模块一致
- 固件信息可正常读取

先核实 sysfs 路径的 PCI 地址与 I210 相同，再执行后续命令。若 CONFIG_IGB=m 且驱动未载入，运行 sudo modprobe igb 后复查；不能把其他 Intel 网卡误认为 I210。保存链路速率、Link detected 和驱动版本。网络连通性还要按现场静态地址或 DHCP 配置用 ping 对端验证。

### 8.3 非 root 访问权限（先完成 root 验收，再启用）

本节是可选权限配置。首次执行时先跳到阶段 7，完成阶段 8 的 root 测试后再回来；若最终始终由 root 启动测试工具，可以跳过并在验收表注明。普通用户权限不是阶段 6 启动验收的前置条件。

创建专用组并记录 GID：

~~~bash
getent group xenomai >/dev/null || sudo groupadd --system xenomai
getent group xenomai
sudo usermod -aG xenomai "$USER"
getent group xenomai | cut -d: -f3
~~~

记下实际 GID。用 sudoedit /etc/default/grub 修改现有 GRUB_CMDLINE_LINUX_DEFAULT，**追加**参数 xenomai.allowed_group=实际GID，保留该行已有的根文件系统、安全和控制台参数；再执行：

~~~bash
sudo update-grub
grep '^GRUB_CMDLINE_LINUX_DEFAULT=' /etc/default/grub
sudo tee /etc/udev/rules.d/99-xenomai-rtdm.rules >/dev/null <<'EOF'
SUBSYSTEM=="rtdm", GROUP="xenomai", MODE="0660"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=rtdm
sudo tee /etc/security/limits.d/99-xenomai.conf >/dev/null <<'EOF'
@xenomai soft memlock unlimited
@xenomai hard memlock unlimited
@xenomai soft rtprio 99
@xenomai hard rtprio 99
EOF
~~~

重启并重新登录目标用户；在 GRUB 中再次手动选新内核。确认 Debian 的 PAM 会话启用了 pam_limits.so，且 udev 已按新规则处理设备：

~~~bash
cat /proc/cmdline
id
ulimit -l
ulimit -r
grep -Rns 'pam_limits\.so' /etc/pam.d
find /dev/rtdm -maxdepth 1 -type c -printf '%u %g %m %p\n' 2>/dev/null
~~~

确认当前登录方式对应的 PAM 服务实际加载了 pam_limits.so，不能仅凭目录中搜索到一行就判通过；SSH、login、图形会话的配置入口可能不同。确认 id 含 xenomai 组、/proc/cmdline 有正确 GID、ulimit -l 为 unlimited、ulimit -r 达到所需优先级，且需要的 /dev/rtdm 节点属于 xenomai 组。udev 规则对已存在节点未生效时，检查 udevadm info，必要时重启或定向触发该子系统；不要给所有设备 chmod 777。最后以该用户去掉 sudo 重跑 10.2 和 10.3；root 正常但普通用户失败时，先定位权限和 PAM。

阶段 6 通过条件：

- Cobalt 启动日志正确；
- I210 仍由 igb 驱动；
- 旧内核可回滚；
- 普通用户权限配置如启用，另按 8.3 留记录并验收；
- 没有出现网卡丢失、磁盘不可挂载或温度保护异常。

此时保持旧内核为默认。新内核通过用户库、实时性能、优化和长时间稳定性验收后，才按第 16 节切换默认启动项。

## 9. 阶段 7：编译和安装 Xenomai 3.3.1 用户空间库

### 9.1 编译

这份压缩包由 Git 快照生成，同样需要先运行 bootstrap 产生 configure，再使用独立目录构建用户库：

~~~bash
cd ~/src/xenomai-j6412/xenomai-3.3.1
./scripts/bootstrap || exit 1
mkdir -p ../build-xenomai
cd ../build-xenomai
../xenomai-3.3.1/configure \
  --with-core=cobalt \
  --enable-smp \
  --enable-pshared \
  --enable-tls \
  --disable-registry \
  --prefix=/usr/xenomai || exit 1
set -o pipefail
make -j4 2>&1 | tee ../userspace-build.log
~~~

此阶段不启用 FUSE registry，避免引入额外依赖；若未来需要对象注册表，可另行启用并补齐 libfuse 开发包及内核 FUSE 支持。检查配置摘要中必须是 Cobalt、x86_64、SMP。若报内核接口不匹配，先核对当前启动内核与 Xenomai 源码。

### 9.2 暂存安装并验收文件

先安装到暂存目录：

~~~bash
STAGE=$(mktemp -d)
make DESTDIR="$STAGE" install
test -x "$STAGE/usr/xenomai/bin/latency"
test -x "$STAGE/usr/xenomai/bin/xeno-test"
find "$STAGE/usr/xenomai" -maxdepth 2 -type f | head -30
~~~

确认至少有：

- usr/xenomai/bin/latency
- usr/xenomai/bin/xeno-test
- usr/xenomai/lib
- usr/xenomai/include
- Cobalt 的动态库（例如 libcobalt.so）

正式安装：

~~~bash
sudo make install
echo /usr/xenomai/lib | sudo tee /etc/ld.so.conf.d/xenomai.conf
sudo ldconfig
/usr/xenomai/bin/xeno-config --version --core
/usr/xenomai/bin/xeno-test --help | head -30
~~~

验证库和工具：

~~~bash
ldconfig -p | grep libcobalt
ls -l /usr/xenomai/bin/latency /usr/xenomai/bin/xeno-test
ldd /usr/xenomai/bin/latency | tee ~/xenomai-migration-record/latency-ldd.txt
! grep -q 'not found' ~/xenomai-migration-record/latency-ldd.txt
~~~

阶段 7 通过条件：

- xeno-config 报告 3.3.1/cobalt，latency 与 xeno-test 均存在；
- 动态库能被 ldconfig 找到；
- 当前运行内核已经是刚才通过 Cobalt 验收的内核；
- 用户空间库不是另一台机器编译后未经校验复制的版本。

若 /usr/xenomai 之前已安装其他版本，应先备份并选择独立前缀，不能将不同版本库混装。make install 不受 Debian 包管理器追踪，保留 build-xenomai、config.log 和安装清单；卸载时要用同一构建目录的 sudo make uninstall，并清理相应 ld.so 配置后运行 ldconfig。

## 10. 阶段 8：实时基线测试

### 10.1 参数和日志约定

已按 Xenomai 3.3.1 源码核对：-t 0 是 Cobalt 用户态任务，-p 单位为微秒，-c 3 把测量线程绑定 CPU3，-P 80 为实时优先级，-T 单位为秒，-h 打印直方图，-g 写直方图文件。latency 没有标准 --help 选项，不能把 -h 当成帮助。最终控制台 RTS 行给出 min/avg/max、overrun、msw；直方图文件不能代替完整日志。先准备记录目录：

~~~bash
mkdir -p ~/xenomai-migration-record/tests
cd ~/xenomai-migration-record/tests
set -o pipefail
test -r /proc/xenomai/version
lscpu -e=CPU,ONLINE | grep '^[[:space:]]*3[[:space:]]'
~~~

以下命令先以 root 运行，排除普通用户权限影响；非 root 权限按 8.3 独立验收。每次检查命令退出码、RTS 行、dmesg 和温度。

### 10.2 空闲条件 1 ms

建议先连续运行 10 分钟：

~~~bash
sudo /usr/xenomai/bin/latency -t 0 -c 3 -p 1000 -P 80 -T 600 -h \
  -g latency-idle-1ms.histo 2>&1 | tee latency-idle-1ms.log
test "${PIPESTATUS[0]}" -eq 0
grep '^RTS|' latency-idle-1ms.log
~~~

记录：

~~~bash
uname -a | tee latency-idle-1ms-system.txt
cat /proc/cmdline | tee -a latency-idle-1ms-system.txt
sudo dmesg | tail -100 | tee -a latency-idle-1ms-system.txt
~~~

通过条件：

- 测试持续运行，不挂死；
- RTS 的 overrun=0；
- RTS 的 msw=0，或有分析记录；如有模式切换，应检查系统调用、缺页和权限后重测；
- 最大延迟、平均延迟和测试条件已保存；
- 没有新增 kernel warning、硬件错误或网卡重置。

### 10.3 空闲条件 125 us

按微秒指定 125 us 周期：

~~~bash
sudo /usr/xenomai/bin/latency -t 0 -c 3 -p 125 -P 80 -T 600 -h \
  -g latency-idle-125us.histo 2>&1 | tee latency-idle-125us.log
test "${PIPESTATUS[0]}" -eq 0
grep '^RTS|' latency-idle-125us.log
~~~

最少重复 3 次，文件名附测试序号，保留原始日志。-t 0 不依赖 timerbench 设备，只有 -t 1/-t 2 才需检查 CONFIG_XENO_DRIVERS_TIMERBENCH 与 xeno_timerbench 模块。125 us 是唤醒测试周期，不等于完整应用周期。

阶段 8 通过条件建议：

| 项目 | 通过判定 |
|---|---|
| Cobalt 状态 | 运行中，无错误 |
| 125 us 测试 | 能完整运行，overrun=0 |
| 模式切换 | 0 为目标；非 0 必须定位原因 |
| 最坏延迟 | 应小于项目给实时唤醒分配的预算；未定预算时可暂以 25 us（125 us 的 20%）作为排查阈值，不能视为硬件保证 |
| 可重复性 | 至少 3 次测试结果趋势一致 |

这里的 20% 是项目临时排查阈值，不是 Xenomai/J6412 官方保证。完整周期必须满足实测唤醒延迟最大值 + 应用最坏执行时间 + I/O/通信时间 + 安全余量 < 125 us；有限时长测试的观测最大值不能证明理论最坏情况，也不能证明 EtherCAT 周期。

## 11. 阶段 9：压力、温度和普通网络负载

### 11.1 CPU/内存压力

先在终端 A 中执行下面的压力命令，使普通 Linux 负载位于 CPU0-2。压力持续 35 分钟，给预热及切换终端预留 5 分钟：

~~~bash
cd ~/xenomai-migration-record/tests
set -o pipefail
date -Is | tee stress-short-start.txt
taskset -c 0-2 stress-ng --cpu 3 --vm 1 --vm-bytes 30% \
  --timeout 35m --metrics-brief 2>&1 | tee stress-short.log
STRESS_RC=${PIPESTATUS[0]}
date -Is | tee stress-short-end.txt
test "$STRESS_RC" -eq 0
~~~

压力稳定运行至少 60 秒后，在终端 B 执行下面的 30 分钟 CPU3 实时测试；必须在压力开始后 4 分钟内启动。若准备超时，重新启动一轮压力。核对起止时间，确保整个 latency 测量区间都有负载：

~~~bash
cd ~/xenomai-migration-record/tests
set -o pipefail
date -Is | tee latency-stress-start.txt
sudo /usr/xenomai/bin/latency -t 0 -c 3 -p 125 -P 80 -T 1800 -h \
  -g latency-stress-cpu3-125us.histo 2>&1 | tee latency-stress-cpu3-125us.log
LATENCY_RC=${PIPESTATUS[0]}
date -Is | tee latency-stress-end.txt
test "$LATENCY_RC" -eq 0
grep '^RTS|' latency-stress-cpu3-125us.log
~~~

记录：

~~~bash
sensors 2>/dev/null || true
sudo dmesg | grep -i -E 'thermal|thrott|mce|watchdog|igb|reset'
~~~

通过条件：

- 无 overrun；
- 无机器检查异常、温度保护异常、网卡 reset；
- 温度达到平台允许上限前，测试能够稳定完成；
- 记录室温、机箱状态、散热方式和测试时间。
### 11.2 I210 普通网络负载

确认 I210 接口名称后，在另一台主机运行 iperf3 服务：

~~~bash
iperf3 -s
~~~

工控机终端 A：输入测试对端的真实 IPv4，并选择本轮方向。正向和反向各完整做一轮；每轮网络压力 13 分钟，期间测量 10 分钟 latency：

~~~bash
cd ~/xenomai-migration-record/tests
set -o pipefail
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
read -r -p '请输入 I210 测试网络对端 IPv4：' PEER
test -n "$PEER" || exit 1
printf '%s\n' "$PEER" > ~/xenomai-migration-record/network-peer.txt
ip route get "$PEER"
ip -o route get "$PEER" | grep -F " dev $IF " || exit 1
read -r -p '本轮方向：forward 或 reverse：' NETDIR
case "$NETDIR" in
  forward) NETOPT=() ;;
  reverse) NETOPT=(-R) ;;
  *) echo '方向错误'; exit 1 ;;
esac
printf '%s\n' "$NETDIR" > ~/xenomai-migration-record/network-direction.txt
ethtool -S "$IF" > "i210-$NETDIR-before.txt"
date -Is | tee "network-$NETDIR-start.txt"
taskset -c 0-2 iperf3 -c "$PEER" -t 780 -P 2 "${NETOPT[@]}" \
  2>&1 | tee "iperf3-$NETDIR.log"
NET_RC=${PIPESTATUS[0]}
date -Is | tee "network-$NETDIR-end.txt"
ethtool -S "$IF" > "i210-$NETDIR-after.txt"
test "$NET_RC" -eq 0
~~~

确认流量建立并稳定 30 秒后，在 2 分钟内于终端 B 启动测量：

~~~bash
cd ~/xenomai-migration-record/tests
set -o pipefail
NETDIR=$(cat ~/xenomai-migration-record/network-direction.txt)
date -Is | tee "latency-net-$NETDIR-start.txt"
sudo /usr/xenomai/bin/latency -t 0 -c 3 -p 125 -P 80 -T 600 -h \
  -g "latency-net-$NETDIR.histo" 2>&1 | tee "latency-net-$NETDIR.log"
LATENCY_RC=${PIPESTATUS[0]}
date -Is | tee "latency-net-$NETDIR-end.txt"
test "$LATENCY_RC" -eq 0
grep '^RTS|' "latency-net-$NETDIR.log"
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
grep -F "$IF" /proc/interrupts
~~~

通过条件：

- I210 仍由 igb 驱动；
- 无链路 flap、TX timeout、DMA 错误或驱动 reset；
- latency 结果与无网络负载的差异已记录；
- 明确区分 Linux 普通网络压力与 EtherCAT 周期性能。

ip route get 必须显示走已核实的 I210 接口，否则先修正测试网路由。每个方向都必须满足 overrun=0 和项目唤醒预算，且流量覆盖完整测量区间；提前断流则该轮无效。比较测试前后 ethtool -S 的计数增量，不能把历史错误计数当成新故障。

### 11.3 执行 Xenomai 功能自检

确认内核启用了 CONFIG_XENO_DRIVERS_SWITCHTEST=y、CONFIG_XENO_DRIVERS_HEAPCHECK=y（阶段 4 同时配置）；latency 的内核测试模式还需 CONFIG_XENO_DRIVERS_TIMERBENCH。运行：

~~~bash
cd ~/xenomai-migration-record/tests
set -o pipefail
sudo env PATH="/usr/xenomai/bin:/usr/xenomai/sbin:$PATH" \
  /usr/xenomai/bin/xeno-test \
  -l "taskset -c 0-2 stress-ng --cpu 3 --timeout 960s --metrics-brief" \
  -- -t 0 -c 3 -p 125 -P 80 -T 900 \
  2>&1 | tee xeno-test.log
test "${PIPESTATUS[0]}" -eq 0
~~~

xeno-test 会先运行 smokey、clocktest、switchtest，再运行负载与 latency。检查退出码和日志中的失败/跳过原因，不能仅搜索一次 PASS；硬件无关的可选测试跳过也要写明原因。这里显式替换默认 dohell 负载，不使用 -r（源码说明该选项引入实时竞争后延迟数字不再适合作为基准）。xeno-test 会临时处理 systemd-timesyncd 并把 /proc/xenomai/latency 复位，因此只在隔离的验收环境执行；完成后检查时钟同步服务状态，并重新做阶段 8 的 latency 基线。

## 12. 阶段 10：四核亲和性和抖动优化

优化必须按单变量实验进行。每次修改都要保存一份完整 latency 日志。

### 12.1 固定实时测试核

先选择 CPU3，确认系统有 CPU0 到 CPU3：

~~~bash
lscpu -e=CPU,ONLINE,CORE,SOCKET,NODE
taskset -pc $$
~~~

运行：

~~~bash
sudo /usr/xenomai/bin/latency -t 0 -c 3 -p 125 -P 80 -T 600 -h \
  -g latency-cpu3-125us.histo 2>&1 | tee latency-cpu3-125us.log
~~~

不要把所有中断都放到 CPU3。初始建议：

- CPU3：实时测试任务；
- CPU0-2：普通 Linux 工作和多数设备中断；
- I210 中断：先保持系统默认，测出基线后再调整。

### 12.2 I210 IRQ 亲和性

以 sysfs 的设备归属确定 I210 的 MSI/MSI-X IRQ，不能照抄另一台机器的 IRQ 编号。先保存亲和性：

~~~bash
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
ethtool -i "$IF"
grep -F "$IF" /proc/interrupts
test -d "/sys/class/net/$IF/device/msi_irqs"
IRQ_RECORD=~/xenomai-migration-record/i210-irq-affinity-before.txt
test ! -e "$IRQ_RECORD" || { echo '已有快照，请归档后再采集，避免覆盖原值'; exit 1; }
cat /proc/sys/kernel/random/boot_id > ~/xenomai-migration-record/i210-irq-boot-id.txt
: > "$IRQ_RECORD"
for p in /sys/class/net/"$IF"/device/msi_irqs/*; do
  test -e "$p" || continue
  irq=${p##*/}
  printf '%s %s\n' "$irq" "$(cat /proc/irq/$irq/smp_affinity_list)" >> "$IRQ_RECORD"
done
cat "$IRQ_RECORD"
test -s "$IRQ_RECORD" || exit 1
~~~

若目录不存在或清单为空，先通过 /proc/interrupts 和 PCI MSI 状态查明原因，不能执行后面的空清单配置。记录 irqbalance 状态，临时停止，再把 I210 的普通 Linux IRQ 约束到 CPU0-2：

~~~bash
IRQ_RECORD=~/xenomai-migration-record/i210-irq-affinity-before.txt
test -s "$IRQ_RECORD" || exit 1
cmp -s /proc/sys/kernel/random/boot_id \
  ~/xenomai-migration-record/i210-irq-boot-id.txt || exit 1
systemctl is-enabled irqbalance 2>/dev/null | tee ~/xenomai-migration-record/irqbalance-enabled.txt
systemctl is-active irqbalance 2>/dev/null | tee ~/xenomai-migration-record/irqbalance-active.txt
sudo systemctl stop irqbalance 2>/dev/null || true
while read -r irq affinity; do
  printf '0-2\n' | sudo tee "/proc/irq/$irq/smp_affinity_list"
  printf 'IRQ %s actual: ' "$irq"
  cat "/proc/irq/$irq/effective_affinity_list"
done < "$IRQ_RECORD"
~~~

验证有效亲和性不含 CPU3，并在有网络流量时检查 /proc/interrupts 各列的计数增量。MSI-X 中断可能受 managed IRQ 策略限制：写入失败或 effective_affinity_list 仍包含 CPU3 就算未生效，须排查 IRQ 管理方式，不能只看 tee 输出“0-2”判成功。不要改动 Cobalt 本身的实时中断。

恢复原值：

~~~bash
IRQ_RECORD=~/xenomai-migration-record/i210-irq-affinity-before.txt
test -s "$IRQ_RECORD" || exit 1
cmp -s /proc/sys/kernel/random/boot_id \
  ~/xenomai-migration-record/i210-irq-boot-id.txt || exit 1
while read -r irq affinity; do
  printf '%s\n' "$affinity" | sudo tee "/proc/irq/$irq/smp_affinity_list"
done < "$IRQ_RECORD"
if grep -qx active ~/xenomai-migration-record/irqbalance-active.txt; then
  sudo systemctl start irqbalance
fi
~~~

恢复命令仅适用于采集快照的同一次启动；boot_id 不一致就停止，禁止把旧 IRQ 编号写到重启后的系统。IRQ 编号可能在每次启动变化；若采用持久化服务，应每次按 sysfs 重新枚举并验证有效亲和性。只有证实优化有效后才考虑禁用 irqbalance 的开机启动，并记录恢复方式。调优阶段先采用临时设置，验收记录要表明“重启是否仍生效”。

### 12.3 BIOS 和内核参数

建议按顺序逐项测试并记录：

1. BIOS 先限制深度 CPU/Package C-state；比较修改前后结果，保留 ACPI、风扇与温度保护。
2. 比较 Burst/动态频率策略；不要直接编译掉所有电源和热管理功能。
3. 检查实际 cpufreq 驱动，再试 performance governor。performance 不代表温度或功率限制下频率永远固定。

~~~bash
for p in /sys/devices/system/cpu/cpufreq/policy*; do
  test -d "$p" || continue
  printf '%s driver=%s governor=%s available=%s\n' "$p" \
    "$(cat "$p/scaling_driver")" "$(cat "$p/scaling_governor")" \
    "$(cat "$p/scaling_available_governors")"
done
~~~

若支持 performance，用下面命令临时设置，保存修改前值；若没有 policy 目录，说明该驱动未提供此接口，不强写不存在的节点。

~~~bash
for p in /sys/devices/system/cpu/cpufreq/policy*; do
  test -d "$p" || continue
  printf '%s %s\n' "$p" "$(cat "$p/scaling_governor")"
done > ~/xenomai-migration-record/governor-before.txt
while read -r p governor; do
  grep -qw performance "$p/scaling_available_governors" || continue
  echo performance | sudo tee "$p/scaling_governor"
  cat "$p/scaling_governor"
done < ~/xenomai-migration-record/governor-before.txt
~~~

恢复时把记录文件中的 governor 写回对应 policy 的 scaling_governor。每次改动后重跑空闲、CPU 和网络压力测试；同时检查热降频。

最后评估 Linux CPU 隔离。在 /etc/default/grub 的现有参数后追加下面这组候选参数（只在试验记录中作为独立配置使用），更新 GRUB 并重启到新内核：

~~~text
isolcpus=domain,3 nohz_full=3 rcu_nocbs=3 irqaffinity=0-2
~~~

需要最终内核支持 CONFIG_NO_HZ_FULL=y、CONFIG_RCU_NOCB_CPU=y；CONFIG_NO_HZ_FULL 还依赖上下文跟踪等选项，使用 menuconfig 解决依赖并重新构建。CPU0-2 保留为 Linux housekeeping 核。重启后验证：

~~~bash
cat /proc/cmdline
cat /sys/devices/system/cpu/isolated
cat /sys/devices/system/cpu/nohz_full
grep -E '^CONFIG_(NO_HZ_FULL|RCU_NOCB_CPU)=' /boot/config-$(uname -r)
~~~

预期隔离/全动态时钟集合含 CPU3；仍要以 latency -c 3 绑定 Cobalt 线程，并检查各个设备的有效 IRQ 亲和性。irqaffinity 不能保证移动所有 managed IRQ，isolcpus 也不能消除固件 SMI、NMI、共享缓存和内存带宽干扰。若无改善，删去新加参数，update-grub 后重启恢复。

不要直接复制旧博客中关闭所有 ACPI 的做法，也不要启用 xenomai.smi=enabled 或屏蔽 SMI 来追求漂亮数字；固件事件应通过 BIOS/整机厂商定位。避免把 acpi=off、idle=poll、mitigations=off 作为默认调优参数。最终选择必须经过 12.4 的热稳态长时间测试。

### 12.4 最终配置的 24 h / 72 h 稳定性验收

阶段 8、阶段 9 通过，且阶段 10 的最终 BIOS、启动参数、IRQ 和 governor 已选定后执行。24 h 为建议的工程预验收时长，生产候选配置建议连续 72 h，并至少完成 3 次冷启动复验。测试时长不能作为所有平台最坏响应时间的保证。若冷启动或重新配置 IRQ/governor 使运行配置发生改变，要记录这一点，并从相同条件下重新完成相应长测。

一次长测使用一个新目录；同一轮的所有终端都从记录文件读取目录和时长，避免跨终端变量丢失。测试期间保持该记录文件不变，不能同时启动第二轮。终端 A 先建立本轮记录：

~~~bash
read -r -p '本轮测量时长，输入 24 或 72（小时）：' HOURS
case "$HOURS" in 24|72) ;; *) echo '时长错误'; exit 1 ;; esac
RUN_DIR="$HOME/xenomai-migration-record/tests/final-${HOURS}h-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RUN_DIR"
printf '%s\n' "$RUN_DIR" > ~/xenomai-migration-record/current-final-test.txt
printf '%s\n' "$((HOURS * 3600))" > "$RUN_DIR/duration-seconds.txt"
cd "$RUN_DIR" || exit 1
uname -a > system.txt
cat /proc/cmdline > cmdline.txt
cp /boot/config-$(uname -r) kernel.config
cp ~/src/xenomai-j6412/linux-dovetail-commit.txt .
cp ~/src/xenomai-j6412/xenomai-commit.txt .
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
ethtool -S "$IF" > i210-before.txt
cat /proc/interrupts > interrupts-before.txt
sudo journalctl -k -b > kernel-before.log
~~~

本轮测量时长为 86400 或 259200 秒，全部压力持续时间比它多 1 小时。先启动 CPU、网络及所需磁盘负载，再等待至少 10 分钟，并观察温度趋于稳定，最后启动 latency；latency 必须在最早压力启动后 50 分钟内开始。准备超时则重开本轮。所有压力都必须覆盖完整测量时段，提前退出或断流即判该轮无效。

终端 A：CPU/内存压力，运行前后分别留下时间戳：

~~~bash
RUN_DIR=$(cat ~/xenomai-migration-record/current-final-test.txt)
cd "$RUN_DIR" || exit 1
DURATION=$(cat duration-seconds.txt)
LOAD_SECONDS=$((DURATION + 3600))
set -o pipefail
date -Is | tee stress-start.txt
taskset -c 0-2 stress-ng --cpu 3 --vm 1 --vm-bytes 30% \
  --timeout "${LOAD_SECONDS}s" --metrics-brief 2>&1 | tee stress.log
STRESS_RC=${PIPESTATUS[0]}
date -Is | tee stress-end.txt
test "$STRESS_RC" -eq 0
~~~

终端 B：网络压力。对端保持 iperf3 服务运行；选择与现场相符的方向，两个方向的短测必须已完成。若两个方向均属于关键运行工况，应分别安排完整长测，使用新的轮次目录：

~~~bash
RUN_DIR=$(cat ~/xenomai-migration-record/current-final-test.txt)
cd "$RUN_DIR" || exit 1
DURATION=$(cat duration-seconds.txt)
LOAD_SECONDS=$((DURATION + 3600))
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
PEER=$(cat ~/xenomai-migration-record/network-peer.txt)
test -n "$PEER" || exit 1
ip -o route get "$PEER" | grep -F " dev $IF " || exit 1
read -r -p '本轮网络方向：forward 或 reverse：' NETDIR
case "$NETDIR" in
  forward) NETOPT=() ;;
  reverse) NETOPT=(-R) ;;
  *) echo '方向错误'; exit 1 ;;
esac
printf '%s\n' "$NETDIR" > network-direction.txt
set -o pipefail
date -Is | tee network-start.txt
taskset -c 0-2 iperf3 -c "$PEER" -t "$LOAD_SECONDS" -P 2 "${NETOPT[@]}" \
  2>&1 | tee network.log
NET_RC=${PIPESTATUS[0]}
date -Is | tee network-end.txt
test "$NET_RC" -eq 0
~~~

如现场运行中有磁盘写入，再增加终端 C：在空间充足的测试目录产生文件负载。核对写入介质、可用空间和耐久限制，记录本轮写入量：

~~~bash
RUN_DIR=$(cat ~/xenomai-migration-record/current-final-test.txt)
DURATION=$(cat "$RUN_DIR/duration-seconds.txt")
LOAD_SECONDS=$((DURATION + 3600))
mkdir -p ~/xenomai-io-test
cd ~/xenomai-io-test || exit 1
df -h .
set -o pipefail
date -Is | tee "$RUN_DIR/storage-start.txt"
taskset -c 0-2 stress-ng --hdd 1 --hdd-bytes 512M \
  --timeout "${LOAD_SECONDS}s" --metrics-brief 2>&1 | tee "$RUN_DIR/storage.log"
STORAGE_RC=${PIPESTATUS[0]}
date -Is | tee "$RUN_DIR/storage-end.txt"
test "$STORAGE_RC" -eq 0
~~~

终端 D：从预热开始每分钟记录温度、频率和平台提供的热降频计数，测量及压力全部结束后按 Ctrl+C 停止：

~~~bash
RUN_DIR=$(cat ~/xenomai-migration-record/current-final-test.txt)
cd "$RUN_DIR" || exit 1
while :; do
  date -Is
  sensors 2>/dev/null || true
  for p in /sys/class/thermal/thermal_zone*/temp \
    /sys/devices/system/cpu/cpufreq/policy*/scaling_cur_freq \
    /sys/devices/system/cpu/cpu*/thermal_throttle/*throttle_count; do
    test -r "$p" && printf '%s %s\n' "$p" "$(cat "$p")"
  done
  sleep 60
done | tee temperature-frequency.log
~~~

终端 E：确认负载已建立、预热满足要求，并再次核对当前配置；只运行一个 latency 实例。用 125 us 周期持续测量：

~~~bash
RUN_DIR=$(cat ~/xenomai-migration-record/current-final-test.txt)
cd "$RUN_DIR" || exit 1
DURATION=$(cat duration-seconds.txt)
set -o pipefail
date -Is | tee latency-start.txt
sudo /usr/xenomai/bin/latency -t 0 -c 3 -p 125 -P 80 -T "$DURATION" -h \
  -g latency-final.histo 2>&1 | tee latency-final.log
LATENCY_RC=${PIPESTATUS[0]}
date -Is | tee latency-end.txt
test "$LATENCY_RC" -eq 0
grep '^RTS|' latency-final.log
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
ethtool -S "$IF" > i210-after.txt
cat /proc/interrupts > interrupts-after.txt
sudo journalctl -k -b > kernel-after.log
~~~

长测通过条件：

- latency 实际运行时长达到计划值，最终 RTS 的 overrun=0，max 满足项目分配的唤醒预算；msw=0，非零必须解释并修正后复测。
- CPU/网络及纳入验收的磁盘压力均正常退出，起止时间覆盖整个 latency 时段；核对持续吞吐量，不能只看进程还存在。
- 没有新增 MCE、RCU stall、驱动重置、链路反复断开或温度保护异常；比较计数增量及测试前后日志。
- 温度、频率和降频状态满足整机厂家限制。节点缺失时使用厂家认可的监测方式，不能把“没有读数”判为“没有降频”。记录室温、机箱和散热条件。
- 至少 3 次冷启动后，重新检查 uname、Cobalt、I210、启动参数和有效 IRQ 亲和性，并重做阶段 8、阶段 9 短测；临时 IRQ/governor 设置应重新按实际设备应用，或验证已经审查过的持久化配置。

发生错误时该轮判失败，修正后从完整时长重新开始；保留异常样本。完成后先停止测试负载，再按 16.1 决定是否切换默认启动内核。

## 13. 失败诊断

### 13.1 打补丁失败

现象：

- Hunk FAILED
- patch does not apply
- Dovetail version mismatch

处理：

1. 记录 Linux 提交号和 make -s kernelversion；
2. 检查 Dovetail 补丁目标版本；
3. 确认树是否已经包含 Dovetail；
4. 使用干净源码重新开始；
5. 不使用 patch -f 或忽略失败继续编译。

### 13.2 编译时找不到 Xenomai 菜单

检查：

~~~bash
grep -R -n XENOMAI ~/src/xenomai-j6412/linux-dovetail/Kconfig* \
  ~/src/xenomai-j6412/linux-dovetail/kernel 2>/dev/null | head
test -r ~/src/xenomai-j6412/linux-dovetail/kernel/xenomai/Kconfig
~~~

如果不存在，重新执行 prepare-kernel.sh，并确认脚本路径和 Linux 源码路径没有写反。

### 13.3 启动后没有 Cobalt

检查：

~~~bash
uname -r
dmesg | grep -i -E 'xenomai|cobalt|dovetail|pipeline'
grep -E 'CONFIG_(DOVETAIL|XENOMAI|XENO)' /boot/config-$(uname -r)
cat /proc/cmdline
~~~

常见原因：

- 启动的仍是旧内核；
- 只编译了普通 Linux 内核；
- Dovetail 或 Cobalt 配置没有保存；
- initrd/GRUB 没有更新；
- 启动参数禁用了 Cobalt；
- Xenomai 用户空间与内核版本不匹配。

### 13.4 latency 权限错误

检查：

~~~bash
id
getent group xenomai
ls -l /dev/rtdm 2>/dev/null
grep xenomai.allowed_group /proc/cmdline
~~~

先重新登录，再运行测试。仍失败时检查 Xenomai udev 规则和 CONFIG_DEVTMPFS_MOUNT，不要把设备节点永久设成所有用户可写。

### 13.5 延迟突然升高

按顺序检查：

~~~bash
sudo journalctl -k -b | tail -200
sudo journalctl -k -b | grep -i -E 'igb|watchdog|reset|mce|thermal|thrott'
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
grep -F "$IF" /proc/interrupts
systemctl is-active irqbalance
~~~

然后分别比较：

- 无负载；
- CPU/内存负载；
- I210 网络负载；
- BIOS C-state/Burst 配置；
- 是否固定到 CPU3；
- 是否运行了图形界面、日志服务或磁盘写入任务。

## 14. 阶段验收表

将下面表格复制到项目记录中，每一项附命令输出或日志文件。

| 阶段 | 验收内容 | 结果 |
|---|---|---|
| 0 | 本地控制台、旧内核、GRUB 回滚 | 待验收 |
| 1 | 编译工具、磁盘空间、4 核识别 | 待验收 |
| 2 | Linux/Dovetail/Xenomai 提交号和哈希 | 待验收 |
| 2 | 两个源码包 SHA256 为 OK，SHA256SUMS 和 SOURCE_MANIFEST.json 已归档 | 待验收 |
| 3 | Dovetail/Cobalt 接入无冲突 | 待验收 |
| 3（选做） | 第 5.3 节补丁导出、干净副本预检查与重放 | 待验收或不适用 |
| 4 | .config 含 Dovetail、Cobalt、SMP、HRT、IGB | 待验收 |
| 5 | 新内核包、模块、initrd、GRUB 条目 | 待验收 |
| 6 | 新内核启动，dmesg 出现 Cobalt | 待验收 |
| 6 | I210 由 igb 驱动，链路正常 | 待验收 |
| 7 | libcobalt、latency、xeno-test 可执行 | 待验收 |
| 8 | 空闲 1 ms latency | 待验收 |
| 8 | 空闲 125 us latency | 待验收 |
| 9 | CPU/内存压力下 125 us | 待验收 |
| 9 | I210 普通网络压力下 125 us | 待验收 |
| 9 | xeno-test 功能自检及自检后基线 | 待验收 |
| 10 | CPU 亲和性/IRQ 优化前后对比 | 待验收 |
| 10 | 12.4 的温度、频率、24 h/72 h 长测及 3 次冷启动复验 | 待验收 |
| 回滚 | 旧内核可重新启动 | 待验收 |

## 15. 建议的结果记录格式

每次测试至少保存：

~~~text
日期：
设备序列号：
BIOS版本：
CPU/内存：
Linux提交号：
Linux kernelversion：
Dovetail固定标签、提交号或补丁：
Xenomai提交号：
uname -a：
启动参数：
BIOS C-state/Turbo：
I210接口名：
I210驱动和版本：
实时任务CPU：
I210 IRQ亲和性：
测试周期：
测试时长（含重复次数）：
CPU/内存负载：
网络负载：
latency min/avg/max：
overrun：
msw：
温度/频率/降频：
异常日志：
结论：
~~~

本手册没有在实际 J6412 工控机上执行编译、启动和 latency 测试，因此所有性能结果均应标为“待验收”。只有在目标硬件上完成长时间、压力和网络负载测试后，才能决定是否满足 125 us 应用周期。

## 16. 回滚

如果新内核无法启动：

1. 重启，在 GRUB 高级选项选择旧内核；
2. 启动旧内核后确认网卡和磁盘正常；
3. 保存新内核的启动日志，供后续分析；
4. 不要删除旧内核，直到新内核连续通过验收；
5. 若只是恢复旧内核默认项，不需要卸载任何包。用第 5 阶段记录的旧菜单 ID 执行：

~~~bash
OLDKREL=$(cat ~/xenomai-migration-record/old-kernelrelease.txt)
OLD_ENTRY=$(awk -F"'" -v r="$OLDKREL" '
  /^submenu / { sid=$4 }
  /^[[:space:]]*menuentry / {
    if (index($4, "gnulinux-" r "-advanced-")==1) {
      print sid ">" $4; exit
    }
  }' /boot/grub/grub.cfg)
test -n "$OLD_ENTRY" || exit 1
sudo grub-set-default "$OLD_ENTRY"
sudo grub-editenv list
sudo update-grub
~~~

6. 若需要移除失败内核，先确认当前不是该内核：

~~~bash
uname -r
dpkg -l 'linux-image*' 'linux-headers*' | grep '^ii'
~~~

选择正确的失败版本后，先模拟卸载，确认不会移除当前或旧内核：

~~~bash
KREL=$(cat ~/src/xenomai-j6412/planned-kernelrelease.txt)
test "$(uname -r)" != "$KREL" || exit 1
dpkg-query -W "linux-image-$KREL" "linux-headers-$KREL"
sudo apt-get -s remove "linux-image-$KREL" "linux-headers-$KREL"
~~~

模拟结果确认无误后执行：

~~~bash
sudo apt-get remove "linux-image-$KREL" "linux-headers-$KREL"
sudo update-grub
~~~

若新内核只是性能不达标，优先保留包和日志，回到第 12 节逐项调整；删除包会失去复现条件。

回滚验收必须再次确认：

~~~bash
uname -r
IF=$(cat ~/xenomai-migration-record/i210-interface.txt)
lspci -Dnnk | grep -A4 -i ethernet
ip -br link
ethtool -i "$IF"
~~~

回滚验收要求 uname -r 为旧版本、I210 仍由 igb 驱动、链路和存储可用，且 saved_entry 指向旧内核。删除试验内核前，至少保留一份构建源码、版本清单和失败日志。

### 16.1 全部验收通过后切换默认项

先确认当前运行的是已通过验收的目标版本，再设置默认菜单：

~~~bash
KREL=$(cat ~/src/xenomai-j6412/planned-kernelrelease.txt)
test "$(uname -r)" = "$KREL" || exit 1
NEW_ENTRY=$(awk -F"'" -v r="$KREL" '
  /^submenu / { sid=$4 }
  /^[[:space:]]*menuentry / {
    if (index($4, "gnulinux-" r "-advanced-")==1) {
      print sid ">" $4; exit
    }
  }' /boot/grub/grub.cfg)
test -n "$NEW_ENTRY" || exit 1
sudo grub-set-default "$NEW_ENTRY"
sudo grub-editenv list
~~~

确认 saved_entry 指向新内核；冷启动后检查 uname -r、/proc/xenomai/version、网卡和 125 us 短测。保留旧内核菜单及 10 秒菜单等待，不因验收通过立即删除恢复通道。

## 17. 与 EtherCAT 的边界

完成并通过本文验收后，可确认：

- J6412 上的 Dovetail+Cobalt 可以启动；
- Xenomai 用户空间库可用；
- I210 的 Linux igb 驱动正常；
- 125 us 的 Xenomai 调度测试结果可重复。

它没有证明：

- EtherCAT 主站周期已经达到 125 us；
- I210 已具备 EtherCAT 硬实时帧收发；
- PDO、DC 同步、从站状态机或线缆拓扑正常；
- 控制应用在真实 I/O 负载下不会超时。

EtherCAT 应在本手册所有阶段通过后另立文档，单独选择主站、网卡驱动、IRQ 策略、分布式时钟和周期验收方法。

## 18. 参考与核对依据

本文根据博客的移植流程编写，并按 Xenomai 3.3.1 源码核对命令。以下来源是操作依据，不能替代目标机验收：

| 来源 | 用途 |
|---|---|
| [用户指定博客](https://www.cnblogs.com/wsg1100/p/12864199.html) | Linux 中断流水线、Cobalt 与用户库的分步构建思路 |
| [Xenomai 3.3.1 安装文档](https://github.com/xenomai-ci/xenomai/blob/v3.3.1/doc/asciidoc/README.INSTALL.adoc) | Cobalt 所需内核、prepare-kernel、用户库和启动参数 |
| [prepare-kernel.sh](https://github.com/xenomai-ci/xenomai/blob/v3.3.1/scripts/prepare-kernel.sh) | --arch=x86、Dovetail 检测、源码链接与 outpatch |
| [Kconfig 入口](https://github.com/xenomai-ci/xenomai/blob/v3.3.1/scripts/Kconfig.frag) | CONFIG_XENOMAI 选择 Dovetail |
| [Cobalt Kconfig](https://github.com/xenomai-ci/xenomai/blob/v3.3.1/kernel/cobalt/Kconfig) | VFILE、STATS、调度统计依赖 |
| [latency 源码](https://github.com/xenomai-ci/xenomai/blob/v3.3.1/testsuite/latency/latency.c) | 周期单位、CPU 绑定、日志统计、用户/内核模式 |
| [xeno-test 脚本](https://github.com/xenomai-ci/xenomai/blob/v3.3.1/testsuite/xeno-test/xeno-test.in) | 功能自检顺序、负载选项、时钟同步处理 |
| [RTDM udev 规则](https://github.com/xenomai-ci/xenomai/blob/v3.3.1/kernel/cobalt/udev/rtdm.rules) | RTDM 设备的 xenomai 组访问权限 |
| [Dovetail 固定内核标签](https://source.denx.de/Xenomai/linux-dovetail/-/tree/v6.1.170-cip55-dovetail1) | 本文指定的 6.1.170-cip55 + Dovetail 源码 |
| [Debian 12 pahole 软件包](https://packages.debian.org/bookworm/pahole) | Debian 编译依赖包名核对 |

官方 Git 查询核对的 Linux/Dovetail 提交为 6062891c48355c4295db5e33d47341a3770dde38，Xenomai 提交为 2f15f8c5970b23428a00ccd17249de26a240dc33；下载后逐一复核。源码入口已经确认，本文未宣称完整厂商包名已找到，也未宣称 J6412 的编译、启动或延迟已通过实机测试。

