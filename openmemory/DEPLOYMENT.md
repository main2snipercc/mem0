# OpenMemory Docker 部署指南

本指南将帮助您将OpenMemory项目打包到DockerHub并实现一键部署。

## 📋 前置要求

- Docker 20.10+
- Docker Compose 2.0+
- DockerHub账户
- OpenAI API密钥

## 🚀 快速开始

### 1. 准备环境

```bash
# 克隆项目
git clone https://github.com/mem0ai/mem0.git
cd mem0/openmemory

# 创建环境配置文件
cp env.example .env
```

### 2. 配置环境变量

编辑 `.env` 文件，填入必要的配置：

```bash
# 必需配置
USER_ID=your_user_id
OPENAI_API_KEY=your_openai_api_key

# 可选配置
DATABASE_URL=sqlite:///./data/openmemory.db
QDRANT_URL=http://qdrant:6333
```

### 3. 构建和推送镜像

```bash
# 给脚本执行权限
chmod +x build-and-push.sh

# 构建并推送到DockerHub
./build-and-push.sh [version] [dockerhub-username]

# 示例
./build-and-push.sh v1.0.0 myusername
```

### 4. 更新Docker Compose配置

编辑 `docker-compose.prod.yml`，将 `your-dockerhub-username` 替换为您的DockerHub用户名：

```yaml
services:
  openmemory-api:
    image: your-dockerhub-username/openmemory-api:latest
  openmemory-ui:
    image: your-dockerhub-username/openmemory-ui:latest
```

### 5. 启动服务

```bash
# 启动所有服务
docker-compose -f docker-compose.prod.yml up -d

# 查看服务状态
docker-compose -f docker-compose.prod.yml ps

# 查看日志
docker-compose -f docker-compose.prod.yml logs -f
```

## 🔧 服务配置

### 服务端口

- **UI界面**: http://localhost:3000
- **API接口**: http://localhost:8765
- **API文档**: http://localhost:8765/docs
- **Qdrant管理**: http://localhost:6333/dashboard

### 数据持久化

数据将存储在以下Docker卷中：

- `qdrant_data`: Qdrant向量数据库数据
- `api_data`: API服务数据（SQLite数据库）

### 环境变量说明

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `USER_ID` | 是 | - | 用户ID，用于标识记忆的所有者 |
| `OPENAI_API_KEY` | 是 | - | OpenAI API密钥 |
| `DATABASE_URL` | 否 | sqlite:///./data/openmemory.db | 数据库连接URL |
| `QDRANT_URL` | 否 | http://qdrant:6333 | Qdrant服务地址 |

## 🛠️ 高级配置

### 使用PostgreSQL数据库

```yaml
# 在docker-compose.prod.yml中添加PostgreSQL服务
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: openmemory
      POSTGRES_USER: openmemory
      POSTGRES_PASSWORD: your_password
    volumes:
      - postgres_data:/var/lib/postgresql/data

# 更新API服务环境变量
openmemory-api:
  environment:
    - DATABASE_URL=postgresql://openmemory:your_password@postgres:5432/openmemory
```

### 使用Nginx反向代理

```bash
# 启动包含Nginx的完整服务
docker-compose -f docker-compose.prod.yml --profile nginx up -d
```

### 配置HTTPS

1. 将SSL证书放置在 `ssl/` 目录下
2. 更新 `nginx.conf` 中的证书路径
3. 启动Nginx服务

## 📊 监控和维护

### 健康检查

所有服务都配置了健康检查：

```bash
# 检查服务健康状态
docker-compose -f docker-compose.prod.yml ps

# 查看健康检查日志
docker inspect openmemory-api | grep -A 10 Health
```

### 日志管理

```bash
# 查看所有服务日志
docker-compose -f docker-compose.prod.yml logs

# 查看特定服务日志
docker-compose -f docker-compose.prod.yml logs openmemory-api

# 实时查看日志
docker-compose -f docker-compose.prod.yml logs -f --tail=100
```

### 数据备份

```bash
# 备份数据库
docker exec openmemory-api sqlite3 /usr/src/openmemory/data/openmemory.db ".backup /backup/backup.db"

# 备份Qdrant数据
docker run --rm -v qdrant_data:/data -v $(pwd)/backup:/backup alpine tar czf /backup/qdrant_backup.tar.gz -C /data .
```

## 🔄 更新和升级

### 更新镜像

```bash
# 拉取最新镜像
docker-compose -f docker-compose.prod.yml pull

# 重启服务
docker-compose -f docker-compose.prod.yml up -d
```

### 回滚版本

```bash
# 修改docker-compose.prod.yml中的镜像标签
# 然后重启服务
docker-compose -f docker-compose.prod.yml up -d
```

## 🐛 故障排除

### 常见问题

1. **服务启动失败**
   ```bash
   # 检查日志
   docker-compose -f docker-compose.prod.yml logs
   
   # 检查端口占用
   netstat -tulpn | grep :3000
   netstat -tulpn | grep :8765
   ```

2. **数据库连接问题**
   ```bash
   # 检查数据库服务状态
   docker-compose -f docker-compose.prod.yml ps qdrant
   
   # 检查数据库连接
   docker exec openmemory-api python -c "from app.database import engine; print(engine.url)"
   ```

3. **内存不足**
   ```bash
   # 检查系统资源
   docker stats
   
   # 清理未使用的镜像
   docker system prune -a
   ```

### 性能优化

1. **调整工作进程数**
   ```yaml
   openmemory-api:
     environment:
       - API_WORKERS=8  # 根据CPU核心数调整
   ```

2. **配置资源限制**
   ```yaml
   openmemory-api:
     deploy:
       resources:
         limits:
           memory: 2G
           cpus: '1.0'
   ```

## 📚 相关文档

- [OpenMemory官方文档](https://docs.mem0.ai)
- [Docker Compose文档](https://docs.docker.com/compose/)
- [Qdrant文档](https://qdrant.tech/documentation/)

## 🤝 支持

如果您遇到问题，请：

1. 查看本文档的故障排除部分
2. 检查GitHub Issues
3. 加入Discord社区获取帮助

## 📄 许可证

本项目采用Apache 2.0许可证。详情请参阅[LICENSE](../LICENSE)文件。
