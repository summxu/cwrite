# cwrite 项目开发规则

## 技术栈

- **框架**: Next.js (App Router)
- **语言**: TypeScript 5.x
- **后端服务**: Appwrite
- **样式**: Tailwind CSS + PostCSS
- **UI 组件**: 不使用任何 第三方UI组件

## 构建和开发配置

### 开发环境启动
```bash
# 使用 Turbopack 启动开发服务器（推荐）
pnpm run dev

# 构建生产版本
pnpm run build

# 启动生产服务器
pnpm start

# 代码检查
pnpm run lint
```

### 关键配置文件

#### TypeScript 配置 (tsconfig.json)
- **目标**: ES2017
- **模块系统**: ESNext with bundler resolution
- **JSX**: preserve (Next.js 处理)
- **路径别名**: `@/*` 映射到项目根目录
- **严格模式**: 已启用

#### Next.js 配置 (next.config.ts)
- 使用默认配置
- 支持 TypeScript
- 自动优化和代码分割

#### Tailwind CSS 配置
- 使用 Tailwind CSS 4.x
- PostCSS 插件: `@tailwindcss/postcss`
- 自定义 CSS 变量用于主题切换
- 支持深色模式 (`.dark` 类)

## 核心技术要点

### Appwrite 集成
- **强制使用Appwrite**: 所有CRUD操作、用户认证、权限管理必须使用Appwrite的SDK
- 开发时使用appwrite-cli进行同步操作

### 开发注意事项
- 不要过分扩展功能，保持项目简单和可维护
- 该项目是一个Demo性质的项目，不考虑未来的扩展和维护

*此文档应随项目发展持续更新，确保信息的准确性和完整性。*