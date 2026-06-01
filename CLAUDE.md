# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

个人面板项目，包含三个独立后端服务和统一前端：

- **frontend/** — 前端应用，负责博客展示和量化面板 UI
- **backend-blog/** — 博客后端服务（文章 CRUD、分类、标签等）
- **backend-quant/** — 量化面板后端服务（策略数据、指标计算等）
- **backend-embedding/** — 代码语义向量化搜索服务（代码片段向量化、语义检索等）
- **nginx/** — 反向代理配置，将请求路由到各服务
- **docker-compose.yml** — 容器编排，统一管理所有服务的启动

## 架构

```
用户 → nginx(:80/:443) → frontend (静态资源)
                         → backend-blog (:8000)
                         → backend-quant (:8001)
                         → backend-embedding (:8002)
```

nginx 作为唯一入口，按路径前缀将请求分发到对应的后端服务。

## 开发须知

- 这是一个从零开始的项目，每一步都要解释清楚再做下一步
- 做技术选型时先说明理由，再动手实现
- 新增依赖或引入工具链时，说明它解决什么问题
