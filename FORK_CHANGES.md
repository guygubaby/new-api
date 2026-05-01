# Fork 改动记录

本文档记录 fork 自 [QuantumNous/new-api](https://github.com/QuantumNous/new-api) 后的所有自定义改动。

---

## 改动概览

| 类别 | 文件 | 说明 |
|------|------|------|
| Docker CI | `.github/workflows/docker-build.yml` | 镜像名变更 + 流程简化 |
| Docker CI | `.github/workflows/docker-image-alpha.yml` | Alpha 版本流程简化 |
| Bug Fix | `relay/channel/claude/relay-claude.go` | Claude required 字段修复 |
| Bug Fix | `relay/channel/gemini/relay-gemini.go` | Gemini required 字段修复 |
| Test | `relay/channel/gemini/relay_gemini_usage_test.go` | 新增测试用例 |

---

## 1. Docker 镜像名变更

### 改动内容
将所有 Docker 镜像引用从上游的 `calciumion/new-api` 改为自己的 `guygubaby/new-api`。

### 涉及文件
- `.github/workflows/docker-build.yml`
- `.github/workflows/docker-image-alpha.yml`

### 改动示例
```yaml
# 之前
tags: calciumion/new-api:${{ env.TAG }}

# 之后
tags: guygubaby/new-api:${{ env.TAG }}
```

---

## 2. Docker CI/CD 简化

### 移除的功能
- GHCR (GitHub Container Registry) 发布
- cosign 镜像签名
- SBOM (Software Bill of Materials) 生成
- provenance 来源证明
- 分平台标签 (`-amd64`, `-arm64`)
- workflow_dispatch 手动输入参数
- 额外的权限声明 (`packages: write`, `id-token: write`)
- Job summary 输出

### 简化后的结构
**之前（复杂）：**
- 2 个 job：`build_single_arch` + `create_manifests`
- 分别构建 amd64/arm64 单平台镜像
- 手动创建 manifest 合并

**之后（简洁）：**
- 1 个 job：`build`
- 直接构建 `linux/amd64,linux/arm64` 多平台镜像
- Docker Buildx 自动创建 manifest
- 只推送 2 个标签：
  - `guygubaby/new-api:<version>`
  - `guygubaby/new-api:latest`

### 最终 Workflow 配置

#### docker-build.yml（Release 版本）
```yaml
name: Publish Docker image

on:
  push:
    tags:
      - '*'
      - '!nightly*'
  workflow_dispatch:
    inputs:
      tag:
        description: 'Tag name to build (e.g., v0.10.8-alpha.3)'
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Check out
        uses: actions/checkout@v4
        with:
          fetch-depth: ${{ github.event_name == 'workflow_dispatch' && 0 || 1 }}
          ref: ${{ github.event.inputs.tag || github.ref }}

      - name: Resolve tag
        id: version
        run: |
          if [ -n "${{ github.event.inputs.tag }}" ]; then
            TAG="${{ github.event.inputs.tag }}"
            if ! git rev-parse "refs/tags/$TAG" >/dev/null 2>&1; then
              echo "::error::Tag '$TAG' does not exist"
              exit 1
            fi
          else
            TAG=${GITHUB_REF#refs/tags/}
          fi
          echo "TAG=${TAG}" >> $GITHUB_ENV
          echo "tag=${TAG}" >> $GITHUB_OUTPUT
          echo "${TAG}" > VERSION

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & push multi-arch image
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            guygubaby/new-api:${{ env.TAG }}
            guygubaby/new-api:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

#### docker-image-alpha.yml（Alpha 版本）
```yaml
name: Publish Docker image (alpha)

on:
  push:
    branches:
      - alpha
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Check out
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Determine alpha version
        id: version
        run: |
          VERSION="alpha-$(date +'%Y%m%d')-$(git rev-parse --short HEAD)"
          echo "$VERSION" > VERSION
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & push multi-arch image
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            guygubaby/new-api:alpha
            guygubaby/new-api:${{ env.VERSION }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 3. Gemini/Claude Required 字段修复

### 问题背景
Gemini API 拒绝 JSON Schema 中 `required` 字段为 `null`，导致请求失败。

### 解决方案
将 `required: null` 转换为 `required: []`（空数组）。

### Claude 适配（relay/channel/claude/relay-claude.go）

**位置：** `RequestOpenAI2ClaudeMessage` 函数

```go
// 之前
claudeTool.InputSchema["required"] = params["required"]

// 之后
// Normalize required field: convert null to empty array
if params["required"] == nil {
    claudeTool.InputSchema["required"] = []any{}
} else {
    claudeTool.InputSchema["required"] = params["required"]
}
```

### Gemini 适配（relay/channel/gemini/relay-gemini.go）

**新增函数：**
```go
// normalizeRequiredArray normalizes the "required" field in JSON Schema.
// Gemini rejects null values for "required", so we convert null to empty array.
func normalizeRequiredArray(schema map[string]interface{}) {
    if schema == nil {
        return
    }

    // Handle "required" field: convert null to empty array
    if required, exists := schema["required"]; exists && required == nil {
        schema["required"] = []interface{}{}
    }

    // Recursively normalize nested schemas
    for _, value := range schema {
        switch v := value.(type) {
        case map[string]interface{}:
            normalizeRequiredArray(v)
        case []interface{}:
            for _, item := range v {
                if itemMap, ok := item.(map[string]interface{}); ok {
                    normalizeRequiredArray(itemMap)
                }
            }
        }
    }
}
```

**调用位置：**
1. `cleanFunctionParametersWithDepth()` — 深度清理时调用
2. `cleanFunctionParametersShallow()` — 浅层清理时调用

```go
// 在 normalizeGeminiSchemaTypeAndNullable(cleanedMap) 之后添加
normalizeRequiredArray(cleanedMap)
```

### 测试覆盖（relay/channel/gemini/relay_gemini_usage_test.go）

新增 5 个测试用例：
```go
func TestNormalizeRequiredArray(t *testing.T) {
    t.Parallel()

    tests := []struct {
        name     string
        input    map[string]interface{}
        expected map[string]interface{}
    }{
        {
            name:     "nil schema",
            input:    nil,
            expected: nil,
        },
        {
            name:     "required is null",
            input:    map[string]interface{}{"required": nil, "type": "object"},
            expected: map[string]interface{}{"required": []interface{}{}, "type": "object"},
        },
        {
            name:     "required is array",
            input:    map[string]interface{}{"required": []string{"name"}, "type": "object"},
            expected: map[string]interface{}{"required": []string{"name"}, "type": "object"},
        },
        {
            name:     "required not present",
            input:    map[string]interface{}{"type": "object"},
            expected: map[string]interface{}{"type": "object"},
        },
        {
            name: "nested schema with null required",
            input: map[string]interface{}{
                "type":       "object",
                "required":   nil,
                "properties": map[string]interface{}{"nested": map[string]interface{}{"required": nil}},
            },
            expected: map[string]interface{}{
                "type":       "object",
                "required":   []interface{}{},
                "properties": map[string]interface{}{"nested": map[string]interface{}{"required": []interface{}{}}},
            },
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            normalizeRequiredArray(tt.input)
            if tt.expected == nil {
                require.Nil(t, tt.input)
                return
            }
            require.Equal(t, tt.expected, tt.input)
        })
    }
}
```

---

## Commit 历史

| Commit | 说明 |
|--------|------|
| `b81a435b` | simplify: remove per-arch tags, only push multi-arch version and latest |
| `4f329bb2` | fix: normalize required null to empty array for Gemini/Claude, simplify Docker CI workflows |

---

## 同步上游更新

当上游有新更新时，使用以下命令同步：

```bash
# 添加上游仓库（如果还没添加）
git remote add upstream https://github.com/QuantumNous/new-api.git

# 拉取上游更新
git fetch upstream

# Rebase 到上游 main
git rebase upstream/main

# 推送到自己的仓库
git push origin main --force-with-lease
```

---

## 后续维护注意事项

1. **Docker 镜像名** — 每次同步上游后，检查 CI workflow 是否有新增镜像名引用，确保使用 `guygubaby/new-api`
2. **Required 字段修复** — 上游如果修改了 Gemini/Claude 的参数处理逻辑，需要确认修复是否仍然有效
3. **新 Workflow 文件** — 上游可能新增其他 CI 配置，需要检查并做相应调整