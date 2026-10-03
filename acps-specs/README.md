**[English](README_en.md) | [中文](README.md)**

# Agent Collaboration Protocols Specifications

## 1. 项目介绍

本项目作为智能体互联开源生态的一部分，由北京邮电大学牵头，依托中国电子技术标准化研究院开发，于2025年11月发布第一版。

文档编写单位：北京邮电大学人工智能学院，中国电子技术标准化研究院

文档编写者：刘军（北京邮电大学），高歌（中国电子技术标准化研究院），李珂（北京邮电大学），陈科良（北京邮电大学），禹可（北京邮电大学），胡晓峰（北京邮电大学），马镝（北京邮电大学）。

## 2. 协议规范索引

智能体协作协议体系（Agent Collaboration Protocols，ACPs）是基于AIP为保障异构智能体之间高效协作、支持多样化智能体互联应用而设计与实现的标准化交互协议体系，主要包含如下协议文档：

(1)[智能体协作协议体系概述（Agent Collaboration Protocols，ACPs）](01-ACPs-spec-overview/ACPs-spec-overview.md)

(2)[智能体身份码（Agent Identity Code，AIC）规范](02-ACPs-spec-AIC/ACPs-spec-AIC.md)

(3)[智能体能力描述（Agent Capability Specification，ACS）规范](03-ACPs-spec-ACS/ACPs-spec-ACS.md)

(4)[智能体可信注册（Agent Trusted Registration，ATR）规范](04-ACPs-spec-ATR/ACPs-spec-ATR.md)

(5)[智能体身份认证（Agent Identity Authentication，AIA）规范](05-ACPs-spec-AIA/ACPs-spec-AIA.md)

(6)[智能体发现协议（Agent Discovery Protocol，ADP）规范](06-ACPs-spec-ADP/ACPs-spec-ADP.md)

(7)[智能体交互协议（Agent Interaction Protocol，AIP）规范](07-ACPs-spec-AIP/ACPs-spec-AIP.md)

(8)[数据同步协议（Data Synchronization Protocol，DSP）规范](08-ACPs-spec-DSP/ACPs-spec-DSP.md)

(9)[智能体监控协议（Agent Monitoring Protocol，AMP）规范](09-ACPs-spec-AMP/ACPs-spec-AMP.md)

(10)[智能体访问控制（Agent Access Control，AAC）规范](10-ACPs-spec-AAC/ACPs-spec-AAC.md)

## 3. 多语言文档约定

本目录的每个规范文档采用「中文原文 + 同目录英文版」的方式维护，英文版文件名在中文文件名后加 `_en` 后缀，例如 `ACPs-spec-AIC.md` 与 `ACPs-spec-AIC_en.md`。

两个版本均在文档顶部提供语言切换入口（紧跟在 `[首页](../README.md)` 之后）：

```markdown
**[English](ACPs-spec-AIC_en.md) | [中文](ACPs-spec-AIC.md)**
```

翻译约定：图片路径与代码块保持原样（英文版与中文版位于同一目录，共用同一批图片）；指向本目录其他规范的相对链接改为对应的 `_en` 版本，因此英文版之间始终互相跳转而不会跳到中文版；正文中引用的章节锚点同步改为英文标题对应的锚点。本目录 11 个文档均已提供英文版（`README_en.md` 及 `01`～`10` 各规范目录下的 `*_en.md`），覆盖 1～10 全部规范。
