# Phase 0：项目初始化 — 详细执行指南

> 预计周期：第 1 周 | 编制日期：2026-02-14

---

## 〇、Phase 0 目标

完成后端 Spring Boot 与前端 uni-app 的脚手架搭建、本地开发环境配置、数据库初始化、Docker 编排以及 CI/CD 基础流水线，使团队在 Phase 1 开始时即可直接进入业务功能开发。

### 交付物清单

| # | 交付物 | 验收标准 |
|---|--------|---------|
| 1 | Spring Boot 后端项目 | 启动成功，`/swagger-ui.html` 可访问，健康检查 `/actuator/health` 返回 `UP` |
| 2 | uni-app 前端项目 | `npm run dev:h5` 浏览器打开正常；`npm run dev:mp-weixin` 微信开发者工具打开正常 |
| 3 | MySQL 数据库 | 全部 10 张核心表创建完成，Flyway 版本管理生效 |
| 4 | Docker 开发环境 | `docker-compose up` 一键启动 MySQL + 后端 + 前端 + Nginx |
| 5 | CI/CD 流水线 | 代码 push 后自动执行构建与测试 |
| 6 | 项目 README | 包含环境搭建、启动步骤、开发规范说明 |

---

## 一、环境准备

### 1.1 开发机必备软件

| 软件 | 版本要求 | 用途 | 安装验证命令 |
|------|----------|------|-------------|
| **JDK** | 21+ (推荐 Eclipse Temurin) | 后端运行时 | `java -version` |
| **Maven** | 3.9+ | 后端依赖管理 & 构建 | `mvn -version` |
| **Node.js** | 18.x LTS 或 20.x LTS | 前端构建 | `node -v && npm -v` |
| **MySQL** | 8.0 | 数据库（可用 Docker 代替本地安装） | `mysql --version` |
| **Docker Desktop** | 最新稳定版 | 容器化开发 & 部署 | `docker -v && docker compose version` |
| **Git** | 2.40+ | 版本控制 | `git --version` |
| **IDE (后端)** | IntelliJ IDEA / VS Code + Java Extension Pack | Java 开发 | — |
| **IDE (前端)** | HBuilderX 或 VS Code + Volar | uni-app 开发 | — |
| **微信开发者工具** | 最新稳定版 | 小程序调试 | — |
| **Navicat / DBeaver** | 任意版本 | 数据库可视化管理 | — |

### 1.2 环境变量 & 端口规划

```
后端 API 服务：        localhost:8080
MySQL 数据库：         localhost:3306
前端 H5 开发服务器：    localhost:5173 (Vite 默认)
Swagger UI：          localhost:8080/swagger-ui.html
API JSON：            localhost:8080/v3/api-docs
```

---

## 二、后端脚手架搭建 (Spring Boot 3.x)

### 2.1 使用 Spring Initializr 生成项目

访问 [https://start.spring.io](https://start.spring.io) 或通过 IDE 创建，参数如下：

| 配置项 | 值 |
|--------|-----|
| Project | Maven |
| Language | Java |
| Spring Boot | 3.x (最新稳定版) |
| Group | `com.henrywang.ems` |
| Artifact | `backend` |
| Name | `EmployeeManagementSystem` |
| Package name | `com.henrywang.ems` |
| Packaging | Jar |
| Java | 21 |

**勾选 Starter 依赖：**
- Spring Web
- Spring Security
- Spring Validation
- Spring Boot Actuator
- Spring Data JPA
- MySQL Driver
- Flyway Migration
- Lombok

### 2.2 额外 Maven 依赖 (pom.xml)

在 `<dependencies>` 中补充以下依赖（Spring Initializr 不提供的部分）：

```xml
<!-- SpringDoc OpenAPI / Swagger UI -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.4</version>
</dependency>

<!-- JWT (jjwt) -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>

<!-- Apache POI (Excel 导出) -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.2.5</version>
</dependency>

<!-- OpenPDF (PDF 导出) -->
<dependency>
    <groupId>com.github.librepdf</groupId>
    <artifactId>openpdf</artifactId>
    <version>2.0.3</version>
</dependency>

<!-- Hutool 工具类 -->
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.8.34</version>
</dependency>

<!-- 阿里云 OSS SDK -->
<dependency>
    <groupId>com.aliyun.oss</groupId>
    <artifactId>aliyun-sdk-oss</artifactId>
    <version>3.18.1</version>
</dependency>
```

### 2.3 后端项目目录结构

在 `backend/src/main/java/com/henrywang/ems/` 下创建以下包结构：

```
com.henrywang.ems
├── EmsApplication.java              # 启动类
├── config/                          # 配置类
│   ├── CorsConfig.java              # CORS 跨域配置
│   ├── SecurityConfig.java          # Spring Security 配置（Phase 0 先放行所有请求）
│   ├── SwaggerConfig.java           # SpringDoc/Swagger 配置
│   ├── JpaConfig.java               # JPA 配置（审计等）
│   └── JacksonConfig.java           # JSON 序列化配置（日期格式等）
├── controller/                      # REST 控制器
├── service/                         # 业务逻辑接口
│   └── impl/                        # 业务逻辑实现
├── repository/                      # Spring Data JPA Repository 接口
├── entity/                          # 数据库实体类
├── dto/                             # 请求数据传输对象
├── vo/                              # 响应视图对象
├── common/                          # 通用模块
│   ├── result/                      # 统一响应封装 (R<T>)
│   ├── exception/                   # 全局异常处理
│   ├── constant/                    # 常量定义
│   └── util/                        # 工具类
└── security/                        # 安全模块（JWT 等，Phase 1 实现）
```

### 2.4 配置文件

#### `application.yml` — 主配置

```yaml
server:
  port: 8080
  servlet:
    context-path: /

spring:
  profiles:
    active: dev
  application:
    name: ems-backend
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    default-property-inclusion: non_null

# SpringDoc / Swagger
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    tags-sorter: alpha
    operations-sorter: method
  info:
    title: 农民工管理系统 API
    version: 1.0.0
    description: 面向建筑行业的农民工管理系统 RESTful API

# JPA
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # 生产环境使用validate，由Flyway管理表结构
    show-sql: true        # 开发阶段打印SQL
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect
    open-in-view: false   # 禁用OSIV，避免懒加载问题
```

#### `application-dev.yml` — 开发环境

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ems?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
    username: root
    password: root123456
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
    encoding: UTF-8

logging:
  level:
    com.henrywang.ems: debug
    org.hibernate.SQL: debug
    org.hibernate.type.descriptor.sql.BasicBinder: trace
```

#### `application-prod.yml` — 生产环境

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/ems?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=Asia/Shanghai
    username: ${DB_USER:ems_user}
    password: ${DB_PASSWORD}
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
  jpa:
    show-sql: false       # 生产环境关闭SQL日志

logging:
  level:
    com.henrywang.ems: info
    org.hibernate.SQL: warn
```

### 2.5 Phase 0 必须实现的基础类

#### 2.5.1 统一响应封装 `R<T>`

```java
// com.henrywang.ems.common.result.R.java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class R<T> implements Serializable {
    private int code;        // 业务状态码：200 成功，其他为错误
    private String message;  // 提示信息
    private T data;          // 数据

    public static <T> R<T> ok(T data) {
        return new R<>(200, "success", data);
    }

    public static <T> R<T> ok() {
        return new R<>(200, "success", null);
    }

    public static <T> R<T> fail(int code, String message) {
        return new R<>(code, message, null);
    }

    public static <T> R<T> fail(String message) {
        return new R<>(500, message, null);
    }
}
```

#### 2.5.2 全局异常处理器

```java
// com.henrywang.ems.common.exception.GlobalExceptionHandler.java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public R<?> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining("; "));
        return R.fail(400, message);
    }

    @ExceptionHandler(BusinessException.class)
    public R<?> handleBusinessException(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return R.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public R<?> handleException(Exception e) {
        log.error("系统异常", e);
        return R.fail(500, "系统内部错误");
    }
}
```

#### 2.5.3 自定义业务异常

```java
// com.henrywang.ems.common.exception.BusinessException.java
@Getter
public class BusinessException extends RuntimeException {
    private final int code;

    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }

    public BusinessException(String message) {
        this(500, message);
    }
}
```

#### 2.5.4 CORS 跨域配置

```java
// com.henrywang.ems.config.CorsConfig.java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("*")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

#### 2.5.5 Spring Security 配置（Phase 0 暂时放行）

```java
// com.henrywang.ems.config.SecurityConfig.java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .authorizeHttpRequests(auth -> auth
                .anyRequest().permitAll()  // Phase 0: 放行所有请求，Phase 1 实现 JWT 认证
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

#### 2.5.6 Swagger 配置

```java
// com.henrywang.ems.config.SwaggerConfig.java
@Configuration
public class SwaggerConfig {
    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("农民工管理系统 API")
                        .description("面向建筑行业的农民工管理系统 RESTful API 文档")
                        .version("1.0.0")
                        .contact(new Contact().name("EMS Team")))
                .addSecurityItem(new SecurityRequirement().addList("Bearer"))
                .components(new Components()
                        .addSecuritySchemes("Bearer",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")
                                        .description("JWT 认证，请输入 token")));
    }
}
```

#### 2.5.7 JPA 配置

```java
// com.henrywang.ems.config.JpaConfig.java
@Configuration
@EnableJpaRepositories(basePackages = "com.henrywang.ems.repository")
@EnableJpaAuditing
public class JpaConfig {
    // JPA审计配置，自动填充创建时间、更新时间等
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            // 这里可以从SecurityContext获取当前用户
            // Phase 0暂时返回系统用户
            return Optional.of("system");
        };
    }
}
```

#### 2.5.8 健康检查控制器（验证项目启动）

```java
// com.henrywang.ems.controller.HealthController.java
@RestController
@Tag(name = "系统", description = "系统健康检查")
public class HealthController {

    @GetMapping("/api/health")
    @Operation(summary = "健康检查", description = "返回系统运行状态")
    public R<Map<String, Object>> health() {
        Map<String, Object> info = new HashMap<>();
        info.put("status", "UP");
        info.put("timestamp", LocalDateTime.now());
        info.put("version", "1.0.0");
        return R.ok(info);
    }
}
```

---

## 三、数据库初始化 (Flyway)

### 3.1 创建本地 MySQL 数据库

```sql
-- 使用 root 账号执行
CREATE DATABASE IF NOT EXISTS ems
    DEFAULT CHARACTER SET utf8mb4
    DEFAULT COLLATE utf8mb4_unicode_ci;

-- 创建专用账号（可选，开发阶段可直接用 root）
CREATE USER IF NOT EXISTS 'ems_user'@'%' IDENTIFIED BY 'ems_password';
GRANT ALL PRIVILEGES ON ems.* TO 'ems_user'@'%';
FLUSH PRIVILEGES;
```

### 3.2 Flyway 迁移脚本

在 `backend/src/main/resources/db/migration/` 目录下按顺序创建 SQL 文件。

> **命名规范：** `V{版本号}__{描述}.sql`，双下划线分隔，版本号递增。

#### `V1__create_tables.sql` — 创建全部核心表

```sql
-- ============================================
-- V1: 农民工管理系统 核心表创建
-- ============================================

-- 1. 系统用户表
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE COMMENT '登录用户名',
    password VARCHAR(255) COMMENT '密码哈希(BCrypt)',
    real_name VARCHAR(50),
    phone VARCHAR(20),
    avatar VARCHAR(500) COMMENT '头像 URL',
    role ENUM('SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN', 'WORKER') NOT NULL,
    worker_id BIGINT COMMENT '关联工人ID（工人角色时）',
    openid VARCHAR(100) COMMENT '微信小程序 openid',
    unionid VARCHAR(100) COMMENT '微信 unionid',
    wx_nickname VARCHAR(100) COMMENT '微信昵称',
    wx_avatar VARCHAR(500) COMMENT '微信头像',
    session_key VARCHAR(200) COMMENT '微信 session_key (加密存储)',
    status TINYINT DEFAULT 1 COMMENT '1:正常 0:禁用',
    last_login_at DATETIME COMMENT '最后登录时间',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_openid (openid),
    INDEX idx_phone (phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='系统用户表';

-- 2. 工种表
CREATE TABLE work_type (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL UNIQUE COMMENT '工种名称',
    default_daily_wage DECIMAL(10,2) COMMENT '默认日薪',
    description VARCHAR(200),
    status TINYINT DEFAULT 1 COMMENT '1:启用 0:停用',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='工种字典表';

-- 3. 项目/工地表
CREATE TABLE project (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL COMMENT '项目名称',
    district VARCHAR(50) COMMENT '区域',
    location VARCHAR(200) COMMENT '项目地址',
    general_contractor VARCHAR(100) COMMENT '总承包企业',
    subcontractor VARCHAR(100) COMMENT '分包企业',
    project_manager_name VARCHAR(50) COMMENT '项目负责人姓名',
    longitude DECIMAL(10,7) COMMENT '经度（打卡围栏）',
    latitude DECIMAL(10,7) COMMENT '纬度（打卡围栏）',
    fence_radius INT DEFAULT 500 COMMENT '打卡围栏半径(米)',
    description TEXT,
    start_date DATE,
    expected_end_date DATE,
    actual_end_date DATE,
    status ENUM('ACTIVE', 'COMPLETED', 'PAUSED') DEFAULT 'ACTIVE',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='项目/工地表';

-- 4. 工人表（需在 team 表之前创建，因 team 引用 worker）
CREATE TABLE worker (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    worker_no VARCHAR(20) COMMENT '用工编号',
    name VARCHAR(50) NOT NULL COMMENT '姓名',
    id_card VARCHAR(255) COMMENT '身份证号(AES加密存储)',
    phone VARCHAR(20) COMMENT '手机号',
    gender TINYINT COMMENT '1:男 2:女',
    birth_date DATE,
    native_place VARCHAR(100) COMMENT '籍贯',
    photo_url VARCHAR(500) COMMENT '照片URL(OSS)',
    work_type_id BIGINT COMMENT '工种',
    pay_type ENUM('DAILY', 'MONTHLY') DEFAULT 'DAILY' COMMENT '计薪方式',
    daily_wage DECIMAL(10,2) COMMENT '个人日薪',
    monthly_wage DECIMAL(12,2) COMMENT '个人月薪',
    current_project_id BIGINT COMMENT '当前项目',
    current_team_id BIGINT COMMENT '当前班组',
    emergency_contact VARCHAR(50) COMMENT '紧急联系人',
    emergency_phone VARCHAR(20) COMMENT '紧急联系电话',
    bank_account VARCHAR(30) COMMENT '银行卡号',
    bank_name VARCHAR(100) COMMENT '开户行（具体到分行）',
    default_payment_method VARCHAR(20) DEFAULT '总包代发' COMMENT '默认发放方式',
    status ENUM('ACTIVE', 'LEFT', 'IDLE') DEFAULT 'ACTIVE' COMMENT '在岗/离场/待岗',
    join_date DATE COMMENT '入场日期',
    leave_date DATE COMMENT '离场日期',
    remark VARCHAR(500),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (work_type_id) REFERENCES work_type(id),
    FOREIGN KEY (current_project_id) REFERENCES project(id),
    INDEX idx_name (name),
    INDEX idx_phone (phone),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='工人信息表';

-- 5. 班组表
CREATE TABLE team (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL COMMENT '班组名称',
    project_id BIGINT NOT NULL COMMENT '所属项目',
    leader_id BIGINT COMMENT '工长/班组长(工人ID)',
    description VARCHAR(200),
    status TINYINT DEFAULT 1 COMMENT '1:启用 0:停用',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (project_id) REFERENCES project(id),
    FOREIGN KEY (leader_id) REFERENCES worker(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='班组表';

-- worker 表补充外键（班组）
ALTER TABLE worker ADD FOREIGN KEY (current_team_id) REFERENCES team(id);

-- 6. 工人调动记录表
CREATE TABLE worker_transfer (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    worker_id BIGINT NOT NULL,
    from_project_id BIGINT,
    to_project_id BIGINT,
    from_team_id BIGINT,
    to_team_id BIGINT,
    transfer_date DATE NOT NULL,
    reason VARCHAR(200),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (worker_id) REFERENCES worker(id),
    FOREIGN KEY (from_project_id) REFERENCES project(id),
    FOREIGN KEY (to_project_id) REFERENCES project(id),
    FOREIGN KEY (from_team_id) REFERENCES team(id),
    FOREIGN KEY (to_team_id) REFERENCES team(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='工人调动记录表';

-- 7. 考勤/工天记录表
CREATE TABLE attendance (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    worker_id BIGINT NOT NULL,
    project_id BIGINT NOT NULL,
    team_id BIGINT,
    work_date DATE NOT NULL COMMENT '工作日期',
    work_category ENUM('FULL_DAY', 'HALF_DAY', 'OVERTIME', 'LEAVE') NOT NULL COMMENT '出勤类型',
    work_days DECIMAL(3,1) NOT NULL COMMENT '折算工天数',
    daily_wage_snapshot DECIMAL(10,2) NOT NULL COMMENT '当日日薪快照',
    is_overtime TINYINT DEFAULT 0 COMMENT '是否加班',
    overtime_hours DECIMAL(4,1) COMMENT '加班时长(小时)',
    actual_work_output VARCHAR(200) COMMENT '当日实际工作量描述',
    check_in_time DATETIME COMMENT '上班打卡时间',
    check_in_lng DECIMAL(10,7) COMMENT '打卡经度',
    check_in_lat DECIMAL(10,7) COMMENT '打卡纬度',
    check_in_photo VARCHAR(500) COMMENT '打卡照片URL',
    check_out_time DATETIME COMMENT '下班打卡时间',
    check_out_lng DECIMAL(10,7),
    check_out_lat DECIMAL(10,7),
    check_out_photo VARCHAR(500),
    status ENUM('PENDING', 'CONFIRMED', 'REJECTED') DEFAULT 'PENDING',
    confirmed_by BIGINT COMMENT '确认人ID',
    confirmed_at DATETIME,
    remark VARCHAR(500),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (worker_id) REFERENCES worker(id),
    FOREIGN KEY (project_id) REFERENCES project(id),
    FOREIGN KEY (team_id) REFERENCES team(id),
    UNIQUE KEY uk_worker_date (worker_id, work_date),
    INDEX idx_work_date (work_date),
    INDEX idx_project_date (project_id, work_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='考勤/工天记录表';

-- 8. 月度薪资表
CREATE TABLE monthly_salary (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    worker_id BIGINT NOT NULL,
    project_id BIGINT NOT NULL,
    year INT NOT NULL,
    month INT NOT NULL,
    total_work_days DECIMAL(5,1) NOT NULL COMMENT '当月总工天',
    actual_work_output VARCHAR(200) COMMENT '当月实际工作量',
    daily_wage DECIMAL(10,2) COMMENT '当月适用日薪',
    monthly_wage DECIMAL(12,2) COMMENT '当月适用月薪',
    base_salary DECIMAL(12,2) NOT NULL COMMENT '基本薪资',
    overtime_pay DECIMAL(10,2) DEFAULT 0 COMMENT '加班工资',
    gross_salary DECIMAL(12,2) NOT NULL COMMENT '应发金额',
    previous_unpaid DECIMAL(10,2) DEFAULT 0 COMMENT '上月未发金额',
    bonus DECIMAL(10,2) DEFAULT 0 COMMENT '奖金',
    withholding DECIMAL(10,2) DEFAULT 0 COMMENT '代缴金额',
    deduction DECIMAL(10,2) DEFAULT 0 COMMENT '其他扣款',
    advance_payment DECIMAL(10,2) DEFAULT 0 COMMENT '预支/借支',
    actual_salary DECIMAL(12,2) NOT NULL COMMENT '实发金额',
    payment_method VARCHAR(30) DEFAULT '总包代发' COMMENT '发放方式',
    worker_confirmed TINYINT DEFAULT 0 COMMENT '工人签字确认 0:未确认 1:已确认',
    worker_confirmed_at DATETIME COMMENT '工人确认时间',
    -- 多级审批签名
    team_leader_id BIGINT COMMENT '班组长签名',
    team_leader_signed_at DATETIME,
    subcontractor_signer VARCHAR(50) COMMENT '分包单位负责人签名',
    subcontractor_signed_at DATETIME,
    labor_officer_id BIGINT COMMENT '劳资专管员签名',
    labor_officer_signed_at DATETIME,
    finance_signer_id BIGINT COMMENT '项目财务签名',
    finance_signed_at DATETIME,
    business_mgr_id BIGINT COMMENT '项目商务经理签名',
    business_mgr_signed_at DATETIME,
    project_mgr_id BIGINT COMMENT '项目经理签名',
    project_mgr_signed_at DATETIME,
    status ENUM('DRAFT', 'PENDING_APPROVAL', 'APPROVED', 'PAID') DEFAULT 'DRAFT',
    generated_at DATETIME COMMENT '制表时间',
    paid_at DATETIME,
    remark VARCHAR(500),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (worker_id) REFERENCES worker(id),
    FOREIGN KEY (project_id) REFERENCES project(id),
    UNIQUE KEY uk_worker_month (worker_id, project_id, year, month),
    INDEX idx_year_month (year, month)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='月度薪资表';

-- 9. 薪资调整明细表
CREATE TABLE salary_adjustment (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    monthly_salary_id BIGINT NOT NULL,
    type ENUM('BONUS', 'DEDUCTION', 'ADVANCE', 'WITHHOLDING', 'OTHER') NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    reason VARCHAR(200),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (monthly_salary_id) REFERENCES monthly_salary(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='薪资调整明细表';

-- 10. 合同表
CREATE TABLE contract (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    worker_id BIGINT NOT NULL,
    project_id BIGINT,
    contract_no VARCHAR(50),
    file_url VARCHAR(500) COMMENT '合同文件URL(OSS)',
    start_date DATE,
    end_date DATE,
    status ENUM('ACTIVE', 'EXPIRED', 'TERMINATED') DEFAULT 'ACTIVE',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (worker_id) REFERENCES worker(id),
    FOREIGN KEY (project_id) REFERENCES project(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='劳动合同表';
```

#### `V2__seed_data.sql` — 初始种子数据

```sql
-- ============================================
-- V2: 初始种子数据
-- ============================================

-- 超级管理员账号（密码: admin123，BCrypt 加密）
INSERT INTO sys_user (username, password, real_name, phone, role, status)
VALUES ('admin', '$2a$10$EqKcp1WFKVQISheBxmTMGe6/5emu3HVXEN0gEPaF5cLb3VrGmqFxm', '超级管理员', '13800000000', 'SUPER_ADMIN', 1);

-- 常见工种
INSERT INTO work_type (name, default_daily_wage, description) VALUES
('木工',    350.00, '木模板安装、拆除'),
('钢筋工',  340.00, '钢筋绑扎、焊接'),
('泥瓦工',  320.00, '砌墙、抹灰、贴砖'),
('电工',    360.00, '电气安装、布线'),
('水暖工',  350.00, '给排水、暖通安装'),
('架子工',  380.00, '脚手架搭设、拆除'),
('焊工',    400.00, '电焊、气焊作业'),
('油漆工',  300.00, '油漆、涂料施工'),
('普工',    250.00, '搬运、清理、杂工'),
('机械操作工', 400.00, '塔吊、挖掘机等操作');
```

---

## 四、前端脚手架搭建 (uni-app 3.x)

### 4.1 创建 uni-app 项目 (CLI 模式)

> **推荐使用 CLI 模式**（非 HBuilderX 可视化创建），更利于 Git 管理和 CI/CD。

```bash
# 在项目根目录 frontend/ 下执行
# 方式一：使用 degit 拉取官方 Vite + Vue3 模板
npx degit dcloudio/uni-preset-vue#vite-ts frontend
# 如果不使用 TypeScript，可用：
npx degit dcloudio/uni-preset-vue#vite frontend

# 方式二：使用 @dcloudio/create-uni-app
npx @dcloudio/create-uni-app frontend
```

### 4.2 安装前端依赖

```bash
cd frontend

# 核心依赖
npm install pinia                    # 状态管理
npm install dayjs                    # 日期处理
npm install @dcloudio/uni-ui         # uni-app 官方 UI 组件库

# 开发依赖
npm install -D sass                  # SCSS 支持
```

> **注意：** uView Plus 3.x 和 ECharts 可在后续 Phase 按需引入，Phase 0 只需最小依赖。

### 4.3 前端目录结构

确保 `frontend/src/` 下创建以下目录骨架：

```
frontend/src/
├── pages/                           # 页面
│   ├── login/
│   │   └── index.vue                # 登录页（骨架）
│   ├── home/
│   │   └── index.vue                # 首页（骨架）
│   └── profile/
│       └── index.vue                # 个人中心（骨架）
├── components/                      # 组件
│   ├── elderly-ui/                  # 适老化专用组件（后续实现）
│   │   └── .gitkeep
│   └── common/                      # 通用组件
│       └── .gitkeep
├── store/                           # Pinia 状态管理
│   ├── index.js                     # Pinia 初始化
│   └── modules/
│       └── user.js                  # 用户状态（骨架）
├── api/                             # 接口封装
│   ├── request.js                   # uni.request 拦截器封装
│   └── modules/
│       └── .gitkeep
├── hooks/                           # Composition API Hooks
│   └── .gitkeep
├── utils/                           # 工具函数
│   └── .gitkeep
├── platform/                        # 平台差异代码
│   ├── h5/
│   │   └── .gitkeep
│   └── mp-weixin/
│       └── .gitkeep
├── static/                          # 静态资源（图片、图标等）
│   └── .gitkeep
├── App.vue                          # 根组件
├── main.js                          # 入口文件
├── manifest.json                    # uni-app 平台配置
├── pages.json                       # 路由与页面配置
└── uni.scss                         # 全局样式变量
```

### 4.4 核心配置文件

#### `pages.json` — 路由配置

```json
{
  "pages": [
    {
      "path": "pages/login/index",
      "style": {
        "navigationBarTitleText": "登录"
      }
    },
    {
      "path": "pages/home/index",
      "style": {
        "navigationBarTitleText": "首页"
      }
    },
    {
      "path": "pages/profile/index",
      "style": {
        "navigationBarTitleText": "我的"
      }
    }
  ],
  "globalStyle": {
    "navigationBarTextStyle": "black",
    "navigationBarTitleText": "农民工管理",
    "navigationBarBackgroundColor": "#FFFFFF",
    "backgroundColor": "#F5F5F5"
  },
  "tabBar": {
    "color": "#999999",
    "selectedColor": "#1890FF",
    "backgroundColor": "#FFFFFF",
    "borderStyle": "black",
    "list": [
      {
        "pagePath": "pages/home/index",
        "text": "首页",
        "iconPath": "static/tab/home.png",
        "selectedIconPath": "static/tab/home-active.png"
      },
      {
        "pagePath": "pages/profile/index",
        "text": "我的",
        "iconPath": "static/tab/profile.png",
        "selectedIconPath": "static/tab/profile-active.png"
      }
    ]
  }
}
```

#### `manifest.json` — 关键配置项

确保以下字段正确配置：

```json
{
  "name": "农民工管理系统",
  "appid": "",
  "description": "面向建筑行业的农民工管理系统",
  "versionName": "1.0.0",
  "versionCode": "100",
  "h5": {
    "title": "农民工管理系统",
    "devServer": {
      "port": 5173,
      "proxy": {
        "/api": {
          "target": "http://localhost:8080",
          "changeOrigin": true
        }
      }
    }
  },
  "mp-weixin": {
    "appid": "",
    "setting": {
      "urlCheck": false
    },
    "usingComponents": true
  }
}
```

#### `uni.scss` — 全局样式变量（适老化基准）

```scss
/* ============================================
 * 全局样式变量 — 适老化设计基准
 * ============================================ */

/* === 字体大小 === */
$font-size-xs: 14px;        /* 辅助文字 */
$font-size-sm: 16px;        /* 正文最小 */
$font-size-base: 18px;      /* 正文基准（适老化：比标准大 2px） */
$font-size-lg: 20px;        /* 标题 */
$font-size-xl: 24px;        /* 大标题 */
$font-size-xxl: 28px;       /* 特大标题 */

/* === 颜色 === */
$color-primary: #1890FF;
$color-success: #52C41A;
$color-warning: #FAAD14;
$color-error: #FF4D4F;
$color-text: #333333;        /* 主文字 - 高对比度 */
$color-text-secondary: #666666;
$color-text-light: #999999;
$color-bg: #F5F5F5;
$color-bg-white: #FFFFFF;
$color-border: #E8E8E8;

/* === 间距 === */
$spacing-xs: 8px;
$spacing-sm: 12px;
$spacing-md: 16px;
$spacing-lg: 24px;
$spacing-xl: 32px;

/* === 按钮 === */
$btn-height: 48px;           /* 普通按钮最小高度 */
$btn-height-lg: 56px;        /* 重要操作按钮高度 */

/* === 列表 === */
$list-row-height: 52px;      /* 列表行高 — 大触控区域 */

/* === 圆角 === */
$border-radius-sm: 4px;
$border-radius-md: 8px;
$border-radius-lg: 12px;
```

#### `api/request.js` — 网络请求封装

```javascript
/**
 * uni.request 统一封装
 * - Token 自动注入
 * - 统一错误处理
 * - 响应拦截
 */

const BASE_URL = import.meta.env.VITE_API_BASE_URL || '/api'

const request = (options) => {
  return new Promise((resolve, reject) => {
    const token = uni.getStorageSync('token')

    uni.request({
      url: `${BASE_URL}${options.url}`,
      method: options.method || 'GET',
      data: options.data,
      header: {
        'Content-Type': 'application/json',
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...options.header
      },
      success: (res) => {
        if (res.statusCode === 200) {
          const data = res.data
          if (data.code === 200) {
            resolve(data)
          } else if (data.code === 401) {
            // Token 过期，跳转登录
            uni.removeStorageSync('token')
            uni.reLaunch({ url: '/pages/login/index' })
            reject(data)
          } else {
            uni.showToast({ title: data.message || '请求失败', icon: 'none' })
            reject(data)
          }
        } else {
          uni.showToast({ title: `网络错误 ${res.statusCode}`, icon: 'none' })
          reject(res)
        }
      },
      fail: (err) => {
        uni.showToast({ title: '网络连接失败', icon: 'none' })
        reject(err)
      }
    })
  })
}

// 快捷方法
export const get = (url, data) => request({ url, method: 'GET', data })
export const post = (url, data) => request({ url, method: 'POST', data })
export const put = (url, data) => request({ url, method: 'PUT', data })
export const del = (url, data) => request({ url, method: 'DELETE', data })

export default request
```

#### `store/index.js` — Pinia 初始化

```javascript
import { createPinia } from 'pinia'

const pinia = createPinia()

export default pinia
```

#### `main.js` — 入口文件

```javascript
import { createSSRApp } from 'vue'
import App from './App.vue'
import pinia from './store'

export function createApp() {
  const app = createSSRApp(App)
  app.use(pinia)
  return { app }
}
```

### 4.5 骨架页面

#### `pages/home/index.vue` — 首页骨架

```vue
<template>
  <view class="home-page">
    <view class="header">
      <text class="title">农民工管理系统</text>
      <text class="subtitle">v1.0.0</text>
    </view>
    <view class="status-card">
      <text class="status-text">✅ 系统运行正常</text>
      <text class="status-detail">后端 API 已连接</text>
    </view>
  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { get } from '@/api/request.js'

const status = ref('检查中...')

onMounted(async () => {
  try {
    const res = await get('/health')
    status.value = res.data?.status || 'UP'
  } catch (e) {
    status.value = '连接失败'
  }
})
</script>

<style lang="scss" scoped>
.home-page {
  padding: $spacing-lg;
}
.header {
  text-align: center;
  margin-bottom: $spacing-xl;
  .title {
    font-size: $font-size-xxl;
    font-weight: bold;
    color: $color-text;
    display: block;
  }
  .subtitle {
    font-size: $font-size-sm;
    color: $color-text-light;
    margin-top: $spacing-xs;
  }
}
.status-card {
  background: $color-bg-white;
  border-radius: $border-radius-lg;
  padding: $spacing-lg;
  text-align: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  .status-text {
    font-size: $font-size-lg;
    color: $color-success;
    display: block;
  }
  .status-detail {
    font-size: $font-size-sm;
    color: $color-text-secondary;
    margin-top: $spacing-sm;
  }
}
</style>
```

---

## 五、Docker 开发环境

### 5.1 目录结构

```
docker/
├── docker-compose.yml          # 编排文件
├── docker-compose.dev.yml      # 开发环境覆盖配置
├── Dockerfile.backend          # 后端镜像
├── Dockerfile.frontend         # 前端 H5 构建镜像
├── nginx.conf                  # Nginx 配置
├── .env.example                # 环境变量模板
└── mysql/
    └── init/
        └── 01-create-db.sql    # 数据库初始化（Docker MySQL 用）
```

### 5.2 `docker-compose.yml`

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: ems-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-root123456}
      MYSQL_DATABASE: ems
      MYSQL_USER: ${DB_USER:-ems_user}
      MYSQL_PASSWORD: ${DB_PASSWORD:-ems_password}
      TZ: Asia/Shanghai
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
    command: >
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --default-authentication-plugin=mysql_native_password
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p${DB_ROOT_PASSWORD:-root123456}"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ../backend
      dockerfile: ../docker/Dockerfile.backend
    container_name: ems-backend
    environment:
      SPRING_PROFILES_ACTIVE: dev
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: ${DB_USER:-ems_user}
      DB_PASSWORD: ${DB_PASSWORD:-ems_password}
    ports:
      - "8080:8080"
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    container_name: ems-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - frontend_dist:/usr/share/nginx/html
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  mysql_data:
  frontend_dist:
```

### 5.3 `Dockerfile.backend`

```dockerfile
# ---- 构建阶段 ----
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

# ---- 运行阶段 ----
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 5.4 `Dockerfile.frontend`

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build:h5

FROM nginx:alpine
COPY --from=builder /app/dist/build/h5 /usr/share/nginx/html
EXPOSE 80
```

### 5.5 `nginx.conf`

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # 前端 H5 路由 — history 模式
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 后端 API 反向代理
    location /api/ {
        proxy_pass http://backend:8080/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Swagger UI 反向代理
    location /swagger-ui/ {
        proxy_pass http://backend:8080/swagger-ui/;
        proxy_set_header Host $host;
    }
    location /v3/api-docs {
        proxy_pass http://backend:8080/v3/api-docs;
        proxy_set_header Host $host;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
}
```

### 5.6 `.env.example`

```env
# 数据库配置
DB_ROOT_PASSWORD=root123456
DB_USER=ems_user
DB_PASSWORD=ems_password

# 后端配置
SPRING_PROFILES_ACTIVE=dev
JWT_SECRET=your-jwt-secret-key-change-in-production

# OSS 配置（Phase 1+ 使用）
OSS_ENDPOINT=oss-cn-chengdu.aliyuncs.com
OSS_ACCESS_KEY_ID=
OSS_ACCESS_KEY_SECRET=
OSS_BUCKET_NAME=ems-files

# 微信小程序配置（Phase 5 使用）
WX_APPID=
WX_APP_SECRET=
```

---

## 六、Git 仓库与分支策略

### 6.1 `.gitignore`

确保项目根目录的 `.gitignore` 包含以下内容：

```gitignore
# === IDE ===
.idea/
.vscode/
*.iml
*.iws

# === Java / Maven ===
backend/target/
*.class
*.jar
*.log

# === Node.js / 前端 ===
frontend/node_modules/
frontend/dist/
frontend/unpackage/
.DS_Store

# === Docker ===
docker/.env

# === 操作系统 ===
Thumbs.db
Desktop.ini
```

### 6.2 分支策略 (Git Flow 简化版)

```
main          ← 生产分支，仅通过 PR 合并
  └── develop ← 开发主分支，日常合并
        ├── feature/phase-0-scaffold    ← Phase 0 脚手架
        ├── feature/phase-1-auth        ← Phase 1 认证
        ├── feature/phase-1-basic-info  ← Phase 1 基本信息
        └── ...
```

**操作步骤：**
```bash
# 初始化仓库
git init
git add .
git commit -m "chore: 项目初始化"

# 创建远程仓库（Gitee 或 GitHub）后
git remote add origin <仓库地址>
git push -u origin main

# 创建 develop 分支
git checkout -b develop
git push -u origin develop

# Phase 0 开发分支
git checkout -b feature/phase-0-scaffold
```

### 6.3 提交规范 (Conventional Commits)

```
类型: 描述

feat:     新功能
fix:      修复 bug
docs:     文档变更
style:    代码格式（不影响逻辑）
refactor: 重构
test:     测试相关
chore:    构建/工具/依赖
ci:       CI/CD 变更

示例：
  feat: 添加工种字典 CRUD 接口
  fix: 修复分页查询工人列表 NPE
  chore: 升级 Spring Data JPA 版本
  docs: 补充 Phase 1 API 文档
```

---

## 七、CI/CD 基础流水线

### 7.1 GitHub Actions 配置

创建 `.github/workflows/ci.yml`：

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  # ===== 后端构建与测试 =====
  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root123456
          MYSQL_DATABASE: ems_test
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h localhost"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build & Test
        run: mvn clean verify -B
        env:
          SPRING_DATASOURCE_URL: jdbc:mysql://localhost:3306/ems_test?useSSL=false&allowPublicKeyRetrieval=true
          SPRING_DATASOURCE_USERNAME: root
          SPRING_DATASOURCE_PASSWORD: root123456

  # ===== 前端构建 =====
  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Build H5
        run: npm run build:h5

      - name: Build MP-WEIXIN
        run: npm run build:mp-weixin
```

### 7.2 Gitee（备选）

如使用 Gitee，在 `cicd/` 目录下创建 `Jenkinsfile` 或使用 Gitee Go：

```yaml
# cicd/gitee-go.yml
name: CI
displayName: '持续集成'
triggers:
  push:
    branches:
      include: [main, develop]
stages:
  - name: build
    displayName: '构建'
    strategy: naturally
    trigger: auto
    steps:
      - step: build@maven
        name: 后端构建
        displayName: 'Maven Build'
        jdkVersion: 21
        mavenVersion: 3.9
        commands:
          - cd backend && mvn clean package -DskipTests -B
      - step: build@nodejs
        name: 前端构建
        displayName: 'npm Build'
        nodeVersion: 20.11.0
        commands:
          - cd frontend && npm ci && npm run build:h5
```

---

## 八、开发规范速查

### 8.1 后端编码规范

| 规范 | 说明 |
|------|------|
| 包名 | 全小写，`com.henrywang.ems.模块名` |
| 类名 | 大驼峰 `WorkerService` |
| 方法/变量名 | 小驼峰 `getWorkerById` |
| 常量 | 全大写下划线 `MAX_PAGE_SIZE` |
| 数据库字段 | 下划线命名 `work_type_id`，JPA 实体使用驼峰命名属性 |
| Controller | 返回 `R<T>` 统一响应，使用 Swagger 注解 (`@Tag`, `@Operation`, `@Schema`) |
| Service | 接口 + 实现分离 |
| 参数校验 | 使用 `@Valid` + `@NotBlank` / `@NotNull` / `@Size` 等注解 |
| 日志 | 使用 Lombok `@Slf4j`，禁止 `System.out.println` |

### 8.2 前端编码规范

| 规范 | 说明 |
|------|------|
| 文件名 | 短横线命名 `attendance-input.vue` |
| 组件名 | 大驼峰 `ElderlyButton.vue` |
| API 方法 | 小驼峰 `getWorkerList()` |
| CSS | 使用 SCSS + BEM 命名，优先使用 `uni.scss` 全局变量 |
| 单位 | 尺寸使用 `rpx`（小程序兼容），字体使用 `px` |
| 条件编译 | `#ifdef H5` / `#ifdef MP-WEIXIN` 处理平台差异 |
| 状态管理 | 全局状态用 Pinia，页面局部状态用 `ref` / `reactive` |

### 8.3 API 接口规范

| 项目 | 规范 |
|------|------|
| URL 前缀 | 统一 `/api/` |
| HTTP 方法 | `GET` 查询 / `POST` 创建 / `PUT` 更新 / `DELETE` 删除 |
| 分页参数 | `?page=1&size=20` |
| 响应格式 | `{ "code": 200, "message": "success", "data": {...} }` |
| 错误码 | `200` 成功 / `400` 参数错误 / `401` 未认证 / `403` 无权限 / `500` 系统错误 |
| 时间格式 | `yyyy-MM-dd HH:mm:ss` |

---

## 九、Phase 0 检查清单

完成以下所有检查项后，Phase 0 视为交付完成：

### 后端

- [ ] Spring Boot 项目创建完成，`mvn clean compile` 无错误
- [ ] `application.yml` / `application-dev.yml` / `application-prod.yml` 配置完成
- [ ] MySQL 数据库连接成功
- [ ] Flyway 迁移执行成功，10 张核心表已创建
- [ ] 种子数据插入成功（admin 账号 + 工种字典）
- [ ] Swagger UI (`/swagger-ui.html`) 可正常访问
- [ ] 健康检查接口 (`/api/health`) 返回 `{"code":200,"data":{"status":"UP"}}`
- [ ] 统一响应封装 `R<T>` 与全局异常处理器生效
- [ ] JPA 配置完成（审计、Repository扫描）
- [ ] Spring Security 暂时放行所有请求（Phase 1 实现认证）
- [ ] 项目包结构完整（config / controller / service / repository / entity / dto / vo / common / security）

### 前端

- [ ] uni-app CLI 项目创建完成
- [ ] `npm install` 依赖安装成功
- [ ] `npm run dev:h5` 浏览器正常打开首页
- [ ] `npm run dev:mp-weixin` 微信开发者工具正常打开
- [ ] `pages.json` 路由配置正确
- [ ] `manifest.json` H5 代理配置指向后端 `localhost:8080`
- [ ] `uni.scss` 全局样式变量（适老化基准）定义完成
- [ ] `api/request.js` 网络请求封装完成（Token 注入 + 错误处理）
- [ ] Pinia 状态管理初始化完成
- [ ] 首页骨架页面可调用后端 `/api/health` 接口
- [ ] 目录骨架完整（pages / components / store / api / hooks / utils / platform / static）

### DevOps

- [ ] Git 仓库初始化，`.gitignore` 配置完成
- [ ] `main` + `develop` 分支已创建
- [ ] Docker Compose 一键启动 MySQL（`docker-compose up mysql`）
- [ ] `Dockerfile.backend` 构建后端镜像成功
- [ ] `nginx.conf` 反向代理配置完成
- [ ] `.env.example` 环境变量模板文件创建
- [ ] CI/CD 流水线配置文件提交（GitHub Actions / Gitee Go）
- [ ] `README.md` 包含快速开始指南

---

## 十、快速启动步骤（开发模式）

```bash
# 1. 克隆仓库
git clone <仓库地址> EmployeeManagementSystem
cd EmployeeManagementSystem

# 2. 启动 MySQL（Docker）
cd docker
cp .env.example .env       # 按需修改配置
docker compose up -d mysql  # 等待 MySQL 健康检查通过

# 3. 启动后端
cd ../backend
mvn spring-boot:run         # 首次启动 Flyway 自动建表
# 验证：浏览器访问 http://localhost:8080/swagger-ui.html

# 4. 启动前端（新终端）
cd ../frontend
npm install                 # 首次安装依赖
npm run dev:h5              # H5 开发模式
# 验证：浏览器访问 http://localhost:5173

# 5. （可选）小程序开发
npm run dev:mp-weixin
# 用微信开发者工具打开 frontend/dist/dev/mp-weixin 目录
```

---

> **Phase 0 完成后，即可进入 Phase 1：用户认证 + 基本信息管理。**
