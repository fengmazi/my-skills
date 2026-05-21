---
name: project-startup-config
description: 项目启动项与打包配置规范。当新建项目的 scripts、.env 文件、代理配置、端口设置，或迁移老项目配置时使用。
metadata:
  version: "2026.5.21"
---

# 项目启动项与打包配置规范

## 核心约定

### 端口规则

| 模式 | 端口范围 | 说明 |
|------|----------|------|
| `local` | 30xx | 连本地后端电脑 |
| `onlineTest` | 90xx | 连测试环境 |

### 常用程度

| 等级 | 脚本 | 说明 |
|------|------|------|
| 常用 | `dev:local`、`dev:onlineTest` | 日常开发 |
| 常用 | `build-prod` | 打生产包 |
| 常用 | `deploy` | 部署到测试环境 |
| 常用 | `preview` | 预览构建结果 |
| 很少用 | `dev`、`build` | 默认命令，配置不全且老旧 |

---

## Scripts 模板

### Vite 项目（Vue 3）

```json
{
  "scripts": {
    "dev": "vite --host",
    "dev:local": "vite --host --mode dev-local",
    "dev:onlineTest": "vite --host --mode dev-onlineTest",
    "build": "vue-tsc && vite build",
    "build-prod": "vue-tsc --noEmit && vite build --mode prod && node ./zip.js",
    "preview": "vite preview",
    "deploy": "pnpm build && node ./deploy.js",
    "prepare": "husky install",
    "commit": "cz"
  }
}
```

### Vue CLI 项目（Vue 2 / Webpack）

```json
{
  "scripts": {
    "dev": "vue-cli-service serve",
    "dev:local": "vue-cli-service serve --mode dev-local",
    "dev:onlineTest": "vue-cli-service serve --mode dev-onlineTest",
    "build": "vue-cli-service build",
    "build-prod": "vue-cli-service build --mode prod && node ./zip.js",
    "deploy": "npm run build && node ./deploy.js"
  }
}
```

---

## .env 文件对应关系

| 文件 | 触发命令 | 用途 | 状态 |
|------|----------|------|------|
| `.env.development` | `dev` | 默认开发 | 很少用，配置老旧 |
| `.env.dev-local` | `dev:local` | 本地开发 | **常用** |
| `.env.dev-onlineTest` | `dev:onlineTest` | 连接测试环境 | **常用** |
| `.env.production` | `build` | 默认打包 | 很少用，配置老旧 |
| `.env.prod` | `build-prod`（`--mode prod`） | 生产环境打包 | **常用** |

---

## .env 文件内容模板

### .env.dev-local（本地开发，端口 30xx）

```env
ENV = 'development'
VUE_APP_ENV = 'dev'              # Vue CLI
VITE_APP_ENV = 'dev'             # Vite

# base api（axios baseURL）
VUE_APP_BASE_URL = '/api'        # Vue CLI
VITE_APP_BASE_API = '/api'       # Vite

# 代理目标 → 本地后端电脑
VUE_APP_PROXY_TARGET = 'http://192.168.1.2:8013/xxx'
VITE_APP_PROXY_TARGET = 'http://192.168.1.2:8013'

# 端口（30 开头）
VUE_APP_PORT = 3012
VITE_APP_PORT = 3005

# 其他通用配置（OSS、钉钉等）
VUE_APP_OSS_PATH = 'http://oss.erp12580.com/'
VUE_APP_DINGDING_APPID = 'xxx'
VUE_APP_ENABLE_BAIDU_TONGJI = ''
```

### .env.dev-onlineTest（测试环境，端口 90xx）

```env
ENV = 'development'
VUE_APP_ENV = 'dev'
VITE_APP_ENV = 'dev'

VUE_APP_BASE_URL = '/api'
VITE_APP_BASE_API = '/api'

# 代理目标 → 测试环境
VUE_APP_PROXY_TARGET = 'http://xxx.erp12580.com/xxx'
VITE_APP_PROXY_TARGET = 'http://xxx.erp12580.com'

# 端口（90 开头）
VUE_APP_PORT = 9012
VITE_APP_PORT = 9005
```

### .env.prod（生产打包）

```env
ENV = 'production'
NODE_ENV = 'production'
VUE_APP_ENV = 'prod'
VITE_APP_ENV = 'prod'

VUE_APP_BASE_URL = '/xxx'
VITE_APP_BASE_API = '/xxx'

# 生产 OSS 路径（可能不同于开发）
VUE_APP_OSS_PATH = 'https://xxx.com:5001/'
VITE_APP_OSS_PATH = 'http://10.10.2.30:9000/'

VUE_APP_SHOW_INFO = 'Y'          # 是否显示公司标题/icon
VITE_APP_SHOW_INFO = 'Y'
```

---

## 配置文件要点

### Vite（vite.config.ts）

```ts
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  return {
    server: {
      port: Number(env.VITE_APP_PORT) || 3005,
      proxy: {
        [env.VITE_APP_BASE_API]: {
          target: env.VITE_APP_PROXY_TARGET,
          changeOrigin: true,
        },
      },
    },
    // ...
  }
})
```

### Vue CLI（vue.config.js）

```js
const port = process.env.VUE_APP_PORT || 3012
const proxyTarget = process.env.VUE_APP_PROXY_TARGET || 'http://192.168.1.2:8013/xxx'
const proxyPath = process.env.VUE_APP_BASE_URL || '/api'

module.exports = {
  devServer: {
    port: Number(port),
    proxy: {
      [proxyPath]: {
        target: proxyTarget,
        changeOrigin: true,
        pathRewrite: { ["^" + proxyPath]: "" },
      },
    },
  },
}
```

---

## deploy.js 部署脚本

- 通过 SSH 连接测试服务器，备份旧 `dist/`（保留最近 2 份），SCP 上传新 `dist/`
- 服务器配置放在 `serverConfig.mjs`（gitignore）
- 存放路径通常为 `/home2/test_envs/<project>/frontends/`

## zip.js 打包脚本

- `build-prod` 构建后自动调用
- Windows 用 Bandizip (`bz.exe`)，Mac 用 7z
- 打包后打开文件夹定位到 `dist.zip`
