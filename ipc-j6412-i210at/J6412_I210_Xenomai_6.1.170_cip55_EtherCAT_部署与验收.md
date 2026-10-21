# J6412 + I210：Linux 6.1.170-cip55 / Xenomai 3.3.1 / EtherCAT 部署与阶段验收

**编写日期：2026-10-19**  
**目标硬件：** Intel J6412 四核 x86_64 工控机、独立 Intel I210 EtherCAT 网口、另一独立管理网口。  
**目标：** 先稳定运行 1 ms/500 µs/250 µs EtherCAT，再判断 125 µs 是否能够在实际从站、PDO 长度及系统负载下验收。

> 本文是可执行的工程流程，不是已经在目标板上完成的测试报告。所有示例以 Debian 12 amd64 为基础；对 Ubuntu 可调整依赖包和 GRUB 命令。涉及网口接管、内核安装和 BIOS 改动时，先保留远程管理口及旧内核启动项。

## 0. 最重要的结论

**下载状态说明：** 本轮在线查询没有返回可审计的网页内容，工作区访问 GitLab 也未成功，因此**尚未独立确认指定组合、下列候选标签和精确兼容性**。下表提供上游 Git 入口及需要在联网构建机验证的候选版本，不能当作已经下载、验证成功的厂商 BSP。完整字符串 <code>Linux 6.1.170-cip55-xenomai-3.3.1-j6412-rtipc</code> 没有找到可确认的公开来源；其中 <code>-j6412-rtipc</code> 可能是板厂/项目设置的本地版本后缀或额外补丁。**不应仅靠修改 LOCALVERSION 伪装成该交付物。**

| 部件 | Git 入口 / 待验证的候选标签 | 说明 |
|---|---|---|
| Linux + Dovetail | https://gitlab.com/Xenomai/linux-dovetail.git ，候选 <code>v6.1.170-cip55-dovetail1</code> | 标签及 CIP 基线必须先验真；若已包含 Dovetail，不再叠加不匹配的 I-pipe/PREEMPT_RT 补丁。 |
| Xenomai | https://gitlab.com/Xenomai/xenomai3/xenomai.git ，<code>v3.3.1</code> | 用其 <code>scripts/prepare-kernel.sh</code> 将 Cobalt 代码集成至内核树，并编译配套用户态。 |
| IgH EtherCAT Master | https://gitlab.com/etherlab.org/ethercat.git ，候选 <code>1.6.13</code> | 标签、Linux 6.1 原生 igb 驱动和 Xenomai 3 RTDM 接口须在取得的源码中确认；不能仅凭版本号保证兼容。 |
| <code>j6412-rtipc</code> | **未找到可信的公开 Git 标签/仓库** | 若要求与现有设备字节级一致，必须向交付方索取仓库 URL、commit SHA、内核 <code>.config</code>、补丁序列、模块及构建记录。 |

本文不提供未经远端核对的 commit SHA 或发布日期。部署清单以远端标签核验和本地 <code>git rev-parse HEAD</code> 得到的**完整 SHA**为准。选择 3.3.1 是复现用户指定版本，不表示它是最新版本。候选标签不等于已确认的下载资源。

**没有证据证明上述候选组件在此板卡上的组合已经通过编译和硬实时验收。** RTIPC 用于实时进程间通信，与 I210 网卡的 EtherCAT 收发数据路径是不同部分。若板厂在 <code>rtipc</code> 名下另有私有实现，必须单独核对。文末列出工程师需要联网访问的资料入口；本文已确认的现场事实限于本地项目代码检查。

## 1. 项目代码当前状态及阻断点

本工作区 <code>cnchmi2</code> 同时包含 SOEM 与 IgH 后端。部署前必须选择其一，不可让两套主站同时占用 I210。

| 文件/位置 | 当前行为 | 对目标的影响 |
|---|---|---|
| <code>cnchmi2/NCK/ethercat/rt_timer.c</code> 的三个 <code>rt_timer_create()</code> 分支 | <code>period_us &lt; 1000</code> 时强制设为 <code>1000</code> | 即使配置 125 µs，实际定时周期仍是 1 ms。 |
| <code>cnchmi2/NCK/ethercat/ecatuser.c</code> 与 <code>ecatuser_igh.c</code> | <code>ecat_set_pdo_period_us()</code> 只接受 250–4000 µs | 125 µs 调用会被忽略；DC SYNC0 仍可能为旧周期。 |
| <code>cnchmi2/NCK/build_nck_libs_linux.sh</code> | 无论 SOEM/IgH 都定义 <code>NCK_HAS_PREEMPT_RT</code>；IgH 默认链接 <code>-lethercat</code> | 单独给 HMI 加 <code>CONFIG+=xenomai</code> 不会把 NCK EtherCAT 库切为 Cobalt/RTDM。 |
| <code>cnchmi2/NCK/build_igh_master_linux.sh</code> | 期望 <code>cnchmi2/ethercat-master/ethercat-master/</code>；此工作区未见该 vendored 源码；脚本默认未明确 <code>--enable-igb</code> | 原脚本不能直接构建本方案的 I210 原生驱动。 |
| <code>cnchmi2/NCK/hal/ecatlinux_MACPHY.c</code> | SOEM 走普通 AF_PACKET；当前驱动经 Linux 网络栈 | Cobalt 定时器实时，不意味着收发路径硬实时。 |
| <code>cnchmi2/cnchmi/DEPLOY_LINUX_RT.md</code> 第 10 节 | 声称 AF_PACKET 可直接按 <code>rt_dev_socket</code> 使用、<code>rt_task_shadow</code> 可替代显式锁内存、抖动必小于 10 µs | 这些不能作为该目标机的验收结论；本手册以实测和驱动路径为准。 |
| <code>rt_timer.c</code> 的 <code>T_FPU / T_CPU(1)</code>、<code>RTIME overruns</code> | 使用旧 API/旧亲和性写法；CPU 固定为 1 | 与本文 CPU2/3 掩码冲突；Xenomai 3.3.1 应使用其正式头文件和 <code>rt_task_set_affinity()</code>，<code>rt_task_wait_period()</code> 参数是 <code>unsigned long *</code>。 |

**推荐目标拓扑：** Xenomai Cobalt 定时线程 + IgH 1.6.13 RTDM 用户接口 + IgH 原生 <code>ec_igb</code> 接管 I210 + EtherCAT DC。先在隔离测试机验证，再合入 <code>cnchmi2</code>。原生驱动接管后，I210 不再作为普通 IP 网口使用；管理网络必须是另一块网卡。

## 2. 阶段 A：硬件清点与基线备份

### 操作

1. 接入本地键盘/显示器或带外管理，确认旧内核 GRUB 项可用；保存系统镜像。
2. 用以下命令记录 CPU、BIOS、I210 的 PCI 地址、设备 ID、MAC 和当前驱动；**记录后替换所有示例中的 <code>enpXsY</code>**。

~~~bash
date -Is
cat /etc/os-release
uname -a
lscpu
sudo dmidecode -t bios | head -40
lspci -nn | grep -i -E 'ethernet|network'
ip -br link
ethtool -i enpXsY
cat /sys/class/net/enpXsY/address
readlink -f /sys/class/net/enpXsY/device/driver
sudo ethtool enpXsY
sudo efibootmgr -v
~~~

3. I210 必须是板载或 PCIe 原生接口，而非 USB 网卡。核实 PCI vendor/device ID 与 <code>igb</code> 匹配；把 EtherCAT 口的 MAC 粘到机箱和验收记录中。
4. 禁止在 EtherCAT 口配置 IP、DHCP、NetworkManager 自动接管；管理口保留网络连接。

### 阶段验收 A

- 能从旧内核进入系统；保留镜像和 GRUB 回滚入口。
- <code>lspci -nn</code> 确认 I210；<code>ethtool -i</code> 显示当前 Linux 驱动为 <code>igb</code>。
- 目标 EtherCAT MAC、PCI BDF、BIOS 版本、管理口名称均被记录。
- 未确认网口身份之前，**不启动** EtherCAT 服务或卸载 <code>igb</code>。

## 3. 阶段 B：获取和固定 Git 源码

建议在可联网的 Debian 12 x86_64 构建机上执行；目标机可只部署安装包。**首先验证标签存在，网络错误或标签不存在时停止；不要擅自选择近似标签。**

~~~bash
set -euo pipefail
git ls-remote --exit-code --tags https://gitlab.com/Xenomai/linux-dovetail.git \
  refs/tags/v6.1.170-cip55-dovetail1 'refs/tags/v6.1.170-cip55-dovetail1^{}'
git ls-remote --exit-code --tags https://gitlab.com/Xenomai/xenomai3/xenomai.git \
  refs/tags/v3.3.1 'refs/tags/v3.3.1^{}'
git ls-remote --exit-code --tags https://gitlab.com/etherlab.org/ethercat.git \
  refs/tags/1.6.13 'refs/tags/1.6.13^{}'
~~~

标签缺失时，先列出远端标签供人工核对，不继续 clone 本文候选组合。要严格复现完整 <code>-j6412-rtipc</code> 构建，仍需其提供方的 Git URL、tag/commit、配置与补丁；本手册不能补造这些资源。以下 clone、configure 和编译步骤均以该验真通过为前提。

~~~bash
sudo apt update
sudo apt install -y git patch build-essential bc bison flex libssl-dev libelf-dev \
  libncurses-dev dwarves fakeroot rsync cpio kmod initramfs-tools \
  autoconf automake libtool pkg-config libsystemd-dev dpkg-dev ethtool pciutils rt-tests \
  stress-ng

mkdir -p ~/rt-stack && cd ~/rt-stack
git clone --depth 1 --branch v6.1.170-cip55-dovetail1 https://gitlab.com/Xenomai/linux-dovetail.git
git clone --depth 1 --branch v3.3.1 https://gitlab.com/Xenomai/xenomai3/xenomai.git
git clone --depth 1 --branch 1.6.13 https://gitlab.com/etherlab.org/ethercat.git

git -C linux-dovetail switch --detach v6.1.170-cip55-dovetail1
git -C xenomai switch --detach v3.3.1
git -C ethercat switch --detach 1.6.13

for p in linux-dovetail xenomai ethercat; do
  echo "=== $p ==="
  git -C "$p" describe --tags --always --dirty
  git -C "$p" rev-parse HEAD
  git -C "$p" status --short
done | tee source-manifest.txt
~~~

若 GitLab 无法下载，先在有权限的机器下载官方标签归档/镜像再离线搬运，不要混用别的 6.1.x 内核包。若供应商提供了 **exact** <code>j6412-rtipc</code> 仓库，应另外记录：Git URL、分支/tag、完整 SHA、submodule SHA、补丁顺序、配置、构建工具链、SHA256 和许可证；对照官方标签做 <code>git diff</code>。仅有 <code>uname -r</code> 字符串不够复现。

### 阶段验收 B

- 三个仓库为固定的 detached tag，<code>git status --short</code> 无意外改动。
- 保存 <code>source-manifest.txt</code>，有供应商补丁时记录其 commit 和补丁 SHA256。
- 确认上游 Dovetail 版本号为 6.1.170-cip55；如果供应商交付源与之不一致，先停止组合构建并解释差异。

## 4. 阶段 C：构建 Dovetail + Xenomai Cobalt 内核

Xenomai 官方采用“Dovetail 内核树 + Xenomai 源码树”的拆分模型。对已经包含 Dovetail 的官方标签，运行 <code>prepare-kernel.sh</code> 时**不再指定另一个 Dovetail 补丁文件**。

~~~bash
cd ~/rt-stack/linux-dovetail
# 生成可归档的 Cobalt 集成补丁；从干净的 Dovetail 标签树执行一次
../xenomai/scripts/prepare-kernel.sh --linux="$PWD" --arch=x86 \
  --outpatch=../xenomai-3.3.1-cobalt-integration.patch
patch --dry-run -p1 < ../xenomai-3.3.1-cobalt-integration.patch
patch -p1 < ../xenomai-3.3.1-cobalt-integration.patch
sha256sum ../xenomai-3.3.1-cobalt-integration.patch

# 以下假定在目标工控机本机编译；独立构建机要先复制目标机的 /boot/config-*。
# 使用构建机自身配置可能漏掉目标机磁盘/网卡驱动。
cp /boot/config-$(uname -r) .config
make olddefconfig
make menuconfig

# menuconfig 中启用 Dovetail/IRQ pipeline、Xenomai Cobalt、RTDM；
# 保留 NVMe/SATA、文件系统、I210 igb、串口和管理网卡驱动。
# RTIPC 仅在项目用到 XDDP/IDDP/BUFP 或供应商 BSP 要求时启用。

make -j"$(nproc)" bzImage modules
make -s kernelrelease
grep -E 'CONFIG_(DOVETAIL|IRQ_PIPELINE|XENOMAI|XENO_DRIVERS_RTIPC|HIGH_RES_TIMERS|PCI|IGB)=' .config
~~~

重要配置原则：

- 通过 <code>make menuconfig</code> 选择 Xenomai 配置，不要猜测 Kconfig 名称后直接覆盖 <code>.config</code>。
- 不将 Dovetail/Cobalt 与独立的 PREEMPT_RT 补丁叠加。
- <code>CONFIG_IGB=m</code> 用于初期普通网卡基线测试并允许后续替换；**不要设为 y**，否则标准服务脚本无法卸载内建 igb、加载原生 <code>ec_igb</code>。如已设为 y，重新构建为模块或制定专门的设备接管补丁。
- <code>LOCALVERSION</code> 可用于标识实验构建，如 <code>-j6412-lab1</code>，但不能用它替代厂商补丁。<code>make -s kernelrelease</code> 输出应与 <code>/lib/modules/...</code> 目录一致。
- 在测试阶段不建议禁用所有看门狗、NMI、机器检查功能；先保持可诊断性，取得基线后单项优化。

需要 RTIPC/XDDP 时，核对该版本真实 Kconfig（不是在 uname 后缀中加字）：

~~~text
CONFIG_XENO_DRIVERS_RTIPC=y
CONFIG_XENO_DRIVERS_RTIPC_XDDP=y
~~~

IDDP/BUFP 按应用选用；XDDP 用于实时域与普通 Linux 进程交换数据。启用它不自动创建应用端点、不自动提高 EtherCAT 性能。本文使用 <code>--outpatch</code> 生成并应用实体文件补丁，便于内核源码和 headers 打包。若改用不带 <code>--outpatch</code> 的默认集成，脚本会建立到 Xenomai 树的符号链接，须保留两树原路径并核查打包后的 headers 没有失效链接；不要在已集成树上重复混用两种方式。

建议检查的配置如下（以 menuconfig 和 olddefconfig 最终结果为准）：

~~~text
CONFIG_SMP=y
CONFIG_DOVETAIL=y
CONFIG_XENOMAI=y
CONFIG_HIGH_RES_TIMERS=y
CONFIG_IGB=m
CONFIG_PROC_FS=y
CONFIG_XENO_OPT_VFILE=y
CONFIG_XENO_OPT_STATS=y
CONFIG_NO_HZ_FULL=y
CONFIG_RCU_NOCB_CPU=y
~~~

后两项用于后续隔离实验；保留目标机所需的 initramfs、根文件系统、存储、管理网卡驱动。<code>CONFIG_XENO_OPT_STATS</code> 用于下面的任务模式切换统计。选项若被依赖关系关闭，应查明原因，不要仅保留配置文本中的手工输入值。

安装时保留旧内核：

~~~bash
cd ~/rt-stack/linux-dovetail
sudo make modules_install
sudo make install
KREL="$(make -s kernelrelease)"
if [ -e "/boot/initrd.img-$KREL" ]; then
  sudo update-initramfs -u -k "$KREL"
else
  sudo update-initramfs -c -k "$KREL"
fi
sudo update-grub
sudo reboot
~~~

以上 <code>make install</code> 命令只能在**准备部署的测试目标机**上执行。独立构建机采用打包方式，安装目标路径改为目标机：

~~~bash
# 构建机，源码已按前述步骤配置并完整编译
make -j"$(nproc)" bindeb-pkg
# 把生成的、版本匹配的 linux-image*.deb 与 linux-headers*.deb 搬至目标机。
# 目标机按实际完整文件名安装，勿把旧包或其他架构的包混在同一通配符中：
# sudo dpkg -i ./linux-image-<本次版本>_<包版本>_amd64.deb \
#              ./linux-headers-<本次版本>_<包版本>_amd64.deb
# sudo update-grub
~~~

内核、headers 与 <code>Module.symvers</code> 必须同源同配置；尤其启用 <code>CONFIG_MODVERSIONS</code> 时，只有 <code>modules_prepare</code> 不等于完成一次可用的外部模块构建环境。若出现证书路径或 BTF/pahole 错误，修正配置/构建依赖后重编，不能带错误继续安装。

重启后：

~~~bash
uname -r
dmesg | grep -Ei 'Xenomai|Dovetail|irq pipeline' | tail -60
test -r /proc/xenomai/version && cat /proc/xenomai/version
ls -ld /lib/modules/$(uname -r)/build
~~~

### 阶段验收 C

- 新内核可正常启动、存储和管理口可用；能切回旧内核。
- <code>dmesg</code> 报告 Cobalt 启动；<code>/proc/xenomai/version</code> 可读；<code>uname -r</code> 与模块目录一致。
- 使用旧内核 GRUB 项验证回滚一次。若机器失去管理网口、磁盘或启动能力，停止后续部署。

## 5. 阶段 D：安装 Xenomai 3.3.1 用户态与调度基线

~~~bash
cd ~/rt-stack/xenomai
./scripts/bootstrap
mkdir -p ../build-xenomai && cd ../build-xenomai
../xenomai/configure --with-core=cobalt --prefix=/usr/xenomai
make -j"$(nproc)"
sudo make install
echo /usr/xenomai/lib | sudo tee /etc/ld.so.conf.d/xenomai.conf
sudo ldconfig

/usr/xenomai/bin/xeno-config --version
sudo /usr/xenomai/bin/latency --help
sudo /usr/xenomai/bin/latency
~~~

上面的默认 latency 用于首次冒烟检查，观察约 60 秒后按 Ctrl+C 结束，再运行下面带固定时长的正式测试。

随后按 CPU2 验收 125 µs 调度；先确认该次编译的 <code>latency --help</code> 与下面选项一致：

~~~bash
mkdir -p ~/rt-evidence && cd ~/rt-evidence
sudo /usr/xenomai/bin/latency -t 0 -p 125 -c 2 -P 80 \
  -T 1800 -h -s -g latency-125us-idle.hist 2>&1 | tee latency-125us-idle.log

# 在另一终端给 Linux housekeeping CPU0/1 施加压力；内存大小须低于空闲内存
taskset -c 0,1 stress-ng --cpu 2 --vm 1 --vm-bytes 256M --timeout 2h

# 压力期间另行运行并保存日志
sudo /usr/xenomai/bin/latency -t 0 -p 125 -c 2 -P 80 \
  -T 7200 -h -s -g latency-125us-load.hist 2>&1 | tee latency-125us-load.log
cat /proc/xenomai/sched/stat
cat /proc/cmdline
~~~

**单位提醒：** <code>latency -p 125</code> 为 125 微秒；Alchemy 默认纳秒时间基准下的 <code>rt_task_set_periodic(..., 125000)</code> 才是 125000 纳秒。不要把这两个参数混用。<code>cyclictest</code> 的普通 Linux 线程结果不能代替 Cobalt 测试。<code>/proc/xenomai/sched/stat</code> 字段以实际表头为准；若不存在，列出本次内核的 <code>/proc/xenomai/</code> 接口定位。

对 RT 任务选一个独占逻辑 CPU。J6412 是四核，但 CPU 编号以 <code>lscpu -e</code> 为准。初期可把 CPU0/1 用于 Linux 和管理、CPU2/3 作为候选，先比较只用 CPU2 与 CPU2/3；避免一开始同时更改所有参数。建议首轮 BIOS 关闭深 C-State、EEE、PCIe ASPM，CPU governor 设为 performance；保留温度保护。启动参数实验样例：

~~~text
isolcpus=domain,managed_irq,2-3 nohz_full=2-3 rcu_nocbs=2-3 irqaffinity=0-1
~~~

<code>xenomai.supported_cpus</code> 的格式和启动 CPU 约束须以取得的固定版本源码核实。首轮省略该参数使用默认配置；若要限制 CPU，先检查 kernel/cobalt/init.c 和启动日志，不直接复制仅 CPU2/3 的掩码。示例 0x0f 包含 CPU0–3，但不用于强制任务绑核；应用仍需显式绑定 CPU2。isolcpus 隔离的是 Linux 调度，不能代替 Cobalt 线程亲和性。主板 SMI 可能承担温度或看门狗任务，须与板厂确认后再调整。

设置步骤：

~~~bash
sudo cp -a /etc/default/grub /etc/default/grub.before-xenomai
sudoedit /etc/default/grub
# 在现有 GRUB_CMDLINE_LINUX_DEFAULT 的引号内追加上面的参数，保留原有启动项。
sudo update-grub
sudo reboot
# 重启后检查参数实际生效
cat /proc/cmdline
lscpu -e=CPU,CORE,ONLINE
cat /sys/devices/system/cpu/isolated

# 若 cpufreq 接口存在，将每个策略切到 performance；无接口时查 BIOS/pstate 驱动。
for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do
  [ -e "$f" ] || continue
  echo performance | sudo tee "$f"
done
~~~

performance governor 不等于频率绝对固定；仍需在热稳态下记录实际频率及降频。系统服务和 Qt/HMI 等普通任务应安排在 CPU0/1，CPU2 用于 EtherCAT，CPU3 作为第二实时任务候选；IRQ/cpuset 不能让实时线程指定的 CPU 与 Cobalt 允许掩码相冲突。持久化 governor、IRQ/cpuset 的脚本在基线通过后再安装，并归档恢复方法。

### 阶段验收 D

- <code>latency</code> 在 125 µs 周期附近运行无错误；记录最小/平均/最大和分布，至少 30 min 空载与 2 h 混合压力。
- 若项目使用 RTIPC/XDDP，把其列为独立验收：运行固定标签中提供的 XDDP 示例，核对实时端与 Linux 端双向数据、消息长度、序号和非实时消费者停顿时的行为。先用 <code>git -C ~/rt-stack/xenomai ls-files | grep -Ei 'xddp|rtipc'</code> 定位该标签例程；不要把存在 <code>/dev/rtpN</code> 或 uname 后缀当作数据通路已通过。RT 发送端必须有有界缓冲/超时策略，Linux/HMI 暂停时不能拖住 EtherCAT 周期。
- 记录 CPU 频率、温度、dmesg、CPU 亲和性和 Xenomai mode-switch/警告信息。
- 该阶段只证明**本机唤醒**；不证明 EtherCAT 收发满足 125 µs。

## 6. 阶段 E：I210 专用口与 IgH 主站

先用 <code>lspci -nn</code>、<code>ethtool -i</code> 确认是 I210/igb。在取得的 IgH 固定源码中核对 Linux 6.1 的原生 <code>igb</code> 驱动、I210 设备 ID 与 RTDM 支持后，再按下面模板构建。使用普通 Linux <code>igb</code> + IgH generic 可以用于联机排障，但不直接作为 125 µs 硬实时验收结论。

~~~bash
cd ~/rt-stack/ethercat
./bootstrap
./configure --prefix=/opt/etherlab --sysconfdir=/etc \
  --with-systemdsystemunitdir=/etc/systemd/system \
  --with-linux-dir=/lib/modules/$(uname -r)/build \
  --enable-igb --enable-rtdm --with-xenomai-dir=/usr/xenomai \
  --disable-generic --disable-8139too --disable-eoe --disable-rt-syslog \
  --enable-cycles --enable-hrtimer
make -j"$(nproc)"
make -j"$(nproc)" modules
sudo make install
sudo make modules_install
sudo depmod -a
echo /opt/etherlab/lib | sudo tee /etc/ld.so.conf.d/etherlab.conf
sudo ldconfig

find /lib/modules/$(uname -r) -name 'ec_*igb*.ko*' -o -name 'ec_master*.ko*'
find /opt/etherlab -name 'libethercat_rtdm.so*' -o -name 'libethercat.so*'
systemctl cat ethercat.service
modinfo -F vermagic ec_master
modinfo -F vermagic ec_igb
~~~

如果安装实际生成 <code>lib64</code>，动态库配置改为实际目录。<code>--with-systemdsystemunitdir</code> 显式给出服务文件安装路径；构建日志须明确 service 已安装。不要只执行 <code>systemctl daemon-reload</code> 却跳过安装 service 文件。模块 vermagic 须对应当前目标内核。

**停止点：** <code>--enable-igb</code>、<code>--enable-rtdm</code>、<code>--with-xenomai-dir</code> 必须被当前 checkout 的 <code>./configure --help</code> 识别。若编译时报 RTDM API 或驱动结构不兼容，应固定失败日志，修正对应版本的源码/补丁；不要静默降级到 generic。上述命令也可能因具体 1.6.13 构建系统而需调整；以该标签内的 INSTALL/README 和 <code>./configure --help</code> 为准。

**管理网口同样使用 igb 时：** 若当前版本 <code>ethercatctl</code> 通过卸载整个普通 <code>igb</code> 模块再加载 <code>ec_igb</code> 实施接管，会短暂影响同驱动的所有网口，须先审阅实际脚本；MAC 绑定并不能防止这个卸载动作。<code>ec_igb</code> 可将非 EtherCAT 口接回普通网络栈，但首次切换和恢复都必须在本地/带外控制台操作。若要求管理口全程无中断，应使用不同驱动的独立管理网卡，或经过验证的板厂设备级接管方案。不要全局 blacklist igb 来碰运气。

配置 EtherCAT 服务前，先确认 <code>/etc/ethercat.conf</code> 路径和服务文件来自当前安装版本。以**专用 I210 MAC**而非网口名绑定，避免改名后误接管管理口：

~~~bash
cat /sys/class/net/enpXsY/address
sudoedit /etc/ethercat.conf
# 配置文件中填写实际值：
# MASTER0_DEVICE="aa:bb:cc:dd:ee:ff"
# DEVICE_MODULES="igb"

sudo systemctl daemon-reload
sudo systemctl start ethercat
sudo systemctl status ethercat --no-pager
sudo journalctl -u ethercat -b --no-pager | tail -80
/opt/etherlab/bin/ethercat master
/opt/etherlab/bin/ethercat slaves
~~~

若本地工具位于别的安装路径，用 <code>command -v ethercat</code> 和 <code>find /opt/etherlab -name ethercat -type f</code> 核查。原生 <code>ec_igb</code> 接管后，目标口可能不再表现为普通 Linux IP 接口，因此不要依赖运行时 <code>ethtool -C/-K</code> 对其施加配置；在接管前记录链路和固件状态，驱动专用参数以 IgH 文档为准。**不要给 EtherCAT 口设置 MTU 9000，也不要强制 1000 Mb/s；从站链路通常是 100 Mb/s，必须按从站及拓扑协商。**

### 阶段验收 E

- 日志明确显示 IgH 主站接管的是指定 I210 MAC，<code>DEVICE_MODULES=igb</code> 生效；管理口仍可远程访问。
- <code>ethercat master</code> 显示 link up；<code>ethercat slaves</code> 能发现预期数量、顺序和厂商/产品 ID。
- 可将从站依次推进 INIT → PREOP → SAFEOP → OP，且无持续 WKC 错误/链路 flap。
- 不允许以 generic 驱动结果冒充原生 <code>ec_igb</code> 结果。

注意：仅启动 EtherCAT 服务通常只完成总线发现，**不会自动建立实际 PDO/DC 配置并维持 OP**。OP 阶段需由真实主站应用完成；<code>ethercat states OP</code> 不能代替应用配置和持续周期收发。

## 7. 阶段 F：应用对接与 RTDM 数据路径

对于本工作区，推荐使用已有的 IgH 后端 <code>ecatuser_igh.c</code>，但需要把“构建时选择 IgH”与“周期线程保持 Cobalt primary mode”分别验证。

1. 同一构建中确保 NCK 和 HMI 都定义 <code>NCK_HAS_XENOMAI</code>，并移除 NCK 构建脚本无条件的 <code>-DNCK_HAS_PREEMPT_RT</code>。编译期可以根据 <code>--xenomai</code> 选项切换，不能只改 HMI 的 qmake 选项。
2. 使用 <code>cnchmi2/NCK/build_nck_libs_linux.sh --igh</code> 前，先把已固定标签的 IgH 源码放到脚本要求的路径，或修正构建脚本源路径；将 <code>build_igh_master_linux.sh</code> 的 configure 选项补足 <code>--enable-igb</code> 与 Xenomai 路径。
3. **最关键：** 当前 NCK IgH 分支链接 <code>-lethercat</code>。若打算从 Cobalt 主实时线程直接调用 IgH API，应改成 <code>-lethercat_rtdm</code>，并用 <code>ldd</code>、<code>readelf -d</code>、实际 mode-switch 计数验证最终装载。取得源码后核对 RTDM 例程的链接方式；若该版本没有此库或接口不同，停止并更新方案，不能猜测库名继续。
4. 检查 EtherCAT 周期内的 Qt 调用、文件/网络日志、malloc/free、SDO/邮箱操作、阻塞等待和普通 POSIX 系统调用；将非实时工作放到 Linux 后台线程，通过预分配 SPSC 队列或项目确需的 RTIPC/XDDP 通道传数据。
5. 保持 SOEM 与 IgH 后端互斥；不要让 SOEM AF_PACKET socket 和 IgH 主站同时访问同一 I210。

6. 编译 Alchemy 任务时使用 <code>xeno-config --skin=alchemy --cflags/--ldflags</code> 输出的全部选项；现有 HMI 的 <code>-lcobalt -lpthread -lrt</code> 不包含完整 Alchemy 库和初始化参数。NCK 是共享库时还需区分 <code>--auto-init-solib</code> 与主程序的初始化，参考官方例程实现一次正确初始化。
7. 主周期进入前显式执行并检查 <code>mlockall(MCL_CURRENT|MCL_FUTURE)</code>，预触页、预分配报文与线程栈；权限/内存不足时禁止投入控制。<code>rt_task_shadow()</code> 不应作为跳过这些验收的理由。
8. Xenomai 3.3.1 的正常 Alchemy API 不使用现有旧写法 <code>T_FPU | T_CPU(1)</code>；审核/改为 <code>rt_task_shadow(..., 0)</code> 加 <code>rt_task_set_affinity()</code>，例如 CPU2，并核对每个返回值。<code>unsigned long overruns = 0;</code> 与 <code>rt_task_wait_period(&overruns)</code> 的类型须匹配头文件。不能仅依靠兼容宏来宣称代码已可编译。
9. 定时器创建/绑定/RTDM 打开失败时，生产控制禁止沿用当前 <code>usleep(1000)</code> 静默降级逻辑；应拒绝伺服使能、上报故障。运行中 deadline miss 的处置按设备控制策略实施；日志由非实时线程写出。

在修改后的应用构建完成后执行（按实际路径替换）：

~~~bash
/usr/xenomai/bin/xeno-config --skin=alchemy --cflags
/usr/xenomai/bin/xeno-config --skin=alchemy --ldflags
ldd ./nck_ethercat.so | grep -E 'ethercat|alchemy|cobalt|not found'
readelf -d ./nck_ethercat.so | grep NEEDED
cat /proc/xenomai/sched/stat
~~~

这里 <code>libethercat_rtdm</code> 是必要检查之一；**链接对了仍须验证周期线程不切入 Linux 域**。先初始化/配置 PDO、DC、domain 及缓存，再转入稳定周期。在周期内使用非阻塞 receive/process → 读入 PDO → 有界控制计算 → queue/send；没有新数据时按 WKC/deadline 策略处理，不做无限等待。

DC 配置必须按每类从站 ESI 和手册设置 assignActivate、SYNC0 周期与 shift。125 µs 对应 125000 ns。设定发送相位，使帧在相应 SYNC0 截止时刻前到达；明确定义“读取上一周期输入、发送本周期/下一 SYNC0 输出”的流水线延迟，不能只凭主循环周期就宣称输入至输出延迟也是 125 µs。

在本工程中，下面两个源码上限也必须修改，**否则 125 µs 根本无法进入实测**：

- <code>rt_timer.c</code> 的 Linux/Xenomai <code>rt_timer_create()</code> 不得把 125 µs 提升到 1000 µs；校验目标值时应拒绝非法周期，而不是静默改写。
- <code>ecatuser.c</code> 和 <code>ecatuser_igh.c</code> 的 <code>ecat_set_pdo_period_us()</code> 应允许 125 µs，并审核 DC SYNC0、1 ms 累计器、重连冷却时间、超时、PDO 工作计数与全部从站最短周期。

原应用未修改时先验收 **1 ms**。修正定时器下限、Xenomai 构建和 RTDM 路径后，才逐级进入 **500 µs/250 µs**；再修正 PDO setter 并核对从站支持后进入 125 µs。不得把 Xenomai <code>latency</code> 的 125 µs 测试结果等同于应用 EtherCAT 的 125 µs 结果。

### 阶段验收 F

- NCK、HMI 和 IgH 的二进制链接依赖可追溯；周期路径未链接普通 <code>libethercat</code>。
- 周期开始/结束记录显示周期设置值就是预期值，没有被 1 ms 或 250 µs 限制截断。
- Xenomai mode-switch 计数在稳态周期循环中不持续增长；异常路径的切换单独记录。
- 实时任务没有普通网络 AF_PACKET/文件日志调用；非实时监控任务与实时核分离。

首轮在本地控制台用受控 root 测试，确认内存锁定和设备访问成功后再收紧为专用服务用户。非 root 部署需同时配置：实际 gid 对应的 <code>xenomai.allowed_group=&lt;gid&gt;</code>、memlock/调度权限、IgH RTDM 设备节点权限。systemd 服务的 <code>LimitMEMLOCK=infinity</code>、<code>LimitRTPRIO=99</code> 只设置资源限额，不会自动把线程切到 Cobalt，也不能替代 RTDM 设备授权。不要仅靠 <code>chmod 666</code> 开放控制设备。阶段 G 通过后再启用主站和控制应用开机自启；应用应在初始化成功和总线验收后才允许伺服使能。

## 8. 阶段 G：周期性能测试与 125 µs 决策门槛

### 先核算 100 Mb/s 线路时间

对常见 100BASE-TX EtherCAT 链路，定义 L 为以太网 MAC 帧总字节数（含 MAC 头和 FCS，不含前导码/帧间隔），单帧含前导码与帧间隔的线路占用近似：

~~~text
T_wire_us = (L + 8 + 12) * 8 / 100 = 0.08 * (L + 20)
~~~

| L（字节） | 线路占用（µs，计算值） |
|---:|---:|
| 64 | 6.72 |
| 256 | 22.08 |
| 512 | 42.56 |
| 1518 | 123.04 |

这是带固定假设的预算计算，**不含从站转发/线缆/DMA/计算时间**。一帧接近标准满长就已接近 125 µs 的全部线路预算；多帧需逐帧累计，并为 DC/邮箱流量留余量。因此小 PDO 拓扑才适合作为首个 125 µs 验证对象。EtherCAT 为全双工、从站在线处理，不能简单把表中数字再乘 2 当作完整往返延迟，应测实际帧返回时刻。若总线时间先已超预算，优化 CPU 唤醒抖动不能解决。

### 逐级放量

1. **1 ms：** 真实从站拓扑和完整 PDO，先完成状态机、WKC 和 DC 检查。
2. **500 µs：** 保持同一拓扑、PDO，观察过载和最坏延迟。
3. **250 µs：** 修正定时器下限后，使用原 PDO setter 允许的最短周期测量循环 WCET 与分布。
4. **125 µs：** 仅在前述源码限制与 RTDM 链路已修正、所有从站说明书允许 125 µs、足够的总线和计算余量时启动。

### 必须分开测量的量

| 指标 | 定义/采集方法 | 125 µs 初版门槛（建议值，非官方保证） |
|---|---|---|
| 唤醒误差 | 实际进入周期线程时刻减去绝对计划时刻 | 压力测试最大绝对误差 ≤ 10 µs |
| 循环 WCET | 接收 → 处理 PDO → 控制计算 → 提交发送的实测最长时间；另测线上帧/DC 截止时刻 | ≤ 80 µs（建议预算），不等于余下 45 µs 全可用于网络 |
| 结束裕度 | 下一绝对周期时刻减去本周期结束时刻 | 最小值 ≥ 20 µs |
| 周期丢失 | Xenomai overruns、应用 deadline miss | 24 h 为 0；任何一次需定位原因 |
| WKC | 每轮工作计数与期望值比较 | 24 h 无持续性错误，零漏帧目标 |
| DC 偏差 | IgH/DC 监视、从站诊断、示波器测 SYNC0 | 按最严从站规格；未拿到规格不得自定“通过” |
| 网卡/链路 | IgH 日志、内核错误、CRC/丢包统计 | 无重置、重新协商、持续 CRC 增长 |

这些初版门槛需根据伺服安全需求和从站手册确认。125 µs 的物理总线时间随帧数、PDO 字节数、从站数量、转发延迟及线缆变化；仅凭 J6412/I210 型号无法给出端到端最坏时延保证。测量建议包含 GPIO 脉冲+示波器，区分任务唤醒、总线往返、计算、DC SYNC0 四个时间点。内核跟踪与 printf 不应在正式测量的每个周期中开启。

### 压力矩阵

| 轮次 | 场景 | 时长 | 记录 |
|---|---|---:|---|
| G1 | 空载、固定温度、单从站 | ≥ 30 min | 调度与最小闭环基线 |
| G2 | 全部从站、完整 PDO、正常控制计算 | ≥ 2 h | 最大值、分位数、WKC、DC |
| G3 | CPU/内存/管理口网络/磁盘压力，EtherCAT 口独占 | ≥ 8 h | 尖峰、温度、降频、SMI 线索 |
| G4 | 热稳态、全拓扑连续运行，无人为断网 | ≥ 24 h | deadline miss=0、WKC 异常=0、无链路复位 |
| G5 | 从站上下电、断线/恢复、急停与控制进程退出（独立故障注入测试） | 按项目次数要求 | 按定义安全停机；故障期预期掉 WKC 单列，恢复不得自动误使能 |

建议保存原始直方图和全程最大值，不用平均值代替最坏值。上线前还需根据实际产品的连续运行周期延长老化时间。

24 h / 125 µs = **691200000 个周期**。计数器用 64 位便于多日老化，统计预热期/正式期/故障注入期分开。latency 和真实 EtherCAT 控制程序均要测试，但最终重负载验收应让实际应用独占目标实时核；不要把另一条高优先级 8 kHz latency 测试线程叠在同一核后误当作生产负载。

## 9. BIOS、驱动与系统调优注意事项

- **电源管理：** 先关闭 I210 EEE 和 PCIe ASPM，CPU 设 performance，再逐项比较 C-State；确认散热与功耗边界。主板 BIOS 若无关闭选项，记录限制而非声称已关闭。
- **CPU/IRQ：** Cobalt 支持 CPU 掩码与 Linux IRQ affinity 分开配置。原生 IgH <code>ec_igb</code> 的收发模型不同于普通中断驱动；先看实际 <code>/proc/interrupts</code>，不要机械地把全部 IRQ 绑到实时核。
- **线缆与交换：** 首轮使用 I210 直连第一个 EtherCAT 从站；不通过普通以太网交换机，不混入 IP 广播。
- **链路速率：** 按从站协议协商，不强制 1 Gb/s；I210 是千兆网卡不意味着 EtherCAT 从站链路 1 Gb/s。
- **时间同步：** EtherCAT DC/SYNC0 为从站同步基准；I210 的 IEEE 1588 硬件时间戳不是 DC 的直接替代。
- **安全性：** 若掉 WKC 或超 deadline，应进入应用定义的失效安全输出/急停状态，而非在周期线程内无限重试。
- **Secure Boot：** 未签名自编译内核/模块可能被 UEFI 拒绝；优先使用厂商认可的签名流程，测试机按受控流程调整 Secure Boot。

## 10. 回滚和故障定位

### 回滚顺序

1. 停止控制程序，按设备流程使伺服/输出进入安全状态。
2. <code>sudo systemctl stop ethercat</code>；确认管理网口仍可达。
3. 在 GRUB 选择旧内核；若服务开机自动启动，先在离线控制台禁用它。
4. 恢复之前备份的 <code>/etc/ethercat.conf</code>、GRUB、BIOS 和应用二进制。
5. 用阶段 A 的基线命令再次验收管理口和普通 I210 驱动。

### 典型失败对应检查

| 现象 | 优先检查 |
|---|---|
| <code>/proc/xenomai/version</code> 缺失 | 启动了旧内核、Cobalt 未启用或 prepare-kernel/config 未成功。 |
| <code>ec_master.ko</code> 无法加载 | <code>uname -r</code> 与模块 vermagic/headers 不一致、签名/Secure Boot 问题。 |
| I210 不被 IgH 接管 | MAC 配错、普通 <code>igb</code> 被其他网口占用、<code>ec_igb</code> 未构建、服务文件配置错误。 |
| 有从站但 WKC/OP 异常 | PDO/ESI、从站电源、网线、DC 配置、从站支持的最短周期。 |
| Cobalt 周期良好但 EtherCAT 抖动大 | 应用链接普通 <code>libethercat</code>、使用 generic/AF_PACKET、在周期内执行 Linux syscall/日志。 |
| 设置 125 µs 后日志仍为 1 ms | 本项目 <code>rt_timer.c</code> 与 PDO setter 的周期下限尚未修改。 |

## 11. 最终交付清单

### 可复用的阶段签收表

| 阶段 | 交付/测量项 | 通过条件 | 实际结果/证据文件 | 结论 |
|---|---|---|---|---|
| A | CPU/BIOS/PCI ID/MAC、备份与回滚 | 身份明确、旧内核可启动 | 待填写 | 待验收 |
| B | 三仓库 tag/SHA、供应商差异 | 可重复取得、来源可追溯 | 待填写 | 待验收 |
| C | 内核配置、启动日志、vermagic | Cobalt 正常、驱动与存储可用 | 待填写 | 待验收 |
| D | 125 µs 空载/压力调度日志 | 无 overrun；符合约定唤醒界限 | 待填写 | 待验收 |
| E | I210 接管、总线发现、服务日志 | 原生 ec_igb、正确从站清单 | 待填写 | 待验收 |
| F | 二进制链接、实际周期、DC/WKC、模式切换 | 全部构建限制修正，稳定 OP | 待填写 | 待验收 |
| G | 1 ms→500→250→125 µs、G1–G5 | 正式 24 h 无漏周期；故障处置通过 | 待填写 | 待验收 |

每次签收附：板卡序列号、软件完整 SHA、测试人员、日期、温度范围、PDO 字节数/帧数、从站清单、最大唤醒误差、最差完成裕度和异常次数。当前表格均为**待验收**，不能当成实机测试完成证明。

- 内核、Xenomai、IgH、应用的 Git SHA；所有补丁、<code>.config</code>、编译器版本及 SHA256。
- BIOS 版本和选项照片/清单，J6412/I210 PCI ID、MAC、从站型号与 ESI 版本。
- 安装包（内核、模块、Xenomai 用户态、IgH、应用）、服务配置和可执行回滚程序。
- 阶段 A–G 的原始日志、周期直方图、示波器截图、WKC/DC 统计及异常记录。
- 明确最终验收档位：1 ms / 500 µs / 250 µs / 125 µs；若 125 µs 未通过，记录阻断原因，**不得仅凭内核名称宣布通过**。

## 上游资料入口（需联网复核）

本轮网页查询未取得可审计内容。以下是供工程师访问核对的上游入口，标签页面、源码路径和 API 适用性须在阶段 B–F 复核；不能把列出链接当成编译或实机验证。文件检查只验证 Markdown 结构、编码及关键步骤是否完整。

1. Xenomai Dovetail 候选标签检索入口：<https://gitlab.com/Xenomai/linux-dovetail/-/tags?search=6.1.170>（<code>v6.1.170-cip55-dovetail1</code>）。
2. Xenomai 3.3.1 候选标签入口：<https://gitlab.com/Xenomai/xenomai3/xenomai/-/tags/v3.3.1>。
3. Xenomai 3 安装指南：<https://doc.xenomai.org/v3/html/README.INSTALL/index.html>。
4. Xenomai RTIPC 官方说明：<https://doc.xenomai.org/v3/html/xeno3prm/group__rtdm__ipc.html>。
5. IgH 1.6.13 候选标签入口：<https://gitlab.com/etherlab.org/ethercat/-/tags/1.6.13>。
6. IgH 1.6 原生驱动矩阵：<https://docs.etherlab.org/ethercat/1.6/doxygen/devicedrivers.html>。
7. IgH 官方手册（原生驱动、RTDM 库、配置选项、服务配置）：<https://docs.etherlab.org/ethercat/1.6/pdf/ethercat_doc.pdf>。
8. 项目现场代码：<code>cnchmi2/NCK/ethercat/rt_timer.c</code>、<code>ecatuser_igh.c</code>、<code>cnchmi2/NCK/build_nck_libs_linux.sh</code>、<code>build_igh_master_linux.sh</code>；本地代码结论以 2026-10-19 工作区状态为准。
9. Xenomai latency 参数/单位：<https://doc.xenomai.org/v3/html/man1/latency/index.html>。
10. Xenomai 3.3.1 RTIPC 配置源码：<https://gitlab.com/Xenomai/xenomai3/xenomai/-/blob/v3.3.1/kernel/drivers/ipc/Kconfig>。
11. Xenomai 3.3.1 Alchemy API 头文件：<https://gitlab.com/Xenomai/xenomai3/xenomai/-/blob/v3.3.1/include/alchemy/task.h>；构建参数：<https://gitlab.com/Xenomai/xenomai3/xenomai/-/blob/v3.3.1/scripts/xeno-config-cobalt.in>。
12. IgH 1.6.13 配置选项及服务脚本：<https://gitlab.com/etherlab.org/ethercat/-/blob/1.6.13/configure.ac>、<https://gitlab.com/etherlab.org/ethercat/-/blob/1.6.13/script/Makefile.am>、<https://gitlab.com/etherlab.org/ethercat/-/blob/1.6.13/script/ethercatctl.in>。
13. IgH 1.6.13 的 RTDM 库和例程链接定义：<https://gitlab.com/etherlab.org/ethercat/-/blob/1.6.13/lib/Makefile.am>、<https://gitlab.com/etherlab.org/ethercat/-/blob/1.6.13/examples/xenomai_posix/Makefile.am>。
14. Xenomai Cobalt 集成补丁生成选项（固定标签脚本）：<https://gitlab.com/Xenomai/xenomai3/xenomai/-/blob/v3.3.1/scripts/prepare-kernel.sh>。
15. 待核对的 Xenomai CPU 掩码与启动约束源码：<https://gitlab.com/Xenomai/xenomai3/xenomai/-/blob/v3.3.1/kernel/cobalt/init.c>。

本文件包含上游获取与部署模板、本地项目代码的静态检查结果，以及待执行的兼容性和实机验收要求。未在此 Windows 工作区编译 Linux 内核，也未连接目标 J6412/I210 或真实 EtherCAT 从站执行验收；版本资源、API、configure 选项及性能结论均以取得的固定源码和实测结果为准。
