# Lens

## 简介

Lens 是非侵入的，不需要在集群上做任何操作，类似 kubectl 一样，不像 dashboard、Kuboard 等面板工具，基于 Web 端，需要在集群部署相关的服务。

从 2023 年 1 月开始，Lens 需要选择付费版还是免费版，如果从官网下载，需要激活。但是我们可以使用 OpenLens。

OpenLens 是一个开源项目，支持 Lens 主要功能。该代码由 Lens 团队开发人员与社区一起开发，截至目前，它保持免费。

Lens 建立在 OpenLens 项目之上，类似 Ansible Tower 和 AWX 的关系，包括一些具有不同许可证的附加软件和库, Lens 的核心功能将在 Openlens 中可用。

关于 OpenLens, 可以到 github 上了解更多

[OpenLens](https://github.com/MuhammedKalkan/OpenLens 'OpenLens')

## 安装 OpenLens

[OpenLens](https://github.com/MuhammedKalkan/OpenLens/releases 'OpenLens')

下载相应版本并安装，相关的集群操作跟 Lens 差不多。
