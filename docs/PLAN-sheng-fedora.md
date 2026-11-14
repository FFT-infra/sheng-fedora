# sheng-fedora 启动与系统升级设计 (boot-upgrade.md)

> 状态：设计草案。本文只记录已定决策，不代表实现完成。

## 目标与升级模型

系统交付后的目标维护形态定为：设备装好系统后执行 `sudo dnf upgrade`，直接拉取新版用户态驱动和新内核；发布流程由 CI 自动完成，GitHub Releases 退成存档，维护者不用剪 tag，用户不用重刷完整镜像。实现路径选定方案 2。

升级管道拆成独立演进的两条轨：

| 轨道 | 分发形式 | 设备端生效动作 | 依赖条件与约束 |
| :--- | :--- | :--- | :--- |
| 用户态驱动轨 | 静态 yum 仓库托管 RPM（`createrepo_c` 生成元数据） | `dnf` 标准事务安装、更新文件至 rootfs | NVRA 必须带 commit；rootfs 预置 `.repo` 与 GPG 公钥 |
| 内核与启动载荷轨 | CI 构建「内核 Image + DTB + shim payload」RPM | `%posttrans` 在设备端重组并写回 `boot_a` | 写块设备前校验 manifest，留存完整分区备份 |

用户态驱动的 spec 目前把版本写成常数 `Version: 0.1.0`、`Release: 3%{?dist}`。不把 commit 编进 NVRA，两个不同 commit 编出来的 RPM 都叫 `0.1.0-3.fc45`，dnf 分不出新旧。CI 只有在 gate 通过后，才把验过的原包 RPM 推到静态 yum 仓库。

## 启动架构与方案选型

### kShimC 与 UEFI UKI 的机制差异

kShimC 与 nabu 使用的 UKI 是两种不同的启动载荷，当前链路不能直接套用 UKI：

```
[ UEFI UKI 启动路径（nabu） ]
ESP (FAT) -> UEFI 固件 -> 读 PE/EFI 镜像 (.linux / .initrd / .dtb / .cmdline) -> 启动内核

[ sheng 第一阶段启动路径 ]
boot_a -> ABL -> kShimC (裸机 shim) -> 读尾部 manifest (Image + DTB + 可选 initrd)
                                   -> 写 /chosen/bootargs -> 跳 ARM64 Image
```

UKI 是给 UEFI 固件执行的 PE/EFI 文件，固件从 ESP 的 FAT 按节区解析 `.linux`、`.initrd`、`.dtb`、`.cmdline`。kShimC 是 ABL 加载的裸机 shim，从 `boot_a` 末尾的 manifest 读 Linux Image、DTB 和可选 initrd，写 `/chosen/bootargs` 后直接跳 ARM64 Image；它不解析 PE、不读 FAT、不枚举 ESP。把 `.efi` 塞进 `boot_a` 当载荷，ABL 与 kShimC 都起不来。

`UniXNU/uefi-sheng` 是 Project Mu/Silicium 给 DarwinBoot 用的，README 没有 Linux、grub、systemd-boot 或 UKI 路径。要真 UKI，前提是给 sheng 弄一套能引导 Linux 的 UEFI 固件，那是另一个移植项目，前面搜过没有现成的。

方案 2 能达到的同等效果是「开机载荷由 rootfs 里的 RPM 决定」，不是 UEFI UKI。第一阶段启动链用 `tc1911/kShimC4sheng`；SunUEFI 只作为架构文档里的未来切换点。

### 启动链收敛方案（方案 2）

方案 2 保持开机路径与 kShimC 源码不变，内核更新在设备端闭环：

1. 内核 RPM 的 `%posttrans` 读取现役 `boot_a`，保留现役镜像里的 Android BaseImage，只替换 Linux payload（Image、DTB、可选 initrd）。
2. 复用现成原语：`make-boot.sh` 生成 `boot_a.img`、`fastboot flash boot_a` 的烧录逻辑、`packer/manifest.py` 读回 entries、DTB bootargs 与 initrd index。变的是执行地点从 PC 换到设备，不是新增一种启动机制。
3. 新镜像先写临时文件，完整读回 manifest 校验无误、在 rootfs 留一份完整 `boot_a` 备份，再落盘。
4. `dnf` 事务完成、重启后，固件按原路径引导新内核。

### 方案选型改判说明

上一轮按「不写块设备更安全」推荐过 kexec 跳板，漏算了二次启动对 Iris 硬件的影响，这一轮改判为方案 2：

- kexec 在这类高通平板上的二次启动没有现成验证记录。第二内核起来时，Iris VPU、时钟、display splash reservation 都不是冷启动状态。
- `fedora-sheng` 是 mkbootimg 直接跳 Image，nabu 是 UKI 直接跳，piano 也是；`sm8550.config` 里 `CONFIG_KEXEC_HANDOVER` 是关的。
- kexec 还会带出固定跳板内核的长期维护、开机多一次启动、rootfs 里的 boot counter 状态机。
- 方案 2 的原语都是 kShimC 已跑通的。

## 块设备写入安全性与回滚机制

写入块设备是方案 2 唯一真实的硬风险，压法写死：

- payload 从现役 shim 镜像提取 BaseImage 重组，新镜像不超过 `boot_a` 的 192 MiB 上限。
- 事务前查电量；电量不足就在解包前终止安装。
- 重组镜像先写临时路径，完整读回 manifest 校验无误才允许落盘。
- 落盘前把现役 `boot_a` 的完整字节流导出为备份文件存放在 rootfs。
- 任何一步失败，就让 dnf 回滚 RPM，`boot_a` 保持原样。

`dnf` 事务与分区镜像的回滚分开记：

- `dnf history undo` 只回 rootfs 内的 RPM 文件，不动 `boot_a`。
- `boot_a` 的回滚靠写盘前那份备份，恢复路径是 PC 端 `fastboot flash boot_a <backup_image>`。
- piano 的 COPR 文档里 UKI 在线升级至今没有真机记录、也没有自动回滚；我们把 RPM 状态回滚与块设备镜像回滚分成两套操作。

## 内核补丁与构建证据链

### 补丁维护规则

内核补丁按 `AGENTS.md` 的 evidence-before-capability 管理：

- 0001-0004 是针对 `sheng-7.2.6` 写的。跟最新 tip 时逐条 rebase 并 `git apply --check`；上游吸收一条就退役一条，7.3.4 已含 `REQUEST_SYNC_FRAME`，0003 可退役。
- 内核补丁只住 `qualcomm-iris-vaapi/packaging/kernel/`，由 `sheng-fedora` 和 spec 从 tag 消费。
- `sheng-fedora/docs/evidence.md` 只引用 `TEST-RESULTS.md`，不复制。

### CI 验收与落盘 provenance 校验

编译通过不能当验收通过。CI 放出去的必须是在这个 tip 上 rebase 过 0001-0004、模块哈希和完整 gate 都记进 `TEST-RESULTS.md` 的那一份：

```
上游 tip + 0001-0004 rebase -> 编译内核 -> 模块哈希 -> 完整 gate
    -> TEST-RESULTS.md -> 生成 RPM (内嵌 provenance)

设备端 %posttrans 落盘前核对 provenance：
    source commit / driver hash / module hash / boot ID /
    decoder name / frame 与 drop 计数 /
    installed package 还是 build-tree override
    全部吻合 -> 写 boot_a
    任一项不符 -> 不写分区，停在上一版，开 issue
```

- 挂了就停在上一版、开 issue。
- 驱动运行时要按实际探到的 controls 分层报 tier，VA profile 和 capability 表的公布跟 tier 绑定；现在缺这一层。

## 发布基础设施与仓库边界

### 仓库托管与签名

- 公开 COPR 不收二进制 RPM，只能从 SRPM 重建，dnf 装到的就不是 gate 过的那份。用静态 yum 仓库发验过的原包；愿意接受重建才用 COPR。
- 静态托管要一把签名私钥和 CI 写权限，按 `releasever` 分目录；Fedora 45→46 要重编，Mesa/libva 升级时也要重编。
- 驱动 README 以 Fedora 44 为 qualified baseline，`piano-fedora` 和 `fedora-sheng` 默认已经是 45，RPM 按目标 releasever 编，别把 44 的验收结论直接当 45 的证据。

### 驱动仓库接口契约

驱动仓库补 `deploy/manifest.json`，集中声明：

- 输出的 RPM 包名与命名规则。
- 依赖的 kernel patch tag。
- 内核必要配置：`CONFIG_VIDEO_QCOM_IRIS`、`CONFIG_F2FS_FS=y`、`CONFIG_SECURITY_SELINUX=y`。
- 按 `iris_driver` 的 `QUERYCAP` 找节点的规则（运行时解析，不写死节点号）。
- CI 准入执行的 gate 命令。

### 依赖锁定与执行顺序

`sources.lock.json` 记录 commit、sha256、gate run id、`checked_date`，由 CI 每次构建回写。

已定顺序，不改动：

1. 搭 `sheng-fedora` 骨架。
2. 定接口。
3. 改驱动仓库：可区分 commit 的 NVRA、`deploy/manifest.json`、runtime tier。
4. 回填 `sources.lock.json`。

原先排在第 5 步的 A 已完成：PR #9 于 2026-11-14T14:32:28Z 合入 `phxinyang/qualcomm-iris-vaapi:main`，merge commit 为 `db28991`，`fork/main` 落后其 41 个提交。`phxinyang/qualcomm-iris-vaapi` 现在是唯一权威，`sources.lock.json` 直接钉 `db28991` 或其 gate run id，不再临时钉 `FFT-infra/sheng-video-accel`。

内核补丁 tag 的归属还没跟上这次合并：`iris-kernel-patches-v1`（`580d6b9`）目前只存在于 `FFT-infra/sheng-video-accel`，`phxinyang/qualcomm-iris-vaapi` 没有任何 tag。下游 spec 按 `packaging/kernel/README.md` 取补丁会拉到旧仓库坐标。把 tag 迁到主仓库（或在主仓库打新 tag，并同步更新 `packaging/kernel/README.md` 与 `test/iris-kernel-patch-check.sh` 的引用）是定接口的前置条件。

### 项目环境现状

| 对象 | 当前状态 | 约束 |
| :--- | :--- | :--- |
| `FFT-infra/sheng-fedora` | 已有本文档，其余待建 | 本文是后续骨架的输入 |
| `piano-fedora` | 有用户未提交改动 | 不动、不覆盖 |
| `qualcomm-iris-vaapi` | `main` @ `db28991`，工作树干净 | A 已完成；尚无 `deploy/`、Release 与可下载 RPM |

## 最脆的前提与真机验收

方案最脆的前提：CI 产物能不能从现役 shim 镜像正确保留 BaseImage、重组成可启动的 `boot_a`。目前只有源码层面的推断——`packer/manifest.py` 能读回 entries 和 DTB bootargs，不等于在设备上重组后 ABL 肯加载（`inferred: did not run`）。这条验通之前不算完成；验不过就退回「内核不进 dnf、随镜像发布」。

两条硬性分支规则：

- 改回方案 1（kexec 跳板）：真机上 kexec 后 Iris 完整跑过下面四项 gate，且二次启动时间能接受。那时方案 1 更好，因为 `boot_a` 之后几乎不用再动。
- 停掉方案 2：`dd` 中途断电的恢复路径验不过，内核只能退回随镜像发布。

真机验收项：

| 验证项 | 判定 |
| :--- | :--- |
| 重组镜像一致性 | 设备端重组的临时镜像经 `packer/manifest.py` 读回，entries、DTB bootargs、initrd index 与大小逐项匹配 |
| 分区刷入与引导 | 写入 `boot_a` 后重启，`uname -r`、module hash、boot ID 与预期一致 |
| 双系统切链 | Android 与 Linux 双向切换都能起，BaseImage 未被破坏 |
| Iris gate 1 | 逐帧 MD5 与软件解码一致 |
| Iris gate 2 | 动态分辨率切换（DRC）通过 |
| Iris gate 3 | film grain 通过 |
| Iris gate 4 | frame/drop 计数符合验收记录 |
| 异常断电恢复 | 模拟写入中断后，`fastboot flash boot_a <backup_image>` 能把设备拉起 |
