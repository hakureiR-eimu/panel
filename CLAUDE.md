# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

个人面板项目，规划包含三个独立后端服务和统一前端，目前阶段已实现 nginx 静态资源服务：

- **static/** — 静态 HTML 页面（已部署，通过 nginx 直接提供服务）
- **nginx/** — 反向代理配置，将请求路由到各服务
- **frontend/** — 前端应用（待开发）
- **backend-blog/** — 博客后端服务（待开发）
- **backend-quant/** — 量化面板后端服务（待开发）
- **backend-embedding/** — 代码语义向量化搜索服务（待开发）

## 架构

```
用户 → nginx(:8080→:80) → /static/*  → static/ (静态 HTML)
                           (预留)
                           → /api/blog/*    → backend-blog (:8000)
                           → /api/quant/*   → backend-quant (:8001)
                           → /api/embed/*   → backend-embedding (:8002)
```

nginx 作为唯一入口，目前仅配置了 `/static/` 路径的静态资源服务。后端路由待各服务开发后再添加。

## 开发须知

- **小步快跑**：不要一次性生成大量代码，每次只推进一个明确的步骤（如生成项目骨架、实现某个指定功能），完成后确认再继续下一步
