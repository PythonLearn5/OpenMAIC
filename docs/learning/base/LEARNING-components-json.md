# OpenMAIC `components.json` 配置说明

`components.json` 是 shadcn/ui 的组件生成配置文件，用于统一 UI 组件的样式、路径别名、图标库，以及第三方组件注册源。

## 1. 基础配置

| 配置项                   | 说明                                   |
| --------------------- | ------------------------------------ |
| `$schema`             | 指定配置文件的 JSON Schema，用于校验配置和提供编辑器提示。  |
| `style: "radix-vega"` | 指定组件样式预设，影响生成组件的视觉风格。                |
| `rsc: true`           | 使用 React Server Components（RSC）兼容模式。 |
| `tsx: true`           | 生成 TypeScript + JSX 格式的 `.tsx` 组件文件。 |

## 2. Tailwind CSS 配置

```
"tailwind": {
  "config": "",
  "css": "app/globals.css",
  "baseColor": "neutral",
  "cssVariables": true,
  "prefix": ""
}
```

| 配置项                    | 说明                                                 |
| ---------------------- | -------------------------------------------------- |
| `config: ""`           | 不指定传统 Tailwind 配置文件路径，通常适用于 Tailwind CSS v4 等配置方式。 |
| `css`                  | 全局 CSS 文件路径，组件样式和主题变量在这里集中管理。                      |
| `baseColor: "neutral"` | 使用中性色作为基础配色。                                       |
| `cssVariables: true`   | 使用 CSS 变量管理颜色等主题值，便于统一调整主题。                        |
| `prefix: ""`           | Tailwind CSS 类名不使用额外前缀。                            |

## 3. 图标与菜单样式

| 配置项                     | 说明               |
| ----------------------- | ---------------- |
| `iconLibrary: "lucide"` | 使用 Lucide 图标库。   |
| `menuColor: "default"`  | 使用默认菜单配色方案。      |
| `menuAccent: "subtle"`  | 菜单强调效果较柔和，不突出显示。 |

## 4. 路径别名 `aliases`

```
"aliases": {
  "components": "@/components",
  "utils": "@/lib/utils",
  "ui": "@/components/ui",
  "lib": "@/lib",
  "hooks": "@/hooks"
}
```

作用：让导入路径更简洁，避免使用大量相对路径。

例如：

```
import { Button } from "@/components/ui/button";
import { cn } from "@/lib/utils";
```

| 别名           | 对应目录或文件                    |
| ------------ | -------------------------- |
| `components` | `@/components`，通用业务组件      |
| `utils`      | `@/lib/utils`，工具函数         |
| `ui`         | `@/components/ui`，基础 UI 组件 |
| `lib`        | `@/lib`，通用库和业务基础代码         |
| `hooks`      | `@/hooks`，React Hooks      |

注意：`@` 的实际映射通常由 `tsconfig.json` 中的 `paths` 配置决定。

## 5. 组件注册源 `registries`

```
"registries": {
  "@ai-elements": "https://registry.ai-sdk.dev/{name}.json"
}
```

配置一个名为 `@ai-elements` 的第三方组件注册源。

- `{name}`：组件名称的占位符。
- `registry.ai-sdk.dev`：AI SDK 相关组件的注册地址。
- 用途：让 shadcn CLI 等工具能够按名称获取和添加注册源中的组件。

## 总结

这个文件主要解决四件事：

1. 统一 UI 风格：规定组件样式、基础颜色和图标库。
2. 统一样式管理：通过 Tailwind CSS 和 CSS 变量控制主题。
3. 统一代码导入路径：通过 `aliases` 简化组件、工具函数和 Hooks 的引用。
4. 扩展组件来源：通过 `registries` 引入 AI 相关组件。

学习时可以重点关注 `aliases`、`tailwind.css` 和 `registries`，它们最直接影响日常开发与添加组件的方式。