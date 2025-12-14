# 项目部署文档

## 项目概述

这是一个基于 Mastra 框架的垃圾分类 AI 应用，支持多种部署方式：

- **本地开发部署**：使用 PM2 进程管理器
- **Cloudflare Workers 部署**：无服务器边缘计算平台

## 部署架构

### 1. 本地/服务器部署架构

```
┌─────────────────────────────────────┐
│             PM2 Process             │
├─────────────────────────────────────┤
│        Mastra Application           │
├─────────────────────────────────────┤
│    - wasteClassifier Agent          │
│    - simpleClassifier Agent         │
│    - Classification Workflow        │
├─────────────────────────────────────┤
│           Node.js Runtime           │
└─────────────────────────────────────┘
```

### 2. Cloudflare Workers 部署架构

```
┌─────────────────────────────────────┐
│       Cloudflare Edge Network      │
├─────────────────────────────────────┤
│         Worker Runtime              │
├─────────────────────────────────────┤
│       worker.ts Entry Point         │
│    - API路由处理                    │
│    - Agent调用                      │
│    - CORS处理                       │
└─────────────────────────────────────┘
```

## 部署配置文件

### 1. package.json 脚本

| 脚本命令 | 功能说明 | 使用场景 |
|----------|----------|----------|
| `npm run dev` | 启动开发模式 | 本地开发调试 |
| `npm run build` | 构建项目 | 生产环境准备 |
| `npm start` | 启动生产服务 | PM2部署时使用 |
| `npm run build:worker` | 构建Worker文件 | CF Workers部署准备 |
| `npm run deploy` | 部署到CF Workers | 生产环境部署 |
| `npm run deploy:dev` | 部署到CF Workers开发环境 | 开发环境部署 |
| `npm run wrangler:dev` | 本地Worker开发模式 | Workers本地调试 |

### 2. PM2 配置 (ecosystem.config.json)

```json
{
  "apps": [{
    "name": "waste-classification",
    "script": "npm",
    "args": "start",
    "instances": 1,
    "exec_mode": "fork",
    "env": {
      "NODE_ENV": "production",
      "PORT": "4111"
    },
    "autorestart": true,
    "max_memory_restart": "1G",
    "restart_delay": 4000,
    "max_restarts": 10,
    "log_file": "./logs/combined.log",
    "error_file": "./logs/error.log",
    "out_file": "./logs/out.log"
  }]
}
```

### 3. Cloudflare Workers 配置 (wrangler.toml)

```toml
name = "waste-classification-api"
main = "dist/worker.js"
compatibility_date = "2024-09-23"
compatibility_flags = ["nodejs_compat"]

[vars]
NODE_ENV = "production"
OPENAI_API_KEY = "your-api-key"

[env.production]
vars = { NODE_ENV = "production" }

[env.development]
vars = { NODE_ENV = "development" }
```

## 部署流程

### 方式一：本地/服务器部署 (PM2)

#### 1. 环境准备
```bash
# 安装依赖
npm install

# 全局安装PM2
npm install -g pm2

# 创建日志目录
mkdir -p logs
```

#### 2. 构建项目
```bash
npm run build
```

#### 3. 启动服务
```bash
# 使用PM2启动
pm2 start ecosystem.config.json

# 查看运行状态
pm2 status

# 查看日志
pm2 logs waste-classification
```

#### 4. 进程管理
```bash
# 重启服务
pm2 restart waste-classification

# 停止服务
pm2 stop waste-classification

# 删除进程
pm2 delete waste-classification

# 保存PM2配置
pm2 save

# 设置开机自启
pm2 startup
```

### 方式二：Cloudflare Workers 部署

#### 1. 环境准备
```bash
# 安装依赖
npm install

# 登录Cloudflare账户
npx wrangler login
```

#### 2. 配置环境变量
```bash
# 设置生产环境密钥
npx wrangler secret put OPENAI_API_KEY

# 或在wrangler.toml中配置（不推荐敏感信息）
```

#### 3. 构建Worker
```bash
npm run build:worker
```

#### 4. 部署
```bash
# 部署到生产环境
npm run deploy

# 或部署到开发环境
npm run deploy:dev
```

#### 5. 本地调试
```bash
# 启动本地Worker开发服务器
npm run wrangler:dev
```

## API 接口

### 基础路由

| 路径 | 方法 | 功能 | 示例 |
|------|------|------|------|
| `/health` | GET | 健康检查 | `GET /health` |
| `/api/agents/{agent}/run` | POST | 运行Agent | `POST /api/agents/wasteClassifier/run` |
| `/api/agents/{agent}/chat` | POST | Agent对话 | `POST /api/agents/simpleClassifier/chat` |

### 可用的 Agents

1. **wasteClassifier** - 详细垃圾分类器
2. **simpleClassifier** - 简单垃圾分类器

### 请求示例

```bash
# 垃圾分类请求
curl -X POST https://your-domain.com/api/agents/wasteClassifier/run \
  -H "Content-Type: application/json" \
  -d '{
    "input": "请帮我分类这个物品：塑料瓶"
  }'
```

## 环境变量配置

### 本地部署

在项目根目录创建 `.env` 文件：

```env
NODE_ENV=production
PORT=4111
OPENAI_API_KEY=your-openai-api-key
DATABASE_URL=your-database-url  # 可选
```

### Cloudflare Workers

通过 `wrangler.toml` 或 Cloudflare Dashboard 配置：

- `OPENAI_API_KEY`: OpenAI API密钥
- `DATABASE_URL`: 数据库连接字符串（可选）
- `NODE_ENV`: 环境标识

## 监控与日志

### PM2 部署监控

```bash
# 查看进程监控
pm2 monit

# 查看详细信息
pm2 show waste-classification

# 实时日志
pm2 logs --lines 200
```

### Cloudflare Workers 监控

- 访问 Cloudflare Dashboard
- 查看 Workers 分析数据
- 监控请求量、错误率、延迟等指标

## 故障排除

### 常见问题

1. **端口占用**
   ```bash
   # 查找占用端口的进程
   lsof -i :4111
   
   # 修改端口配置
   # 编辑 ecosystem.config.json 中的 PORT 值
   ```

2. **内存不足**
   ```bash
   # 查看内存使用
   pm2 show waste-classification
   
   # 调整内存限制
   # 修改 ecosystem.config.json 中的 max_memory_restart
   ```

3. **Worker 部署失败**
   ```bash
   # 检查wrangler配置
   npx wrangler whoami
   
   # 验证代码语法
   npm run build:worker
   
   # 查看部署日志
   npx wrangler tail
   ```

## 性能优化建议

### 1. PM2 部署优化

- 根据服务器CPU核心数调整 `instances` 值
- 启用 `cluster` 模式提高并发处理能力
- 合理设置内存重启阈值
- 配置日志轮转防止日志文件过大

### 2. Cloudflare Workers 优化

- 利用边缘缓存减少响应时间
- 优化代码大小，减少冷启动时间
- 合理使用 KV 存储缓存静态数据
- 监控请求配额，避免超限

## 安全注意事项

1. **API 密钥管理**
   - 使用环境变量存储敏感信息
   - 定期轮换 API 密钥
   - 避免在代码中硬编码密钥

2. **访问控制**
   - 配置适当的 CORS 策略
   - 实施速率限制
   - 添加请求验证和日志记录

3. **网络安全**
   - 使用 HTTPS 加密传输
   - 配置防火墙规则
   - 定期更新依赖包

## 扩展建议

1. **监控告警**
   - 集成 APM 工具（如 New Relic、DataDog）
   - 设置关键指标告警
   - 实施健康检查端点

2. **CI/CD 集成**
   - 添加自动化测试
   - 配置部署管道
   - 实施蓝绿部署或滚动更新

3. **数据持久化**
   - 集成数据库存储
   - 实施数据备份策略
   - 配置读写分离