# WASM 组件 GOARCH 文件名冲突 Bug 分析报告

## 问题现象

升级 Dapr 到 v1.16.0 后，使用 WASM binding 组件时启动报错：

```
FATA[0000] Fatal error from runtime: process component wasm error:
[INIT_COMPONENT_FAILURE]: initialization error occurred for wasm
(bindings.wasm/v1): couldn't find binding wasm (bindings.wasm/v1)
```

该错误在 v1.15.0 中不存在，WASM binding 和 middleware 在 v1.15.0 中工作正常。

## 影响范围

- **影响版本**: v1.16.0, v1.16.1 ~ v1.16.8, v1.17.0
- **影响组件**: WASM output binding (`bindings.wasm`), WASM HTTP middleware (`middleware.http.wasm`)
- **影响平台**: 所有非 `GOARCH=wasm` 平台（即所有实际生产环境: amd64, arm64, arm）
- **与构建标签无关**: 无论使用 `allcomponents` 还是 `stablecomponents` 都受影响

## 排查过程

### 第一步：对比发布说明

搜索 v1.11.0 至 v1.17.0 所有版本发布说明中的 wasm 相关内容：

| 版本 | WASM 相关变化 |
|------|-------------|
| v1.11.0 | WASM binding 首次引入 (alpha), middleware 配置 `path` → `url` |
| v1.12.0 | wasi-http 支持、strict sandboxing、HTTP(S) 加载 WASM 文件 |
| v1.13.0 | 无变化 |
| v1.14.0 | http-wasm host v0.6.0, wazero v1.7.0, TinyGo 0.28.1 |
| v1.15.0 ~ v1.17.0 | 发布说明中无任何 WASM 相关记录 |

v1.16.0 的发布说明中完全未提及 WASM，Breaking Changes 中也没有相关条目。

### 第二步：检查组件注册代码

定位到错误来源 `pkg/runtime/processor/binding/binding.go`:

```go
func (b *binding) Init(ctx context.Context, comp compapi.Component) error {
    // ...
    if b.registry.HasOutputBinding(comp.Spec.Type, comp.Spec.Version) {
        // ...
        found = true
    }
    if !found {
        return fmt.Errorf("couldn't find binding %s", comp.LogName())
    }
}
```

`HasOutputBinding` 返回 false，意味着 WASM binding 没有被注册到 registry 中。

### 第三步：检查注册文件

WASM binding 的注册在 `cmd/daprd/components/bindings_wasm.go`:

```go
//go:build allcomponents

func init() {
    bindingsLoader.DefaultRegistry.RegisterOutputBinding(wasm.NewWasmOutput, "wasm")
    components.RegisterWasmComponentType(components.CategoryBindings, "wasm")
}
```

构建标签 `allcomponents` 看起来没问题。

### 第四步：对比 v1.15.0 和 v1.16.0 的代码差异

对比 registry、processor、category 等核心代码，**全部无变化**：

```bash
git diff v1.15.0 v1.16.0 -- pkg/components/bindings/registry.go    # 无差异
git diff v1.15.0 v1.16.0 -- pkg/runtime/processor/processor.go     # 无差异
git diff v1.15.0 v1.16.0 -- pkg/components/category.go             # 无差异
git diff v1.15.0 v1.16.0 -- pkg/components/wasm.go                 # 无差异
```

wazero 依赖版本也相同 (v1.7.0)。

### 第五步：追踪文件重命名历史

检查 PR #9009 (`feat: bump contrib dep + fix component registrations`) 的变更：

```bash
git show f402ff455 --stat | grep binding
```

发现该 PR 对所有 binding 文件做了批量重命名（`binding_*` → `bindings_*`），其中：

```
binding_webassembly.go  →  bindings_wasm.go      (0 行代码改动)
middleware_http_webassembly.go  →  middleware_http_wasm.go  (0 行代码改动)
```

### 第六步：发现历史修复记录

搜索 git 历史，发现这个问题**曾经被修复过两次**：

**2022年 PR #5486** — 修复 middleware:

```bash
git log 14d10fb6b -1
# "Renames wasm middleware file as it clashes with GOARCH=wasm"
# middleware_http_wasm.go → middleware_http_webassembly.go
```

原作者 Adrian Cole 在 commit 信息中明确写道：

> "While testing, I noticed the wasm middleware wasn't loading. This was
> due to the file naming convention, as it is the same as a valid GOARCH."

**2023年 PR #6439** — 修复 binding:

```bash
git log d2184c414 -1
# "fix wasm binding register"
# binding_wasm.go → binding_webassembly.go
```

### 第七步：确认根因

Go 编译器对文件名有 [隐式构建约束](https://pkg.go.dev/cmd/go#hdr-Build_constraints)：

> 如果文件名（去掉扩展名和可能的 `_test` 后缀后）匹配 `*_GOARCH` 模式，
> Go 会隐式添加对应架构约束。

`wasm` 是合法的 `GOARCH` 值（用于 `js/wasm`, `wasip1/wasm` 等目标），因此：

- `bindings_wasm.go` → 隐式约束 `GOARCH=wasm` → 在 amd64/arm64 上**不编译**
- `middleware_http_wasm.go` → 同理 → 在 amd64/arm64 上**不编译**

即使显式指定了 `//go:build allcomponents`，隐式约束也会叠加。
两个条件都要满足才会编译该文件。

**PR #9009 在 v1.16.0 中的批量重命名无意中回退了 #5486 和 #6439 的修复。**

## 修复方法

将文件重命名回不含 `_wasm` 后缀的名称，文件内容无需改动：

```bash
mv cmd/daprd/components/bindings_wasm.go cmd/daprd/components/bindings_webassembly.go
mv cmd/daprd/components/middleware_http_wasm.go cmd/daprd/components/middleware_http_webassembly.go
```

## 验证结果

修复前（v1.16.0 原始文件名）:

```
FATA[0000] Fatal error from runtime: process component wasm error:
[INIT_COMPONENT_FAILURE]: initialization error occurred for wasm
(bindings.wasm/v1): couldn't find binding wasm (bindings.wasm/v1)
```

修复后（重命名为 `_webassembly` 后缀）:

```
INFO[0000] successful init for output binding (wasm-binding (bindings.wasm/v1))
INFO[0000] Component loaded: wasm-binding (bindings.wasm/v1)
INFO[0000] dapr initialized. Status: Running. Init Elapsed 456ms
```

## 关键时间线

| 时间 | 事件 | PR |
|------|------|-----|
| 2022-11 | 发现 middleware 文件名冲突并修复: `_wasm` → `_webassembly` | [#5486](https://github.com/dapr/dapr/pull/5486) |
| 2023-06 | 发现 binding 文件名冲突并修复: `_wasm` → `_webassembly` | [#6439](https://github.com/dapr/dapr/pull/6439) |
| 2025-08 | PR #9009 批量重命名回退了上述两个修复: `_webassembly` → `_wasm` | [#9009](https://github.com/dapr/dapr/pull/9009) |
| 2025-08 | v1.16.0 发布，包含此回归 Bug | — |
| 2026-02 | v1.17.0 发布，Bug 仍然存在 | — |

## 相关文件

- `cmd/daprd/components/bindings_wasm.go` (需重命名)
- `cmd/daprd/components/middleware_http_wasm.go` (需重命名)
- `pkg/components/bindings/registry.go` (注册表逻辑，无需修改)
- `pkg/runtime/processor/binding/binding.go` (错误来源，无需修改)

## 参考链接

- Go 构建约束文档: https://pkg.go.dev/cmd/go#hdr-Build_constraints
- 原始修复 PR (middleware): https://github.com/dapr/dapr/pull/5486
- 原始修复 PR (binding): https://github.com/dapr/dapr/pull/6439
- 引入回归的 PR: https://github.com/dapr/dapr/pull/9009
