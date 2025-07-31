# ddm 目录 Go 函数概览

本文档汇总 `ddm` 目录及其子目录中定义的 Go 函数，帮助快速了解各文件的功能。

## 顶层目录

### declaration.go
- `(*Declaration) Valid() bool`：对声明结构进行基本校验。
- `ParseDeclaration(raw []byte) (*Declaration, error)`：从原始 JSON 解析声明。

### items.go
- `ManifestType(t string) string`：根据声明类型字符串返回所属的 Manifest 类型。

### path.go
- `ParseDeclarationPath(path string) (string, string, error)`：从 URL 路径拆分出声明类型和 Identifier。

### status.go
- `parseStatusDeclarations(v *fastjson.Value) ([]DeclarationStatus, []StatusError, error)`：解析声明状态数组。
- `parseErrors(v *fastjson.Value) ([]StatusError, error)`：解析状态报表中的错误列表。
- `parseStatusReportValue(v *fastjson.Value, values *[]StatusValue, path, container string) error`：递归解析其他状态值。
- `valueHandler(s *StatusReport) jsonpath.HandlerFunc`：生成用于处理状态值的 JSONPath 处理函数。
- `declarationHandler(s *StatusReport) jsonpath.HandlerFunc`：生成用于处理声明状态的 JSONPath 处理函数。
- `errorHandler(s *StatusReport) jsonpath.HandlerFunc`：生成用于处理错误的 JSONPath 处理函数。
- `RegisterStatusHandlers(mux *jsonpath.PathMux, s *StatusReport)`：向 mux 注册默认的状态处理函数。
- `ParseStatusUsingMux(raw []byte, mux *jsonpath.PathMux) ([]string, error)`：使用给定的 mux 解析原始状态报告。
- `ParseStatus(raw []byte) ([]string, *StatusReport, error)`：使用默认 mux 解析状态报告。

### token.go
- （仅包含结构定义，无函数）

## build 子目录

### build.go
- `tokenHashFinalize(h hash.Hash) string`：返回哈希结果的十六进制字符串。
- `tokenHashWrite(h hash.Hash, d *ddm.Declaration)`：将声明的 ServerToken 写入哈希。

### di.go
- `NewDIBuilder(newHash NewHash) *DIBuilder`：创建用于构建 DeclarationItems 的 Builder。
- `(b *DIBuilder) Add(d *ddm.Declaration)`：向构建器添加一个声明。
- `(b *DIBuilder) Finalize()`：完成构建并计算 DeclarationsToken。

### token.go
- `NewTokensBuilder(newHash NewHash) *TokensBuilder`：创建同步 token 构建器。
- `(b *TokensBuilder) Add(d *ddm.Declaration)`：向 token 构建器添加声明。
- `(b *TokensBuilder) Finalize()`：计算最终的 DeclarationsToken 与时间戳。

## 测试文件

### declaration_test.go
- `TestUnmarshal(t *testing.T)`：验证 ParseDeclaration 的基本解析功能。
- `TestUnmarshalPayload(t *testing.T)`：验证带 Payload 的声明解析。

### items_test.go
- `TestManifestType(t *testing.T)`：测试 ManifestType 的各种输入情况。

### path_test.go
- `TestPathSplit(t *testing.T)`：测试 ParseDeclarationPath 的多种路径场景。

### status_test.go
- `TestStatusParse(t *testing.T)`：从示例 JSON 验证状态解析流程。

### build/di_test.go
- `TestBuilder(t *testing.T)`：测试 DIBuilder 的添加与 Finalize 行为。

以上即为 `ddm` 目录下所有 Go 文件中函数的简要说明。
