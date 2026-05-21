# 启动项配置模板

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

## .env 文件模板

### .env.dev-local（本地开发，连本地后端电脑，端口 30xx）

```env
# Vite 项目用 VITE_APP_* 前缀，Vue CLI 项目用 VUE_APP_* 前缀

ENV = 'development'
VUE_APP_ENV = 'dev'
VITE_APP_ENV = 'dev'

# base api（axios baseURL，也是代理匹配路径）
VUE_APP_BASE_URL = '/api'
VITE_APP_BASE_API = '/api'

# 代理目标 → 本地后端电脑
VUE_APP_PROXY_TARGET = 'http://192.168.1.2:8013/项目路径'
VITE_APP_PROXY_TARGET = 'http://192.168.1.2:8013'

# 端口（30 开头）
VUE_APP_PORT = 3012
VITE_APP_PORT = 3005

# OSS 路径
VUE_APP_OSS_PATH = 'http://oss.erp12580.com/'
VITE_APP_OSS_PATH = 'http://oss.erp12580.com/'

# 百度统计（开发环境通常关闭）
VUE_APP_ENABLE_BAIDU_TONGJI = ''

# 钉钉 AppID
VUE_APP_DINGDING_APPID = 'xxx'
VITE_APP_DINGDING_APPID = 'xxx'

# 公司信息（开发环境通常关闭）
VUE_APP_SHOW_INFO = 'N'
VITE_APP_SHOW_INFO = 'N'
```

### .env.dev-onlineTest（测试环境，端口 90xx）

```env
ENV = 'development'
VUE_APP_ENV = 'dev'
VITE_APP_ENV = 'dev'

VUE_APP_BASE_URL = '/api'
VITE_APP_BASE_API = '/api'

# 代理目标 → 测试环境
VUE_APP_PROXY_TARGET = 'http://项目名.erp12580.com/项目路径'
VITE_APP_PROXY_TARGET = 'http://项目名.erp12580.com'

# 端口（90 开头）
VUE_APP_PORT = 9012
VITE_APP_PORT = 9005

VUE_APP_OSS_PATH = 'http://oss.erp12580.com/'
VITE_APP_OSS_PATH = 'http://oss.erp12580.com/'
VUE_APP_ENABLE_BAIDU_TONGJI = ''
VUE_APP_DINGDING_APPID = 'xxx'
VITE_APP_DINGDING_APPID = 'xxx'
VUE_APP_SHOW_INFO = 'N'
VITE_APP_SHOW_INFO = 'N'
```

### .env.prod（生产环境打包，--mode prod）

```env
ENV = 'production'
NODE_ENV = 'production'
VUE_APP_ENV = 'prod'
VITE_APP_ENV = 'prod'

# base api（生产环境直接用后端路径，不走代理）
VUE_APP_BASE_URL = '/项目路径'
VITE_APP_BASE_API = '/项目路径'

# 生产 OSS 路径（通常不同于开发）
VUE_APP_OSS_PATH = 'https://项目域名.com:5001/'
VITE_APP_OSS_PATH = 'http://10.10.2.30:9000/'

# 百度统计（生产环境按需开启）
VUE_APP_ENABLE_BAIDU_TONGJI = ''

# 钉钉 AppID（生产环境用正式 appid）
VUE_APP_DINGDING_APPID = '正式appid'
VITE_APP_DINGDING_APPID = '正式appid'

# 公司信息（生产环境开启）
VUE_APP_SHOW_INFO = 'Y'
VITE_APP_SHOW_INFO = 'Y'
```

---

## vite.config.ts 模板

```ts
import { defineConfig, loadEnv } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'
import { fileURLToPath, URL } from 'url'

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
    resolve: {
      alias: {
        '@': fileURLToPath(new URL('./src', import.meta.url)),
      },
    },
    plugins: [vue(), vueJsx()],
  }
})
```

关键点：
- `loadEnv(mode, ...)` 根据 `--mode` 加载对应的 `.env.[mode]` 文件
- `VITE_APP_BASE_API` 既做 axios baseURL 也做代理匹配路径
- Vite 代理不写 pathRewrite，默认透传完整路径

---

## vue.config.js 模板

```js
const port = process.env.VUE_APP_PORT || 3012
const proxyTarget = process.env.VUE_APP_PROXY_TARGET || 'http://192.168.1.2:8013/项目路径'
const proxyPath = process.env.VUE_APP_BASE_URL || '/api'

module.exports = {
  publicPath: '/',
  outputDir: 'dist',
  productionSourceMap: false,
  devServer: {
    port: Number(port),
    proxy: {
      [proxyPath]: {
        target: proxyTarget,
        changeOrigin: true,
        pathRewrite: {
          ['^' + proxyPath]: '',
        },
      },
    },
  },
}
```

关键点：
- Vue CLI 通过 `process.env` 读取 `.env.*` 文件中的 `VUE_APP_*` 变量
- Vue CLI 代理通常需要 `pathRewrite` 去掉前缀（因为后端路径前缀与前端 axios baseURL 不同）
- `--mode` 参数决定加载哪个 `.env.[mode]` 文件

---

## deploy.js（部署到测试服务器）

```js
// 通过 SSH + SCP 部署 dist/ 到测试服务器
// 1. 备份远程现有 dist/ → dist_YYYYMMDDHHmmss.tar.gz
// 2. 保留最近 2 份备份，删除旧的
// 3. SCP 上传本地 dist/ 到远程
```

- 服务器配置放在 `serverConfig.mjs`（gitignore）：
  ```js
  export default {
    host: '192.168.1.2',
    port: 27645,
    username: 'alios',
    password: 'xxx',
  }
  ```
- 远程路径通常为 `/home2/test_envs/<项目名>/frontends/`

## zip.js（构建后打包）

- `build-prod` 构建完成后自动调用
- Windows：Bandizip (`bz.exe`) 打包为 `dist.zip`
- Mac：7z 打包
- 打包后打开文件管理器定位到 `dist.zip`
