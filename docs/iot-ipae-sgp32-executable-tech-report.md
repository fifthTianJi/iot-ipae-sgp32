# IoT-IPAe 技术方案报告

| 项 | 内容 |
|----|------|
| 文档名称 | IoT-IPAe 技术方案报告（SGP.32 可执行版） |
| 版本 | 1.4 |
| 日期 | 2026-11-14（Asia/Shanghai） |
| 依据 | GSMA SGP.32 v1.3（2026-05-22） |
| 关联 | SGP.31、SGP.22、SGP.33、ETSI TS 102 221 / 102 223 |
| 性质 | 技术方案报告：需求、架构、详细设计、流程、接口、安全、实现、测试 |
| 说明 | 解读与落地指南。互通与认证以 GSMA 正式文本为准。 |

---

## 摘要

本报告给出基于 GSMA SGP.32 的 **IPAe（eUICC 内 IoT Profile Assistant）** 技术方案。方案在消费级 RSP（SGP.22）之上引入 eIM 与 IPA，去掉本地用户界面，用远程编排管理无 UI、网络受限的 IoT 设备。

推荐实现形态为：**IPA Services（ES10b 语义）单内核**；IPAd 经 APDU 适配，IPAe 经卡内调用。模式由 Terminal Capability 与 `IpaeActivationRequest`（BF42）唯一确定。IPAe 经 ISD-R 注册的 CAT 事件（STATUS / Location / Timer）唤醒，经 BIP 建立承载后完成 DNS、TLS/SNI 与 ESipa。下载分 Direct（通知直达 SM-DP+）与 Indirect（通知经 eIM 再转 SM-DP+）。eIM 须同时支持 ASN.1 与 JSON 绑定。

报告按通用技术报告结构组织，并附可执行步骤、验收用例 T01–T21 与实现任务包 M1–M8。

**关键词：** SGP.32；IPAe；IPAd；eIM；ESipa；ES9+；ES9+'；ES10b；TLS；BIP

---

## 1 引言

### 1.1 编写目的

统一 EUM、模组、eIM 与测试对 IPAe 方案的理解，使设计可直接拆成开发与测试任务。每个关键机制按「前置条件 → 步骤 → 数据/Tag → 成功判据 → 失败分支」描述。

### 1.2 范围

**包含：** IPAe 架构与选型；能力协商与激活；与 IPAd 共存及 ES10b 复用；CAT/BIP 事件注册；TLS/DNS 与忙处理；ESipa 与 ES9+/ES9+' 函数与报文；Package/PSMO/eCO；Direct/Indirect 下载与通知；安全要点；测试矩阵。

**不包含：** SM-DP+ 内部订单实现、运营商商务流程、EUM 私有 COS 指令码、完整 ASN.1 模块原文。

### 1.3 读者

| 读者 | 重点章节 |
|------|----------|
| 架构 / 方案 | 2、3、11 |
| EUM 固件 | 4、5、6、7、8 |
| 模组 / 设备 | 4.1–4.4、5.1–5.2 |
| eIM / 平台 | 5.3–5.5、6、7 |
| 测试 | 9、10 |

### 1.4 术语与缩略语

| 术语 | 含义 |
|------|------|
| IPA / IPAd / IPAe | IoT Profile Assistant；位于设备 / 位于 eUICC |
| eIM | eSIM IoT Remote Manager |
| ESipa / ESep | eIM↔IPA；eIM↔eUICC 逻辑端到端 |
| ES10b | IPA↔ISD-R（IPA Services） |
| ES9+ / ES9+' | IPA↔SM-DP+ / eIM↔SM-DP+ |
| PSMO / eCO | Profile 状态操作 / eIM 配置操作 |
| BPP | Bound Profile Package |
| NCD | Network Constrained Device |
| BIP | Bearer Independent Protocol（OPEN CHANNEL 等） |
| PIR | Profile Installation Result |

### 1.5 文档约定

- 「必须 / 不得」对应规范 SHALL / SHALL NOT 的落地要求。
- Tag 为 BER/DER 十六进制（如 BF42、E1）。
- 验收勾选供研发自测；正式认证以 SGP.33 为准。

---

## 2 背景与需求分析

### 2.1 背景

GSMA 已有 M2M（SGP.02）与消费（SGP.22）两套 RSP。消费方案依赖 LUI、ES10c 与用户扫码，不适合无界面、功耗与带宽受限、生命周期很长的 IoT。SGP.32 复用 SM-DP+ 与 ES8+，新增 eIM 与 IPA，把决策放到远程。

### 2.2 问题与目标

| 问题 | 目标 |
|------|------|
| 设备无 UI，无法本地选 Profile | eIM 远程 PSMO / 触发下载 |
| 设备固件难改、算力极低 | IPA 可下沉到 eUICC（IPAe） |
| 受限网络难以直连 SM-DP+ | Indirect：eIM 代理 ES9+'，设备只维持一条到 eIM 的通道 |
| 同一芯片要覆盖「带 IPA 固件」与「哑设备」 | 双模共存，ES10b 只实现一次 |
| 卡端主动式命令与取包争用 | 明确忙处理，避免并行 RSP |

### 2.3 需求与约束

1. 任一时刻仅一种 IPA 模式（IPAd 或 IPAe）。
2. Terminal Capability 必须在 SELECT ISD-R 之前发送。
3. IPAe 至少能被 STATUS、Location status、Timer 之一触发，并具备 Annex A.2 所列 CAT/BIP。
4. ESipa 传输须具备机密性与完整性；HTTPS/CoAPS 为规范路径，底层 DM（LwM2M/MQTT）可承载 ASN.1。
5. eIM 必须同时支持 ASN.1 与 JSON。
6. 同一 eUICC Package 不得混装 PSMO 与 eCO。
7. IPAe + Retrieve HTTPS/CoAPS 应配置 TLS 信任锚，否则握手易失败。

### 2.4 方案比选与结论

| 维度 | IPAd | IPAe |
|------|------|------|
| 集成方 | OEM / 模组 | EUM |
| 设备改动 | 完整 IPA + ES10 | CAT/BIP + 能力声明 |
| 安全评估 | IPAd 独立路径 | 并入 eUICC |
| 定制 | 高 | 低 |
| 典型场景 | 可维护固件的网关 | 表计、标签、超受限模组 |

**结论（决策树）：**

```
固件能否长期维护完整 IPA？
  ├─ 能 → IPAd（Terminal Capability b1=1，不发 BF42）
  │        受限网络 → Indirect + 可选 minimizeEsipaBytes
  │        宽带稳定 → Direct 或 Indirect
  └─ 不能 → IPAe（b1=0 自动激活，或显式 BF42）
             必选 Annex A.2；建议 Indirect + trustedCertificateTls
             传输：CoAPS 或 LwM2M/MQTT + ASN.1
```

---

## 3 总体方案

### 3.1 设计原则

1. **单内核：** IPA Services 只实现一套 ES10b 语义。
2. **模式门控：** Mode Manager 根据 Terminal Capability 与 BF42 开关外部 ES10 与 IPAe 引擎。
3. **传输与业务分离：** BIP/TLS 属于承载；PSMO/下载属于 IPA Services。
4. **通知与下载模式绑定：** Direct 直送 SM-DP+；Indirect 经 eIM。
5. **会话串行：** 不并行 RSP；触发在忙时排队。

### 3.2 系统角色与架构

| 角色 | 职责 |
|------|------|
| Operator / SM-DP+ / SM-DS | 订单、Profile 准备、事件登记；与 SGP.22 同类 |
| eIM | 远程编排、PSMO/eCO、可代理 SM-DP+/SM-DS |
| IPA | 发现、下载编排、转包、通知 |
| eUICC | ISD-R（IPA Services）、ISD-P、ECASD、Profile |

**IPAe 拓扑要点：** 设备变薄。ESipa 经 Modem/BIP 出卡；ES10 在卡内；ES8+ 仍是 SM-DP+ 与 eUICC 的端到端保护。Indirect 时设备不建到 SM-DP+ 的 TLS。

### 3.3 接口总览

| 接口 | 端点 | 用途 |
|------|------|------|
| ESipa | eIM—IPA | 触发、eIM Package、间接下载消息 |
| ESep | eIM—eUICC（逻辑） | 签名 eUICC Package，经 ESipa 隧道 |
| ES9+ | SM-DP+—IPA | Direct 下载与直送通知 |
| ES9+' | SM-DP+—eIM | Indirect 代理 |
| ES8+ | SM-DP+—eUICC | BPP 保护 |
| ES10a/b | IPA—ISD-R | IPAd=APDU；IPAe=内调 |
| ES11 / ES11' | IPA 或 eIM—SM-DS | Event 发现 |
| ES6 | Operator—eUICC | OTA，含连通参数更新 |

### 3.4 软件分层

```
Mode Manager（Terminal Cap + BF42 → ipaMode）
        ├─ ES10b APDU Adapter（仅 IPAd）
        └─ IPAe Engine（ESipa / CAT / BIP / DNS / TLS）
                    │
                    ▼
              IPA Services（ES10b 语义内核，ISD-R）
```

### 3.5 生命周期概览

制造预置 → 上电能力协商与激活 → 事件注册 → 触发建链 → 关联 eIM → 下载 Profile → 运营 PSMO / Fallback → 异常回退与通知清理。检查项见第 8 章。

---

## 4 详细设计

### 4.1 能力协商

#### 4.1.1 设备：Terminal Capability

eUICC 在 MF FCP 置位，要求设备发送 Terminal Capability（TS 102 221）。**必须先于 SELECT ISD-R。**

模板 `A9` 内 tag `84` 首字节：

| b1 | 含义 | 卡侧处理 |
|----|------|----------|
| 1 | 支持 IPAd | 启用对外 ES10；默认不激活 IPAe（除非 BF42） |
| 0 | 不支持 IPAd | 禁止对外 ES10；若 `ipaeSupported` 则自动激活 IPAe |

#### 4.1.2 卡：EUICCInfo2（BF22）

| 字段 | 要求 |
|------|------|
| 基线 | profileVersion、svn、固件、uiccCapability、rspCapability、CI PKId 等按 SGP.22 |
| `ipaMode [16]` | `ipad(0)` / `ipae(1)`，反映当前模式 |
| `iotSpecificInfo [20]` | 必填；`iotVersion` 至少一项；可有 eCall、Fallback 标志 |
| Fallback | 若支持：`fallbackSupported` 与 `fallbackAllowedUpdateSupported` 均出现 |
| `iotSpecificMetadataSupport` | 本版本置 0 |
| 未用字段 | `euiccCiPKIdListForSigningV3`、`additionalEuiccInfo`、`highestSvn` 不得带出 |

**上电处理：** 记录 b1 → SELECT 解析 E1 → GetEUICCInfo2 → 校验 `iotSpecificInfo` 与 `ipaMode` → 缓存供 eIM（A8/BF22）与本地下载策略。

#### 4.1.3 IPA Capabilities（与 eIM 对齐）

`ipaFeatures`：

| 位 | 名称 | 置 1 |
|----|------|------|
| 0 | directRspServerCommunication | Direct，通知直送 SM-DP+ |
| 1 | indirectRspServerCommunication | Indirect，通知经 eIM |
| 2 | eimDownloadDataHandling | AC/Event 留在 eIM |
| 3 | eimCtxParams1Generation | eIM 生成 CtxParams1 |
| 4 | eimProfileMetadataVerification | eIM 校验 Metadata |
| 5 | minimizeEsipaBytes | compact，省略对端已知字段 |

协议位（IPA 与 eIM 对称）：RetrieveHttps、RetrieveCoaps、InjectHttps、InjectCoaps、Proprietary。

**画像：** 受限 IPAe 建议 indirect + minimize + eimDownloadDataHandling + CoAPS 或专有承载。网关 IPAd 可同时开 Direct 与 Indirect、HTTPS 拉取。eIM 须先取得 tag A8（或带外），协议位无交集则拒绝业务。

### 4.2 SELECT ISD-R 与 IPAe 激活

#### 4.2.1 选择步骤

| 步 | 命令 | 说明 |
|----|------|------|
| 1 | MANAGE CHANNEL | 开逻辑通道（可 OMAPI） |
| 2 | SELECT ISD-R | 仅 IPAd 或初始化路径；其它应用不得选择 |
| 3 | 解析 FCI | GP 标准对象之后为 IoT 专有模板 |

#### 4.2.2 FCI 专有信息

`ISDRProprietaryApplicationTemplateIoT`：`[PRIVATE 1]`，Tag **E1**。

```
euiccConfiguration BIT STRING {
  ipaeSupported(0),
  enabledProfile(1)
}
```

#### 4.2.3 激活指令 BF42

| 项 | 内容 |
|----|------|
| 名称 | IpaeActivationRequest / Response |
| Tag | BF42 `[66]` |
| 传输 | 与 SGP.22 §5.7.2 相同（如 STORE DATA） |
| 前提 | Annex A.2（IPAe 列）；FCI `ipaeSupported=1` |
| 请求 | `ipaeOption`：`activateIpae(0)` |
| 响应 | `ok(0)` / `notSupported(1)` |

**判定：**

| b1 | BF42 | ipaeSupported | 结果 |
|----|------|---------------|------|
| 1 | 否 | * | 保持 IPAd，不得激活 IPAe |
| 1 | 是 | 1 | 激活 IPAe，`ipaMode=ipae`，关闭对外 ES10 |
| 1 | 是 | 0 | notSupported |
| 0 | 可不发 | 1 | 自动 IPAe |
| 0 | * | 0 | 无可用 IPA（异常产品） |

**切回 IPAd：** 复位 → Terminal Capability b1=1 → 不发 BF42 → 确认对外 ES10 与 `ipaMode=ipad`。

### 4.3 双模共存与 ES10b 复用

同一时刻一种模式。映射：

| ES10b | IPAd | IPAe |
|-------|------|------|
| LoadEuiccPackage | APDU | ESipa 收包后内调 |
| GetEUICCInfo / Challenge / EID / Certs | APDU | 组包前内调 |
| AuthenticateServer / PrepareDownload / LoadBoundProfilePackage / CancelSession | APDU | 下载状态机内调 |
| Retrieve / Remove Notification | APDU | 通知状态机内调 |
| GetEimConfigurationData / GetConnectivityParameters | APDU | 建链前内读 |
| ImmediateEnable / Rollback / Fallback / Emergency | APDU | 策略引擎内调 |
| AddInitialEim / MemoryReset | 产线 APDU | 少用；制造预置或 eCO |

**Mode Manager：**

```
on_boot:
  if tc.ipad_bit == 0 and ipae_supported:
    activate_ipae(); disable_external_es10()
  elif tc.ipad_bit == 1:
    enable_external_es10(); ipae_engine.suspend()

on_bf42(activate):
  if not ipae_supported: return notSupported
  activate_ipae(); disable_external_es10(); return ok
```

**ES10b 函数全集：** LoadEuiccPackage, GetEUICCInfo, GetEUICCChallenge, AddInitialEim, eUICCMemoryReset, AuthenticateServer, PrepareDownload, LoadBoundProfilePackage, CancelSession, GetCerts, GetEID, RetrieveNotificationsList, RemoveNotificationFromList, GetRAT, GetProfilesInfo, ImmediateEnable, ProfileRollback, ConfigureImmediateProfileEnabling, GetEimConfigurationData, ExecuteFallbackMechanism, ReturnFromFallback, Enable/DisableEmergencyProfile, GetConnectivityParameters, SetDefaultDpAddress。

### 4.4 事件注册、STATUS 与 BIP

目标：IPAe 能被 STATUS、Location status、Timer 至少之一唤醒，并用 BIP 数据通道跑 DNS/TLS/ESipa。事件通过 **TERMINAL PROFILE**（设备声明）和 **SET UP EVENT LIST**（卡注册，上下文在 ISD-R/IPAe）建立。

| 能力 | IPAd | IPAe |
|------|------|------|
| TERMINAL PROFILE / SET UP EVENT LIST | 必 | 必 |
| REFRESH（UICC Reset / Profile State Change）；至少 mode 4 | 必 | 必 |
| LOCAL INFO：IMEI / Location / 时区 | — | 必 |
| Location status EVENT DOWNLOAD | — | 必 |
| TIMER MANAGEMENT / EXPIRATION | — | 必 |
| OPEN/CLOSE CHANNEL、SEND/RECEIVE DATA、通道状态 | 必 | 必 |
| SMS-PP DOWNLOAD | 必 | 必 |

**注册步骤：**

1. 设备 TERMINAL PROFILE。  
2. ISD-R/IPAe 发 SET UP EVENT LIST（含 Location status 等）。  
3. 若用 Timer：TIMER MANAGEMENT 设定周期。  
4. 保存已注册触发集；与 USIM 事件并存时规定路由，并规定 BIP 是否与 proactive 会话互斥。  
5. 触发进入第 4.5 节状态机。

### 4.5 会话状态机与忙处理

**空闲路径：** Idle → 读 eimFqdn / TLS 锚 / 连通参数 → OPEN CHANNEL → DNS+TLS → GetEimPackage 循环 → 通知投递 → CLOSE CHANNEL → Idle。

**状态：** `Idle | Backoff | WaitBearer | DnsTls | EsipaSession | NotifDelivery`  
**规则：** 非 Idle 的触发只置 `pendingPoll`；通道失败或 catBusy 进 Backoff（T, 2T, 4T 封顶）；回到 Idle 且 pending 则立即再入 WaitBearer。

| 忙类型 | 现象 | 处理 |
|--------|------|------|
| proactive 占用 | catBusy(5) | A：返回后重试；B：卡内结束会话，收齐未完成 TERMINAL RESPONSE 后再 REFRESH |
| BIP 占用 | OPEN CHANNEL 失败 | 排队 + 指数退避 |
| RSP 进行中 | 下载未完又触发 | 禁止并行；新 Challenge 丢弃旧会话态 |
| TLS 中再触发 | Timer 与 Location 并发 | 不抢占，结束后补拉 |
| 无待办 | noEimPackageAvailable(1) | 正常休眠 |
| DNS/TLS 失败 | 解析、证书、alert | 有限重试与 SNI 回退，再等下一周期 |
| 掉电 / 拔卡 | 中断 | 恢复后可重启流程 |

### 4.6 数据设计

#### 4.6.1 eIM Package 四类

| 对象 | Tag | 用途 |
|------|-----|------|
| EuiccPackageRequest | BF51 | 签名 PSMO 或 eCO |
| IpaEuiccDataRequest | BF52 | 拉取数据 |
| EimAcknowledgements | BF53 | ACK 序号 |
| ProfileDownloadTriggerRequest | BF54 | 触发下载 |

签名覆盖 `euiccPackageSigned ‖ associationToken`；无 token 时用 `84 01 00`。`counterValue` 每包加 1，卡至少支持到 0x7FFFFF；禁止在不更换公钥时把 counter 调低。

**包约束：** PSMO 与 eCO 不混装；enable/disable 各至多一次；listProfileInfo 不得位于 enable/disable/delete 之后；执行中断须恢复原状态。

**BF52 tagList：** A0 通知；81 default SM-DP+；A2 Package Result；BF20/BF22 Info；83 Root SM-DS；84 association token（必须能提供）；A5/A6 证书；A8 Capabilities（必须）；A9 DeviceInfo（需 bit3）。

**GetEimPackage 附加：** `notifyStateChange`；`stateChangeCause` = otherEim(0)、fallback(1)、emergencyProfile(2)、local(3)、reset(4)、immediateEnableProfile(5)、deviceChange(6)、undefined(127)；`rPLMN` 3 字节 MCC+MNC。

**BF54：** 空对象仅当 bit2=1；否则 activationCode、contactDefaultSmdp 或 contactSmds。

#### 4.6.2 EimConfigurationData 要点

eimId（1..128，卡内唯一）；eimFqdn（类型为 FQDN 时禁止重复填写，可为 Broker）；eimIdType oid/fqdn/proprietary；counter；可选 associationToken（请求新值传 -1，Memory Reset 不能清零全局计数）；签名公钥或证书；TLS 信任锚（IPAe+Retrieve 强烈建议，证书须含 SKI，CA 须 cA=TRUE）；协议位；euiccCiPKId；indirectProfileDownload。

**eCO：** addEim 必填 id、counter、公钥；delete 仅 id；update 禁止改 id、禁止带 token；list 为空序列。

#### 4.6.3 Connectivity Parameters

DGI `3A07`，内层 `A1`（HTTP/CoAP）：Bearer、NAN/APN、Login、Password；TS 102 223 TLV，CR=0。Enabled Profile 含则用于建链，否则 ES6 更新。IPAd 用 GetConnectivityParameters；IPAe 读取方式须在设计中写明。

#### 4.6.4 PSMO 一览

enable（可 rollbackFlag）、disable、delete、listProfileInfo（BF2D）、getRAT、configureImmediateEnable、set/unsetFallbackAttribute、setDefaultDpAddress（BF65）。

---

## 5 关键流程

### 5.1 上电与激活

1. 设备 Terminal Capability。  
2. MANAGE CHANNEL + SELECT ISD-R，解析 E1。  
3. 按第 4.2 节判定表进入 IPAd 或 IPAe。  
4. GetEUICCInfo2，确认 `ipaMode` 与 `iotSpecificInfo`。  
5. TERMINAL PROFILE → SET UP EVENT LIST →（可选）Timer。

### 5.2 DNS 解析与 TLS 建链

本节只讲 IPA 连上 eIM 之前的名字解析和握手。依据是 SGP.32 §2.6.3、§2.6.4.4、§2.6.5 与 §2.7.3。ESipa 业务报文从握手成功之后才开始。直接下载另有一条到 SM-DP+ 的 HTTPS，SNI 和证书链用的是 SM-DP+ 的 FQDN 与 CERT.DPauth，见 5.2.6。

#### 5.2.1 连接用的名字

DNS 查询名和 ClientHello 里的 SNI 使用同一个 FQDN，来自该 Associated eIM 的配置：

| eimIdType | 取值 | 用来解析和填写 SNI 的名字 | eimFqdn 字段 |
|-----------|------|---------------------------|--------------|
| eimIdTypeFqdn | 2 | `eimId` 本身（它就是 FQDN） | 禁止再填，避免和 eimId 重复 |
| eimIdTypeOid | 1 | `eimFqdn` | 必须提供 |
| eimIdTypeProprietary | 3（未带类型时卡内也按此保存） | `eimFqdn` | 必须提供 |
| 中间节点 | 任意 | `eimFqdn` 填 MQTT Broker 等中间服务器的 FQDN | DNS 与 SNI 都对着这台中间服务器，而不是 eIM 应用层的另一名字 |

`eimId` 长度 1..128。证书 CN 按 RFC 5280 一般不超过 64 个字符，部署时让 eimId 与 CERT.EIM.TLS 的 CN 对齐，并满足 DNS 限制：单个标签不超过 63，整名不超过 255。国际名先转成 A-label（Punycode），DNS 查询和 SNI 都用这份编码。

IPAe 走 Retrieve HTTPS 或 Retrieve CoAPS 时，同一条配置里应带 `trustedCertificateTls`（或 `trustedEimPkTls`）。没有信任锚时，名字能解析，握手也会在验链处失败。

#### 5.2.2 DNS 解析

前置：已按第 4 章完成模式判定；BIP 已 OPEN CHANNEL，承载来自连通参数 DGI `3A07`（Bearer、APN、登录）。解析器地址用该承载提供的 DNS，规范没有单独的 DNS 服务器字段。

IPAe 的查询经 BIP 发出，IPAd 用设备解析器。步骤：

1. 按 5.2.1 选出唯一 FQDN。多个 Associated eIM 时只解析当前选中的那一个。  
2. 查 A 与 AAAA。只使用承载实际具备的地址族。  
3. 多个地址按应答顺序尝试，每地址有限次连接。TTL 内可以缓存；`updateEim` 改了 FQDN 或信任锚之后丢弃缓存。  
4. 部署默认端口：HTTPS 443，CoAPS 5684。配置里若另有端口，以配置为准。  
5. 地址连通后才发 ClientHello。DNS 失败不进入 TLS，也不记成证书错误。

| 结果 | 含义 | 处理 |
|------|------|------|
| 得到地址 | 可以握手 | 进入 5.2.3 |
| NXDOMAIN | 配置的名字不存在 | 停止本轮，记配置错误，等下一触发周期 |
| SERVFAIL / 超时 | 解析器或无线路径暂时失败 | 有限重试，然后进入 Backoff |
| 只有对端地址族没有的记录 | 例如只有 AAAA 而承载是 IPv4 | 按解析失败处理，不改去连别的 eIM |

#### 5.2.3 握手顺序

ESipa 只做服务器认证。下限 TLS 1.2 / DTLS 1.2；ESipa 可以协商 1.3。强制曲线 NIST P-256（secp256r1，namedCurve 23）。强制套件：1.2 为 `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256` 与 `AES_256_GCM_SHA384`；1.3 为 `TLS_AES_128_GCM_SHA256` 与 `TLS_AES_256_GCM_SHA384`。

```
OPEN CHANNEL
    → DNS（5.2.2）
    → TCP 443 或 UDP 5684
    → ClientHello（必须带 server_name = 连接用 FQDN）
    → ServerHello + Certificate（SNI 选定的那一条链）
    → 验链（5.2.5）通过
    → Finished
    → HTTP：User-Agent = gsma-rsp-ipae 或 gsma-rsp-ipad
            X-Admin-Protocol = gsma/rsp/v<x.y.z>
```

ClientHello 至少带：版本、上述套件、`supported_groups` 含 secp256r1、签名算法，以及 `server_name`。NCD 在 TLS 1.2 还应带 `max_fragment_length`（RFC 6066）。对端 RSP Server 若不应答该扩展，规范建议改走间接下载，避免在设备上分片失败。DTLS 另加 Connection ID（RFC 9146）和会话恢复；CoAP 用 block-wise。

验链完成之前不发送 ESipa 请求。会话未结束时，新的 STATUS / Timer 只置 pendingPoll，不插入第二条握手。

#### 5.2.4 SNI

SNI 是 RFC 6066 的 `server_name` 扩展，类型为 host_name。同一 IP 上可以挂多张证书、多条链。eIM 用客户端送来的 FQDN 选出 **IPA 能够验证的那一条 TLS 证书链**。支持该机制的 IPA 与 eIM 都要按本节实现；本节对 IPAe 同样适用。

**IPA（客户端）**

1. 每次新建握手，ClientHello 都带 `server_name`，内容等于 5.2.1 的连接用 FQDN，与 DNS 查询名相同。  
2. 服务器对**不认识的 FQDN** 回 fatal `unrecognized_name(112)`。IPA 结束这条连接，**不再**去掉 SNI 重试。112 表示名字本身错误，去掉 SNI 只会拿到默认域的另一条链。  
3. 服务器回**其它** fatal alert（扩展不被接受、中间盒切断等）：IPA **必须**再握手一次，这次 ClientHello **不带** `server_name`。只重试这一次。  
4. 第二次仍失败：关闭连接，进入 Backoff，把原因记成「SNI 回退后仍失败」，与 112、与验链失败分开。

**eIM（服务器）**

1. 必须实现 `server_name`，用它选择证书和整条链。最终编码后的域名不得超过 DNS 长度限制。  
2. ClientHello 里的 FQDN 不在本机证书表中：发送 fatal `unrecognized_name(112)` 并中止。不要改发另一租户的链。  
3. ClientHello **没有** `server_name`：应出示预先绑定到**默认域**的那张证书及其链。  
4. 默认域证书仍然要能被该 IPA 的信任锚验过，并且叶证书的 `dNSName` 覆盖客户端原本要连的 FQDN。做不到时，客户端会在 5.2.5 拒绝。默认证书不是绕过校验的通道。

| 服务器行为 | IPA 下一步 | 验收 |
|------------|------------|------|
| 认识该 FQDN，出示对应链 | 用原 FQDN 验 SAN 和链 | 进入 ESipa |
| fatal 112 | 停止，不重试 | T09 |
| 其它 fatal | 去掉 SNI 再握手一次 | T10 |
| 无 SNI，出示默认域链 | 仍用原 FQDN 验 SAN；链必须收到预置锚 | T21 |

#### 5.2.5 TLS 证书链

传输用的是 **CERT.EIM.TLS**（DTLS 可以是同一张，记为 CERT.EIM.DTLS）。它和签 eUICC Package 的 **CERT.EIM.ECDSA** 不是同一张：签名证书不必带 SAN，也不应带 CRL/OCSP；TLS 证书必须带 SAN 和 EKU。`trustedCertificateTls` 只锚传输链，不代替 `eimPublicKeyData`。

服务器在 Certificate 消息里按这个顺序给链。根通常不放进握手，因为锚已经在客户端：

```
[0] 叶证书 CERT.EIM.TLS     ← SNI 选中的这一张
[1] 中间 CA
[2] 更上层中间 CA（若有）
锚：trustedCertificateTls 或 trustedEimPkTls
    可以是叶本身、路径上的某一张中间 CA，或根 CA
```

`trustedCertificateTls` 的规则：

- 它是「终于叶证书的那条链」上客户端事先信任的一张。  
- 必须带 Subject Key Identifier。  
- 若它是根或中间 CA，BasicConstraints 的 `cA` 必须为 TRUE。  
- IPAe + Retrieve HTTPS/CoAPS 应在 addEim 时写入。IPAd 可以忽略该字段，改用设备信任库；需要时也可通过 GetEimConfigurationData 把锚取到设备。

客户端收口时，叶必须链到这张锚，或链到 `trustedEimPkTls` 所表示的那把公钥。锚是中间 CA 时，叶和锚之间的每一张都要验签，锚以上的根可以不出现。

**叶证书逐项核对（X.509 v3，RFC 5280）：**

| 字段 | 要求 |
|------|------|
| version | v3 |
| subject | CN、C 必填；O、OU、L 可选。CN 与 eimId 对齐 |
| subjectPublicKeyInfo | 算法按 §2.6.4.2，强制能验 P-256 |
| keyUsage | critical，且含 digitalSignature |
| extendedKeyUsage | critical，**只有** serverAuth 与 clientAuth，多一个 EKU 就拒绝 |
| subjectAltName | 至少一个 dNSName，**恰好一个** registeredID。dNSName 可以按 RFC 2818 使用通配符 `*` |
| authorityKeyIdentifier | 非 critical，用来指向签发它的 CA 公钥 |
| 有效期 | notBefore / notAfter 覆盖当前时间。IPAe 用 PROVIDE LOCAL INFORMATION 取日期时间后再判 |

**名字怎么对上 SNI：** 用 5.2.1 的连接用 FQDN 去匹配叶证书里的某个 `dNSName`。通配符只出现在最左标签（`*.example.com` 匹配 `a.example.com`，不匹配 `a.b.example.com`，也不匹配裸域 `example.com`）。无 SNI 的第二次握手也用这同一个 FQDN 去匹配，不改成「服务器随便给的默认名」。`registeredID` 有且仅有一个；eimIdType 为 OID 时，部署应让它等于该 eIM 的 OID，避免同一 VIP 上的默认链被当成目标 eIM。

**链的检查顺序：**

1. 时间可用，且叶与每一张 CA 都在有效期内。  
2. 自叶向锚逐跳验签，AKI 与上一张的 SKI 对应。  
3. CA 证书 `cA=TRUE`，叶证书 `cA` 不为 TRUE。  
4. 叶的 EKU、Key Usage、SAN 符合上表。  
5. SAN 的某个 dNSName 覆盖连接用 FQDN。  
6. 路径终止于 `trustedCertificateTls` 或 `trustedEimPkTls`。终止不了就失败，不回落到公共 CA，除非 IPAd 的产品规格明确使用设备信任库。

**吊销：** IPAd 的 TLS 必须能通过 CRL 或 OCSP 检查叶证书及路径上的 CA，eIM 可以做 OCSP stapling。CRL 里的 Hold 不算撤销，只有明确撤销才拒绝。DTLS **不做**吊销检查。吊销信息不要下发到 eUICC；卡内验的是预置锚、有效期和 SAN。IPAe 若要做吊销，放在设备侧代查，失败则不把通道交给卡内握手。

任一检查失败：发送合适的 alert 或直接关闭，不发 GetEimPackage。日志分开记「SAN 不匹配」「EKU 非法」「链不到锚」「已撤销」「112」。

#### 5.2.6 到 SM-DP+ 的另一条 TLS

直接下载时，IPA 再对 SM-DP+ 做一条 HTTPS，规则是 SGP.22 §2.6.6，IPA 扮演 LPA。SNI 填激活码或配置里的 **SM-DP+ FQDN**，链回到 CERT.DPauth.ECDSA，不用 eIM 的 `trustedCertificateTls`。间接下载时设备不对 SM-DP+ 建 TLS，这条链由 eIM 在 ES9+' 上完成。Immediate Enable 时，卡还核对 server 证书的 OID/FQDN 与卡内 default SM-DP+ 一致，该核对发生在 ES10b.AuthenticateServer，不属于 ESipa 这条链。

#### 5.2.7 本环节的完成判据

- 查询名、SNI、叶证书某个 dNSName 三者一致。  
- 握手使用的链终止于该 eIM 预置的锚。  
- 112 不会触发「去掉 SNI 再试」；其它 fatal 只会去掉 SNI 再试一次。  
- 验链失败与 `noEimPackageAvailable` 不会被记成同一类结果。  
- 对应用例：T06、T09、T10、T13、T19、T20、T21。

### 5.3 eIM Package 投递

**拉取：** 安全连接 → GetEimPackage(EID，可选 notify/cause/rPLMN) → 处理 BF51/52/54 → ProvideEimPackageResult 或 HandleNotification → 直至 noEimPackageAvailable。

**注入：** TransferEimPackage（BF4E）携带 BF51/52/53/54，返回结果或带关联 ID 的接收确认。

### 5.4 Profile 下载：直接与间接

两种方式安装到卡内的最后一步相同：ES10b 完成双向认证材料、PrepareDownload、LoadBoundProfilePackage，ES8+ 仍是 SM-DP+ 与 eUICC 之间的端到端保护。差异在**谁和 SM-DP+ 建立 HTTPS、报文走哪条接口、谁解析激活码与 Metadata、通知发给谁**。IPAe 与 IPAd 的呼叫语义相同；IPAe 的 ES10b 在卡内调用。

#### 5.4.1 共同前提

下载开始前须满足 SGP.22 公共双向认证的起始条件，并且：

- Emergency Profile 未处于启用。
- 寻址三选一，互斥：
  - **(a) Activation Code**：按 SGP.22 §4.1 编码，含 SM-DP+ 地址、Activation Code Token、可选 SM-DP+ OID。
  - **(b) SM-DS**：已取得 SM-DP+ 地址与 EventID（Direct 可由 IPA 经 ES11 或由 eIM 经 ES11'；规范中间接流程的起始条件写的是 eIM 已完成 ES11'）。
  - **(c) default SM-DP+**：地址来自 ES10a.GetEuiccConfiguredAddresses，或 eIM 用 IpaEuiccDataRequest 读到。
- 触发包为 `ProfileDownloadTriggerRequest`（BF54），经拉取或注入到达 IPA。格式非法或缺字段时，IPA 返回 `invalidPackageFormat` 并停止。

**MatchingID（写入 ctxParams1，供 eUICC 签名）：**

| 寻址 | MatchingID |
|------|------------|
| (a) 激活码 | Activation Code Token |
| (b) SM-DS | EventID |
| (c) default SM-DP+ | 缺省（不带） |

若已打开 Immediate Enable，eUICC 在 AuthenticateServer 时核对本次下载的 SM-DP+ OID 与 FQDN 是否等于卡内 default SM-DP+；一致才授予本次安装后的立即启用。

#### 5.4.2 差异总表

| 维度 | 直接下载 Direct | 间接下载 Indirect |
|------|-----------------|-------------------|
| 规范 | §3.2.3.1 | §3.2.3.2；取消见 §3.2.3.3 |
| 谁连 SM-DP+ | **IPA**（IPAe 经设备承载） | **eIM** |
| 对 SM-DP+ 的接口 | ES9+（IPA 扮演 LPA） | ES9+'（eIM 扮演 LPA） |
| 设备侧 RSP 通道 | 到 SM-DP+ 的 HTTPS；若由 eIM 触发，另有到 eIM 的 ESipa | **只有到 eIM 的一条安全通道**（HTTPS 或 CoAPS/底层） |
| 能力位 | `directRspServerCommunication` | `indirectRspServerCommunication`；eIM 配置含 `indirectProfileDownload` |
| 激活码由谁解析 | eIM 或 IPA 都可；触发包里应带 AC | 可在触发包中带给 IPA；若 `eimDownloadDataHandling=1`，触发体可为空，地址/Token/OID 留在 eIM |
| 地址与 Info1 由谁提供给 SM-DP+ | IPA 在 ES9+ 中提供 | IPA 可省略 FQDN/euiccInfo1，由 eIM 补齐；eIM 自填地址时必须与 `serverSigned1` 中的 FQDN 一致，否则停止 |
| OID 核对 | IPA 在公共认证中核对证书 OID 与 AC 中的 OID | AC 在 eIM 手中时，由 eIM 核对证书 OID |
| CtxParams1 | IPA 构造 | IPA 构造；若 `eimCtxParams1Generation=1`，eIM 在 InitiateAuthentication 响应里下发 |
| Metadata / PPR | **IPA** 用 RAT 与已装 Profile 校验（可先 GetRAT / GetProfilesInfo） | IPA 能校验则 eIM 把 Metadata 交给 IPA；`eimProfileMetadataVerification=1` 时 **eIM** 校验且不把 Metadata 给 IPA（RAT 需事先用 PSMO 取到） |
| 空口体积 | 全量 ES9+ 对象 | 可用 `minimizeEsipaBytes`：compactEuiccSigned1/2、省略证书，eIM 用缓存拼回再调 ES9+' |
| 取 BPP | ES9+.GetBoundProfilePackage | ESipa.GetBoundProfilePackage → ES9+'.GetBoundProfilePackage |
| 取消会话 | ES10b.CancelSession 后 **ES9+.CancelSession** | ES10b.CancelSession 后 **ESipa.CancelSession → ES9+'.CancelSession** |
| 安装结果 PIR | **ES9+.HandleNotification** 直送 SM-DP+；若本次由 eIM 触发，另用 ESipa 把 ProfileDownloadTriggerResult 告诉 eIM | **ESipa.HandleNotification** 给 eIM，eIM ACK 后 IPA 才 Remove，再由 eIM **ES9+'.HandleNotification** 给 SM-DP+ |
| TLS 形态 | 设备必须能对 RSP Server 做 TLS；NCD 还依赖对端 `max_fragment_length` | 设备可以用 CoAP/DTLS 或 MQTT/LwM2M；**eIM 对 SM-DP+ 仍是 HTTPS**，由 eIM 做协议转换 |
| 适用 | 宽带、固件能跑完整 HTTPS 客户端 | 受限设备、车队由 eIM 统一编排；**IPAe 默认推荐** |
| 失败面 | 设备直连公网 SM-DP+，DNS/证书/分片都在设备侧 | 多一跳；eIM 必须缓存会话以便还原 compact |

卡内函数两条路径相同：GetEUICCChallenge、AuthenticateServer、PrepareDownload、LoadBoundProfilePackage、CancelSession、ImmediateEnable、Retrieve/RemoveNotification。

#### 5.4.3 直接下载步骤

参与者：Operator、SM-DP+、eIM（仅触发与结果）、IPA、eUICC。

| 步 | 动作 | 接口与数据 |
|----|------|------------|
| 1 | 若 AC 在 eIM：解析出地址、Token、可选 OID；格式非法则停止 | eIM 本地 |
| 2 | 若由 eIM Package 触发：先与 eIM 建安全连接，拉取（GetEimPackage）或注入（TransferEimPackage） | ESipa |
| 3 | IPA 识别 BF54。 (a) 必须含 AC；(b) 含地址+EventID（按 AC 形式）；(c) 指示使用 default SM-DP+。非法则 `invalidPackageFormat` | ESipa |
| 4 | IPA 解析 AC（来自步骤 3 或带外） | IPA |
| 5 | 与 SM-DP+ 做 SGP.22 §3.1.2 公共双向认证。ESXX=ES9+，证书为 CERT.DPauth.ECDSA。IPA 核对证书 OID 与 AC 中 OID。按上表填 MatchingID。Immediate Enable 时卡核对 default SM-DP+ | ES9+ / ES10b.AuthenticateServer |
| 6 | SM-DP+：订单存在且 MatchingID/EID 匹配；Profile 为 Released，重试时可为 Downloaded；未超最大尝试次数；用 Device Info 与 euiccInfo2 做资格检查（含是否还能再装一个 Profile） | SM-DP+ |
| 7 | 可选：ES2+.HandleDownloadProgressInfo 通知运营商（资格检查点） | ES2+ |
| 8 | 失败则 Profile 置 Error 并停。成功则生成 smdpSigned2（TransactionID、是否要确认码、可选上次 otPK）、用 SK.DPpb.ECDSA 签名 | SM-DP+ |
| 9 | 返回 TransactionID、ProfileMetadata、smdpSigned2、smdpSignature2、CERT.DPpb.ECDSA | ES9+ |
| 10 | IPA 按 SGP.22 校验 Metadata/PPR。无 UI 的设备不应处理需要用户确认的 PPR。可事先取 RAT 与 Profile 列表 | ES10b.GetRAT / GetProfilesInfo |
| 11 | ES10b.PrepareDownload，随后走 SGP.22 下载确认子流程。PrepareDownload 失败则卡结束 RSP 会话 | ES10b / ES9+ |
| 12 | 按 SGP.22 安装子流程步骤 1–6 把 BPP 装入 eUICC | ES10b.LoadBoundProfilePackage，保护在 ES8+ |
| 13 | 若本次由 eIM 触发：把 ProfileDownloadTriggerResult 用 ESipa.HandleNotification 回给 eIM（步骤 5–12 出错也要回） | ESipa |
| 14 | ES9+.HandleNotification，SM-DP+ 继续 SGP.22 安装子流程步骤 8–10 | ES9+ |
| 15–17 | 仅 default SM-DP+：IPA 可调 ES10b.ImmediateEnable。若步骤 5 已授权，卡启用 Profile 并产生 enable 通知。IPA 按 §3.7 把新通知交给接收方，并在适用时 GetEimPackage，`notifyStateChange` 且 `stateChangeCause=immediateEnableProfile(5)` | ES10b / ES9+ 或 ESipa |

确认码如何送到设备不在本规范范围内。v1.3 之前的实现可能把 Emergency Profile 以 metadataMismatch 拒绝。

#### 5.4.4 间接下载步骤

前置除 5.4.1 外：IPA 已声明 `indirectRspServerCommunication`；eIM 已声明 `indirectProfileDownload`。SM-DS 路径的起始条件是 eIM 已用 ES11' 取得地址与 EventID。

| 步 | 动作 | 谁执行 | 接口 |
|----|------|--------|------|
| 1 | AC 在 eIM 则解析地址、Token、OID | eIM | — |
| 2–3 | 取 BF54。若无 profileDownloadData（仅 bit2=1 允许），直接去步骤 5；否则 IPA 解析 AC / default / SM-DS 指示 | IPA + eIM | ESipa |
| 4 | IPA 若自己拿到 AC 则再解析一次 | IPA | — |
| 5–6 | 需要时 GetEUICCInfo（euiccInfo1），然后 GetEUICCChallenge | IPA→eUICC | ES10b |
| 7 | 若尚未建链则建立 IPA—eIM 安全连接 | 双方 | TLS 或 DTLS |
| 8 | ESipa.InitiateAuthentication：必带 euiccChallenge；FQDN、euiccInfo1、eimTransactionId 可选。缺的由 eIM 补 | IPA | ESipa BF39 |
| 9 | eIM 与 SM-DP+ 建 **HTTPS、仅服务器认证** | eIM | TLS |
| 10 | ES9+'.InitiateAuthentication。eIM 自填的 FQDN 必须等于 serverSigned1 里的 FQDN；AC 在 eIM 时核对 OID 与 serverCertificate | eIM | ES9+' |
| 11 | 把 ES9+' 响应转成 ESipa 响应。若 IPA 不生成 ctxParams1，eIM 必须附上 | eIM | ESipa |
| 12 | AuthenticateServer。Immediate Enable 规则同直接下载 | IPA + eUICC | ES10b |
| 13 | ESipa.AuthenticateClient：euiccSigned1 或 compact、签名、可选两张证书。compact 或缺证书时 eIM 拼回完整 euiccSigned1 | IPA | ESipa BF3B |
| 14 | ES9+'.AuthenticateClient | eIM | ES9+' |
| 15 | 把结果转给 IPA。出错则 IPA 以 `undefinedReason` 取消会话。Metadata：IPA 能校验则 eIM 下发 Metadata，失败由 IPA 取消；bit4=1 则 eIM 校验且不下发 Metadata，失败码驱动 IPA 以 `pprNotAllowed` 取消 | eIM / IPA | ESipa / ES10b |
| 16 | PrepareDownload；失败则停并结束卡内 RSP 会话 | IPA + eUICC | ES10b |
| 17–18 | ESipa.GetBoundProfilePackage（euiccSignature2 + euiccSigned2 或 compact）→ eIM 调 ES9+'.GetBoundProfilePackage | IPA，eIM | ESipa BF3A / ES9+' |
| 19 | 转发 BPP。应复核 Metadata 是否与步骤 15 一致；变化则 `metadataMismatch` 取消 | eIM 或 IPA | ESipa |
| 20 | 安装步骤 1–6。有 PIR 则进入步骤 23；否则以 `loadBppExecutionError` 取消 | IPA + eUICC | ES10b |
| 21–22 | 仅 default SM-DP+ 可 ImmediateEnable；已授权则启用并产生通知 | IPA + eUICC | ES10b |
| 23 | PIR：ESipa.HandleNotification → eIM ACK → RemoveNotificationFromList → eIM 用 ES9+'.HandleNotification → SM-DP+ 执行 SGP.22 步骤 8–10。compact PIR 由 eIM 还原。PIR 也可以在 ImmediateEnable 之前送出 | IPA，eIM | ESipa / ES9+' |
| 24 | 启用产生的其它通知按 §3.7 送出（Indirect 走 eIM）。适用时再拉取，cause=immediateEnableProfile | IPA | ESipa |

**间接取消（AuthenticateClient 或 GetBPP 之后）：**

1. ES10b.CancelSession(TransactionID, reason)，卡对 {TransactionID, reason} 签名。  
2. reason 为 `undefinedReason` 时 IPA 停止继续上送（规范时序中的停止点）。  
3. 否则 ESipa.CancelSession，eIM 再 ES9+'.CancelSession。SM-DP+ 验签、核对 OID，按 SGP.22 处理原因。  

原因码：PPR 不允许用 `pprNotAllowed`；其它 AuthenticateClient 错误用 `undefinedReason`；BPP 中 Metadata 与先前不一致用 `metadataMismatch`；安装失败且无 PIR 用 `loadBppExecutionError`。

#### 5.4.5 能力位如何切开两条路径

| 位 | 对下载的影响 |
|----|----------------|
| 0 direct | 允许 ES9+ 直连，通知直送 SM-DP+ |
| 1 indirect | 允许 ESipa 三段代理，通知经 eIM |
| 2 eimDownloadDataHandling | 间接时触发包可为空，eIM 持有 AC/Event/default 地址 |
| 3 eimCtxParams1Generation | eIM 生成 ctxParams1，并因此需要 DeviceInfo（tag A9） |
| 4 eimProfileMetadataVerification | Metadata/PPR 与 BPP 一致性由 eIM 做 |
| 5 minimizeEsipaBytes | 上送 compact，eIM 用同批次静态数据、已读 euiccInfo、SM-DP+ 会话数据还原 |

eIM 在下发 BF54 之前必须已经知道这些位（带外或 BF52 的 A8）。位 0 与位 1 可以同时为 1，由本次触发和网络条件选择路径；受限 IPAe 应只承诺位 1。

#### 5.4.6 选型与实现注意

- 设备无法稳定 HTTPS 到 SM-DP+，或担心 RSP Server 不支持 `max_fragment_length`：用间接。  
- 需要 eIM 集中解析激活码、统一重试与审计：用间接，并打开位 2。  
- 实验室或宽带网关、希望少一跳延迟：用直接。  
- 两条路径不要并行开第二个 RSP 会话；新的 GetEUICCChallenge 会丢掉旧会话（一次性密钥可保留供重试）。  
- 间接的 eIM 必须按 TransactionID 保存 serverSigned1、证书和 Metadata，否则无法还原 compact，也无法做 metadataMismatch 检查。  
- IPAe 不单独实现第二套下载状态机：Engine 只负责 ESipa 与字节搬运，卡内步骤仍调用第 4.3 节的 IPA Services。

### 5.5 通知投递

公共内核：`RetrieveNotificationsList` → 投递并等 ACK → `RemoveNotificationFromList(seqNumber)`。空间不足时按最小序号淘汰旧结果与通知。

| 模式 | 能力位 | IPA 发往 | 下一跳 |
|------|--------|----------|--------|
| Direct | bit0 | SM-DP+：ES9+.HandleNotification | SM-DP+ 处理 |
| Indirect | bit1 | eIM：ESipa.HandleNotification | eIM 还原 compact 后 ES9+' 转到接收方 |

| 类型 | Direct | Indirect |
|------|--------|----------|
| ProfileInstallationResult | SM-DP+ | eIM → SM-DP+（TransactionID） |
| OtherSignedNotification | 元数据中的 NotificationAddress | 常经 eIM 转发 |
| ProfileDownloadTriggerResult | — | eIM |
| eUICC Package Result | — | eIM（Provide 或 HandleNotification） |

ACK 之前不得删除。HandleNotification 成功时 HTTP 为 204。

### 5.6 回退

| 机制 | 时机 | 动作 |
|------|------|------|
| Rollback | 刚 Enable 后无法连通 | enable.rollbackFlag；ProfileRollback；切回先前 Profile |
| Fallback | 运行期判定永久失联（检测实现定义） | setFallbackAttribute；ExecuteFallbackMechanism / ReturnFromFallback |

---

## 6 接口说明

本章展开两条常被混在一起的接口：

- **ES9+ / ES9+'**：面向 SM-DP+ 的 Profile 下载传输。ES9+ 的调用方是 IPA，ES9+' 的调用方是 eIM。函数集合与 SGP.22 §5.6 相同，HTTP 绑定遵循 SGP.22 §6。
- **ESipa**：eIM 与 IPA 之间的接口。它既代理上述下载函数，又单独承载 eIM Package（PSMO/eCO、拉数、触发、回执）。ESipa 不代替 ES8+，也不代替 ESep 签名。

### 6.1 ES9+ 与 ES9+'

#### 6.1.1 定位与安全

| 项 | ES9+ | ES9+' |
|----|------|-------|
| 规范 | SGP.32 §5.6，等同 SGP.22 §5.6（IPA 扮演 LPA） | SGP.32 §5.7（eIM 扮演 LPA） |
| 端点 | IPA — SM-DP+ | eIM — SM-DP+ |
| 出现场景 | 直接下载、直接投递通知 | 间接下载、eIM 转发通知与取消 |
| 传输 | **必须 HTTPS，仅服务器认证** | 同左 |
| TLS | SGP.22 §2.6.6；SGP.32 另要求 ES9+' 使用服务器认证 | 同左 |
| 不负责 | 不解密 Profile。机密性在 ES8+（SM-DP+ ↔ eUICC） | 同左 |

AuthenticateClient 里的 `EUICCInfo2` 必须用 SGP.32 §5.9.2 的定义（含 `ipaMode`、`iotSpecificInfo`），不能用未扩展的消费版结构。

服务器身份用 **CERT.DPauth.ECDSA**（公共双向认证）。绑定 Profile 的 **smdpSignature2** 使用 **SK.DPpb.ECDSA / CERT.DPpb.ECDSA**。两把密钥用途不同，不能混用。

#### 6.1.2 函数（五者相同，调用方不同）

SGP.22 §6 的 JSON 路径如下。ASN.1 绑定时请求都发到 `/gsma/rsp2/asn1`，用外层 Tag 区分函数。

| 函数 | 提供方 | JSON 路径 | 调用方 ES9+ | 调用方 ES9+' | 作用 |
|------|--------|-----------|-------------|--------------|------|
| InitiateAuthentication | SM-DP+ | `/gsma/rsp2/es9plus/initiateAuthentication` | IPA | eIM | 用 eUICC Challenge 启动对 SM-DP+ 的认证 |
| AuthenticateClient | SM-DP+ | `/gsma/rsp2/es9plus/authenticateClient` | IPA | eIM | 把卡签过的材料交给 SM-DP+，换回 Metadata 与 smdpSigned2 |
| GetBoundProfilePackage | SM-DP+ | `/gsma/rsp2/es9plus/getBoundProfilePackage` | IPA | eIM | 上送 PrepareDownload 结果，取回 Bound Profile Package |
| HandleNotification | SM-DP+ | `/gsma/rsp2/es9plus/handleNotification` | IPA | eIM | 投递 PIR 或其它已签名通知；成功响应体为空 |
| CancelSession | SM-DP+ | `/gsma/rsp2/es9plus/cancelSession` | IPA | eIM | 取消进行中的 RSP 会话 |

#### 6.1.3 各函数携带的数据

**InitiateAuthentication**

- 请求：`euiccChallenge`（16 字节，来自 ES10b.GetEUICCChallenge）、`euiccInfo1`、`smdpAddress`（FQDN）。
- 响应成功：`transactionId`（SM-DP+ 生成）、`serverSigned1`、`serverSignature1`（Tag 5F37）、`euiccCiPKIdToBeUsed`、`serverCertificate`（CERT.DPauth）。
- 随后 IPA 或卡用这些字段调用 ES10b.AuthenticateServer。Immediate Enable 时，卡在此核对 server OID/FQDN 是否等于 default SM-DP+。

**AuthenticateClient**

- 请求 Tag BF3B：`transactionId` + `authenticateServerResponse`（ES10b.AuthenticateServer 的响应，内含 euiccSigned1、euiccSignature1、证书）。SGP.32 明确该响应里的 EUICCInfo2 用本规范扩展结构。
- 响应成功：`profileMetadata`（StoreMetadataRequest，BF25）、`smdpSigned2`、`smdpSignature2`、`smdpCertificate`（CERT.DPpb）、可选 `hashCc`。
- SM-DP+ 在此做订单、EID、Profile 状态和资格检查。失败则不应进入 GetBoundProfilePackage。

**GetBoundProfilePackage**

- 请求：`transactionId` + `prepareDownloadResponse`（ES10b.PrepareDownload 的结果，含 euiccSigned2 / euiccSignature2，Tag BF3A）。
- 响应：`boundProfilePackage`。IPA 再调用 ES10b.LoadBoundProfilePackage。BPP 内的段由卡用 ES8+ 会话密钥解开，ES9+ 只负责把密文运到 IPA。

**HandleNotification**

- 请求：`pendingNotification`（PIR 或 OtherSignedNotification）。
- 直接下载：IPA 把 PIR 发给本次会话的 SM-DP+。OtherSignedNotification 发给元数据里的 NotificationAddress。
- 间接下载：eIM 收到 ESipa 上的通知后，PIR 必须发给**参与本次间接会话的那台 SM-DP+**；其它通知发给 NotificationAddress。字段不全时 eIM 用会话缓存补齐再调 ES9+'。
- SM-DP+ 收到 PIR 后继续 SGP.22 安装子流程步骤 8–10（确认安装、更新订单状态）。

**CancelSession**

- 请求：`transactionId` + 卡签名的 `cancelSessionResponse`（ES10b.CancelSession 输出）。
- SM-DP+ 验 euiccCancelSessionSignature 与 OID，再按原因码结束订单侧会话。
- 间接路径上 IPA 不直接调用本函数，而调用 ESipa.CancelSession，由 eIM 转发。

#### 6.1.4 ES9+ 与 ES9+' 的仅有差别

函数名、HTTPS 和 SM-DP+ 侧处理相同。差别是：

1. TLS 客户端是 IPA 还是 eIM。
2. ES9+' 允许 eIM 在转发前补齐 IPA 未上送的字段（地址、euiccInfo1、证书、compact 还原后的签名结构）。
3. eIM 自填 SM-DP+ FQDN 时，必须与响应 `serverSigned1` 中的 FQDN 一致；AC 中若有 OID，必须与 serverCertificate 的 OID 一致。任一不符即停止，不能继续 AuthenticateClient。

### 6.2 ESipa

#### 6.2.1 定位与安全

ESipa 是 eIM 与 IPA 的逻辑接口（SGP.32 §2.3、§5.14）。它做两件事：

1. **触发并运送下载**：把 ProfileDownloadTrigger 交给 IPA；在间接下载中代理 ES9+' / ES11'。
2. **运送 eUICC Package**：PSMO 与 eCO 的签名包走 ESipa，端到端验证在 ESep（卡用 eIM 公钥验签）。ESipa 本身只保证传输上的机密性与完整性。

传输三选一，双方必须事先用同一模式（§3.1.1、§3.1.2）：

| 模式 | 安全 | 典型设备 |
|------|------|----------|
| HTTP over TLS | 仅服务器认证，TLS ≥1.2，规则见 §2.6.3.2 | 有 TCP 的设备；规范推荐 TCP 可用时用 HTTPS |
| CoAP over DTLS | 仅服务器认证；建议 CID 与会话恢复 | NCD / LPWAN |
| 底层传输 | 由 MQTT、LwM2M 等提供同等机密性与完整性 | 已有设备管理通道；互通责任在集成方 |

IPAe 且使用 Retrieve HTTPS/CoAPS 时，eIM 配置里的 `trustedCertificateTls` 应预先写入，否则卡内没有信任锚，握手会失败。证书吊销：IPAd 的 TLS 要能做 CRL/OCSP；DTLS 不做吊销检查。

HTTP 头：`User-Agent` 为 `gsma-rsp-ipad` 或 `gsma-rsp-ipae`；`X-Admin-Protocol: gsma/rsp/v<x.y.z>` 表示发送方支持的最高 SGP.32 版本。

#### 6.2.2 两类函数

**A. 下载代理（仅间接下载 / 间接 Event 发现时使用）**

这些函数的提供方是 **eIM**，调用方是 **IPA**。eIM 收到后调用对应的 ES9+' 或 ES11'，再把响应改回 ESipa。

| 函数 | Tag | JSON 路径 | 请求要点 | eIM 接着调用 |
|------|-----|-----------|----------|--------------|
| InitiateAuthentication | BF39 | `/gsma/rsp2/esipa/initiateAuthentication` | euiccChallenge 必填；euiccInfo1、smdpAddress 可选；若触发包带了 eimTransactionId 则必须原样带回 | ES9+'.InitiateAuthentication 或 ES11'.InitiateAuthentication |
| AuthenticateClient | BF3B | `.../authenticateClient` | transactionId + authenticateServerResponse（可 compact） | ES9+'.AuthenticateClient 或 ES11' |
| GetBoundProfilePackage | BF3A | `.../getBoundProfilePackage` | transactionId + prepareDownloadResponse（可 compact） | ES9+'.GetBoundProfilePackage |
| CancelSession | BF41 | `.../cancelSession` | transactionId + 卡签名的 cancelSessionResponse | ES9+'.CancelSession |
| HandleNotification | BF3D | `.../handleNotification` | pendingNotification **或** provideEimPackageResult，二者只出现一个 | PIR/其它通知则 ES9+'；若是包结果则 eIM 自己处理 |

`minimizeEsipaBytes=1` 时 IPA 不送 euiccInfo1 和 smdpAddress，eIM 用已有数据补上。`eimDownloadDataHandling=1` 且使用激活码时，IPA **不得**再送 smdpAddress。未知的 eimTransactionId：eIM 返回无效关联（规范状态码 8.31.2 / 3.10），不能当成新会话。

**B. eIM Package（ES9+ 没有对应函数）**

| 函数 | Tag | 方向 | JSON 路径 | 作用 |
|------|-----|------|-----------|------|
| GetEimPackage | BF4F | IPA → eIM | `.../getEimPackage` | 按 EID 拉一条待处理包；可带 notifyStateChange、stateChangeCause、rPLMN |
| TransferEimPackage | BF4E | eIM → IPA | `.../transferEimPackage` | 注入 BF51/BF52/BF53/BF54 之一 |
| ProvideEimPackageResult | BF50 | IPA → eIM | `.../provideEimPackageResult` | 回传执行结果；可附带通知列表以取得序号 ACK |

GetEimPackage 的响应是四者之一：BF51 签名包、BF52 数据请求、BF54 下载触发，或错误。错误码：`noEimPackageAvailable(1)`、`eidNotFound(2)`、`invalidEid(3)`、`missingEid(4)`、`undefinedError(127)`。空响应 `noEimPackageAvailable` 是正常结束，不是故障。

同一对 eIM 与 IPA 只能约定一种投递模式（拉取或注入）和一种安全连接，不能在一次部署里混用拉取与注入。

#### 6.2.3 与 ES9+ 的字段对应

间接下载时，ESipa 不是另一套认证，而是 ES9+' 的设备侧封装：

| 阶段 | IPA 发给 eIM | eIM 发给 SM-DP+ | 卡内 |
|------|--------------|-----------------|------|
| 启动 | ESipa.InitiateAuthentication | ES9+'.InitiateAuthentication | 之前已 GetEUICCChallenge |
| 卡认证服务器 | — | — | ES10b.AuthenticateServer |
| 服务器认证卡 | ESipa.AuthenticateClient | ES9+'.AuthenticateClient | — |
| 准备下载 | — | — | ES10b.PrepareDownload |
| 取包 | ESipa.GetBoundProfilePackage | ES9+'.GetBoundProfilePackage | 之后 LoadBoundProfilePackage |
| 结果 | ESipa.HandleNotification | ES9+'.HandleNotification | RemoveNotificationFromList |
| 取消 | ESipa.CancelSession | ES9+'.CancelSession | 之前 ES10b.CancelSession |

直接下载不经过上表的 ESipa 代理列：IPA 自己调用 ES9+。即便直接下载，若下载是 eIM Package 触发的，触发和 ProfileDownloadTriggerResult 仍然走 ESipa，只有与 SM-DP+ 的五次调用走 ES9+。

#### 6.2.4 错误码（实现时按函数区分）

**ESipa.InitiateAuthentication：** invalidDpAddress(1)、euiccVersionNotSupportedByDp(2)、ciPKIdNotSupported(3)、smdpAddressMismatch(50)、smdpOidMismatch(51)、invalidEimTransactionId(52)、undefinedError(127)。

**ESipa.AuthenticateClient：** 证书或签名 1–5、matchingIdRefused(6)、eidMismatch(7)、noEligibleProfile(8)、ciPKUnknown(9)、invalidTransactionId(10)、insufficientMemory(11)、downloadOrderExpired(18)、pprNotAllowed(50)、eventIdUnknown(56)、127。

**ESipa.GetBoundProfilePackage：** euiccSignatureInvalid(1)、确认码 2–4、bppRebindingRefused(5)、metadataMismatch(50)、invalidTransactionId(95)、127。

**包结果：** invalidPackageFormat(1)、unknownPackage(2)、127。卡侧验签失败还包括 replayError、counterValueOutOfRange、sizeOverflow、ecallActive。

ES9+ 的 JSON 错误如何放进 ProfileDownloadTriggerResult.errorResponse，规范仍标为 FFS。排障时以 ASN.1 错误码和 SM-DP+ / eIM 日志为准。

### 6.3 ASN.1 与 JSON

| 项 | ASN.1 | JSON |
|----|-------|------|
| HTTP | POST `/gsma/rsp2/asn1`；`application/x-gsma-rsp-asn1` | 按函数分路径；`application/json;charset=UTF-8` |
| 选择函数 | 外层 CHOICE Tag | URL |
| 二进制 | 原始 DER | 默认 base64(完整 DER)；euiccChallenge 仅 value |
| 头 | User-Agent；`X-Admin-Protocol: gsma/rsp/v<x.y.z>` | 相同 |
| CoAP / LwM2M | 仅 ASN.1 | 规范不定义 JSON 绑定 |
| eIM | 必须支持 | 必须支持 |

ES9+ 的 JSON 错误映射到下载触发结果仍为规范未完成项（FFS）；排障优先看 ASN.1。

### 6.4 CoAP 与 LwM2M

- 版本选项号 2056，三字节（v1.3.0 = `01 03 00`）。  
- Content-Format 省略或 42；大包 block-wise + Request-Tag；Echo 做新鲜性。  
- LwM2M Object 3443：0 Status、1 EID、2 eIM message（写 DER）、3 IPA message（读 DER）、4 确认清空。

---

## 7 安全设计

### 7.1 信任分层

| 层 | 机制 |
|----|------|
| Profile 机密与完整 | ES8+（eUICC ↔ SM-DP+，含前向安全） |
| 管理命令 | ESep：eIM 私钥签名，卡用关联公钥验证 |
| 传输 | TLS/DTLS 或底层同等保护 |
| 模式与通道 | 仅授权应用可选 ISD-R；IPAe 时关闭对外 ES10 |

IPAe 纳入 eUICC 安全评估。Indirect 把 SM-DP+ 会话终结在 eIM，缩小设备暴露面。

### 7.2 TLS / DTLS

建链步骤、SNI 分支和证书链核对见第 5.2 节。这里只保留和业务密钥的分界：

| 对象 | 用途 | 不用于 |
|------|------|--------|
| CERT.EIM.TLS 及其链，锚为 trustedCertificateTls / trustedEimPkTls | ESipa 服务器认证 | 验 eUICC Package 签名 |
| CERT.EIM.ECDSA / eimPublicKeyData | ESep 验签 | TLS 握手 |
| CERT.DPauth.ECDSA | 直连 SM-DP+ 的 ES9+ | 代替 eIM 的 TLS 锚 |

可选曲线：brainpoolP256r1、P-384、Ed25519/x25519。可选套件：TLS 1.2 的 `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`，TLS 1.3 的 `TLS_CHACHA20_POLY1305_SHA256`。强制曲线和强制套件见 5.2.3。

### 7.3 防重放与结果

- 每 Associated eIM 独立 counter；结果带回 eimId、counter、可选事务 ID 与 eUICC 序号。  
- associationToken 防止「清空配置后重放整段 AddInitialEim 序列」。  
- 卡不处理 CRL/OCSP；不得把吊销信息下发到 eUICC。

---

## 8 实现与部署建议

### 8.1 阶段检查清单

| 阶段 | 检查 |
|------|------|
| 制造 | ipaeSupported；初始 eIM（id、公钥、counter、FQDN、TLS 锚）；Bootstrap 与 DGI 3A07 |
| 上电 | b1、E1、BF42 规则、ipaMode |
| 事件 | PROFILE、EVENT LIST、Timer |
| 建链 | 忙检测、DNS、SNI、证书、协议版本头 |
| 能力 | A8 与协议位交集 |
| 下载与通知 | 模式分流、ACK 后删除 |
| 运营 | counter 单调、包约束、Fallback 唯一、多 eIM 选路（实现定义） |
| 双模 | 复位切换后 ES10b 与 ESipa 结果一致 |

### 8.2 任务拆分

| 包 | 交付 |
|----|------|
| M1 Mode Manager | Terminal Cap、BF42、ipaMode、ES10 门控 |
| M2 IPA Services | ES10b 全集与单测 |
| M3 APDU Adapter | IPAd 与 SGP.33 子集 |
| M4 IPAe Engine | 状态机、CAT、BIP、DNS/TLS |
| M5 ESipa Client | ASN.1（及 JSON）、8 函数 |
| M6 通知 | 分流与删除 |
| M7 Package | LoadEuiccPackage、counter、PSMO/eCO |
| M8 集成 | T01–T21、退避与功耗参数 |

### 8.3 部署建议

- 受限设备默认 IPAe + Indirect + ASN.1。  
- eIM 同时实现拉取与注入、ASN.1 与 JSON。  
- addEim 时一并写入 FQDN、SAN 匹配的证书与信任锚。  
- 轮询用 Timer，关键事件作补充，避免只依赖 Location（静止设备可能永不取包）。  
- 在设计说明中写明 IPAe 如何读取 Connectivity Parameters。

---

## 9 测试与验收

策略：研发用下表冒烟；认证按 SGP.33-1/2/3 与 SGP.24 的 IoT 扩展。每条对应用例应记录模式（IPAd/IPAe）、绑定（ASN.1/JSON）与传输（HTTPS/CoAPS）。

| ID | 场景 | 前置 | 期望 |
|----|------|------|------|
| T01 | 自动激活 | b1=0 且支持 IPAe | ipaMode=ipae |
| T02 | 不自动激活 | b1=1 且无 BF42 | ipaMode=ipad，ES10 可用 |
| T03 | BF42 成功 | b1=1 且支持 | ok，对外 ES10 关闭 |
| T04 | BF42 失败 | 不支持 IPAe | notSupported |
| T05 | FCI | SELECT ISD-R | 解析出 E1 两位 |
| T06 | Timer 建链 | IPAe | 通道 + TLS + GetEimPackage |
| T07 | 忙不抢占 | 会话中再触发 | 结束后补拉 |
| T08 | catBusy | proactive 中 Enable | 可恢复成功 |
| T09 | SNI 112 | 错误主机名 | 握手中止 |
| T10 | SNI 回退 | 非 112 fatal | 无 SNI 重试行为符合设计 |
| T11 | Indirect 通知 | bit1 | ESipa → ES9+' → Remove |
| T12 | Direct 通知 | bit0 | ES9+ 直送 → Remove |
| T13 | 缺信任锚 | IPAe Retrieve HTTPS | 预期失败 |
| T14 | 内核复用 | 双模同一 PSMO | APDU 与 ESipa 结果一致 |
| T15 | 事件 | EVENT LIST + Location | IPAe 收到触发 |
| T16 | CoAP ASN.1 | CoAPS | GetEimPackage 成功 |
| T17 | JSON | HTTP | 同一函数成功 |
| T18 | 切回 IPAd | 复位且 b1=1 | ipad 模式 |
| T19 | SAN 不覆盖 SNI | 叶证书 dNSName 与连接 FQDN 不一致 | 握手失败，不发 ESipa |
| T20 | 链不到锚 | 路径终止证书不是 trustedCertificateTls | 拒绝，不回落到公共 CA |
| T21 | 无 SNI | ClientHello 不带 server_name | 服务器出示默认域链；客户端仍用原 FQDN 验 SAN |

**设计内验收点（并入上表即可）：** b1 与 ES10 门控；E1.bit0；ACK 前不删通知；compact 可由 eIM 还原；会话中 Timer 不打断 TLS；空轮询不导致过高 OPEN CHANNEL 频率。

---

## 10 风险与故障分析

### 10.1 故障树

```
取包失败
├─ 模式错误 → Terminal Cap、E1、BF42、ipaMode
├─ 无触发 → EVENT LIST、Timer、STATUS
├─ 通道失败 → BIP、APN(3A07)、占用
├─ DNS → eimFqdn / Broker
├─ TLS → SNI（112 不重试 / 其它 fatal 去 SNI 再试一次）、证书链、SAN、套件
├─ 协议 → User-Agent、X-Admin-Protocol、DER/JSON
└─ 无包 → noEimPackageAvailable（正常）

通知残留
├─ 分流与能力位不一致
├─ 未等 ACK 就 Remove
└─ compact 未还原

下载失败
├─ 能力无交集（先读 A8）
├─ Metadata / PPR 校验方错误
└─ 并行 RSP
```

### 10.2 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| 只依赖 Location 触发 | 静止设备长期不取包 | Timer 为主 |
| eIM 只实现 JSON | CoAP/LwM2M 不通 | 双绑定作为准入 |
| 多 eIM 无优先级 | 行为不一致 | 产品规格写明选路，并在 IPA 实现中固定 |
| NCD 走 Direct | 分片与互通失败 | 默认 Indirect |
| 声明 IPAd 却期望自动 IPAe | 现场无法开通 | 出厂配置与 T02 卡控 |
| BIP 与 USIM 争用 | catBusy 风暴 | 状态机退避，必要时内部结束 proactive |

---

## 11 结论与建议

1. IPAe 适合固件不可承载 IPA 的设备；对 eIM 与 SM-DP+ 的业务语义与 IPAd 相同，差异在承载与 ES10 调用位置。  
2. 可落地的产品结构是 Mode Manager + 单一 IPA Services + IPAe 引擎；用 BF42 与 Terminal Capability 做硬门控。  
3. 受限场景应固定为 Indirect、ASN.1、TLS 信任锚预置，以及「Timer + 忙队列」的取包策略。  
4. 通知路径必须与下载模式一致，否则 SM-DP+ 收不到 PIR 或 eIM 状态发散。  
5. 建议先完成 M1–M4 与 T01–T10、T19–T21（含 SNI 与证书链），再做下载与通知（T11–T18），最后对照 SGP.33 做认证子集。

---

## 参考文献

| 文献 | 用途 |
|------|------|
| GSMA SGP.32 v1.3 | 主规范：架构、流程、ASN.1、安全 |
| GSMA SGP.31 v1.3 | 需求与角色 |
| GSMA SGP.22 | ES8+/ES9+、认证、通知与 HTTP 基线 |
| GSMA SGP.33-1/2/3 | eUICC / IPA / eIM 测试 |
| ETSI TS 102 221 / 102 223 | Terminal Capability、CAT、BIP |
| RFC 6066、8446、9146、9147、7959、9175 | SNI、TLS 1.3、CID、DTLS、分块、Echo |
| TCA IoT eSIM Technical Guide（2026-10） | 产业解读 |

---

## 附录 A 章节对照

旧稿按主题平铺为 22 节。本版按通用技术报告归并如下。

| 通用章节 | 并入的原内容 |
|----------|----------------|
| 摘要、1 引言 | 目的、读者、术语 |
| 2 背景与需求 | 问题空间、选型决策树 |
| 3 总体方案 | 架构、接口地图、分层 |
| 4 详细设计 | 能力、SELECT/激活、双模 ES10b、CAT/BIP、忙处理、Package/Capabilities/连通参数 |
| 5 关键流程 | 上电、DNS/TLS、取包、Direct/Indirect、通知、Fallback |
| 6 接口说明 | ESipa Tag/路径、ASN.1/JSON、CoAP、LwM2M、错误码 |
| 7 安全设计 | TLS 细则、证书、防重放、信任分层 |
| 8 实现与部署 | 检查清单、M1–M8、部署建议 |
| 9 测试与验收 | T01–T21 与分节验收点 |
| 10 风险与故障 | 故障树、陷阱 |
| 11 结论 | 方案结论与实施顺序 |
| 参考文献、附录 | 文献与修订 |

## 附录 B 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2026-11-14 | 首版可执行内容（主题平铺） |
| 1.1 | 2026-11-14 | 按通用技术报告章节重组，技术内容保留并归并 |
| 1.2 | 2026-11-14 | 第 5.4 节展开直接/间接下载步骤、报文分工与差异 |
| 1.3 | 2026-11-14 | 第 6 章补充 ES9+/ES9+' 与 ESipa 的函数、报文和安全 |
| 1.4 | 2026-11-14 | 第 5.2 节单独展开 DNS、SNI 与 TLS 证书链 |

---

*文档结束。*
