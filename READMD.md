# 个人面板

1. nginx服务与静态资源
2. 量化面板后端

## 快速启动

使用 Docker Compose 一键启动所有服务：

```bash
# 启动所有服务（后台运行）
docker compose up -d

# 查看服务状态
docker compose ps

# 查看实时日志
docker compose logs -f

# 停止所有服务
docker compose down

# 重新构建并启动（代码有变更时）
docker compose up -d --build
```

启动后访问 [http://localhost:8080](http://localhost:8080) 即可。