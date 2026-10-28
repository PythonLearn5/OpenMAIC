# OpenMAIC：`middleware.ts` 配置说明

作用： 在请求进入页面或 API 处理逻辑之前，统一执行功能开关检查、匿名用户身份初始化和访问码验证。

## 1. 核心流程

```
收到请求
   ↓
检查 Workbench 功能是否启用
   ↓
初始化匿名用户身份
   ↓
检查是否配置 ACCESS_CODE
   ↓
没有配置 → 放行
   ↓
白名单接口？→ 放行
   ↓
验证 openmaic_access Cookie
   ↓
有效 → 放行
   ↓
无效
   ├─ API 请求 → 返回 401
   └─ 页面请求 → 放行，由前端显示访问码弹窗
```

## 2. 功能开关检查

```
const workbenchEnabled =
  isProWorkbenchEnabled() &&
  (!canInspectServerRuntime || isAgentRuntimeConfigured());
```

`/workbench` 页面只有在 Pro Workbench 功能启用，并且适用的运行环境检查通过时才允许访问。

- `isProWorkbenchEnabled()`：检查 Workbench 功能开关。
- `isAgentRuntimeConfigured()`：检查 Agent 运行时配置。
- Edge Middleware 无法可靠读取所有服务端部署配置，因此将完整运行时检查留给 Node.js 环境。

功能未启用时，返回真正的 HTTP `404`。

## 3. 匿名用户身份初始化

```
const identity = anonymousOwnerForNavigation({
  method: request.method,
  headers: request.headers,
  pathname,
});
```

为浏览器导航请求建立匿名用户身份，避免每个 API 请求分别创建不同的匿名用户。

辅助函数 `next()` 负责：

- 将匿名身份 Cookie 加入转发请求。
- 通过 `Set-Cookie` 将身份写入浏览器。

这样页面与后续请求能够关联到同一个匿名用户。

## 4. 访问码验证

```
const accessCode = process.env.ACCESS_CODE;
```

如果没有配置 `ACCESS_CODE`，则不启用访问码验证，直接放行。

如果配置了访问码，则按以下顺序检查。

| 检查项                          | 处理方式          |
| ---------------------------- | ------------- |
| `/api/access-code/*`         | 白名单放行         |
| `/api/health`                | 健康检查接口放行      |
| 有效的 `openmaic_access` Cookie | 放行            |
| 未通过验证的 API 请求                | 返回 HTTP `401` |
| 未通过验证的页面请求                   | 放行，由前端显示访问码弹窗 |

验证通过并不是只看 Cookie 是否存在，而是调用 `verifyAccessTokenEdge()` 校验令牌的 HMAC 签名，防止仅伪造 Cookie 值就通过验证。

## 5. 请求转发函数 `next()`

```
function next(
  request: NextRequest,
  identity: NavigationIdentity | undefined
): NextResponse
```

该函数有两种情况：

- 没有匿名身份需要处理：直接调用 `NextResponse.next()`。
- 存在匿名身份：转发包含身份 Cookie 的请求，并在响应中设置 `Set-Cookie`。

注意：这里的 `next()` 是项目自定义的辅助函数，不是 Next.js 的 `middleware` 函数本身。

## 6. `matcher` 请求匹配规则

```
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|logos/).*)'],
};
```

表示中间件匹配大部分请求，但排除：

- `/_next/static`：Next.js 静态资源。
- `/_next/image`：Next.js 图片优化请求。
- `/favicon.ico`：网站图标。
- `/logos/`：Logo 资源。

这样可以避免对这些资源执行不必要的身份和访问码检查。

## 7. 学习重点

1. Middleware：在请求进入业务处理之前执行统一逻辑。
2. NextResponse：控制请求放行、响应状态码和 Cookie。
3. HMAC 验证：验证访问令牌的签名，而非只检查 Cookie 是否存在。
4. 匿名身份管理：让页面与 API 请求共享同一用户身份。
5. Edge 与 Node.js 的区别：Edge 环境存在运行时能力限制，需要区分哪些配置可以检查。

总结： 这份中间件主要承担三项职责：Workbench 功能访问控制、匿名用户身份初始化，以及基于访问码 Cookie 的 API 保护。