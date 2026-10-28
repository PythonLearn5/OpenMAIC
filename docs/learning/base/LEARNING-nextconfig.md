# OpenMAIC 项目 `next.config.ts` 配置说明

这个文件用于配置 OpenMAIC 的 Next.js 构建、依赖处理、部署、安全策略和资源缓存。

## 1. 构建与部署

| 配置                          | 作用                                                 |
| --------------------------- | -------------------------------------------------- |
| `agentRules: false`         | 根据注释，禁止开发环境自动创建或修改 Agent 规则文件。                     |
| `output: 'standalone'`      | 生成适合独立部署的服务端产物，便于 Docker 部署。                       |
| `outputFileTracingIncludes` | 手动指定需要打包的文件、技能目录、动态依赖和 Sharp 原生库。                  |
| `typescript.tsconfigPath`   | 开发使用 `tsconfig.json`，生产环境使用 `tsconfig.build.json`。 |

## 2. 环境变量

```
env: {
  INTERACTIVE_ASSET_BASE_URL:
    process.env.INTERACTIVE_ASSET_BASE_URL?.trim()
    || '/vendor/interactive',
},
```

配置交互式课程资源的基础路径，默认使用 `/vendor/interactive`。

注意：Next.js 的 `env` 配置可能将变量编译进客户端代码，不应存放密钥。

## 3. 依赖处理

### `transpilePackages`

```
transpilePackages: [
  'mathml2omml',
  'pptxgenjs',
  '@openmaic/importer',
],
```

指定需要由 Next.js 编译处理的依赖，主要用于公式转换、PPTX 生成和课程内容导入。

### `serverExternalPackages`

```
serverExternalPackages: [
  '@earendil-works/pi-ai',
  '@earendil-works/pi-agent-core',
  '@openmaic/generation',
  '@aws-sdk/client-s3',
  '@aws-sdk/s3-request-presigner',
],
```

将指定依赖排除在常规服务端打包之外，让 Node.js 在运行时加载。

主要解决 AI Agent 动态导入、课程生成模块及 S3 SDK 的兼容性问题。

## 4. 请求体大小

```
experimental: {
  proxyClientMaxBodySize: '200mb',
},
```

调整相关 Next.js 代理请求的请求体大小限制，用于支持较大的请求数据。

注意：不代表所有 API、上传接口和反向代理都允许 200 MB。

## 5. HTTP 响应头

通过 `async headers()` 配置不同路径的响应头。

| 路径                           | 配置                            | 目的                     |
| ---------------------------- | ----------------------------- | ---------------------- |
| `/vendor/interactive/:path*` | CORS 允许任意来源；缓存一年              | 方便加载交互式课程资源            |
| `/(.*)`                      | CSP `frame-ancestors`         | 控制哪些网站可以通过 iframe 嵌入页面 |
| `/(.*)`                      | `X-Frame-Options: SAMEORIGIN` | 未配置自定义来源时限制跨域嵌入        |
| `/logos/:path*`              | 缓存一天，允许过期后短期使用旧缓存             | 减少模型供应商 Logo 的重复请求     |

`ALLOWED_FRAME_ANCESTORS` 环境变量用于配置额外允许嵌入页面的来源。

## 6. 生产构建校验

```
const PI_CHAT_FLAG_VALUES_STILL_ACCEPTED =
  new Set(['', 'true', '1']);
```

生产构建时检查 `NEXT_PUBLIC_PI_CHAT_ENABLED`：

- 未设置或值为 `''`、`'true'`、`'1'`：通过此项检查。
- 设置为其他值，例如 `'false'`：构建报错。

原因是 OpenMAIC 已移除旧版 `director-graph` 课堂聊天实现，防止旧配置继续影响部署。课堂模型应使用支持工具调用的模型。

## 7. 学习时重点掌握

建议优先理解以下四项：

1. `output: 'standalone'`：Next.js 如何构建和部署服务端应用。
2. `outputFileTracingIncludes`：为什么动态加载的文件需要手动纳入部署包。
3. `serverExternalPackages`：Webpack 打包与 Node.js 原生动态加载的区别。
4. `headers()`：CORS、CSP、iframe 安全策略与 HTTP 缓存。

总结： 这份配置的核心不是实现 OpenMAIC 的业务功能，而是确保课程生成、AI Agent、文件处理及静态资源在生产环境中能够正确运行。