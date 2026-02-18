# OpenAPI 契约优先开发 - 快速上手

## ✅ 配置完成

你的项目已配置为使用 OpenAPI 契约优先开发模式。

## 🚀 快速开始

### 当前状态

✅ 已配置 OpenAPI Generator Maven 插件
✅ 已创建 `swagger-input.yml` 示例文件
✅ 已生成 API 接口和数据模型
✅ 已创建示例 Controller 实现
✅ 应用正在运行: http://localhost:8080

### 访问 Swagger UI

打开浏览器访问: **http://localhost:8080/swagger-ui.html**

你将看到从 `swagger-input.yml` 生成的完整 API 文档。

## 📝 开发流程

### 1️⃣ 编辑 API 规范

```powershell
# 编辑文件
code backend/src/main/resources/openapi/swagger-input.yml
```

### 2️⃣ 生成接口代码

```powershell
cd backend
mvn clean generate-sources
```

### 3️⃣ 实现接口

在 `src/main/java/com/henrywang/ems/controller/` 下创建 Controller：

```java
@RestController
public class YourController implements DefaultApi {
    @Override
    public ResponseEntity<YourResponse> yourMethod(...) {
        // 实现业务逻辑
    }
}
```

### 4️⃣ 运行应用

```powershell
cd backend
mvn spring-boot:run "-Dspring-boot.run.profiles=dev"
```

## 📂 重要文件

| 文件 | 说明 | 是否可编辑 |
|------|------|-----------|
| `src/main/resources/openapi/swagger-input.yml` | API 规范定义 | ✅ 需要维护 |
| `target/generated-sources/openapi/` | 生成的接口和模型 | ❌ 自动生成 |
| `src/main/java/com/henrywang/ems/controller/` | Controller 实现 | ✅ 需要实现 |
| `src/main/java/com/henrywang/ems/config/SwaggerConfig.java` | Swagger 配置 | ⚠️ 一般不需修改 |

## 💡 核心优势

### 不需要写这些注解了

❌ 不需要:
```java
@GetMapping("/api/employees")
@ApiOperation("获取员工列表")
@ApiParam(name = "page", value = "页码")
```

✅ 只需要:
```java
@RestController
public class EmployeeController implements DefaultApi {
    @Override
    public ResponseEntity<EmployeePageResponse> getEmployees(Integer page, Integer size, String keyword) {
        // 实现逻辑
    }
}
```

### 所有路由、验证、文档都从 YAML 生成

```yaml
# swagger-input.yml
paths:
  /api/employees:
    get:
      summary: 获取员工列表
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            minimum: 0
```

## 📖 详细文档

查看完整指南: [docs/openapi-contract-first-guide.md](./openapi-contract-first-guide.md)

## 🔧 Maven 命令

```powershell
# 生成代码
mvn generate-sources

# 清理并重新生成
mvn clean generate-sources

# 编译项目
mvn compile

# 运行应用
mvn spring-boot:run "-Dspring-boot.run.profiles=dev"

# 完整构建
mvn clean install
```

## 📦 已添加的依赖

- `openapi-generator-maven-plugin` - 代码生成器
- `springdoc-openapi-starter-webmvc-ui` - Swagger UI
- `swagger-annotations` - Swagger 注解
- `swagger-parser` - YAML 解析器
- `jackson-databind-nullable` - Jackson 支持

## 🎯 下一步

1. 查看示例 API: http://localhost:8080/swagger-ui.html
2. 修改 `swagger-input.yml` 添加你的 API
3. 运行 `mvn generate-sources` 生成接口
4. 实现生成的接口方法
5. 测试你的 API

## 📚 参考资料

- [OpenAPI Specification](https://swagger.io/specification/)
- [OpenAPI Generator](https://openapi-generator.tech/)
- [完整开发指南](./openapi-contract-first-guide.md)

---

**提示**: 生成的代码在 `target/` 目录下，不要提交到 Git。每次修改 `swagger-input.yml` 后重新生成即可。
