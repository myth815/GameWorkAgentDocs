# 游戏开发AI Agent基建 CLI 技术需求文档（最终版）

## 1. 项目背景与目标

### 1.1 项目背景
- **团队特点**：游戏开发专家团队，SAAS开发经验有限
- **目标用户**：内部游戏开发团队（程序员、美术、策划、QA）
- **核心诉求**：降低SAAS开发门槛，快速搭建AI Agent基础设施

### 1.2 CLI工具目标
- **零门槛启动**：一键式环境搭建，自动化所有配置
- **端到端体验**：从环境安装到服务运行全流程自动化
- **智能化管理**：自动检测问题，提供智能修复建议
- **可视化监控**：统一的Web管理界面，集成所有服务

## 2. 核心功能需求

### 2.1 一键环境搭建

#### 2.1.1 引导脚本设计
**云端引导脚本**：
```bash
# Windows (使用BAT脚本自动处理权限)
curl -o install.bat https://raw.githubusercontent.com/yourorg/game-ai-cli/main/install.bat && install.bat

# install.bat 内容示例：
@echo off
echo 检查PowerShell执行策略...
powershell -Command "Get-ExecutionPolicy" > nul 2>&1
if %errorlevel% neq 0 (
    echo 需要管理员权限，正在提升权限...
    powershell -Command "Start-Process cmd -ArgumentList '/c %~dpnx0' -Verb RunAs"
    exit /b
)
echo 设置PowerShell执行策略...
powershell -Command "Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force" > nul 2>&1
echo 执行安装脚本...
powershell -ExecutionPolicy Bypass -File "%~dp0install.ps1"

# Linux/MacOS
curl -fsSL https://raw.githubusercontent.com/yourorg/game-ai-cli/main/install.sh | bash
```

**脚本功能清单**：
```yaml
环境检测:
  - 操作系统: Windows(WSL2)/Linux/MacOS
  - 系统架构: x86_64/arm64
  - 最小配置检查:
    - CPU: 4核心以上
    - 内存: 16GB以上（开发环境）
    - 磁盘: 100GB可用空间

依赖自动安装:
  - Node.js: v22 LTS (最新LTS)
  - Docker CE: 27.0+ (最新稳定版)
  - Kubernetes: 1.31+ (最新稳定版)
  - Git: 包含LFS支持
  - Python: 3.12 LTS (最新LTS)
  - WSL2: Windows环境自动配置

容器运行时选择:
  primary:
    - Docker CE + Portainer CE
    - 完全开源，无许可限制
  alternative:
    - Podman + Podman Desktop
    - Rancher Desktop

CLI工具安装:
  - 全局安装: npm install -g @gameai/agent-cli
  - 自动更新检查
  - 插件系统支持
```

#### 2.1.2 智能依赖管理
```typescript
// 依赖版本智能处理
class DependencyManager {
  private readonly deps = {
    'node': {
      required: '>=22.0.0',
      installed: null,
      autoFix: () => this.installNode(),
      validator: semver.satisfies
    },
    'docker': {
      required: '>=27.0.0',
      installed: null,
      autoFix: () => this.installDocker(),
      alternative: 'podman',
      validator: semver.satisfies
    },
    'python': {
      required: '>=3.12.0',
      installed: null,
      autoFix: () => this.installPython(),
      validator: semver.satisfies
    }
  };

  async checkDependencies() {
    for (const [name, dep] of Object.entries(this.deps)) {
      dep.installed = await this.getVersion(name);
      
      if (!dep.validator(dep.installed, dep.required)) {
        console.log(`${name} version mismatch: ${dep.installed} < ${dep.required}`);
        
        // 检查替代方案
        if (dep.alternative && await this.checkAlternative(dep.alternative)) {
          console.log(`Using ${dep.alternative} as alternative`);
          continue;
        }
        
        // 提示自动修复
        const { autoFix } = await inquirer.prompt([{
          type: 'confirm',
          name: 'autoFix',
          message: `Auto-install ${name} ${dep.required}?`,
          default: true
        }]);
        
        if (autoFix) {
          await dep.autoFix();
        }
      }
    }
  }
  
  private async installDocker() {
    // 安装Docker CE，不是Docker Desktop
    const platform = process.platform;
    
    if (platform === 'win32') {
      // Windows: WSL2 + Docker CE
      await this.exec('wsl --install');
      await this.exec('wsl -e bash -c "curl -fsSL https://get.docker.com | sh"');
    } else {
      // Linux/Mac: Docker CE
      await this.exec('curl -fsSL https://get.docker.com | sh');
      await this.exec('sudo usermod -aG docker $USER');
    }
  }
}
```

#### 2.1.3 Docker服务清单
**基础设施服务**：
```yaml
# docker-compose.base.yml
services:
  # 数据存储层
  postgresql:
    image: postgres:17-alpine  # 最新LTS
    environment:
      POSTGRES_DB: gameai_agent
      POSTGRES_USER: gameai
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - internal_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U gameai"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine  # Redis作为缓存层
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - internal_net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:RELEASE.2024-12-18  # 指定稳定版本
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - minio_data:/data
    networks:
      - internal_net
      - debug_net
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 30s
      timeout: 20s
      retries: 3

  # 核心服务 - 使用Logto替代Authentik
  temporal:
    image: temporalio/auto-setup:1.25.0  # 最新稳定版
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=gameai
      - POSTGRES_PWD=${DB_PASSWORD}
      - POSTGRES_SEEDS=postgresql
    depends_on:
      postgresql:
        condition: service_healthy
    networks:
      - internal_net
      - debug_net

  logto:
    image: svhd/logto:latest
    environment:
      DB_URL: postgresql://gameai:${DB_PASSWORD}@postgresql:5432/logto
      ENDPOINT: http://localhost:3301
      ADMIN_ENDPOINT: http://localhost:3302
    depends_on:
      postgresql:
        condition: service_healthy
    networks:
      - internal_net
      - public_net
    ports:
      - "3301:3001"  # 用户端口
      - "3302:3002"  # 管理端口

  # 日志系统 - Loki
  loki:
    image: grafana/loki:latest
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - loki_data:/loki
      - ./loki-config.yaml:/etc/loki/local-config.yaml
    networks:
      - internal_net
      - debug_net
    depends_on:
      - minio

  # 容器管理 - Portainer是主要管理工具
  portainer:
    image: portainer/portainer-ce:2.21.0  # 最新版
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    ports:
      - "9000:9000"  # Portainer UI
      - "8000:8000"  # Portainer Edge Agent (可选)
    networks:
      - debug_net
      - public_net

  # 监控系统
  prometheus:
    image: prom/prometheus:v3.0.0  # 最新LTS
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - internal_net
      - debug_net

  grafana:
    image: grafana/grafana:11.4.0  # 最新LTS
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_INSTALL_PLUGINS=redis-datasource
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - internal_net
      - debug_net
    depends_on:
      - prometheus
    ports:
      - "3300:3000"  # 避免与前端冲突

  # 数据库管理工具
  pgadmin:
    image: dpage/pgadmin4:8.14  # 最新版
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@gameai.local
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD}
    networks:
      - debug_net
    depends_on:
      postgresql:
        condition: service_healthy
    ports:
      - "5050:80"

  # 可选的AI服务（根据需要启用）
  milvus:
    image: milvusdb/milvus:v2.4.10  # 最新稳定版
    profiles: ["ai"]  # 仅在AI profile启用
    volumes:
      - milvus_data:/var/lib/milvus
    networks:
      - internal_net
    depends_on:
      - minio  # Milvus依赖MinIO存储

networks:
  public_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
  
  internal_net:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.21.0.0/16
  
  debug_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/16

volumes:
  postgres_data:
  redis_data:
  minio_data:
  prometheus_data:
  grafana_data:
  pgadmin_data:
  portainer_data:
  milvus_data:
  loki_data:
```

### 2.2 环境模式管理

#### 2.2.1 开发环境（dev）
```yaml
特点:
  - 最小资源配置
  - 所有服务单实例
  - 开发调试优先
  - 快速启动

配置调整:
  postgresql:
    resources:
      limits:
        memory: 1Gi
  
  redis:
    maxmemory: 1gb
    maxmemory-policy: allkeys-lru
  
  temporal:
    replicas: 1
  
  # 禁用不必要的服务
  milvus:
    enabled: false
```

#### 2.2.2 调试/内网环境（staging）
```yaml
特点:
  - 接近生产配置
  - 内网访问
  - 完整监控
  - 性能测试

配置调整:
  postgresql:
    resources:
      limits:
        memory: 4Gi
    backup:
      enabled: true
      schedule: "0 2 * * *"
  
  redis:
    maxmemory: 2gb
    maxmemory-policy: allkeys-lru
  
  temporal:
    replicas: 2
  
  monitoring:
    enabled: true
    retention: 7d
```

#### 2.2.3 生产环境（production）
```yaml
特点:
  - 高可用配置
  - 完整安全策略
  - 自动备份
  - 监控告警

Kubernetes部署:
  - PostgreSQL: 使用StatefulSet，PVC持久化
  - Redis: 哨兵模式或集群模式
  - Temporal: 多节点部署
  - MinIO: 分布式模式
  - 所有服务配置资源限制和自动扩缩容
  - 使用Kubernetes Dashboard管理（不是Portainer）
```

### 2.3 Web管理界面

#### 2.3.1 服务监控仪表板设计
**界面布局**：
```typescript
interface ServiceDashboard {
  // 顶部导航栏
  header: {
    logo: "GameWorkAgent";
    quickLinks: ["跳转到外网", "WebSocket连接中"];
    userInfo: {
      systemLoad: "12/12 正常";
      currentEnv: "Dev";
    };
  };
  
  // 服务卡片网格布局
  serviceGrid: {
    layout: "4列 x N行";
    categories: [
      {
        name: "CORE APPS",
        icon: "🚀",
        services: [
          { name: "用户前台", status: "HEALTHY", url: "http://localhost:3000" },
          { name: "管理后台", status: "HEALTHY", url: "http://localhost:3001" },
          { name: "用户服务API", status: "HEALTHY", url: "http://localhost:4001" },
          { name: "调度中心API", status: "HEALTHY", url: "http://localhost:4002" }
        ]
      },
      {
        name: "INFRA STRUCTURE", 
        icon: "🗄️",
        services: [
          { name: "PostgreSQL", status: "HEALTHY", version: "17", db: "gameai" },
          { name: "MinIO存储", status: "HEALTHY", buckets: ["gws-files", "gws-uploads"] },
          { name: "Redis缓存", status: "HEALTHY", memory: "Redis兼容模式" },
          { name: "Temporal Server", status: "HEALTHY", workflows: "工作流引擎" }
        ]
      },
      {
        name: "MONITORING",
        icon: "📊", 
        services: [
          { name: "Grafana", status: "HEALTHY", dashboards: ["系统监控"] },
          { name: "pgAdmin", status: "HEALTHY", manage: "PostgreSQL数据库管理" },
          { name: "Prometheus", status: "HEALTHY", metrics: "指标采集" },
          { name: "Loki", status: "HEALTHY", manage: "日志管理" }
        ]
      },
      {
        name: "DEV TOOLS",
        icon: "🔧",
        services: [
          { name: "Portainer", status: "HEALTHY", info: "Docker容器管理" },
          { name: "NPM Registry", status: "HEALTHY", info: "NPM包仓库(内网部署)" },
          { name: "Docker Registry", status: "HEALTHY", info: "容器镜像仓库" },
          { name: "Logto", status: "HEALTHY", routes: "统一认证服务" }
        ]
      }
    ];
  };
  
  // 服务卡片组件
  serviceCard: {
    elements: {
      icon: string;           // 服务图标
      name: string;          // 服务名称
      description: string;   // 简短描述
      status: {
        indicator: "🟢" | "🟡" | "🔴";  // 健康状态
        text: "HEALTHY" | "DEGRADED" | "UNHEALTHY";
      };
      quickActions: {
        access: () => void;    // 快速访问
        restart: () => void;   // 重启服务
        logs: () => void;      // 查看日志
      };
      metadata: {
        url: string;          // 服务地址
        credentials: {        // 凭据（点击复制）
          username?: string;
          password?: string;
        };
      };
    };
  };
}
```

**界面主题配置（深色科技风格 - 配合Arco Design Vue）**：
```css
/* GameWorkAgent主题样式 - 适配Arco Design Vue */
:root {
  --bg-primary: #0a0e27;      /* 深蓝背景 */
  --bg-secondary: #151932;     /* 卡片背景 */
  --bg-hover: #1f2344;         /* 悬浮背景 */
  --text-primary: #ffffff;     /* 主文字 */
  --text-secondary: #8892b0;   /* 次要文字 */
  --accent-cyan: #00d9ff;      /* 青色强调 */
  --accent-green: #52c41a;     /* 健康绿 */
  --accent-orange: #fa8c16;    /* 警告橙 */
  --accent-red: #f5222d;       /* 错误红 */
  --border-color: #2a3055;     /* 边框色 */
}

.service-card {
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  padding: 16px;
  transition: all 0.3s ease;
}

.service-card:hover {
  background: var(--bg-hover);
  border-color: var(--accent-cyan);
  box-shadow: 0 0 20px rgba(0, 217, 255, 0.3);
}
```

#### 2.3.2 统一入口设计
**Nginx配置示例（支持WebSocket）**：
```nginx
# nginx.conf - 使用Nginx作为边缘入口
server {
    listen 80;
    server_name localhost;

    # 主应用 (Vue 3 SPA)
    location / {
        proxy_pass http://app-frontend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # WebSocket健康检查
    location /ws {
        proxy_pass http://app-backend:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
    
    # API
    location /api {
        proxy_pass http://app-backend:4000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # gRPC
    location /grpc {
        grpc_pass grpc://grpc-server:50051;
    }
    
    # 管理工具路由
    location /portainer {
        proxy_pass http://portainer:9000;
        proxy_set_header Host $host;
    }
    
    location /pgadmin {
        proxy_pass http://pgadmin:80;
        proxy_set_header Host $host;
    }
    
    location /temporal {
        proxy_pass http://temporal-ui:8080;
        proxy_set_header Host $host;
    }
    
    location /minio {
        proxy_pass http://minio:9001;
        proxy_set_header Host $host;
    }
    
    location /grafana {
        proxy_pass http://grafana:3000;
        proxy_set_header Host $host;
    }
    
    location /logto {
        proxy_pass http://logto:3001;
        proxy_set_header Host $host;
    }
}
```

### 2.4 CLI命令设计

#### 2.4.1 智能化命令结构
```bash
# 统一的智能启动/停止 - 使用gwa前缀
gwa start              # 智能启动（自动检测环境）
gwa stop               # 智能停止
gwa status             # 统一状态显示

# 资源导向的命令结构
gwa deploy
  --target=local             # docker-compose
  --target=k8s               # kubernetes
  --services=core            # 只启动核心服务
  --services=all             # 启动所有服务

# 统一的资源管理
gwa resources
  list                        # 列出所有资源
  logs <resource>            # 查看任何资源日志
  exec <resource> <cmd>      # 执行命令
  restart <resource>         # 重启资源
  health                     # 健康检查

# 初始化向导
gwa init [project-name]
  --preset=quickstart        # 快速开始（推荐配置）
  --preset=custom            # 自定义配置
  --preset=import            # 导入已有配置

# 环境管理
gwa env
  setup                    # 一键配置环境
  check                    # 环境检查
  doctor                   # 问题诊断
  switch <env>            # 切换环境（dev/staging/prod）
  export                   # 导出配置
  import <file>           # 导入配置

# 数据库管理
gwa db
  migrate                 # 运行迁移
  seed                    # 填充测试数据
  backup                  # 备份
  restore [file]          # 恢复
  console                 # 进入psql控制台

# Temporal工作流
gwa workflow
  list                    # 列出工作流
  start [workflow]        # 启动工作流
  status [id]            # 查看状态
  history [id]           # 执行历史
  replay [id]            # 重放工作流

# Agent开发辅助
gwa agent
  create [name]          # 创建新Agent
  test [name]            # 测试Agent
  deploy [name]          # 部署Agent
  monitor [name]         # 监控Agent

# 服务声明（对接调度中心）
gwa declare              # 生成service.yaml
  --interactive          # 交互式生成
  --from-proto          # 从proto文件生成
  --validate            # 验证service.yaml

# AI辅助功能
gwa ai <prompt>         # Claude Code集成
  --context=current     # 包含当前项目上下文
  --generate=code       # 生成代码
  --generate=config     # 生成配置
  --generate=test       # 生成测试

# 包管理（连接内网源）
gwa package
  config-npm              # 配置NPM使用内网源
    --registry=http://npm.internal.company.com
  config-pypi            # 配置PyPI使用内网源  
    --index-url=http://pypi.internal.company.com
  config-docker          # 配置Docker使用内网源
    --registry=docker.internal.company.com
  publish [package]      # 发布私有包到内网源
  search [package]       # 从内网源搜索包

# 监控和日志
gwa monitor
  health                 # 健康检查
  metrics               # 查看指标
  alerts                # 告警状态
  
gwa logs
  search [query]        # 搜索日志（Loki）
  tail [service]        # 实时日志
  export --from --to    # 导出日志

# 批处理和脚本模式
gwa batch
  run <file>            # 运行批处理文件
  --                    # 从标准输入读取

# 别名系统
gwa alias
  list                  # 列出所有别名
  add <name> <command>  # 添加别名
  remove <name>         # 删除别名

# 系统管理
gwa system
  update               # 更新CLI
  config               # 配置管理
  info                 # 系统信息
  clean                # 清理缓存
```

#### 2.4.2 智能初始化向导
```typescript
// 智能初始化向导实现
class InitWizard {
  async run() {
    // 1. 智能检测现有环境
    const detected = await this.detectEnvironment();
    
    // 2. 根据检测结果推荐配置
    const recommended = this.getRecommendation(detected);
    
    // 3. 最少询问原则
    const answers = await inquirer.prompt([
      {
        type: 'list',
        name: 'preset',
        message: 'Choose setup:',
        choices: [
          { name: 'Quick Start (Recommended)', value: 'quickstart' },
          { name: 'Custom Configuration', value: 'custom' },
          { name: 'Import Existing', value: 'import' }
        ],
        default: 'quickstart'
      }
    ]);
    
    // Quick Start直接完成，不再询问
    if (answers.preset === 'quickstart') {
      await this.quickSetup(recommended);
      console.log('✅ Setup completed! Run "gwa start" to begin.');
    } else if (answers.preset === 'custom') {
      await this.customSetup();
    } else {
      await this.importSetup();
    }
  }
  
  private async quickSetup(config: any) {
    const spinner = ora('Setting up environment...').start();
    
    // 自动完成所有配置
    await this.installDependencies();
    await this.generateConfig(config);
    await this.pullDockerImages();
    await this.initDatabase();
    
    spinner.succeed('Environment ready!');
  }
}
```

#### 2.4.3 智能错误恢复
```typescript
// 添加智能重试装饰器
function retry(options: RetryOptions = {}) {
  const { 
    attempts = 3, 
    backoff = 'exponential',
    onRetry = () => {} 
  } = options;
  
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      let lastError: Error;
      
      for (let i = 0; i < attempts; i++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error;
          
          // 智能错误处理
          if (error.code === 'ECONNREFUSED') {
            await this.autoFixConnection();
          } else if (error.code === 'EACCES') {
            await this.autoFixPermission();
          } else if (error.code === 'ENOSPC') {
            await this.autoCleanSpace();
          }
          
          onRetry(error, i + 1);
          
          // 退避策略
          if (i < attempts - 1) {
            const delay = backoff === 'exponential' 
              ? Math.pow(2, i) * 1000 
              : 1000;
            await new Promise(r => setTimeout(r, delay));
          }
        }
      }
      
      throw lastError;
    };
  };
}

class ServiceManager {
  @retry({ attempts: 3, backoff: 'exponential' })
  async startService(name: string) {
    // 服务启动逻辑
  }
}
```

### 2.5 配置管理

#### 2.5.1 项目配置文件（支持继承）
```yaml
# .gwa.yml - 使用统一的配置文件名
extends: .gwa.base.yml    # 继承基础配置

project:
  name: game-ai-agent
  version: 1.0.0
  type: agent-platform

# 配置覆盖优先级：
# 1. 命令行参数 (最高)
# 2. 环境变量
# 3. .env.local
# 4. .env.[environment]
# 5. .env
# 6. .gwa.yml (最低)

environments:
  dev:
    docker:
      compose_files:
        - docker-compose.base.yml
        - docker-compose.dev.yml
    services:
      postgresql:
        memory: 1Gi
      redis:
        memory: 1Gi
      milvus:
        enabled: false
    
  staging:
    docker:
      compose_files:
        - docker-compose.base.yml
        - docker-compose.staging.yml
    services:
      postgresql:
        memory: 4Gi
        replicas: 1
      temporal:
        replicas: 2
    
  production:
    kubernetes:
      namespace: game-ai-prod
      context: production-cluster
    helm:
      charts:
        - ./charts/game-ai-agent

services:
  required:
    - postgresql
    - redis
    - temporal
    - logto
    - nginx
    - minio
    - portainer  # 开发环境容器管理
  
  optional:
    - milvus
    - loki
    - prometheus
    - grafana
  
  # 内网源配置（不由CLI管理）
  internal_registries:
    npm: http://npm.internal.company.com
    pypi: http://pypi.internal.company.com/simple
    docker: docker.internal.company.com

networking:
  ingress:
    provider: nginx  # 使用Nginx作为边缘入口
    domains:
      dev: localhost
      staging: staging.gameai.internal
      production: agent.gameai.com

monitoring:
  prometheus:
    retention: 15d
  grafana:
    dashboards:
      - system-overview
      - temporal-workflows
      - agent-performance
  loki:
    retention: 7d
    storage: minio
  
  alerts:
    - service-down
    - high-memory
    - workflow-failed

# Hook配置
hooks:
  'before:start': ./hooks/before-start.js
  'after:start': ./hooks/after-start.js
  'service:health': ./hooks/health-check.js

# 别名配置
aliases:
  st: status
  l: logs
  r: restart
  dev: "env switch dev && start --all"
```

#### 2.5.2 环境变量管理
```bash
# .env.dev
NODE_ENV=development
DB_PASSWORD=dev_password
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
LOGTO_ENDPOINT=http://localhost:3301
LOGTO_ADMIN_ENDPOINT=http://localhost:3302
GRAFANA_PASSWORD=admin
PGADMIN_PASSWORD=admin

# Temporal配置
TEMPORAL_ADDRESS=localhost:7233
TEMPORAL_NAMESPACE=default

# gRPC配置
GRPC_SERVER_PORT=50051
GRPC_CLIENT_TIMEOUT=30s

# NestJS配置
NEST_PORT=4000
NEST_JWT_SECRET=dev_jwt_secret

# 调度中心配置
SCHEDULER_URL=http://localhost:4002
SCHEDULER_API_KEY=dev_scheduler_key
```

### 2.6 进度显示优化

```typescript
// 多层级进度显示
import * as cliProgress from 'cli-progress';
import { PassThrough } from 'stream';

class ProgressManager {
  private mainBar: cliProgress.SingleBar;
  private subBars: cliProgress.MultiBar;
  
  showProgress() {
    // 主进度条
    this.mainBar = new cliProgress.SingleBar({
      format: 'Setup |{bar}| {percentage}% | {stage}',
      barCompleteChar: '\u2588',
      barIncompleteChar: '\u2591',
      hideCursor: true
    });
    
    // 子任务进度
    this.subBars = new cliProgress.MultiBar({
      format: '  {task} |{bar}| {percentage}%',
      clearOnComplete: false,
      hideCursor: true
    });
    
    // 实时日志流
    const logStream = new PassThrough();
    logStream.on('data', (chunk) => {
      // 在进度条下方显示日志
      process.stdout.write(`\r  > ${chunk.toString().trim()}\n`);
      this.redrawBars();
    });
  }
  
  private redrawBars() {
    // 重绘进度条以保持显示
    this.mainBar.render();
    this.subBars.render();
  }
}
```

### 2.7 Hook系统

```typescript
// Hook系统实现
interface CLIHooks {
  // 前置和后置hooks
  'before:command': (cmd: string) => Promise<void>;
  'after:command': (cmd: string, result: any) => Promise<void>;
  'error:command': (cmd: string, error: Error) => Promise<void>;
  
  // 服务相关hooks
  'service:starting': (service: string) => Promise<void>;
  'service:started': (service: string) => Promise<void>;
  'service:health': (service: string, status: string) => Promise<void>;
}

class HookManager {
  private hooks = new Map<string, Function[]>();
  
  register(event: keyof CLIHooks, handler: Function) {
    if (!this.hooks.has(event)) {
      this.hooks.set(event, []);
    }
    this.hooks.get(event)!.push(handler);
  }
  
  async execute(event: string, ...args: any[]) {
    const handlers = this.hooks.get(event) || [];
    for (const handler of handlers) {
      await handler(...args);
    }
  }
}
```

### 2.8 自更新机制

```typescript
class AutoUpdater {
  private updateAvailable = false;
  
  // 检查更新但不中断工作
  async checkUpdate() {
    const currentVersion = this.getCurrentVersion();
    const latestVersion = await this.getLatestVersion();
    
    if (semver.gt(latestVersion, currentVersion)) {
      this.updateAvailable = true;
      
      // 后台下载新版本
      this.downloadInBackground(latestVersion);
      
      // 完成当前命令后提示
      process.on('beforeExit', () => {
        if (this.updateAvailable) {
          console.log(chalk.yellow(
            '\n📦 New version available! Run "gwa system update" to upgrade.'
          ));
        }
      });
    }
  }
  
  // 平滑升级
  async update() {
    const spinner = ora('Updating CLI...').start();
    
    try {
      // 保存当前状态
      await this.saveState();
      
      // 执行更新
      await this.performUpdate();
      
      // 恢复状态
      await this.restoreState();
      
      // 运行迁移脚本
      await this.runMigrations();
      
      spinner.succeed('CLI updated successfully!');
    } catch (error) {
      spinner.fail('Update failed');
      // 回滚
      await this.rollback();
      throw error;
    }
  }
  
  private async saveState() {
    // 保存当前配置、别名、hooks等
    const state = {
      config: await this.getConfig(),
      aliases: await this.getAliases(),
      version: this.getCurrentVersion()
    };
    
    await fs.writeJson('.gwa.state.json', state);
  }
}
```

## 3. 技术实现（确定选型）

### 3.1 编码规范（第一优先级）

#### 3.1.1 TypeScript编码标准
```typescript
// .eslintrc.json - 强制执行的编码规范
{
  "extends": [
    "@typescript-eslint/recommended",
    "prettier"
  ],
  "rules": {
    // 命名规范
    "naming-convention": [
      "error",
      {
        "selector": "interface",
        "format": ["PascalCase"],
        "prefix": ["I"]
      },
      {
        "selector": "class",
        "format": ["PascalCase"]
      },
      {
        "selector": "variable",
        "format": ["camelCase", "UPPER_CASE"]
      }
    ],
    // 代码结构
    "max-lines-per-function": ["error", 50],
    "max-depth": ["error", 3],
    "complexity": ["error", 10],
    // 类型安全
    "no-any": "error",
    "strict-null-checks": "error",
    "no-non-null-assertion": "error"
  }
}

// prettier.config.js - 格式化标准
module.exports = {
  printWidth: 100,
  tabWidth: 2,
  useTabs: false,
  semi: true,
  singleQuote: true,
  trailingComma: 'all',
  bracketSpacing: true,
  arrowParens: 'always'
};
```

#### 3.1.2 项目结构标准
```
gwa-cli/
├── src/
│   ├── core/              # 核心模块
│   │   ├── cli.ts         # CLI主入口
│   │   ├── config.ts      # 配置管理
│   │   ├── plugin.ts      # 插件系统
│   │   ├── hook.ts        # Hook系统
│   │   ├── progress.ts    # 进度管理
│   │   └── updater.ts     # 自更新系统
│   ├── commands/          # 命令实现
│   │   ├── index.ts       # 命令注册
│   │   ├── start/         # 启动命令
│   │   ├── deploy/        # 部署命令
│   │   ├── resources/     # 资源管理
│   │   ├── declare/       # 服务声明
│   │   ├── ai/           # AI辅助
│   │   └── [command]/     # 其他命令
│   ├── services/          # 业务服务
│   │   ├── docker.ts      # Docker管理
│   │   ├── k8s.ts         # K8s管理
│   │   ├── dependency.ts  # 依赖管理
│   │   ├── scheduler.ts   # 调度中心对接
│   │   └── wizard.ts      # 向导服务
│   ├── utils/             # 工具函数
│   └── types/             # TypeScript类型定义
├── templates/             # 项目模板
├── web/                   # Web管理界面
│   ├── src/
│   │   ├── components/    # Vue组件
│   │   ├── pages/         # 页面
│   │   ├── stores/        # Pinia状态管理
│   │   └── api/           # API调用
│   └── package.json
├── api/                   # NestJS后端
│   ├── src/
│   │   ├── modules/       # 模块
│   │   ├── services/      # 服务
│   │   └── main.ts        # 入口
│   └── package.json
├── tests/                 # 测试文件
└── docs/                  # 文档
```

### 3.2 核心技术栈（最终确定）

#### 3.2.1 CLI框架选型
```json
{
  "framework": "oclif",
  "reason": "企业级CLI框架，插件化支持，自动生成文档",
  
  "dependencies": {
    "@oclif/core": "^4.0.0",
    "@oclif/plugin-help": "^6.2.0",
    "@oclif/plugin-plugins": "^5.4.0",
    "@oclif/plugin-autocomplete": "^3.2.0",
    "@oclif/plugin-warn-if-update-available": "^3.1.0",
    "@oclif/plugin-not-found": "^3.2.0"
  }
}
```

#### 3.2.2 确定的依赖库
```typescript
// 完整依赖列表 - package.json
{
  "dependencies": {
    // CLI核心
    "@oclif/core": "4.0.0",
    "@oclif/plugin-help": "6.2.0",
    "@oclif/plugin-plugins": "5.4.0",
    "@oclif/plugin-autocomplete": "3.2.0",
    "@oclif/plugin-warn-if-update-available": "3.1.0",
    "@oclif/plugin-not-found": "3.2.0",
    
    // Docker管理
    "dockerode": "4.0.2",
    
    // Kubernetes
    "@kubernetes/client-node": "0.21.0",
    
    // 数据库
    "pg": "8.13.0",
    
    // 缓存
    "ioredis": "5.4.1",
    
    // HTTP客户端
    "axios": "1.7.7",
    
    // gRPC
    "@grpc/grpc-js": "1.12.0",
    "@grpc/proto-loader": "0.7.13",
    
    // Temporal
    "@temporalio/client": "1.11.0",
    
    // 日志
    "winston": "3.14.0",
    
    // 配置
    "cosmiconfig": "9.0.0",
    
    // CLI交互
    "inquirer": "10.2.0",
    
    // 进度显示
    "ora": "8.1.0",
    "cli-progress": "3.12.0",
    
    // 文件操作
    "fs-extra": "11.2.0",
    
    // 进程管理
    "execa": "9.5.0",
    
    // 表格显示
    "cli-table3": "0.6.5",
    
    // 颜色输出
    "chalk": "5.3.0",
    
    // YAML解析
    "js-yaml": "4.1.0",
    
    // 模板引擎
    "handlebars": "4.7.8",
    
    // 验证
    "joi": "17.13.3",
    
    // 版本管理
    "semver": "7.6.3"
  }
}
```

#### 3.2.3 Web界面技术栈
```typescript
// 前端确定方案 - 使用Vue 3（与L1文档一致）
{
  "framework": "Vue 3",
  "bundler": "Vite",
  "ui": "Arco Design Vue",  // 与L1文档一致
  "state": "Pinia",
  "charts": "Recharts",
  "flow": "Vue Flow",  // 与L1文档一致
  
  "dependencies": {
    "vue": "3.5.0",
    "@arco-design/web-vue": "2.56.0",
    "vue-router": "4.4.0",
    "pinia": "2.2.0",
    "@vue-flow/core": "1.41.0",
    "vite": "5.4.0",
    "socket.io-client": "4.8.0",
    "axios": "1.7.7",
    "@tanstack/vue-query": "5.59.0"
  }
}

// 后端确定方案
{
  "framework": "NestJS",
  "database": "PostgreSQL",
  "orm": "Prisma",
  "cache": "Redis",
  "auth": "Logto SDK",  // 与L1文档一致
  
  "dependencies": {
    "@nestjs/core": "10.4.0",
    "@nestjs/platform-express": "10.4.0",
    "@nestjs/websockets": "10.4.0",
    "@nestjs/platform-socket.io": "10.4.0",
    "prisma": "5.22.0",
    "@prisma/client": "5.22.0",
    "ioredis": "5.4.1",
    "@logto/node": "2.5.0"  // Logto SDK
  }
}
```

### 3.3 开发环境标准

#### 3.3.1 Node.js版本
```json
{
  "engines": {
    "node": ">=22.0.0",  // 最新LTS
    "npm": ">=10.5.0"
  }
}
```

#### 3.3.2 容器运行时
```yaml
# 开发环境 - 使用Docker CE
docker_version: "27.0.0+"  # 最新稳定版
container_management: "Portainer CE 2.21+"

# 生产环境 - 使用Kubernetes
production:
  orchestrator: "Kubernetes"
  version: "1.31+"  # 最新稳定版
  ingress: "nginx-ingress"
  management: "Kubernetes Dashboard"
```

### 3.4 插件系统实现

#### 3.4.1 插件接口定义（确定方案）
```typescript
// src/types/plugin.interface.ts
export interface IGWAPlugin {
  // 插件元数据
  metadata: {
    name: string;
    version: string;
    author: string;
    description: string;
    keywords: string[];
    dependencies: Record<string, string>;
    peerDependencies?: Record<string, string>;
    engines: {
      node: string;  // >=22.0.0
      'gwa-cli': string;  // >=1.0.0
    };
  };
  
  // 生命周期钩子
  lifecycle: {
    onInstall(context: IPluginContext): Promise<void>;
    onActivate(context: IPluginContext): Promise<void>;
    onDeactivate(context: IPluginContext): Promise<void>;
    onUninstall(context: IPluginContext): Promise<void>;
    onUpdate?(from: string, to: string): Promise<void>;
  };
  
  // 扩展点
  extensions: {
    commands?: IPluginCommand[];
    services?: IPluginService[];
    hooks?: IPluginHook[];
    middleware?: IPluginMiddleware[];
  };
  
  // 配置架构
  configSchema?: {
    type: 'object';
    properties: Record<string, any>;
    required?: string[];
  };
}
```

### 3.5 监控系统集成

#### 3.5.1 Prometheus配置（确定方案）
```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Node Exporter - 系统指标
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    
  # cAdvisor - 容器指标
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
  
  # PostgreSQL Exporter
  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']
  
  # Temporal Metrics
  - job_name: 'temporal'
    static_configs:
      - targets: ['temporal:9090']
  
  # 应用自定义指标
  - job_name: 'app'
    static_configs:
      - targets: ['app-backend:4000/metrics']

# 告警规则
rule_files:
  - '/etc/prometheus/rules/*.yml'
```

## 4. 部署策略（确定方案）

### 4.1 开发环境部署
```bash
# 固定的启动流程
gwa init my-agent-project --preset=quickstart
cd my-agent-project
gwa start
gwa status

# 访问地址（固定端口）
- 主应用: http://localhost:3000
- API服务: http://localhost:4000
- Portainer: http://localhost:9000
- Temporal UI: http://localhost:8080
- MinIO Console: http://localhost:9001
- Grafana: http://localhost:3300
- pgAdmin: http://localhost:5050
- Logto: http://localhost:3301
```

### 4.2 生产环境部署
```bash
# Kubernetes部署（使用Helm）
gwa deploy --target=k8s --env=production

# Helm Chart固定结构
charts/game-ai-agent/
  ├── Chart.yaml           # version: 1.0.0, appVersion: 1.0.0
  ├── values.yaml          # 默认值
  ├── values.prod.yaml     # 生产覆盖
  └── templates/
      ├── deployment.yaml  # 所有服务的Deployment
      ├── service.yaml     # Service定义
      ├── configmap.yaml   # 配置
      ├── secret.yaml      # 密钥
      ├── ingress.yaml     # nginx-ingress配置
      └── hpa.yaml         # 自动扩缩容
```

## 5. 安全规范（确定实现）

### 5.1 认证与授权
```yaml
# 使用Logto作为唯一IAM方案（与L1文档一致）
认证方式:
  primary: OAuth2.0  # 主要方式
  secondary: 企业微信  # 企业集成
  
角色定义（RBAC）:
  - name: admin
    permissions: ["*"]
    description: 完全控制权限
    
  - name: developer
    permissions: 
      - "services:*"
      - "workflows:*"
      - "logs:read"
    description: 开发和部署权限
    
  - name: qa
    permissions:
      - "services:read"
      - "workflows:execute"
      - "logs:read"
      - "metrics:read"
    description: 测试和监控权限
    
  - name: viewer
    permissions:
      - "*:read"
    description: 只读权限
```

### 5.2 数据安全实现
```typescript
// src/security/encryption.ts
export class SecurityManager {
  private readonly algorithm = 'aes-256-gcm';
  private readonly keyDerivation = 'pbkdf2';
  
  // 环境变量加密
  encryptEnvVar(value: string, masterKey: string): string {
    const salt = crypto.randomBytes(16);
    const key = crypto.pbkdf2Sync(masterKey, salt, 100000, 32, 'sha256');
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, key, iv);
    
    let encrypted = cipher.update(value, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    const authTag = cipher.getAuthTag();
    
    return Buffer.concat([salt, iv, authTag, Buffer.from(encrypted, 'hex')])
      .toString('base64');
  }
}
```

## 6. 故障排查规范

### 6.1 诊断工具实现
```typescript
// src/commands/doctor.ts
export class DoctorCommand {
  private checks = [
    { name: 'Docker', checker: this.checkDocker, fixer: this.fixDocker },
    { name: 'Kubernetes', checker: this.checkK8s, fixer: this.fixK8s },
    { name: 'Network', checker: this.checkNetwork, fixer: this.fixNetwork },
    { name: 'Ports', checker: this.checkPorts, fixer: this.fixPorts },
    { name: 'Dependencies', checker: this.checkDeps, fixer: this.fixDeps },
    { name: 'Disk Space', checker: this.checkDisk, fixer: this.fixDisk }
  ];
  
  async run(options: { fix: boolean }): Promise<void> {
    console.log('🔍 Running system diagnostics...\n');
    
    for (const check of this.checks) {
      const spinner = ora(`Checking ${check.name}...`).start();
      
      try {
        const result = await check.checker();
        
        if (result.status === 'ok') {
          spinner.succeed(`${check.name}: ✓ ${result.message}`);
        } else if (result.status === 'warning') {
          spinner.warn(`${check.name}: ⚠ ${result.message}`);
          
          if (options.fix && check.fixer) {
            await check.fixer(result);
          }
        } else {
          spinner.fail(`${check.name}: ✗ ${result.message}`);
          
          if (options.fix && check.fixer) {
            try {
              await check.fixer(result);
              spinner.succeed(`${check.name}: Fixed!`);
            } catch (e) {
              spinner.fail(`Could not fix ${check.name}: ${e.message}`);
            }
          }
        }
      } catch (error) {
        spinner.fail(`${check.name}: Error - ${error.message}`);
      }
    }
  }
}
```

## 7. 最终技术栈确认

### 完整依赖清单（锁定版本）
```json
{
  "name": "@gameai/gwa-cli",
  "version": "1.0.0",
  "engines": {
    "node": ">=22.0.0",
    "npm": ">=10.5.0"
  },
  "dependencies": {
    // CLI框架
    "@oclif/core": "4.0.0",
    "@oclif/plugin-help": "6.2.0",
    "@oclif/plugin-plugins": "5.4.0",
    "@oclif/plugin-autocomplete": "3.2.0",
    "@oclif/plugin-warn-if-update-available": "3.1.0",
    "@oclif/plugin-not-found": "3.2.0",
    
    // 容器和编排
    "dockerode": "4.0.2",
    "@kubernetes/client-node": "0.21.0",
    
    // 数据库和缓存
    "pg": "8.13.0",
    "ioredis": "5.4.1",
    "@prisma/client": "5.22.0",
    
    // 网络和通信
    "axios": "1.7.7",
    "@grpc/grpc-js": "1.12.0",
    "@grpc/proto-loader": "0.7.13",
    "@temporalio/client": "1.11.0",
    
    // 工具库
    "winston": "3.14.0",
    "cosmiconfig": "9.0.0",
    "inquirer": "10.2.0",
    "ora": "8.1.0",
    "cli-progress": "3.12.0",
    "chalk": "5.3.0",
    "cli-table3": "0.6.5",
    "fs-extra": "11.2.0",
    "execa": "9.5.0",
    "js-yaml": "4.1.0",
    "handlebars": "4.7.8",
    "joi": "17.13.3",
    "semver": "7.6.3"
  }
}
```

### 确定的架构决策记录（ADR）

| 决策项 | 选择 | 理由 | 备选方案（已否决） |
|--------|------|------|-------------------|
| CLI框架 | Oclif | 企业级、插件化、文档自动生成 | Commander.js（功能不足） |
| 容器运行时 | Docker CE | 完全开源、无许可限制 | Docker Desktop（许可问题） |
| 容器管理 | Portainer CE | 开源免费、功能完整 | Rancher（过于复杂） |
| 前端框架 | Vue 3 | 与L1文档一致、游戏开发者友好 | React（与主架构不一致） |
| UI组件库 | Arco Design Vue | 与L1文档一致、现代科技风 | Element Plus（风格不符） |
| 状态管理 | Pinia | Vue生态标准、简单 | Vuex（已过时） |
| 后端框架 | NestJS | 企业级、模块化、TypeScript | Express（缺少结构） |
| ORM | Prisma | 类型安全、迁移简单 | TypeORM（配置复杂） |
| 构建工具 | Vite | 快速、简单 | Webpack（配置复杂） |
| 流程图 | Vue Flow | 与L1文档一致、Vue生态 | ReactFlow（框架不匹配） |
| 认证系统 | Logto | 与L1文档一致、TypeScript原生 | Auth0（成本高） |
| 边缘入口 | Nginx | 与L1文档一致、成熟稳定 | Caddy（团队不熟悉） |

## 8. 详细实施计划（12周）

### Phase 1: 基础架构搭建（第1-2周）

#### 1.1 开发环境准备
**任务清单**：
```yaml
任务ID: ENV-001
任务名: 开发环境配置
前置条件: 无
交付物:
  - 开发机器配置文档
  - IDE配置文件（.vscode/settings.json）
  - Git仓库初始化
验收标准:
  - Node.js 22+ 安装完成
  - Docker CE安装并运行
  - Git配置完成（含.gitignore）
测试要求:
  - node --version 输出v22或更高
  - docker --version 输出v27或更高
  - git status 无错误
```

#### 1.2 项目初始化
```yaml
任务ID: INIT-001
任务名: Oclif项目初始化
前置条件: ENV-001
交付物:
  - 基础项目结构
  - package.json配置
  - tsconfig.json配置
验收标准:
  - oclif generate命令成功创建项目
  - npm install无错误
  - npm run build成功
测试要求:
  自动化测试:
    - npm test通过
    - ./bin/dev --version输出版本号
```

#### 1.3 编码规范配置
```yaml
任务ID: LINT-001
任务名: ESLint和Prettier配置
前置条件: INIT-001
交付物:
  - .eslintrc.json
  - .prettierrc
  - husky hooks配置
验收标准:
  - ESLint规则完整配置
  - Prettier格式化规则配置
  - Git hooks自动执行
测试要求:
  自动化测试:
    - npm run lint无错误
    - git commit触发pre-commit hook
    - CI/CD集成lint检查
```

### Phase 2: 核心功能开发（第3-4周）

#### 2.1 依赖管理模块
```yaml
任务ID: DEP-001
任务名: 智能依赖检测
前置条件: LINT-001
交付物:
  - src/services/dependency.ts
  - 依赖版本检测逻辑
  - 自动修复功能
验收标准:
  - 检测Node/Docker/Python版本
  - 版本不匹配时提示修复
  - 支持替代方案检测
测试要求:
  单元测试:
    - 版本比较测试用例
    - Mock依赖检测测试
  集成测试:
    - 真实环境依赖检测
```

#### 2.2 Docker服务管理
```yaml
任务ID: DOCKER-001
任务名: Docker服务编排
前置条件: DEP-001
交付物:
  - docker-compose配置文件
  - src/services/docker.ts
  - 服务健康检查逻辑
验收标准:
  - 所有服务定义完整
  - 健康检查配置正确
  - 网络隔离实现
测试要求:
  集成测试:
    - docker-compose config验证
    - 服务启动顺序测试
    - 健康检查测试
```

#### 2.3 Portainer集成
```yaml
任务ID: DOCKER-002
任务名: Portainer集成配置
前置条件: DOCKER-001
交付物:
  - Portainer服务配置
  - 自动初始化脚本
  - 堆栈模板配置
验收标准:
  - Portainer可访问localhost:9000
  - Docker服务可通过Portainer管理
  - 堆栈模板预配置完成
测试要求:
  - Portainer API测试
  - 容器管理功能测试
  - 权限控制测试
```

#### 2.4 命令系统基础
```yaml
任务ID: CMD-001
任务名: 基础命令实现
前置条件: DOCKER-002
交付物:
  - src/commands/start.ts
  - src/commands/stop.ts
  - src/commands/status.ts
验收标准:
  - gwa start命令启动所有服务
  - gwa stop命令停止所有服务
  - gwa status显示服务状态
测试要求:
  功能测试:
    - gwa start --all
    - gwa stop
    - gwa status
  自动化测试:
    - 命令执行返回码测试
    - 输出格式验证
```

### Phase 3: 调度中心集成（第5周）

#### 3.1 服务声明支持
```yaml
任务ID: SCHED-001
任务名: 服务声明接口
前置条件: CMD-001
交付物:
  - src/commands/declare.ts
  - service.yaml模板生成器
  - 声明文件解析器
验收标准:
  - gwa declare命令可用
  - 生成的service.yaml符合L1规范
  - 支持交互式和命令行两种模式
测试要求:
  - 模板生成测试
  - YAML解析测试
  - 验证逻辑测试
```

#### 3.2 运行态注册
```yaml
任务ID: SCHED-002
任务名: 运行态注册机制
前置条件: SCHED-001
交付物:
  - src/services/scheduler.ts
  - 健康检查上报逻辑
  - 容量信息更新机制
验收标准:
  - 服务能自动注册到调度中心
  - 健康状态定期上报
  - 支持优雅下线
测试要求:
  - 注册流程测试
  - 心跳机制测试
  - 异常恢复测试
```

#### 3.3 资源生命周期管理
```yaml
任务ID: SCHED-003
任务名: 资源管理集成
前置条件: SCHED-002
交付物:
  - 资源申请客户端
  - 生命周期管理逻辑
  - SpiceDB集成
验收标准:
  - 支持ephemeral和persistent资源
  - 资源自动回收机制
  - 权限校验功能
测试要求:
  - 资源申请测试
  - 生命周期测试
  - 权限验证测试
```

### Phase 4: Web管理界面前端（第6周）

#### 4.1 Vue 3项目搭建
```yaml
任务ID: WEB-001
任务名: 前端项目初始化
前置条件: SCHED-003
交付物:
  - web/目录结构
  - Vue 3 + Vite配置
  - Arco Design Vue集成
验收标准:
  - 项目可正常启动
  - 路由配置完成
  - UI框架集成成功
测试要求:
  - npm run dev启动测试
  - 基础组件渲染测试
  - 路由跳转测试
```

#### 4.2 服务监控界面
```yaml
任务ID: WEB-002
任务名: 监控Dashboard开发
前置条件: WEB-001
交付物:
  - 服务卡片组件
  - 实时状态更新逻辑
  - WebSocket连接管理
验收标准:
  - 4列卡片布局实现
  - 服务状态实时更新
  - 深色主题应用
测试要求:
  - 组件渲染测试
  - WebSocket重连测试
  - 响应式布局测试
```

#### 4.3 工作流编辑器
```yaml
任务ID: WEB-003
任务名: Vue Flow集成
前置条件: WEB-002
交付物:
  - 工作流编辑器页面
  - 节点模板库
  - 流程导出功能
验收标准:
  - 拖拽编辑功能正常
  - 节点连接逻辑正确
  - 可导出Temporal定义
测试要求:
  - 编辑器功能测试
  - 节点操作测试
  - 导出格式验证
```

### Phase 5: Web管理界面后端（第7周）

#### 5.1 NestJS项目搭建
```yaml
任务ID: API-001
任务名: 后端API服务初始化
前置条件: WEB-003
交付物:
  - api/目录结构
  - NestJS基础配置
  - Prisma集成
验收标准:
  - 服务正常启动
  - 数据库连接成功
  - 基础中间件配置完成
测试要求:
  - 服务启动测试
  - 数据库连接测试
  - 健康检查接口测试
```

#### 5.2 服务管理API
```yaml
任务ID: API-002
任务名: RESTful API开发
前置条件: API-001
交付物:
  - 服务管理模块
  - Docker操作接口
  - 状态查询接口
验收标准:
  - CRUD操作完整
  - 错误处理规范
  - API文档生成
测试要求:
  - 单元测试覆盖80%
  - 集成测试通过
  - API响应时间<100ms
```

#### 5.3 WebSocket服务
```yaml
任务ID: API-003
任务名: 实时通信实现
前置条件: API-002
交付物:
  - WebSocket网关
  - 事件推送机制
  - 客户端管理
验收标准:
  - 双向通信稳定
  - 支持断线重连
  - 消息广播功能
测试要求:
  - 连接稳定性测试
  - 并发连接测试
  - 消息传输测试
```

#### 5.4 认证集成
```yaml
任务ID: API-004
任务名: Logto认证集成
前置条件: API-003
交付物:
  - 认证中间件
  - 权限守卫
  - 用户会话管理
验收标准:
  - OAuth流程完整
  - 权限控制生效
  - Token刷新机制
测试要求:
  - 认证流程测试
  - 权限验证测试
  - 会话管理测试
```

### Phase 6: 智能化功能（第8周）

#### 6.1 初始化向导
```yaml
任务ID: WIZ-001
任务名: 智能初始化向导
前置条件: API-004
交付物:
  - src/services/wizard.ts
  - 项目模板文件
  - 配置生成逻辑
验收标准:
  - 三种预设模式可选
  - 环境自动检测
  - 配置文件自动生成
测试要求:
  交互测试:
    - Quick Start一键完成
    - Custom模式交互测试
    - Import模式导入测试
```

#### 6.2 智能错误恢复
```yaml
任务ID: RETRY-001
任务名: 错误重试机制
前置条件: WIZ-001
交付物:
  - src/decorators/retry.ts
  - 错误分类处理逻辑
  - 自动修复策略
验收标准:
  - 重试装饰器实现
  - 指数退避策略
  - 错误自动修复
测试要求:
  单元测试:
    - 重试逻辑测试
    - 退避策略测试
    - 错误分类测试
```

#### 6.3 AI命令集成
```yaml
任务ID: AI-001
任务名: Claude Code集成
前置条件: RETRY-001
交付物:
  - src/commands/ai.ts
  - 上下文管理器
  - 代码生成器
验收标准:
  - gwa ai命令可用
  - 支持多种生成目标
  - 上下文感知
测试要求:
  - 命令执行测试
  - 生成质量验证
  - 上下文管理测试
```

### Phase 7: 高级功能（第9周）

#### 7.1 配置管理系统
```yaml
任务ID: CFG-001
任务名: 配置继承和覆盖
前置条件: AI-001
交付物:
  - src/core/config.ts
  - 配置优先级逻辑
  - 继承链解析
验收标准:
  - 支持extends配置
  - 六层优先级正确
  - 环境变量覆盖
测试要求:
  单元测试:
    - 继承链测试
    - 优先级测试
    - 合并逻辑测试
```

#### 7.2 Hook系统
```yaml
任务ID: HOOK-001
任务名: 生命周期Hook
前置条件: CFG-001
交付物:
  - src/core/hook.ts
  - Hook注册机制
  - Hook执行引擎
验收标准:
  - 支持所有生命周期
  - 配置文件注册
  - 异步Hook支持
测试要求:
  集成测试:
    - Hook触发测试
    - 执行顺序测试
    - 错误处理测试
```

#### 7.3 批处理模式
```yaml
任务ID: BATCH-001
任务名: 批处理和脚本支持
前置条件: HOOK-001
交付物:
  - src/commands/batch.ts
  - 批处理解析器
  - 管道模式支持
验收标准:
  - 批处理文件执行
  - 标准输入支持
  - 错误处理机制
测试要求:
  功能测试:
    - 批处理文件测试
    - 管道输入测试
    - 错误恢复测试
```

### Phase 8: Kubernetes支持（第10周）

#### 8.1 K8s客户端集成
```yaml
任务ID: K8S-001
任务名: Kubernetes客户端
前置条件: BATCH-001
交付物:
  - src/services/k8s.ts
  - 集群连接管理
  - 资源操作封装
验收标准:
  - 多集群支持
  - 认证机制完整
  - 资源CRUD操作
测试要求:
  - 连接测试
  - 资源操作测试
  - 错误处理测试
```

#### 8.2 Helm集成
```yaml
任务ID: K8S-002
任务名: Helm Charts开发
前置条件: K8S-001
交付物:
  - charts/目录结构
  - Kubernetes资源模板
  - Values配置文件
验收标准:
  - Deployment模板完成
  - Service/Ingress配置
  - ConfigMap/Secret管理
测试要求:
  Helm测试:
    - helm lint通过
    - helm template验证
    - dry-run测试
```

#### 8.3 部署命令
```yaml
任务ID: K8S-003
任务名: K8s部署命令
前置条件: K8S-002
交付物:
  - src/commands/deploy.ts
  - 部署流程实现
  - 回滚机制
验收标准:
  - gwa deploy --target=k8s可用
  - 支持蓝绿部署
  - 回滚功能正常
测试要求:
  - 部署流程测试
  - 回滚功能测试
  - 多环境测试
```

### Phase 9: 插件系统（第11周）

#### 9.1 插件框架
```yaml
任务ID: PLUGIN-001
任务名: 插件系统架构
前置条件: K8S-003
交付物:
  - src/core/plugin.ts
  - 插件接口定义
  - 插件加载器
验收标准:
  - 插件生命周期管理
  - 依赖解析机制
  - 沙箱隔离实现
测试要求:
  单元测试:
    - 插件加载测试
    - 生命周期测试
    - 隔离性测试
```

#### 9.2 示例插件
```yaml
任务ID: PLUGIN-002
任务名: 示例插件开发
前置条件: PLUGIN-001
交付物:
  - plugins/目录
  - 三个示例插件
  - 插件文档
验收标准:
  - 命令扩展插件
  - 服务扩展插件
  - Hook扩展插件
测试要求:
  功能测试:
    - 插件安装测试
    - 功能验证测试
    - 卸载测试
```

### Phase 10: 测试与优化（第12周）

#### 10.1 单元测试完善
```yaml
任务ID: TEST-001
任务名: 单元测试覆盖
前置条件: PLUGIN-002
交付物:
  - 测试用例集
  - 覆盖率报告
  - 测试文档
验收标准:
  - 覆盖率>80%
  - 关键路径100%覆盖
  - CI集成完成
测试要求:
  - Jest测试通过
  - 覆盖率达标
  - CI/CD集成
```

#### 10.2 集成测试
```yaml
任务ID: TEST-002
任务名: 端到端测试
前置条件: TEST-001
交付物:
  - E2E测试套件
  - 测试环境配置
  - 测试报告
验收标准:
  - 核心流程覆盖
  - 多环境测试通过
  - 性能基准建立
测试要求:
  - 完整流程测试
  - 环境切换测试
  - 压力测试
```

#### 10.3 性能优化
```yaml
任务ID: PERF-001
任务名: 启动性能优化
前置条件: TEST-002
交付物:
  - 性能分析报告
  - 优化方案实施
  - 性能监控
验收标准:
  - CLI启动<500ms
  - 命令响应<100ms
  - 内存占用<100MB
测试要求:
  - 性能基准测试
  - 负载测试
  - 内存泄漏检测
```

#### 10.4 文档与发布
```yaml
任务ID: RELEASE-001
任务名: 发布准备
前置条件: PERF-001
交付物:
  - README.md
  - 用户指南
  - API文档
  - 发布脚本
验收标准:
  - 文档完整
  - 示例可运行
  - 发布流程自动化
测试要求:
  - 文档审核
  - 示例测试
  - 发布演练
```

### 交付里程碑

```yaml
里程碑计划:
  M1 - CLI基础版（第4周）:
    - 环境检测和安装
    - Docker服务管理  
    - 基础命令可用
    - Portainer集成
    
  M2 - 平台集成版（第5周）:
    - 调度中心对接
    - 服务声明支持
    - 资源管理集成
    
  M3 - 管理界面版（第7周）:
    - Web监控界面
    - API服务完整
    - 实时通信功能
    
  M4 - 智能化版（第9周）:
    - AI命令支持
    - 智能错误恢复
    - 配置管理完善
    
  M5 - 生产就绪版（第12周）:
    - Kubernetes支持
    - 插件系统完成
    - 测试覆盖完整
    - 文档齐全
```

## 总结

本技术需求文档提供了一个**与L1文档完全一致**的CLI工具实施方案，包含：

### 核心特性
1. **技术栈统一** - 严格遵循L1文档的技术选型（Vue 3、Arco Design Vue、Logto、Nginx等）
2. **命令前缀一致** - 使用gwa作为统一命令前缀
3. **完整功能覆盖** - CLI工具 + Web管理界面 + 后端API服务
4. **调度中心集成** - 支持服务声明、运行态注册、资源管理
5. **智能化支持** - Claude Code集成、智能错误恢复、初始化向导

### 技术保障
1. **架构一致性** - 所有技术决策与L1文档保持一致
2. **版本兼容性** - 所有组件版本已验证兼容
3. **服务依赖清晰** - 启动顺序和健康检查完善
4. **环境隔离** - 开发、预发、生产环境配置分离

### 实施保障
1. **12周详细计划** - 每个任务都有明确交付物和验收标准
2. **5个关键里程碑** - 渐进式交付，降低风险
3. **完整测试覆盖** - 单元测试、集成测试、性能测试
4. **文档完备** - 用户指南、API文档、开发文档
