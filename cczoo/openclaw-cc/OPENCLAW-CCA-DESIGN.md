# OpenClaw-CCA 部署方案设计

> 基于 Intel TDX 版 OpenClaw-CC 方案，针对 ARM CCA（Confidential Compute Architecture）平台的迁移与增强设计。
> 目标 Guest OS：**openEuler**（鲲鹏 ARM 服务器主流发行版，与 GTA RBS 同属 openEuler 社区生态）。
> Rev: 0.2（draft，待评审）

---

## 1. 概述

本方案在 OpenClaw-CC（rev-0.6）的 Intel TDX 部署基础上，针对 ARM CCA 平台做出适配，并补齐两个原方案中的信任链断点：

1. **LUKS 卷的解锁密码** —— 改为通过远程证明（CCA Realm Attestation Token, RAT）绑定下发，消除"密码静态保管"的暗门；
2. **LLM 服务 API Key** —— 改为运行期通过 RBS 拉取并注入 tmpfs，消除"明文 API Key 持久化在配置盘"的风险。

整体技术栈：

```
ARM CCA Realm (硬件 TEE) / 过渡期可用 virtCCA (Kunpeng)
    + Guest OS: openEuler (建议 22.03 LTS SP3 或更新)
    + rbc (Resource Broker Client, 已有, openEuler 社区)
    + RBS / GTA (Resource Broker Service, 已有, openEuler 社区)
    + LUKS2 (磁盘加密, 沿用)
    + systemd-cryptsetup + prepare-keyfile (LUKS 自动解锁; openEuler 不支持 keyscript)
    + tmpfs + EnvironmentFile (API Key 运行期注入)
    + SELinux (默认 enforcing, 需配套策略)
    + OpenClaw (上层应用,基本零改动)
```

---

## 2. 背景与设计目标

### 2.1 现状分析（Intel TDX 版方案）

参见 `README.md` / `README_CN.md`。该方案已经实现：

- ✓ OpenClaw 运行在 TDVM 中，运行时内存受 TDX 加密保护
- ✓ OpenClaw 配置与状态目录存放在 LUKS 加密卷
- ✓ 提供 3 个 CC TDX Skills（`check_td_runtime` / `get_td_eventlog` / `get_td_quote`）用于运行时自证
- ✓ 可对接 Trustee 远程证明服务

**两个待补齐的断点**：

| 断点 | 现状 | 风险 |
|---|---|---|
| LUKS 解锁密码 | 通过 `cryptsetup luksFormat` 交互式或脚本静态提供 | 谁拿到 host + 密码就能挂载，绕过 TEE 保护 |
| LLM API Key | 写在 `openclaw.json` 中（位于 LUKS 卷上） | 一旦 LUKS 解锁，配置文件中存在持久化的明文 Key |

### 2.2 ARM CCA 适配要点

| 维度 | TDX | CCA |
|---|---|---|
| 证明证据 | DCAP Quote | Realm Attestation Token (RAT, COSE_Sign1 + CBOR) |
| 度量寄存器 | MRTD + RTMR[0..3] | RIM (Realm Initial Measurement) + REM[0..3] |
| 取证接口 | `/dev/tdx_guest` ioctl | `RSI_ATTEST_TOKEN` (经 RMM) |
| Guest OS | Ubuntu / CentOS | **openEuler**（推荐 22.03 LTS SP3+；ARM 架构原生支持） |
| 远程证明服务 | Trustee | **RBS / GTA**（openEuler 社区项目，本方案直接复用） |
| 平台过渡 | — | 可在 **virtCCA**（Kunpeng 鲲鹏专有）上先期落地，再向标准 ARMv9 CCA 迁移；二者在 RAT 格式和 RSI 接口上保持兼容 |

注：本方案中"远程证明 + 资源下发"统一由 **GTA RBS + rbc 客户端**承担，不再依赖 Trustee。GTA 与 rbc 均为 openEuler 社区项目，与 Guest OS 选型天然契合。

### 2.3 设计目标

| # | 目标 | 含义 |
|---|---|---|
| G1 | **端到端证明绑定** | 从磁盘解锁到应用启动，每一步关键密钥的释放都受 Realm 度量约束 |
| G2 | **密钥不持久化于明文形态** | 任何敏感密钥不出现在 Realm 内存以外的位置（含加密磁盘上的明文配置项） |
| G3 | **应用零侵入** | OpenClaw 自身代码不需要改造；通过 systemd / 环境变量完成集成 |
| G4 | **故障可恢复** | RBS 不可用时有救援通路；度量升级有迁移路径 |
| G5 | **可审计可追溯** | 关键操作（取密钥、解锁、注入）有清晰的 systemd journal 记录 |

---

## 3. OpenClaw 数据资产分类

### 3.1 静态数据（Data-at-rest）—— LUKS 加密保护

| 类别 | 内容 | 保护机制 |
|---|---|---|
| 配置数据 `openclaw.json` | LLM 服务端点、各类工具/技能配置（**不再含 API Key**） | LUKS2 卷加密 |
| 长期记忆 | 用户对话历史、用户画像、知识库与向量嵌入 | LUKS2 卷加密 |
| 会话状态 | 持久化会话快照、缓存中间结果 | LUKS2 卷加密 |
| 运行时元数据 | 日志、审计记录、Skill workspace | LUKS2 卷加密 |

### 3.2 动态数据（Data-in-use）—— Realm 内存隔离保护

| 类别 | 内容 | 在内存中的特征 |
|---|---|---|
| 解密后的凭证 | LLM API Key、Channel 服务令牌（运行时从 RBS 拉取） | 长生命周期常驻 |
| LUKS 主密钥 | dm-crypt 内核态持有 | 卷打开期间常驻内核内存 |
| 会话上下文 | 当前活跃会话、Chain-of-Thought 推理状态 | 短~中生命周期 |
| 模型交互数据 | 发往 LLM 的 prompt（含用户隐私）、LLM 返回结果 | 短生命周期，密集 |
| 工具调用 I/O | Tool 调用参数与返回结果 | 短生命周期 |
| 检索片段 | 从加密记忆解密出的 RAG 片段 | 短生命周期 |
| 证明证据 | CCA RAT、REM 度量值、JWE 临时密钥对 | 临时（rbc Session Drop 时归零） |

### 3.3 传输数据（Data-in-transit）—— TLS + 证明绑定通道

- 用户 ↔ Channel Service（TLS）
- OpenClaw ↔ LLM Service（TLS；API Key 用于鉴权）
- OpenClaw ↔ Tool / Skill Service（TLS）
- **rbc ↔ RBS（TLS + JWE attestation-bound channel；本方案的核心引入）**

### 3.4 数据生命周期视角

> **关键洞察**：AI Agent 与传统单次推理任务的本质差异是 —— 大量"动态数据"在内存中**长期常驻**（API Key、当前对话上下文、LUKS 主密钥）。这正是 CCA Realm 比一次性加解密更不可替代的原因；也对密钥派发与轮换机制提出了更高要求。

---

## 4. 威胁模型与保护目标

### 4.1 威胁假设

可信：
- ✓ ARM CCA 硬件根（CPU + RMM）
- ✓ Realm 镜像构建管线（RIM 来源可信）
- ✓ RBS 服务自身（运维侧严格管控）

不可信：
- ✗ Host OS / Hypervisor / 云运维
- ✗ 同节点其他 Realm / 普通 VM
- ✗ Realm 启动镜像的传输路径（依赖 RIM 校验）
- ✗ LLM 服务端（必然看到 prompt 与 API Key 明文，本方案不在此处提供保护）

### 4.2 保护目标

| ID | 目标 | 实现机制 |
|---|---|---|
| P1 | host 重启 Realm 也无法读取 LUKS 数据 | LUKS 密码受 RBS 度量绑定下发 |
| P2 | host 内存嗅探无法读取运行时数据 | CCA Realm 内存加密 |
| P3 | 镜像被篡改后无法启动到正常状态 | RIM/REM 度量与 RBS 策略校验 |
| P4 | 密钥不被运维人员持有 | 密钥仅存于 RBS 与 Realm 内存 |
| P5 | 单次密钥泄露的影响最小化 | 密钥与 Realm 实例生命周期绑定，不写入持久存储 |

### 4.3 不在保护范围内

- ✗ LLM 服务端的数据使用与日志（属于其他信任域，需另行通过 Confidential Inference 解决）
- ✗ Prompt Injection 等应用层攻击
- ✗ 工具/插件供应链审查（属于工程治理范畴）

---

## 5. 总体架构

### 5.1 组件清单

| 组件 | 位置 | 职责 |
|---|---|---|
| ARM CCA Realm | 云端节点 | 硬件 TEE 容器 |
| Realm Root FS | Realm 内 | 包含 rbc-cli / cryptsetup / systemd / OpenClaw 等基础组件 |
| LUKS2 卷 | Realm 内 `/home/encrypted_storage` | 持久化 OpenClaw 配置、状态、长期记忆 |
| **rbc-cli** | Realm 内 `/usr/local/bin/` | 基于 rbc Rust SDK 封装的 CLI 工具，启动期向 RBS 拉取密钥 |
| **rbc-keyscript** | Realm 内 `/usr/local/sbin/` | systemd-cryptsetup 调用的 keyscript，包装 rbc-cli |
| systemd 单元链 | Realm 内 `/etc/systemd/system/` | 编排 loop bind → LUKS 解锁 → 挂载 → API Key 注入 → OpenClaw 启动 |
| OpenClaw daemon | Realm 内 | 上层 Agent 业务（基本不感知 CC 改造） |
| CC CCA Skills | OpenClaw workspace | 运行期自证能力（替代原 CC TDX Skills） |
| **GTA RBS** | 受信运维域 | 远程证明 + 资源下发服务 |

### 5.2 信任传递链

```
ARM CCA Hardware Root  (CPU + RMM 签发 Platform Token)
        │
        ▼
Realm Boot Measurement (RIM/REM 反映镜像与启动序列)
        │
        ▼
rbc 在 Realm 内调 RSI_ATTEST_TOKEN  →  得到 RAT
        │
        ▼
RAT 提交给 RBS,RBS 验证:
    - Platform Token 链路签名
    - Realm Token 签名 + 度量是否匹配策略
        │
        ▼
RBS 用 JWE 加密资源 (LUKS passphrase / API Key)
       并下发,只有 Realm 内的临时密钥能解密
        │
        ▼
Realm 内消费资源:
    - LUKS passphrase  → cryptsetup → 解锁卷
    - API Key          → tmpfs       → OpenClaw 进程 env

任意一环度量不符  →  RBS 拒绝下发  →  Realm 进入 emergency,无法工作
```

### 5.3 架构图

```
┌── 受信运维域 ─────────────────────────────────┐
│  RBS (GTA Resource Broker Service)            │
│   ├─ 资源: luks-passphrase-openclaw-prod      │
│   ├─ 资源: openai-api-key-prod                │
│   └─ 策略: 绑定 Realm 度量 + 速率限制         │
└─────────────────────┬─────────────────────────┘
                      │ HTTPS + JWE
                      │ (RAT-bound)
                      │
┌── ARM CCA Realm (Hardware TEE) ───────────────┴──┐
│                                                   │
│  ┌── Realm Root FS (内置) ───────────────────┐  │
│  │  /usr/local/bin/rbc-cli                    │  │
│  │  /usr/local/sbin/rbc-keyscript             │  │
│  │  /etc/rbc/{rbc.yaml, rbs-ca.crt}           │  │
│  │  /usr/sbin/cryptsetup                      │  │
│  │  /etc/systemd/system/openclaw-*.service    │  │
│  └────────────────────────────────────────────┘  │
│                                                   │
│  ┌── 启动期 (Bootstrap, 一次性) ────────────────┐ │
│  │ ① losetup /root/vfs                         │ │
│  │ ② cryptsetup luksOpen --keyscript=rbc-...   │ │
│  │    (内部: rbc-cli → RAT → RBS → passphrase) │ │
│  │ ③ mount /home/encrypted_storage             │ │
│  │ ④ rbc-cli get-resource openai-api-key       │ │
│  │    → /run/openclaw/llm.env  (tmpfs, 0600)   │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  ┌── 运行期 ──────────────────────────────────┐ │
│  │ openclaw daemon                             │ │
│  │   - EnvironmentFile=/run/openclaw/llm.env   │ │
│  │   - 读  /home/encrypted_storage/.openclaw/  │ │
│  │   - CC CCA Skills (运行时自证)              │ │
│  │   - 调 LLM Service (HTTPS,持有 API Key)     │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  ┌── /home/encrypted_storage (LUKS2 mounted) ──┐ │
│  │  .openclaw/openclaw.json  (无 API Key 字段) │ │
│  │  .openclaw/workspace/                       │ │
│  │  .openclaw/sessions/, logs/, memory/        │ │
│  └─────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────┘
```

---

## 6. 关键设计决策

### 6.1 取密钥不能由 OpenClaw Agent 路径完成

**问题**：OpenClaw 是依赖 LLM 推理的 Agent，"通过 Skill 调 RBS 取 API Key"会形成循环依赖（取 Key → 需 main agent 路由消息 → 需 LLM → 需 API Key）。

**决策**：**取密钥放在 OpenClaw 启动之前的 Bootstrap 阶段**，由独立的 systemd 单元 + rbc-cli 完成，与 Agent loop 解耦。

**类比**：与 nginx 不应自己调 KBS 取 TLS 证书、而由 init 流程把证书放好同理 —— 取密钥是 infrastructure-layer 职责，不是 application-layer 职责。

**对现有 CC Skills 的影响**：CC TDX Skills（`check_td_runtime` / `get_td_eventlog` / `get_td_quote`）的角色重新定位为 **运行期/用户主动触发的自证能力**，不再承担启动期密钥获取。CCA 版相应改名为 `check_realm_runtime` / `get_realm_eventlog` / `get_realm_token`。

### 6.2 LUKS 解锁密码也纳入 rbc

**问题**：仅保护 API Key、LUKS 仍用静态密码，相当于"前门指纹锁，后门花盆下藏钥匙"——攻击者拿到 host + 密码即可绕过整个 TEE 信任链挂载卷。

**决策**：LUKS 解锁密码作为 RBS 资源（`luks-passphrase-openclaw-prod`），与 API Key 同样受 Realm 度量绑定释放。

**收益**：
- Realm 度量不符 → RBS 拒绝下发 → 卷物理不可解
- 没有任何运维人员持有该密码（生成后立刻仅交给 RBS）
- 升级镜像需更新 RBS 策略，而非重设每个卷的密码

**首次部署关键约束**：密码生成后立刻只交给 RBS，运维终端不留副本（参见 §8）。

### 6.3 运行时密钥落 tmpfs 而非 LUKS 卷

**问题**：rbc 拉到的 API Key 写到 `openclaw.json`（LUKS 卷上）虽然简单，但创建了"超出 Realm 实例生命周期的持久副本"，弱化了"证明绑定"的语义（首次启动证明 → 之后任意重启都用旧 Key）。

**决策**：API Key 写入 **tmpfs（`/run/openclaw/llm.env`）**，OpenClaw 通过 systemd `EnvironmentFile=` 接收。

**关键性质**：
- tmpfs 文件存储在 Realm 加密内存页中
- Realm shutdown → 内存释放 → 文件彻底消失，**无任何持久副本**
- 每次 Realm 启动都重新走 RBS 证明流程，密钥与当次实例严格绑定

**必须配套**：**关闭 swap**（或确保 swap 在 Realm 加密边界内），否则 tmpfs 页可能被换出到普通磁盘。

### 6.4 LUKS 解锁路径选型：openEuler 上必须用 prepare-keyfile

**问题**：`/etc/crypttab` 的 `keyscript=` 选项在不同发行版中由不同程序处理：

| 发行版 | keyscript 处理者 | 是否支持 |
|---|---|---|
| Debian / Ubuntu | `cryptsetup-initramfs` 提供的 `cryptdisks_start` | ✓ |
| RHEL / CentOS / Fedora / **openEuler** | `systemd-cryptsetup`（不识别 keyscript=） | ✗ |

**决策**：**openEuler 上以 prepare-keyfile (systemd-native) 为唯一主路径**。流程：
1. `openclaw-prepare-luks-key.service` 调 rbc-cli 把密码写入 `/run/cryptsetup/openclaw_data.key`（tmpfs，0600）；
2. systemd-cryptsetup 用 `key-file=` 解锁；
3. `openclaw-luks-unlock.service` 解锁完成后立即 `shred -u` 销毁该文件。

**与 keyscript 路径的取舍**：

| 维度 | keyscript（不可用于 openEuler） | prepare-keyfile（openEuler 主路径） |
|---|---|---|
| 密码是否落文件 | ✗ 完全不落 | ✓ 短暂落 tmpfs（毫秒~秒级） |
| 失败原子性 | 无中间状态 | 用 `RuntimeDirectory=` + `ExecStartPost=shred -u` 原子化 |
| openEuler 兼容性 | ✗ | ✓ |

prepare-keyfile 路径中"密码短暂落 tmpfs"的暴露窗口由 systemd 单元链严格约束：
- 文件仅存在于 unlock 过程中（通常 < 1 秒）；
- 文件位置 `/run/cryptsetup/`（tmpfs，物理上是 Realm 加密内存）；
- 解锁完成后立刻 `shred -u`；
- 同 Realm 内只有 root 可读（mode 0600）。

可接受。

---

## 7. 详细设计

### 7.1 启动信任链（systemd 依赖图）

```
network-online.target ──┐
                        ├──> openclaw-loop.service
                        │       (losetup /root/vfs)
                        │
                        ├──> openclaw-luks-unlock.service
                        │       (cryptdisks_start openclaw_data)
                        │       (内部 keyscript → rbc-cli → RBS)
                        │
                        ├──> home-encrypted_storage.mount
                        │       (mount /dev/mapper/openclaw_data)
                        │
                        ├──> openclaw-fetch-apikey.service
                        │       (rbc-cli → /run/openclaw/llm.env)
                        │
                        ├──> openclaw.service
                        │       (EnvironmentFile=...; openclaw daemon)
                        ▼
                multi-user.target
```

任意一步失败 → systemd 不进入 multi-user.target → Realm 进入 emergency mode，可介入救援（见 §10）。

### 7.2 LUKS 解锁（openEuler / systemd-native 路径）

#### 7.2.0 绑定 loop 设备

OpenClaw-CC 的加密卷做在虚拟磁盘文件 `/root/vfs` 中（沿用原 TDX 方案）。`cryptsetup` 只能作用于块设备，所以需要先把文件绑定到 loop 设备：

```ini
# /etc/systemd/system/openclaw-loop.service
[Unit]
Description=Bind LUKS-backing file to loop device for OpenClaw
DefaultDependencies=no
After=local-fs.target
Before=openclaw-prepare-luks-key.service

[Service]
Type=oneshot
RemainAfterExit=yes
# 生产建议:固定 loop 设备号,与 crypttab 第 2 字段保持一致
# 避免 --find 自动选号导致与 crypttab 写死的设备号不匹配
ExecStart=/sbin/losetup /dev/loop7 /root/vfs
ExecStop=/sbin/losetup --detach /dev/loop7

[Install]
WantedBy=multi-user.target
```

> **loop 号一致性的坑**：`losetup --find --show` 会自动挑空闲号（loop0/loop1/loop7…），但 `crypttab` 里写死了 `/dev/loop0`，一旦 loop0 被别的进程占用就会错位。**生产固定写死 loop 号**（本设计统一用 `/dev/loop7`），并同步修改 §7.2.1 crypttab 中的设备号。
>
> **为何用文件而非真盘**：沿用 TDX 版方案的灵活部署假设——节点不必预留独立分区/云盘。如果未来切换到独立块设备（如挂载云盘 `/dev/vdb1`），此 service 可整体移除，对其他单元无影响。

#### 7.2.1 crypttab

**crypttab**（openEuler）：

```
# /etc/crypttab
openclaw_data  /dev/loop7  /run/cryptsetup/openclaw_data.key  luks,noauto,tries=1,timeout=30
```

第 3 字段是 keyfile 路径（不是 RBS resource_id），由 `openclaw-prepare-luks-key.service` 在解锁前一步写入。

#### 7.2.2 单元链

```ini
# /etc/systemd/system/openclaw-prepare-luks-key.service
[Unit]
Description=Prepare LUKS key in tmpfs for OpenClaw
Requires=openclaw-loop.service network-online.target
After=openclaw-loop.service network-online.target
Before=openclaw-luks-unlock.service

[Service]
Type=oneshot
RemainAfterExit=yes
RuntimeDirectory=cryptsetup
RuntimeDirectoryMode=0700
ExecStart=/usr/local/bin/rbc-cli \
    --config /etc/rbc/rbc.yaml \
    get-resource --resource luks-passphrase-openclaw-prod \
    --output /run/cryptsetup/openclaw_data.key
```

```ini
# /etc/systemd/system/openclaw-luks-unlock.service
[Unit]
Description=Unlock OpenClaw LUKS volume
Requires=openclaw-prepare-luks-key.service
After=openclaw-prepare-luks-key.service
Before=home-encrypted_storage.mount

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/systemctl start systemd-cryptsetup@openclaw_data.service
# 解锁完成立即销毁 keyfile
ExecStartPost=/usr/bin/shred -u /run/cryptsetup/openclaw_data.key
ExecStartPost=-/usr/bin/rmdir /run/cryptsetup
ExecStop=/usr/bin/systemctl stop systemd-cryptsetup@openclaw_data.service
```

> **不支持 keyscript 的影响范围**：仅是 openEuler 上的实现路径变化，对外接口（rbc-cli 调用语义、RBS 资源策略、Realm 度量绑定）完全不变。Debian/Ubuntu 的 keyscript 实现仅作为附录参考保留（见附录 C）。

### 7.3 API Key 注入

```ini
# /etc/systemd/system/openclaw-fetch-apikey.service (摘要)
[Service]
Type=oneshot
RemainAfterExit=yes
RuntimeDirectory=openclaw          # 创建 /run/openclaw/, mode=0700
User=openclaw
Group=openclaw
ExecStart=/usr/local/bin/rbc-cli get-resource \
    --resource openai-api-key-prod \
    --output /run/openclaw/llm.env.key
ExecStartPost=/bin/sh -c 'printf "OPENAI_API_KEY=%s\n" "$(cat /run/openclaw/llm.env.key)" > /run/openclaw/llm.env && chmod 0600 /run/openclaw/llm.env && shred -u /run/openclaw/llm.env.key'
```

```ini
# /etc/systemd/system/openclaw.service (摘要)
[Service]
EnvironmentFile=/run/openclaw/llm.env
Environment=OPENCLAW_STATE_DIR=/home/encrypted_storage/.openclaw
Environment=OPENCLAW_CONFIG_PATH=/home/encrypted_storage/.openclaw/openclaw.json
ExecStart=/usr/bin/openclaw daemon
```

**前置假设**：OpenClaw 支持从环境变量（如 `OPENAI_API_KEY`）读取 LLM 凭证，或支持 `${ENV:OPENAI_API_KEY}` 占位符语法。若不支持，需在 OpenClaw 侧做最小改造（**待源码确认**，列入后续工作）。

### 7.4 Realm 镜像内置组件清单

启动期所需的组件**必须**打包到 Realm Root FS（不能放在 LUKS 卷内）：

| 组件 | 路径 | openEuler 包名 / 说明 |
|---|---|---|
| rbc-cli 二进制 | `/usr/local/bin/rbc-cli` | 静态链接优先；从 rbc 仓库构建 |
| rbc 配置 | `/etc/rbc/rbc.yaml` | RBS endpoint + provider 配置 |
| RBS CA 证书 | `/etc/rbc/rbs-ca.crt` | TLS trust anchor |
| cryptsetup | `/usr/sbin/cryptsetup` | `dnf install cryptsetup` |
| dracut | `/usr/bin/dracut` | `dnf install dracut`（initramfs 构建） |
| systemd units | `/etc/systemd/system/openclaw-*.service` | 启动链 |
| SELinux 策略 | `/etc/selinux/targeted/.../openclaw.pp` | 自定义模块（见 §9.6） |
| Node.js + pnpm | `/usr/bin/node`, `/usr/local/bin/pnpm` | NodeSource RPM 或 openEuler 仓库 |
| OpenClaw 主程序 | `/usr/bin/openclaw` | 已通过 `pnpm link --global` 安装 |
| CCA 取证内核接口 | `/dev/`（kernel module） | openEuler 内核需启用 `CONFIG_ARM_CCA_GUEST` 等 |

**openEuler 镜像构建建议**：用 `dracut` 生成包含上述组件的 initramfs，rootfs 保持精简；或用 `mkosi` / `osbuild` 构建标准化镜像。详细构建步骤见 §7.5。Realm Initial Measurement (RIM) 即由该镜像计算得出，需在升级 RBS 策略时同步更新。

### 7.5 镜像构建：将启动期组件塞进 initramfs

#### 7.5.1 为什么要进 initramfs

现有方案中，`/home/encrypted_storage` 是 LUKS 加密卷，而 rootfs 不加密——理论上启动期组件可以仅放在 rootfs 上。但仍然推荐**至少把关键启动组件打进 initramfs**，原因：

1. **直接受 RIM 度量保护** —— initramfs 内容是 Realm 初始内存的一部分，会被 RMM 计入 RIM。任何篡改即被 RBS 策略校验拒绝下发密钥。
2. **减少对 rootfs 完整性的依赖** —— 在未启用 dm-verity 之前（本设计 P0 工作项，见 §12），rootfs 完整性无法保证；打进 initramfs 等价于"用 RIM 替代 dm-verity"作为完整性保障。
3. **缩短"未度量"窗口** —— rootfs 切换之前的所有逻辑都被 RIM 覆盖，缩短攻击者可利用的时窗。

#### 7.5.2 dracut 配置

openEuler 默认用 `dracut` 生成 initramfs。新增配置文件：

```ini
# /etc/dracut.conf.d/openclaw-cca.conf

# === 启动期需要的二进制 ===
install_items+=" /usr/local/bin/rbc-cli "
install_items+=" /usr/sbin/cryptsetup "
install_items+=" /sbin/losetup "
install_items+=" /usr/bin/shred "

# === 启动期需要的配置/证书 ===
install_items+=" /etc/rbc/rbc.yaml "
install_items+=" /etc/rbc/rbs-ca.crt "
install_items+=" /etc/crypttab "

# === 启动期 systemd unit ===
install_items+=" /etc/systemd/system/openclaw-loop.service "
install_items+=" /etc/systemd/system/openclaw-prepare-luks-key.service "
install_items+=" /etc/systemd/system/openclaw-luks-unlock.service "

# === dracut 必需模块 ===
add_dracutmodules+=" crypt systemd network-manager "

# === 关掉不必要模块: 缩小攻击面 + 提高 RIM 可重放性 ===
omit_dracutmodules+=" plymouth lvm mdraid dm-multipath i18n "

# === 通用镜像而非主机绑定 (Realm 间可复用) ===
hostonly="no"
hostonly_cmdline="no"

# === 压缩算法固定,避免 RIM 漂移 ===
compress="zstd"
```

> **dracut 自动处理依赖**：`install_items` 列出的二进制，dracut 会通过 `ldd` 自动把所需动态库一并打入。但 **rbc-cli 推荐静态链接**（musl 构建），消除运行期对 glibc 版本/路径的依赖，也让 RIM 更稳定。

#### 7.5.3 启用 systemd 单元

打进 initramfs 还不够，要让它们在启动期被 systemd 拉起。在镜像构建期（chroot 中）执行：

```bash
systemctl enable openclaw-loop.service
systemctl enable openclaw-prepare-luks-key.service
systemctl enable openclaw-luks-unlock.service
systemctl enable openclaw-fetch-apikey.service
systemctl enable openclaw.service
```

`systemctl enable` 会在 `/etc/systemd/system/<target>.wants/` 下创建符号链接，这些链接也需要 dracut 一并带入（默认行为）。

#### 7.5.4 重新生成 initramfs

```bash
# 全量重建所有内核版本
sudo dracut --force --regenerate-all

# 或仅针对当前内核
sudo dracut --force /boot/initramfs-$(uname -r).img $(uname -r)

# 验证内容是否打进去
sudo lsinitrd /boot/initramfs-$(uname -r).img | grep -E 'rbc-cli|cryptsetup|openclaw-'
# 期望看到:
#   usr/local/bin/rbc-cli
#   usr/sbin/cryptsetup
#   etc/systemd/system/openclaw-loop.service
#   etc/systemd/system/openclaw-prepare-luks-key.service
#   ...
```

#### 7.5.5 RIM 计算与 RBS 策略联动

initramfs 一旦变化（升级 rbc-cli、补漏 unit、改 dracut 配置），RIM 必然改变。**镜像构建管线必须同步更新 RBS 策略**：

```bash
# 1. 构建期计算预期 RIM
expected_rim=$(rmm-rim-calc \
    --kernel   /boot/vmlinuz-${KVER} \
    --initrd   /boot/initramfs-${KVER}.img \
    --cmdline  "$(cat /etc/kernel/cmdline)" \
    --rpv      ./rpv.bin)
echo "New RIM: $expected_rim"

# 2. 添加到 RBS 资源策略 (灰度期允许新旧并存)
rbs-admin update-policy --resource luks-passphrase-openclaw-prod \
    --add-allowed-rim "$expected_rim"
rbs-admin update-policy --resource openai-api-key-prod \
    --add-allowed-rim "$expected_rim"

# 3. 滚动重启 Realm, 验证新 RIM 能取到密钥
# 4. 全部切换完成后, 从策略中移除旧 RIM
rbs-admin update-policy --resource luks-passphrase-openclaw-prod \
    --remove-allowed-rim "$old_rim"
```

详见 §10.4 度量升级路径。

#### 7.5.6 可重放构建建议

为了**任何人独立构建都能得到相同的 RIM**（也是 RBS 策略可信的前提），需要消除非确定性来源：

| 非确定性来源 | 消除措施 |
|---|---|
| 时间戳 | 设 `SOURCE_DATE_EPOCH=<固定值>` 环境变量 |
| 文件 mtime | dracut 已支持，无需额外操作 |
| 包版本 | 锁定 `dnf install <pkg>-<version>-<release>.<arch>` |
| 依赖库 | `hostonly=no` + 显式 `install_items` |
| 压缩算法 | 固定 `compress=zstd` |
| 构建器版本 | 固定 dracut / kernel / openEuler base 版本 |
| 文件系统排序 | 用 `tar --sort=name` / cpio 排序选项 |

工具链推荐：**`mkosi`** 或 **`osbuild`**（openEuler 都支持），或 Docker BuildKit 配合 `--reproducible` 标志。可重放性是后续做 **SLSA 工件溯源**（见 §12 P3）的前置条件。

#### 7.5.7 验证清单

```bash
[ ] sudo lsinitrd /boot/initramfs-$(uname -r).img | grep rbc-cli      # 存在
[ ] sudo lsinitrd /boot/initramfs-$(uname -r).img | grep cryptsetup   # 存在
[ ] sudo lsinitrd /boot/initramfs-$(uname -r).img | grep openclaw-    # unit 文件存在
[ ] systemctl is-enabled openclaw-loop.service                        # enabled
[ ] systemctl is-enabled openclaw-luks-unlock.service                 # enabled
[ ] 两次独立构建对比 RIM,值应一致 (可重放性)
[ ] 故意改动 initramfs (改一字节后重新打包),RIM 应变化,
    RBS 应拒绝下发密钥 → Realm 进入 emergency
```

### 7.6 OpenClaw 内的 CC CCA Skills

继承原 CC TDX Skills 的设计模式，为 CCA 改写：

| 原 TDX Skill | CCA 版 Skill | 用途 |
|---|---|---|
| `check_td_runtime` | `check_realm_runtime` | 检测当前是否在 CCA Realm 中运行；展示 cc-attest 链路健康度 |
| `get_td_eventlog` | `get_realm_eventlog` | 导出 REM extend 历史（与 RIM 一起替代 TDX event log + RTMR） |
| `get_td_quote` | `get_realm_token` | 取 RAT，可保存为 `realm_token.json` 供独立验证 |
| —（新增） | `refresh_credentials` | 触发 rbc 强制刷新 API Key（应急运维用） |

**职责定位**：这些 Skill 仅服务于"运行期用户/审计员主动触发的自证能力"，与启动期密钥获取严格分离。

---

## 8. 首次部署流程

### 8.1 准备阶段（在受信运维终端，一次性）

```bash
# 0. 在 openEuler 上安装依赖
sudo dnf install -y cryptsetup openssl util-linux e2fsprogs

# 1. 生成 LUKS passphrase 和 API Key
LUKS_PASS=$(openssl rand -base64 32)
# (OPENAI_API_KEY 来自外部 LLM 服务管理后台)

# 2. 创建并 format LUKS2 卷（可选启用 dm-integrity）
sudo dd if=/dev/zero of=/root/vfs bs=1M count=10240
LOOP=$(sudo losetup --find --show /root/vfs)
printf '%s' "$LUKS_PASS" | sudo cryptsetup luksFormat \
    --type luks2 --cipher aes-xts-plain64 --key-size 512 \
    --hash sha256 --use-random --batch-mode --key-file=- "$LOOP"

# 3. 创建 ext4
printf '%s' "$LUKS_PASS" | sudo cryptsetup luksOpen "$LOOP" openclaw_data --key-file=-
sudo mkfs.ext4 /dev/mapper/openclaw_data
sudo cryptsetup luksClose openclaw_data

# 4. 加救援 keyslot（应急通路）
sudo cryptsetup luksAddKey "$LOOP" --key-slot 7   # 输入 rescue passphrase

# 5. 注册资源到 RBS, 绑定 Realm 度量策略
rbs-admin register-resource \
    --id "luks-passphrase-openclaw-prod" \
    --value-stdin <<<"$LUKS_PASS" \
    --policy-file ./openclaw-realm-policy.json

rbs-admin register-resource \
    --id "openai-api-key-prod" \
    --value-stdin <<<"$OPENAI_API_KEY" \
    --policy-file ./openclaw-realm-policy.json

# 6. 销毁运维终端上的所有副本
unset LUKS_PASS OPENAI_API_KEY; history -c
```

> **注意（openEuler）**：默认 `firewalld` 启用、`SELinux` enforcing，需为 rbc-cli 出站到 RBS 放行端口，并为 `/usr/local/bin/rbc-cli`、`/run/openclaw/`、`/run/cryptsetup/` 配置 SELinux 上下文（详见 §9.6）。

### 8.2 RBS 资源策略示例

```json
{
  "platform": "arm-cca",
  "expected_rim": "0x<Realm Initial Measurement>",
  "expected_rems": {
    "0": "0x<REM[0] expected>",
    "1": "0x<REM[1] expected>"
  },
  "audience": ["openclaw-prod"],
  "rate_limit": {
    "max_per_realm_per_day": 50,
    "max_concurrent_sessions": 5
  },
  "valid_until": "2026-12-31T00:00:00Z"
}
```

### 8.3 Realm 启动验证

```bash
[ ] systemctl is-active openclaw.service                # active (running)
[ ] mount | grep /home/encrypted_storage                # 已挂载
[ ] ls /run/openclaw/                                   # llm.env 存在,0600
[ ] ls /run/cryptsetup/ 2>/dev/null                     # 不存在 (key 已销毁)
[ ] grep OPENAI_API_KEY /proc/$(pidof openclaw)/environ # 注入成功
[ ] cat /proc/$(pidof openclaw)/cmdline                 # 命令行无密钥
[ ] journalctl -u openclaw-luks-unlock.service | grep -i passphrase  # 日志无密码出现
```

---

## 9. 安全加固

### 9.1 密钥生命周期

| 层 | 措施 |
|---|---|
| rbc 内部缓冲 | `zeroize` crate 做归零（rbc 已实现） |
| 传给 cryptsetup | 用 stdin pipe 而非命令行参数（避免 `/proc/<pid>/cmdline` 泄露） |
| 必须落文件时 | 仅写 tmpfs；用完 `shred -u` |
| 进程内存 | **关闭 swap** + 禁用 coredump |
| journal 输出 | rbc/keyscript stderr 严禁 dump 密钥 |

### 9.2 系统加固

```bash
# 关 swap
swapoff -a; sed -i 's/^[^#].*swap.*/#&/' /etc/fstab

# 禁 coredump
echo '* hard core 0' >> /etc/security/limits.conf
echo 'kernel.core_pattern=|/bin/false' >> /etc/sysctl.d/99-cc.conf
```

### 9.3 systemd 沙箱

OpenClaw service 启用：
- `ProtectSystem=strict` + `ReadWritePaths=/home/encrypted_storage/.openclaw`
- `PrivateTmp=yes` / `NoNewPrivileges=yes`
- `ProtectKernelTunables=yes` / `ProtectKernelModules=yes`
- `MemoryDenyWriteExecute=yes`
- `RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6`
- `SystemCallArchitectures=native`

### 9.4 文件权限

| 路径 | mode | owner |
|---|---|---|
| `/usr/local/sbin/rbc-keyscript` | 0700 | root:root |
| `/usr/local/bin/rbc-cli` | 0750 | root:root |
| `/etc/rbc/rbc.yaml` | 0600 | root:root |
| `/etc/rbc/rbs-ca.crt` | 0644 | root:root |
| `/run/openclaw/` | 0700 | openclaw:openclaw |
| `/run/openclaw/llm.env` | 0600 | openclaw:openclaw |
| `/home/encrypted_storage/.openclaw/openclaw.json` | 0600 | openclaw:openclaw |

### 9.5 LUKS2 加固

推荐启用 dm-integrity 防止离线密文篡改：

```bash
cryptsetup luksFormat --type luks2 \
    --integrity hmac-sha256 --cipher aes-gcm-random ...
```

写性能下降可接受（OpenClaw IO 不密集）。

### 9.6 SELinux 加固（openEuler 默认 enforcing）

openEuler 默认启用 SELinux targeted 策略，未做配置时 `rbc-cli` / `cryptsetup` / OpenClaw 等可能因策略缺失被拒绝。需要：

#### 标签上下文

```bash
# rbc-cli 二进制
sudo semanage fcontext -a -t bin_t '/usr/local/bin/rbc-cli'
sudo restorecon -v /usr/local/bin/rbc-cli

# 运行期 tmpfs 目录(systemd RuntimeDirectory= 自动创建,但需要正确标签)
sudo semanage fcontext -a -t var_run_t '/run/openclaw(/.*)?'
sudo semanage fcontext -a -t var_run_t '/run/cryptsetup(/.*)?'

# rbc 配置目录
sudo semanage fcontext -a -t etc_t '/etc/rbc(/.*)?'
sudo restorecon -Rv /etc/rbc

# 加密存储挂载点(允许 OpenClaw 在其中读写)
sudo semanage fcontext -a -t user_home_dir_t '/home/encrypted_storage'
sudo semanage fcontext -a -t user_home_t '/home/encrypted_storage(/.*)?'
sudo restorecon -Rv /home/encrypted_storage
```

#### 自定义策略模块

把 OpenClaw / rbc-cli 的网络出站、读取 `/dev/`（CCA 取证设备）等访问以最小集合放开。模板 `openclaw.te`（按实际审计日志增删）：

```
module openclaw 1.0;

require {
    type unconfined_t;
    type bin_t;
    type var_run_t;
    type http_port_t;     # RBS / LLM API
    type self;
    class file { read open execute getattr };
    class tcp_socket { name_connect };
    class capability { ipc_lock };
}

# 允许 rbc-cli 出站到 RBS / LLM API
allow unconfined_t http_port_t:tcp_socket name_connect;

# 允许 mlock 锁页(防止密钥被 swap)
allow unconfined_t self:capability ipc_lock;
```

```bash
# 编译 + 加载
checkmodule -M -m -o openclaw.mod openclaw.te
semodule_package -o openclaw.pp -m openclaw.mod
sudo semodule -i openclaw.pp
```

#### 调试方法

```bash
# 实时看 SELinux denial
sudo ausearch -m avc -ts recent
# 或:
sudo journalctl -t setroubleshoot

# 用 audit2allow 自动生成补丁
sudo ausearch -m avc -ts recent | audit2allow -M openclaw-extra
sudo semodule -i openclaw-extra.pp
```

> **建议**：开发期可先用 `setenforce 0`（permissive）跑通整链路，把所有 denial 收集起来，再用 `audit2allow` 一次性生成最小策略，最后切回 enforcing。

---

## 10. 故障与降级

### 10.1 失败模式

| 失败 | 现象 | 自动行为 | 介入方式 |
|---|---|---|---|
| 网络不通 | rbc-cli 超时 | systemd 单元失败，进入 emergency | 手动用 rescue keyslot |
| RBS 拒绝（度量错） | rbc 返回非 0 | 同上 | 检查策略 vs 实际度量 |
| TLS/CA 校验失败 | TLS handshake 错 | 同上 | 更新 `/etc/rbc/rbs-ca.crt` |
| LUKS keyslot 错 | cryptsetup "No key available" | 同上 | rescue keyslot 或 `luksDump` 排查 |
| loop 设备未绑定 | cryptsetup "device not found" | openclaw-loop 失败 | 检查 `/root/vfs` 路径 |
| **SELinux denial（openEuler）** | rbc-cli/cryptsetup 调用被 SELinux 拒 | systemd 单元失败 | `ausearch -m avc -ts recent` + `audit2allow` |
| **firewalld 拦截** | 出站到 RBS 失败 | rbc-cli 网络错误 | `firewall-cmd --add-rich-rule` 放行 |

### 10.2 救援通路

LUKS keyslot 7 的 rescue passphrase 用 Shamir 分片或离线保险柜保管：

```bash
# Realm emergency console 内
# 注意:正常启动时 openclaw-loop.service 已绑 /dev/loop7;
#       如未绑定(systemd 没起来),先手动绑
losetup /dev/loop7 /root/vfs 2>/dev/null || true
cryptsetup luksOpen /dev/loop7 openclaw_data        # 输入 rescue passphrase
mount /dev/mapper/openclaw_data /home/encrypted_storage
# 决策:RBS 故障等其恢复 / 度量变更修复策略 / 怀疑入侵则销毁重建
```

### 10.3 防止启动期循环

```ini
[Unit]
StartLimitIntervalSec=300
StartLimitBurst=3
[Service]
Restart=no   # 失败即进 emergency,不死循环
```

`tries=1`（在 crypttab）+ `Restart=no`。

### 10.4 度量升级路径

| 策略 | 做法 | 适用 |
|---|---|---|
| 多度量白名单 | RBS 资源策略允许 N 个版本度量同时通过 | 滚动升级期 |
| 预热新度量 | 升级前在 RBS 注册新度量为可信 | 计划内升级 |
| 周期性 re-key | `cryptsetup luksChangeKey` + 同步更新 RBS 资源 | 防长期密码沉淀 |

应急 re-encrypt 用 `cryptsetup-reencrypt`（在线，但慢），仅在密码可能泄露时使用。

---

## 11. 与 TDX 版方案的差异

| 维度 | TDX 版（rev-0.6） | CCA 版（本设计） |
|---|---|---|
| 硬件平台 | Intel TDX | ARM CCA / virtCCA（鲲鹏） |
| Guest OS | Ubuntu / CentOS | **openEuler** (22.03 LTS SP3+) |
| 证明证据 | DCAP Quote | Realm Attestation Token (RAT) |
| 度量寄存器 | MRTD + RTMR[0..3] | RIM + REM[0..3] |
| 取证接口 | `/dev/tdx_guest` ioctl | RSI_ATTEST_TOKEN（经 RMM） |
| 远程证明服务 | Trustee | **GTA RBS**（openEuler 社区） |
| 客户端 SDK | （直调 + Trustee REST） | **rbc Rust SDK + C FFI**（openEuler 社区） |
| LUKS 解锁实现 | 静态密码（人工 / 脚本） | **systemd-cryptsetup + prepare-keyfile + rbc-cli** |
| API Key 存放 | `openclaw.json` 明文（在 LUKS 卷上） | **tmpfs + EnvironmentFile** |
| 启动证明绑定 | 仅运行期可选 | **启动期强制（解锁前）** |
| CC Skills | `*_td_*` × 3 | `*_realm_*` × 3 + `refresh_credentials` |
| Skills 角色 | 运行期自证 | 运行期自证（与启动期密钥获取严格分离） |
| MAC 框架 | AppArmor (Ubuntu) / 无 (CentOS 弱) | **SELinux enforcing**（openEuler 默认） |
| initramfs 工具 | initramfs-tools / dracut | **dracut**（openEuler） |

---

## 12. 后续工作

按优先级：

| P | 工作项 | 估算 | 备注 |
|---|---|---|---|
| P0 | 实现 rbc-cli（rbc 仓库的 bin target） | 2d | 当前 `src/bin/main.rs` 是 stub |
| P0 | 改造 `luks_tools/create_encrypted_vfs.sh`：format + 注册 RBS 一体化 | 1d | 包含密码即销毁逻辑 |
| P0 | 编写 systemd 单元链（openEuler 主路径，prepare-keyfile） | 1d | 含 loop / unlock / mount / fetch-apikey / openclaw |
| P0 | **openEuler SELinux 策略模块** `openclaw.te` | 1-1.5d | permissive 跑通后 audit2allow 收敛 |
| P0 | 验证 OpenClaw 是否支持 env 注入 LLM Key | 0.5d | 否则需小改 OpenClaw |
| P0 | **openEuler Realm 镜像构建脚本（dracut + RIM 计算）** | 2d | 见 §7.5；与 RBS 策略联动 |
| P1 | 实现 CC CCA Skills × 3 + `refresh_credentials` | 3d | 参考现有 CC TDX Skills |
| P1 | 救援 keyslot 流程文档 + 演练 | 0.5d | |
| P1 | RBS 资源策略 schema 标准化 | 1d | 给 LUKS / API Key / Channel Token 统一模板 |
| P1 | virtCCA → 标准 ARMv9 CCA 迁移评估 | 1d | RAT 格式与 RSI 接口兼容性 |
| P2 | LUKS2 + dm-integrity 评估与启用 | 1-2d | |
| P2 | 度量升级运维手册（多度量白名单切换） | 0.5d | |
| P2 | 端到端集成测试 | 3d | virtCCA / mock attester |
| P2 | Debian/Ubuntu 端的 keyscript 实现（附录 C 落地） | 1d | 可选，为生态兼容 |
| P3 | clevis pin (`clevis-pin-cca-rbs`) 标准化集成 | 1-2 周 | 长期方向 |
| P3 | Confidential Inference 对接（解决 LLM 服务端信任问题） | 中长期 | Azure / NVIDIA NIM 等 |
| P3 | 容器化部署支持（openEuler iSulad / Docker / Kubernetes） | 中长期 | 与原 TDX 方案的未来工作一致 |

---

## 附录 A：关键术语

| 缩写 | 全称 |
|---|---|
| CCA | Confidential Compute Architecture (ARM) |
| Realm | ARM CCA 的硬件 TEE 容器，等价于 TDX TDVM |
| RIM | Realm Initial Measurement |
| REM | Realm Extensible Measurement |
| RAT | Realm Attestation Token |
| RMM | Realm Management Monitor |
| RSI | Realm Services Interface（Realm 内部调 RMM 的 API） |
| RBS | Resource Broker Service（GTA 提供的远程证明 + 资源下发服务） |
| rbc | Resource Broker Client（RBS 的客户端 SDK） |
| GTA | Global Trust Authority |
| LUKS | Linux Unified Key Setup |
| LLM | Large Language Model |
| TEE | Trusted Execution Environment |
| TDX | Trust Domain Extensions (Intel) |
| TDVM | Trust Domain Virtual Machine |
| KBS | Key Broker Service |
| JWE | JSON Web Encryption |

## 附录 B：参考资料

- [OpenClaw-CC TDX 版 README](./README.md) / [README_CN](./README_CN.md)
- [rbc CLAUDE.md](../../../RBS/globaltrustauthority-rbs/rbc/CLAUDE.md)
- [openEuler 官网](https://www.openeuler.org/)
- [openEuler GTA (Global Trust Authority)](https://gitee.com/openeuler/global-trust-authority)
- [openEuler virtCCA 项目](https://gitee.com/openeuler/virtCCA_sdk)
- [LUKS2 On-Disk Format Specification](https://gitlab.com/cryptsetup/cryptsetup/-/wikis/LUKS2-On-Disk-Format)
- [systemd-cryptsetup man page](https://www.freedesktop.org/software/systemd/man/systemd-cryptsetup-generator.html)
- [crypttab man page](https://man7.org/linux/man-pages/man5/crypttab.5.html)
- [ARM CCA Architecture Spec](https://developer.arm.com/documentation/den0096/latest)
- [SELinux Policy Writing on RHEL/openEuler](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/index)

## 附录 C：keyscript 实现（仅供 Debian/Ubuntu 等支持的发行版参考，**openEuler 不可用**）

```
# /etc/crypttab (Debian/Ubuntu only; openEuler 上 keyscript= 会被 systemd-cryptsetup 静默忽略)
openclaw_data /dev/loop0 luks-passphrase-openclaw-prod luks,keyscript=/usr/local/sbin/rbc-keyscript,tries=1,timeout=30,noauto
```

```bash
#!/bin/bash
# /usr/local/sbin/rbc-keyscript  (mode=0700, owner=root)
set -eu
resource_id="${1:-}"
log() { echo "[rbc-keyscript] $*" >&2; }
[[ -z "$resource_id" ]] && { log "ERROR: empty resource_id"; exit 64; }
export PATH=/usr/local/bin:/usr/bin:/bin
export LANG=C LC_ALL=C
exec timeout 25s /usr/local/bin/rbc-cli \
    --config /etc/rbc/rbc.yaml \
    get-resource --resource "$resource_id"
```

在 openEuler 上不要走这条路径 —— `systemd-cryptsetup` 不会调 keyscript，会回落到把第 3 字段当 keyfile 路径处理，导致解锁失败且错误信息可能误导排查。
