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

## 开发工作流

### Appwrite 开发
- 开发时

### 新功能开发
1. **文档先行**: 在 `.trae/documents/` 中更新需求和计划文档
2. 确保开发环境正常运行: `pnpm run dev`
3. 创建或修改组件在 `components/` 目录
4. 使用 `cn()` 函数处理样式合并
5. 遵循 TypeScript 严格模式要求
6. 测试构建: `pnpm run build`

### 样式开发
1. 优先使用 Tailwind CSS 实用类
2. 自定义样式添加到 `app/globals.css`
3. 使用 CSS 变量支持主题切换
4. 利用 `cva` 管理组件变体

## 性能优化

### Next.js 优化
- 使用 App Router 的自动代码分割
- 利用服务端组件减少客户端 JavaScript
- 使用 `next/image` 优化图片加载
- 启用 Turbopack 加速开发构建

### Tailwind CSS 优化
- 生产构建自动清除未使用的样式
- 使用 `tw-merge` 避免样式冲突
- CSS 变量减少运行时计算

## 故障排除

### 常见问题
1. **TypeScript 编译错误**: 检查 `tsconfig.json` 配置和类型定义
2. **样式不生效**: 确认 Tailwind CSS 类名正确，检查 PostCSS 配置
3. **组件导入错误**: 验证路径别名配置和文件扩展名
4. **构建失败**: 运行 `pnpm run lint` 检查代码质量问题

### 调试技巧
- 使用 Next.js 开发工具进行性能分析
- 利用浏览器开发者工具检查样式应用
- 检查控制台错误和警告信息
- 使用 TypeScript 编译器进行类型检查

## 依赖管理

### 包管理器
- 项目同时支持 npm 和 pnpm
- 推荐使用 pnpm 以获得更好的性能
- 锁定文件: `package-lock.json` 和 `pnpm-lock.yaml`

### 更新依赖
```bash
# 检查过时的依赖
pnpm outdated

# 更新依赖（谨慎操作）
pnpm update

# 安装新依赖
pnpm install <package-name>

# 添加 shadcn/ui 组件（推荐方式）
pnpx shadcn add [component-name]
```

---

*此文档应随项目发展持续更新，确保信息的准确性和完整性。*