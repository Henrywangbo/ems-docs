# OpenAPI契约优先开发指南

## 概述

本项目已配置为使用契约优先（Contract-First）的开发方式，通过 `swagger-input.yml` 文件定义 API 规范，自动生成接口代码。

## 目录结构

```
backend/
├── src/main/resources/openapi/
│   └── swagger-input.yml          # API 规范定义文件（你需要维护这个文件）
├── src/main/java/com/henrywang/ems/
│   ├── api/                       # 生成的接口代码（自动生成，不要手动修改）
│   │   ├── DefaultApi.java        # 生成的 API 接口
│   │   └── model/                 # 生成的数据模型
│   ├── controller/                # 你的 Controller 实现
│   │   └── EmployeeController.java  # 实现生成的接口
│   └── config/
│       └── SwaggerConfig.java     # Swagger 配置
└── target/generated-sources/openapi/  # 生成的源代码目录
```

## 工作流程

### 1. 定义 API 规范

编辑 `src/main/resources/openapi/swagger-input.yml` 文件来定义你的 API：

```yaml
paths:
  /api/your-endpoint:
    get:
      tags:
        - 你的标签
      summary: 端点摘要
      operationId: yourMethodName  # 这将是生成的接口方法名
      parameters:
        - name: paramName
          in: query
          schema:
            type: string
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/YourResponse'

components:
  schemas:
    YourResponse:
      type: object
      properties:
        field1:
          type: string
        field2:
          type: integer
```

### 2. 生成代码

运行以下命令生成接口和模型代码：

```powershell
cd backend
mvn clean generate-sources
```

这将在 `target/generated-sources/openapi/` 目录下生成：
- API 接口（如 `DefaultApi.java`）
- 数据模型类（在 `model/` 目录下）

### 3. 实现接口

创建 Controller 类实现生成的接口：

```java
package com.henrywang.ems.controller;

import com.henrywang.ems.api.DefaultApi;
import com.henrywang.ems.api.model.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class YourController implements DefaultApi {

    @Override
    public ResponseEntity<YourResponse> yourMethodName(String paramName) {
        // 实现你的业务逻辑
        YourResponse response = new YourResponse();
        // ... 设置响应数据
        return ResponseEntity.ok(response);
    }
}
```

**注意：**
- 不需要写任何 `@RequestMapping`、`@GetMapping` 等注解
- 不需要写 `@ApiOperation`、`@ApiParam` 等 Swagger 注解
- 所有的路由、参数验证、文档都由生成的接口提供

### 4. 编译和运行

```powershell
cd backend
mvn clean compile spring-boot:run "-Dspring-boot.run.profiles=dev"
```

或者在 IDE 中直接运行 `EmsApplication`。

## 访问 Swagger UI

应用启动后，访问：
- Swagger UI: http://localhost:8080/swagger-ui.html
- OpenAPI JSON: http://localhost:8080/v3/api-docs

## 重要提示

### DO（应该做的）
✅ 修改 `swagger-input.yml` 来定义和更新 API
✅ 实现生成的接口方法
✅ 在 Service 层编写业务逻辑
✅ 使用生成的 Model 类作为 DTO

### DON'T（不应该做的）
❌ 不要直接修改 `target/generated-sources/openapi/` 下的代码（每次生成会覆盖）
❌ 不要在 Controller 中写路由注解（接口已经包含）
❌ 不要手动编写与 API 规范重复的验证注解
❌ 不要将生成的代码提交到版本控制（已在 `.gitignore` 中）

## 配置说明

### pom.xml 关键配置

```xml
<!-- OpenAPI Generator 插件 -->
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <configuration>
        <inputSpec>src/main/resources/openapi/swagger-input.yml</inputSpec>
        <generatorName>spring</generatorName>
        <apiPackage>com.henrywang.ems.api</apiPackage>
        <modelPackage>com.henrywang.ems.api.model</modelPackage>
        <configOptions>
            <interfaceOnly>true</interfaceOnly>  <!-- 只生成接口 -->
            <useSpringBoot3>true</useSpringBoot3>
            <useTags>true</useTags>
        </configOptions>
    </configuration>
</plugin>
```

### 自定义生成选项

如果需要调整生成的代码，可以修改 `pom.xml` 中的 `configOptions`：

- `interfaceOnly`: true 只生成接口，false 会生成 Controller 实现
- `useTags`: true 使用 OpenAPI tags 作为文件名
- `useJakartaEe`: true 使用 Jakarta EE (Spring Boot 3+)
- `skipDefaultInterface`: true 跳过默认接口实现

更多选项：https://openapi-generator.tech/docs/generators/spring/

## 最佳实践

### API 版本管理

在 `swagger-input.yml` 中使用路径前缀管理版本：

```yaml
paths:
  /api/v1/employees:
    get: ...
  /api/v2/employees:
    get: ...
```

### 错误响应标准化

定义统一的错误响应模型：

```yaml
components:
  schemas:
    ErrorResponse:
      type: object
      properties:
        success:
          type: boolean
        message:
          type: string
        error:
          type: string
```

在所有端点的错误响应中使用：

```yaml
responses:
  '400':
    description: 请求错误
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'
```

### 安全认证

已在 `swagger-input.yml` 中配置 JWT 认证：

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

需要认证的接口添加：

```yaml
paths:
  /api/protected-endpoint:
    get:
      security:
        - bearerAuth: []
```

## 常见问题

### Q: 修改 swagger-input.yml 后代码没有更新？
A: 运行 `mvn clean generate-sources` 重新生成代码。

### Q: IDE 显示找不到生成的类？
A: 确保运行了 `mvn generate-sources`，并且 IDE 已将 `target/generated-sources/openapi/src/main/java` 标记为源代码目录。

### Q: 如何添加自定义的验证逻辑？
A: 在 Controller 实现方法中添加，或在 Service 层处理。生成的接口只处理基本的参数验证。

### Q: 生成的模型类需要添加数据库注解怎么办？
A: 生成的 Model 是 DTO，不应该直接用作 Entity。创建单独的 Entity 类，使用 MapStruct 或 ModelMapper 进行转换。

## 示例

查看项目中的示例：
- API 规范: `src/main/resources/openapi/swagger-input.yml`
- Controller 实现: `src/main/java/com/henrywang/ems/controller/EmployeeController.java`

## 参考资料

- OpenAPI Specification: https://swagger.io/specification/
- OpenAPI Generator: https://openapi-generator.tech/
- SpringDoc OpenAPI: https://springdoc.org/
