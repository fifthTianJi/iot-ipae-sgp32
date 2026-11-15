# IoT-IPAe 技术方案报告

基于 [GSMA SGP.32 v1.3](https://www.gsma.com/solutions-and-impact/technologies/esim/)（2026-05-22）的 **eSIM IoT / IPAe** 可执行技术报告。面向没有本地界面、网络受限的设备：eIM 远程编排，IPA 放在 eUICC（IPAe）或设备（IPAd），Profile 下载走直接或间接两条路径。

这是一份**解读与落地指南**。互通和认证以 GSMA、ETSI 正式文本为准，本仓库不替代规范，也不附带规范原文。

- 阅读正文：[docs/iot-ipae-sgp32-executable-tech-report.md](docs/iot-ipae-sgp32-executable-tech-report.md)
- 下载 PDF：[docs/iot-ipae-sgp32-executable-tech-report.pdf](docs/iot-ipae-sgp32-executable-tech-report.pdf)
- 版本：1.4 · 2026-11-14（Asia/Shanghai）

## 这篇报告解决什么

消费级 RSP（SGP.22）依赖设备上的 LPA 和用户确认。大量 IoT 设备没有 UI，固件里也不适合再做一套 IPA。SGP.32 把编排放到 eIM，把 IPA 收成两种形态：

- **IPAd**：IPA 在设备上，经 APDU 调 ES10b。
- **IPAe**：IPA 在 eUICC 内，由 Terminal Capability 与 `IpaeActivationRequest`（BF42）决定是否激活。同一时刻只有一种模式。

报告按「需求 → 架构 → 详细设计 → 流程 → 接口 → 安全 → 实现 → 测试」写成，关键步骤写到 Tag、判据和失败分支，并附验收用例 T01–T21、实现任务包 M1–M8。

## 建议阅读顺序

| 读者 | 先看 |
|------|------|
| 架构、方案 | 第 2、3、11 章 |
| eUICC 固件 | 第 4、5、6、7、8 章 |
| 模组、设备 | 4.1–4.4、5.1–5.2 |
| eIM、平台 | 5.3–5.5、第 6、7 章 |
| 测试 | 第 9、10 章 |

按主题可以直接跳到这些节：

| 主题 | 位置 |
|------|------|
| IPAe 激活、SELECT ISD-R 的 E1、与 IPAd 共存 | 第 4 章 |
| DNS、SNI、TLS 证书链 | 5.2 |
| 直接下载与间接下载对比 | 5.4 |
| 通知发给谁 | 5.5 |
| ES9+ / ES9+' 与 ESipa | 第 6 章 |
| 验收用例 | 第 9 章，T01–T21 |

## English

An implementation-oriented reading of GSMA SGP.32 v1.3 for the in-eUICC IoT Profile Assistant (IPAe): one ES10b service kernel shared with IPAd, activation via Terminal Capability and BF42, ESipa to the eIM, ES9+ / ES9+' to the SM-DP+, DNS and TLS with SNI and the eIM certificate chain, and a direct vs indirect profile download comparison. Cite the published GSMA specification for conformance. This repository does not contain the specification text.

## 仓库里有什么

```
README.md
LICENSE          报告正文：CC BY 4.0
NOTICE           第三方规范的权利说明
docs/
  iot-ipae-sgp32-executable-tech-report.md
  iot-ipae-sgp32-executable-tech-report.pdf
```

## 许可与引用

报告正文（作者自己的结构、步骤、表格和判据）采用 [Creative Commons Attribution 4.0](https://creativecommons.org/licenses/by/4.0/)。转载请注明作者与本仓库地址，并保留「以 GSMA 正式文本为准」这句边界。

GSMA、ETSI 规范，以及其中的接口名、Tag 和错误码，权利属于各自发布方。本仓库**不提供** SGP.22 / SGP.31 / SGP.32 / SGP.33 的 PDF 或原文摘录。需要规范请向 GSMA 获取。

引用示例：

> IoT-IPAe 技术方案报告（SGP.32 可执行版）v1.4，2026-11-14。依据 GSMA SGP.32 v1.3。
