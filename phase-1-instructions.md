# Phase 1：用户认证 + 基本信息管理 — 详细实施指南

> 编制日期：2026-02-15  
> 前置条件：Phase 0 已完成（脚手架、数据库迁移、空壳页面就绪）  
> 周期：第 2–3 周（约 10 个工作日）  
> 目标：实现完整的 JWT 认证体系 + 工种/项目/班组/工人 四大模块 CRUD

---

## 一、Phase 1 总览

### 1.1 交付目标

| # | 交付物 | 说明 |
|---|--------|------|
| 1 | JWT 登录认证 | H5 端用户名密码登录，签发 Access Token + Refresh Token |
| 2 | Spring Security 集成 | JWT 过滤器、接口权限控制、角色鉴权 |
| 3 | 工种字典 CRUD | 后端 API + 前端管理页面 |
| 4 | 项目/工地 CRUD | 后端 API + 前端管理页面 |
| 5 | 班组 CRUD | 后端 API + 前端管理页面（含成员管理） |
| 6 | 工人信息 CRUD | 后端 API + 前端管理页面（含照片上传占位） |
| 7 | 工人日薪设置 | 支持个人日薪 / 工种默认日薪 |
| 8 | RBAC 权限体系 | 基于角色的访问控制（4 种角色） |

### 1.2 不在本阶段范围

- ❌ 微信小程序登录（Phase 5）
- ❌ Token 自动刷新（简化为单 Token，后续按需扩展）
- ❌ 照片上传到阿里云 OSS（本阶段仅预留接口，实际存储在 Phase 5）
- ❌ 工人调动功能（P1 优先级，推迟到 Phase 2 或 3）
- ❌ 合同管理（P1 优先级，推迟到 Phase 3）
- ❌ 身份证号 AES 加密存储（本阶段明文存储，后续统一加密）

### 1.3 开发顺序（建议严格遵循）

```
Week 1 (第 2 周):
  Step 1:  统一响应格式 — 解决 R<T> vs OpenAPI 模型的冲突
  Step 2:  JPA Entity 层 — 所有核心实体类
  Step 3:  Repository 层 — 数据访问接口
  Step 4:  JWT 工具类 — Token 签发/解析/校验
  Step 5:  Spring Security + JWT 认证 — 过滤器、UserDetailsService
  Step 6:  登录接口实现 — AuthController
  Step 7:  前端登录对接 — 登录页 → 调用 API → 存 Token → 跳转首页

Week 2 (第 3 周):
  Step 8:  工种字典 CRUD — 后端 Service + Controller + 前端页面
  Step 9:  项目/工地 CRUD — 后端 + 前端
  Step 10: 班组 CRUD — 后端 + 前端（含成员管理）
  Step 11: 工人信息 CRUD — 后端 + 前端（含日薪设置）
  Step 12: 前端导航升级 — 添加管理模块菜单和页面路由
  Step 13: 权限控制 — 前端路由守卫 + 后端接口权限注解
```

---

## 二、后端实现（Spring Boot）

### Step 1：统一响应格式

**背景**：Phase 0 存在两套响应格式——`R<T>` (`{ code, message, data }`) 和 OpenAPI 生成模型 (`{ success, message, data }`)。Phase 1 需要统一。

**决策：统一使用 `R<T>` 格式**，废弃 OpenAPI 生成的 Response 模型。原因：
- `R<T>` 已被 `HealthController`、`GlobalExceptionHandler` 和前端 `request.js` 使用
- 前端已按 `code === 200` 判断成功
- 更符合国内后端开发惯例

**具体操作：**

#### 1.1 不再使用 OpenAPI 生成的 Controller 接口

Phase 0 中 `EmployeeController implements DefaultApi` 的方式会强制使用生成的 Response 模型。从 Phase 1 起，**手写 Controller**，不再实现 `DefaultApi` 接口。OpenAPI YAML 仅作为文档参考，不再驱动代码生成。

> 如果团队倾向于保留 contract-first，可以修改 `swagger-input.yml` 中的 Schema 为 `R<T>` 格式，但这会增加复杂度。建议先手写 Controller，在其上使用 `@Tag`、`@Operation` 等 SpringDoc 注解来生成 Swagger 文档。

#### 1.2 删除或保留 EmployeeController

**删除** `EmployeeController.java`（纯 stub，已无价值）。后续根据业务需求新建 `WorkerController`（对应 `ems_worker` 表，而非泛化的 "Employee"）。

#### 1.3 配置统一响应（可选增强）

若需要自动包装返回值为 `R<T>`，可添加 `ResponseBodyAdvice` 实现。但鉴于本系统规模较小，建议 **在每个 Controller 方法中显式返回 `R.ok(data)`**，更直观可控。

---

### Step 2：JPA Entity 层

在 `com.henrywang.ems.entity` 包下创建实体类，映射 `V1__create_tables.sql` 中的数据库表。

> **命名约定**：实体类名不带 `ems_` 前缀，使用 `@Table(name = "ems_xxx")` 指定表名。

#### 2.1 基础审计字段抽象类

```java
// com.henrywang.ems.entity.BaseEntity
package com.henrywang.ems.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import java.time.LocalDateTime;

@Data
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

#### 2.2 需要创建的实体类（Phase 1 范围）

| # | 实体类 | 对应表 | 核心字段（不含审计字段） | 关联关系 |
|---|--------|--------|------------------------|---------|
| 1 | `SysUser` | `ems_sys_user` | username, password, realName, phone, avatar, role(Enum), workerId, openid, unionid, wxNickname, wxAvatar, sessionKey, status, lastLoginAt | `@ManyToOne → Worker` (可选延迟加载) |
| 2 | `WorkType` | `ems_work_type` | name, defaultDailyWage, description, status | — |
| 3 | `Project` | `ems_project` | name, district, location, generalContractor, subcontractor, projectManagerName, longitude, latitude, fenceRadius, description, startDate, expectedEndDate, actualEndDate, status(Enum) | `@OneToMany → Team` |
| 4 | `Worker` | `ems_worker` | workerNo, name, idCard, phone, gender, birthDate, nativePlace, photoUrl, payType(Enum), dailyWage, monthlyWage, emergencyContact, emergencyPhone, bankAccount, bankName, defaultPaymentMethod, status(Enum), joinDate, leaveDate, remark | `@ManyToOne → WorkType`, `@ManyToOne → Project`, `@ManyToOne → Team` |
| 5 | `Team` | `ems_team` | name, description, status | `@ManyToOne → Project`, `@ManyToOne → Worker (leader)`, `@OneToMany → Worker (members)` |

#### 2.3 枚举类定义

在 `com.henrywang.ems.entity.enums` 包下创建：

```java
// UserRole.java
public enum UserRole {
    SUPER_ADMIN, PROJECT_MANAGER, FOREMAN, WORKER
}

// ProjectStatus.java
public enum ProjectStatus {
    ACTIVE, COMPLETED, PAUSED
}

// WorkerStatus.java
public enum WorkerStatus {
    ACTIVE, LEFT, IDLE
}

// PayType.java
public enum PayType {
    DAILY, MONTHLY
}

// Gender.java  (用于 Worker)
public enum Gender {
    MALE(1), FEMALE(2);
    private final int code;
    Gender(int code) { this.code = code; }
    public int getCode() { return code; }
}
```

> **注意**：数据库中 `gender` 列是 `TINYINT (1=男, 2=女)`，需要自定义 `@Convert(converter = GenderConverter.class)` 或 `@Column(columnDefinition)` 处理映射。推荐使用 JPA `AttributeConverter`。

#### 2.4 Entity 完整示例 — SysUser

```java
package com.henrywang.ems.entity;

import com.henrywang.ems.entity.enums.UserRole;
import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "ems_sys_user")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class SysUser extends BaseEntity {

    @Column(unique = true, length = 50)
    private String username;

    @Column(length = 255)
    private String password;

    @Column(name = "real_name", length = 50)
    private String realName;

    @Column(length = 20)
    private String phone;

    @Column(length = 500)
    private String avatar;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private UserRole role;

    @Column(name = "worker_id")
    private Long workerId;

    @Column(length = 100)
    private String openid;

    @Column(length = 100)
    private String unionid;

    @Column(name = "wx_nickname", length = 100)
    private String wxNickname;

    @Column(name = "wx_avatar", length = 500)
    private String wxAvatar;

    @Column(name = "session_key", length = 200)
    private String sessionKey;

    @Column(columnDefinition = "TINYINT DEFAULT 1")
    private Integer status;

    @Column(name = "last_login_at")
    private LocalDateTime lastLoginAt;
}
```

> 其他实体类（WorkType, Project, Worker, Team）按同样模式创建，字段映射参考 `V1__create_tables.sql`。

#### 2.5 关键注意事项

1. **`@Table(name = "ems_xxx")`**：所有实体必须指定带 `ems_` 前缀的表名
2. **`ddl-auto: validate`**：Hibernate 不会修改表结构，仅校验实体与表是否匹配。如果字段映射有误会启动失败，这是安全保障
3. **`@Enumerated(EnumType.STRING)`**：数据库中角色/状态等 ENUM 列使用字符串形式存储（如 `'ACTIVE'`），JPA 实体也必须用 `EnumType.STRING`
4. **Worker ↔ Team 循环引用**：Worker 有 `current_team_id`，Team 有 `leader_id`（指向 Worker）。建议 Worker 端使用 `@ManyToOne(fetch = LAZY)` + `@JoinColumn(name = "current_team_id")`，Team 端同理

---

### Step 3：Repository 层

在 `com.henrywang.ems.repository` 包下创建 Spring Data JPA Repository。

```java
// SysUserRepository.java
public interface SysUserRepository extends JpaRepository<SysUser, Long> {
    Optional<SysUser> findByUsername(String username);
    Optional<SysUser> findByOpenid(String openid);
    boolean existsByUsername(String username);
    Optional<SysUser> findByPhone(String phone);
}

// WorkTypeRepository.java
public interface WorkTypeRepository extends JpaRepository<WorkType, Long> {
    boolean existsByName(String name);
    List<WorkType> findByStatus(Integer status);
}

// ProjectRepository.java
public interface ProjectRepository extends JpaRepository<Project, Long> {
    List<Project> findByStatus(ProjectStatus status);
    Page<Project> findByNameContaining(String name, Pageable pageable);
}

// TeamRepository.java
public interface TeamRepository extends JpaRepository<Team, Long> {
    List<Team> findByProjectId(Long projectId);
    List<Team> findByLeaderId(Long leaderId);
}

// WorkerRepository.java
public interface WorkerRepository extends JpaRepository<Worker, Long> {
    Page<Worker> findByNameContainingOrPhoneContaining(String name, String phone, Pageable pageable);
    List<Worker> findByCurrentProjectId(Long projectId);
    List<Worker> findByCurrentTeamId(Long teamId);
    List<Worker> findByWorkTypeId(Long workTypeId);
    Optional<Worker> findByIdCard(String idCard);
    Optional<Worker> findByPhone(String phone);
    boolean existsByIdCard(String idCard);
}
```

---

### Step 4：JWT 工具类

在 `com.henrywang.ems.security` 包下实现。

#### 4.1 application.yml 添加 JWT 配置

```yaml
# 在 application.yml 中添加（或 application-dev.yml）
jwt:
  secret: "这里放一个至少256位的随机密钥-开发环境可简写-生产环境必须使用强密钥"
  expiration: 86400000   # 24小时（毫秒）
```

#### 4.2 JwtUtils.java

```java
package com.henrywang.ems.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

@Component
public class JwtUtils {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private long expiration;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    /**
     * 生成 JWT Token
     */
    public String generateToken(Long userId, String username, String role) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", userId);
        claims.put("role", role);

        return Jwts.builder()
                .claims(claims)
                .subject(username)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey())
                .compact();
    }

    /**
     * 从 Token 中获取用户名
     */
    public String getUsernameFromToken(String token) {
        return parseClaims(token).getSubject();
    }

    /**
     * 从 Token 中获取用户 ID
     */
    public Long getUserIdFromToken(String token) {
        return parseClaims(token).get("userId", Long.class);
    }

    /**
     * 从 Token 中获取角色
     */
    public String getRoleFromToken(String token) {
        return parseClaims(token).get("role", String.class);
    }

    /**
     * 校验 Token 是否有效
     */
    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    /**
     * 获取过期时间（秒）
     */
    public long getExpirationSeconds() {
        return expiration / 1000;
    }

    private Claims parseClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

---

### Step 5：Spring Security + JWT 认证

#### 5.1 JwtAuthenticationFilter.java

请求进来时从 `Authorization: Bearer xxx` 头中提取 Token，校验后将用户信息放入 SecurityContext。

```java
package com.henrywang.ems.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtils jwtUtils;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String token = extractToken(request);

        if (StringUtils.hasText(token) && jwtUtils.validateToken(token)) {
            String username = jwtUtils.getUsernameFromToken(token);
            Long userId = jwtUtils.getUserIdFromToken(token);
            String role = jwtUtils.getRoleFromToken(token);

            // 构造 Authentication 对象
            List<SimpleGrantedAuthority> authorities =
                    List.of(new SimpleGrantedAuthority("ROLE_" + role));

            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(username, null, authorities);
            // 将 userId 放入 details，方便后续获取
            authentication.setDetails(userId);

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

#### 5.2 更新 SecurityConfig.java

```java
package com.henrywang.ems.config;

import com.henrywang.ems.security.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity                               // 启用 @PreAuthorize 注解
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .authorizeHttpRequests(auth -> auth
                // 公开接口
                .requestMatchers(
                    "/api/auth/login",
                    "/api/auth/wx-login",
                    "/api/health",
                    "/health",
                    "/swagger-ui/**",
                    "/swagger-ui.html",
                    "/v3/api-docs/**",
                    "/error"
                ).permitAll()
                // 其他接口需要认证
                .anyRequest().authenticated()
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            // 在 UsernamePasswordAuthenticationFilter 之前添加 JWT 过滤器
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

#### 5.3 更新 JpaConfig.java 中的 AuditorAware

```java
// Phase 1: 从 SecurityContext 中获取当前用户
@Bean
public AuditorAware<String> auditorProvider() {
    return () -> {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()
            && !"anonymousUser".equals(auth.getPrincipal())) {
            return Optional.of(auth.getName());
        }
        return Optional.of("system");
    };
}
```

#### 5.4 SecurityUtils 工具类

```java
// com.henrywang.ems.security.SecurityUtils
package com.henrywang.ems.security;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

public class SecurityUtils {

    /**
     * 获取当前登录用户名
     */
    public static String getCurrentUsername() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth != null ? auth.getName() : null;
    }

    /**
     * 获取当前登录用户 ID
     */
    public static Long getCurrentUserId() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getDetails() instanceof Long) {
            return (Long) auth.getDetails();
        }
        return null;
    }

    /**
     * 获取当前用户角色
     */
    public static String getCurrentRole() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && !auth.getAuthorities().isEmpty()) {
            // "ROLE_SUPER_ADMIN" → "SUPER_ADMIN"
            return auth.getAuthorities().iterator().next()
                    .getAuthority().replace("ROLE_", "");
        }
        return null;
    }
}
```

---

### Step 6：登录接口实现

#### 6.1 AuthController.java

```java
package com.henrywang.ems.controller;

import com.henrywang.ems.common.result.R;
import com.henrywang.ems.dto.LoginRequest;
import com.henrywang.ems.dto.LoginResponse;
import com.henrywang.ems.service.AuthService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@Tag(name = "认证管理", description = "登录、登出、获取当前用户信息")
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @Operation(summary = "用户名密码登录")
    @PostMapping("/login")
    public R<LoginResponse> login(@RequestBody @Valid LoginRequest request) {
        return R.ok(authService.login(request));
    }

    @Operation(summary = "获取当前用户信息")
    @GetMapping("/profile")
    public R<LoginResponse.UserInfoVO> getProfile() {
        return R.ok(authService.getCurrentUserInfo());
    }

    @Operation(summary = "登出")
    @PostMapping("/logout")
    public R<Void> logout() {
        // JWT 无状态，客户端删除 Token 即可
        // 如需服务端失效，可引入 Token 黑名单（Redis），本阶段暂不实现
        return R.ok();
    }
}
```

#### 6.2 DTO 定义

```java
// com.henrywang.ems.dto.LoginRequest
package com.henrywang.ems.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class LoginRequest {
    @NotBlank(message = "用户名不能为空")
    private String username;

    @NotBlank(message = "密码不能为空")
    private String password;
}
```

```java
// com.henrywang.ems.dto.LoginResponse
package com.henrywang.ems.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class LoginResponse {
    private String token;
    private String tokenType;     // "Bearer"
    private Long expiresIn;       // 过期时间(秒)
    private UserInfoVO user;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class UserInfoVO {
        private Long id;
        private String username;
        private String realName;
        private String phone;
        private String avatar;
        private String role;
    }
}
```

#### 6.3 AuthService.java

```java
// com.henrywang.ems.service.AuthService (接口)
package com.henrywang.ems.service;

import com.henrywang.ems.dto.LoginRequest;
import com.henrywang.ems.dto.LoginResponse;

public interface AuthService {
    LoginResponse login(LoginRequest request);
    LoginResponse.UserInfoVO getCurrentUserInfo();
}
```

```java
// com.henrywang.ems.service.impl.AuthServiceImpl
package com.henrywang.ems.service.impl;

import com.henrywang.ems.common.exception.BusinessException;
import com.henrywang.ems.dto.LoginRequest;
import com.henrywang.ems.dto.LoginResponse;
import com.henrywang.ems.entity.SysUser;
import com.henrywang.ems.repository.SysUserRepository;
import com.henrywang.ems.security.JwtUtils;
import com.henrywang.ems.security.SecurityUtils;
import com.henrywang.ems.service.AuthService;
import lombok.RequiredArgsConstructor;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
public class AuthServiceImpl implements AuthService {

    private final SysUserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtUtils jwtUtils;

    @Override
    public LoginResponse login(LoginRequest request) {
        // 1. 查找用户
        SysUser user = userRepository.findByUsername(request.getUsername())
                .orElseThrow(() -> new BusinessException(401, "用户名或密码错误"));

        // 2. 校验密码
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new BusinessException(401, "用户名或密码错误");
        }

        // 3. 检查账号状态
        if (user.getStatus() == null || user.getStatus() == 0) {
            throw new BusinessException(403, "账号已被禁用");
        }

        // 4. 生成 Token
        String token = jwtUtils.generateToken(
                user.getId(), user.getUsername(), user.getRole().name());

        // 5. 更新最后登录时间
        user.setLastLoginAt(LocalDateTime.now());
        userRepository.save(user);

        // 6. 构造响应
        return LoginResponse.builder()
                .token(token)
                .tokenType("Bearer")
                .expiresIn(jwtUtils.getExpirationSeconds())
                .user(toUserInfoVO(user))
                .build();
    }

    @Override
    public LoginResponse.UserInfoVO getCurrentUserInfo() {
        Long userId = SecurityUtils.getCurrentUserId();
        SysUser user = userRepository.findById(userId)
                .orElseThrow(() -> new BusinessException(404, "用户不存在"));
        return toUserInfoVO(user);
    }

    private LoginResponse.UserInfoVO toUserInfoVO(SysUser user) {
        return LoginResponse.UserInfoVO.builder()
                .id(user.getId())
                .username(user.getUsername())
                .realName(user.getRealName())
                .phone(user.getPhone())
                .avatar(user.getAvatar())
                .role(user.getRole().name())
                .build();
    }
}
```

#### 6.4 更新 GlobalExceptionHandler

确保 401 错误返回正确的 HTTP 状态码：

```java
// 在 GlobalExceptionHandler 中添加
@ExceptionHandler(BusinessException.class)
public ResponseEntity<R<Void>> handleBusinessException(BusinessException e) {
    int httpStatus = e.getCode() == 401 ? 401 : (e.getCode() == 403 ? 403 : 200);
    // 注意：业务异常仍用 R<T> 格式返回，但 HTTP 状态码需要适当设置
    // 401 和 403 返回对应 HTTP 状态码，便于前端统一拦截
    return ResponseEntity.status(httpStatus).body(R.fail(e.getCode(), e.getMessage()));
}
```

---

### Step 7：后端 CRUD — 工种、项目、班组、工人

Phase 1 的核心业务代码。每个模块遵循统一模式：**DTO → Service → ServiceImpl → Controller**。

#### 7.1 通用分页 DTO

```java
// com.henrywang.ems.dto.PageResponse
package com.henrywang.ems.dto;

import lombok.Builder;
import lombok.Data;
import java.util.List;

@Data
@Builder
public class PageResponse<T> {
    private List<T> content;       // 数据列表
    private long totalElements;    // 总条数
    private int totalPages;        // 总页数
    private int number;            // 当前页码（从0开始）
    private int size;              // 每页大小
    private boolean first;         // 是否首页
    private boolean last;          // 是否末页
}
```

工具方法——从 Spring `Page<E>` 转换：

```java
// com.henrywang.ems.common.util.PageUtils
public class PageUtils {
    public static <T> PageResponse<T> toPageResponse(Page<?> page, List<T> content) {
        return PageResponse.<T>builder()
                .content(content)
                .totalElements(page.getTotalElements())
                .totalPages(page.getTotalPages())
                .number(page.getNumber())
                .size(page.getSize())
                .first(page.isFirst())
                .last(page.isLast())
                .build();
    }
}
```

#### 7.2 模块 A：工种字典管理

**DTO:**
```java
// com.henrywang.ems.dto.WorkTypeDTO
@Data
public class WorkTypeDTO {
    @NotBlank(message = "工种名称不能为空")
    @Size(max = 50)
    private String name;

    @DecimalMin(value = "0.00")
    private BigDecimal defaultDailyWage;

    @Size(max = 200)
    private String description;
}
```

**VO:**
```java
// com.henrywang.ems.vo.WorkTypeVO
@Data
@Builder
public class WorkTypeVO {
    private Long id;
    private String name;
    private BigDecimal defaultDailyWage;
    private String description;
    private Integer status;
    private LocalDateTime createdAt;
}
```

**Service:**
```java
public interface WorkTypeService {
    List<WorkTypeVO> listAll();                            // 全量查询（数据量小）
    WorkTypeVO getById(Long id);
    WorkTypeVO create(WorkTypeDTO dto);
    WorkTypeVO update(Long id, WorkTypeDTO dto);
    void delete(Long id);                                 // 逻辑删除（status=0）
}
```

**Controller:**
```java
@Tag(name = "工种管理")
@RestController
@RequestMapping("/api/work-types")
@RequiredArgsConstructor
public class WorkTypeController {

    private final WorkTypeService workTypeService;

    @Operation(summary = "工种列表")
    @GetMapping
    public R<List<WorkTypeVO>> list() {
        return R.ok(workTypeService.listAll());
    }

    @Operation(summary = "工种详情")
    @GetMapping("/{id}")
    public R<WorkTypeVO> getById(@PathVariable Long id) {
        return R.ok(workTypeService.getById(id));
    }

    @Operation(summary = "新增工种")
    @PostMapping
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    public R<WorkTypeVO> create(@RequestBody @Valid WorkTypeDTO dto) {
        return R.ok(workTypeService.create(dto));
    }

    @Operation(summary = "编辑工种")
    @PutMapping("/{id}")
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    public R<WorkTypeVO> update(@PathVariable Long id, @RequestBody @Valid WorkTypeDTO dto) {
        return R.ok(workTypeService.update(id, dto));
    }

    @Operation(summary = "删除工种")
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public R<Void> delete(@PathVariable Long id) {
        workTypeService.delete(id);
        return R.ok();
    }
}
```

#### 7.3 模块 B：项目/工地管理

**DTO:**
```java
// com.henrywang.ems.dto.ProjectDTO
@Data
public class ProjectDTO {
    @NotBlank(message = "项目名称不能为空")
    @Size(max = 100)
    private String name;

    @Size(max = 50)
    private String district;         // 区域

    @Size(max = 200)
    private String location;         // 地址

    @Size(max = 100)
    private String generalContractor; // 总承包企业

    @Size(max = 100)
    private String subcontractor;     // 分包企业

    @Size(max = 50)
    private String projectManagerName; // 项目负责人姓名

    private BigDecimal longitude;     // 经度
    private BigDecimal latitude;      // 纬度
    private Integer fenceRadius;      // 打卡围栏半径(米)

    private String description;
    private LocalDate startDate;
    private LocalDate expectedEndDate;
}
```

**VO:**
```java
// com.henrywang.ems.vo.ProjectVO
@Data
@Builder
public class ProjectVO {
    private Long id;
    private String name;
    private String district;
    private String location;
    private String generalContractor;
    private String subcontractor;
    private String projectManagerName;
    private BigDecimal longitude;
    private BigDecimal latitude;
    private Integer fenceRadius;
    private String description;
    private LocalDate startDate;
    private LocalDate expectedEndDate;
    private LocalDate actualEndDate;
    private String status;            // ACTIVE / COMPLETED / PAUSED
    private Integer workerCount;      // 在岗工人数（查询时统计）
    private Integer teamCount;        // 班组数（查询时统计）
    private LocalDateTime createdAt;
}
```

**Service:**
```java
public interface ProjectService {
    PageResponse<ProjectVO> list(String keyword, String status, int page, int size);
    ProjectVO getById(Long id);
    ProjectVO create(ProjectDTO dto);
    ProjectVO update(Long id, ProjectDTO dto);
    void delete(Long id);
    // 项目下的工人列表
    List<WorkerVO> getWorkersByProjectId(Long projectId);
    // 项目下的班组列表
    List<TeamVO> getTeamsByProjectId(Long projectId);
}
```

**Controller 路由设计：**
```
GET    /api/projects                      → list(keyword, status, page, size)
POST   /api/projects                      → create(ProjectDTO)
GET    /api/projects/{id}                 → getById(id)
PUT    /api/projects/{id}                 → update(id, ProjectDTO)
DELETE /api/projects/{id}                 → delete(id)
GET    /api/projects/{id}/workers         → getWorkersByProjectId(id)
GET    /api/projects/{id}/teams           → getTeamsByProjectId(id)
```

#### 7.4 模块 C：班组管理

**DTO:**
```java
// com.henrywang.ems.dto.TeamDTO
@Data
public class TeamDTO {
    @NotBlank(message = "班组名称不能为空")
    @Size(max = 100)
    private String name;

    @NotNull(message = "所属项目不能为空")
    private Long projectId;

    private Long leaderId;       // 工长/班组长(工人ID)

    @Size(max = 200)
    private String description;
}
```

**VO:**
```java
// com.henrywang.ems.vo.TeamVO
@Data
@Builder
public class TeamVO {
    private Long id;
    private String name;
    private Long projectId;
    private String projectName;   // 冗余展示
    private Long leaderId;
    private String leaderName;    // 冗余展示
    private String description;
    private Integer status;
    private Integer memberCount;  // 成员数量（查询时统计）
    private LocalDateTime createdAt;
}
```

**Service:**
```java
public interface TeamService {
    List<TeamVO> listByProject(Long projectId);
    TeamVO getById(Long id);
    TeamVO create(TeamDTO dto);
    TeamVO update(Long id, TeamDTO dto);
    void delete(Long id);
    // 成员管理
    void addMember(Long teamId, Long workerId);
    void removeMember(Long teamId, Long workerId);
    List<WorkerVO> getMembers(Long teamId);
}
```

**Controller 路由设计：**
```
GET    /api/teams                          → list(projectId)
POST   /api/teams                          → create(TeamDTO)
GET    /api/teams/{id}                     → getById(id)
PUT    /api/teams/{id}                     → update(id, TeamDTO)
DELETE /api/teams/{id}                     → delete(id)
GET    /api/teams/{id}/members             → getMembers(id)
POST   /api/teams/{id}/members             → addMember(id, workerId)
DELETE /api/teams/{id}/members/{workerId}  → removeMember(id, workerId)
```

#### 7.5 模块 D：工人信息管理

**DTO:**
```java
// com.henrywang.ems.dto.WorkerDTO
@Data
public class WorkerDTO {
    @NotBlank(message = "姓名不能为空")
    @Size(max = 50)
    private String name;

    @Pattern(regexp = "^\\d{17}[\\dXx]$", message = "身份证号格式不正确")
    private String idCard;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;

    private Integer gender;           // 1:男 2:女

    private LocalDate birthDate;

    @Size(max = 100)
    private String nativePlace;       // 籍贯

    private Long workTypeId;          // 工种ID
    private String payType;           // DAILY / MONTHLY

    @DecimalMin(value = "0.00")
    private BigDecimal dailyWage;     // 个人日薪

    @DecimalMin(value = "0.00")
    private BigDecimal monthlyWage;   // 个人月薪

    private Long currentProjectId;    // 当前项目
    private Long currentTeamId;       // 当前班组

    @Size(max = 50)
    private String emergencyContact;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "紧急联系电话格式不正确")
    private String emergencyPhone;

    @Size(max = 30)
    private String bankAccount;       // 银行卡号

    @Size(max = 100)
    private String bankName;          // 开户行

    private String defaultPaymentMethod; // 默认发放方式

    private LocalDate joinDate;
    private String remark;
}
```

**VO (分列表/详情两套):**
```java
// com.henrywang.ems.vo.WorkerVO — 列表展示用（精简字段）
@Data
@Builder
public class WorkerVO {
    private Long id;
    private String workerNo;
    private String name;
    private String phone;
    private Integer gender;
    private String workTypeName;     // 工种名称
    private BigDecimal dailyWage;
    private String currentProjectName;
    private String currentTeamName;
    private String status;           // ACTIVE / LEFT / IDLE
    private LocalDate joinDate;
}

// com.henrywang.ems.vo.WorkerDetailVO — 详情用（全量字段）
@Data
@Builder
public class WorkerDetailVO {
    private Long id;
    private String workerNo;
    private String name;
    private String idCard;           // 脱敏展示：110***********1234
    private String phone;
    private Integer gender;
    private LocalDate birthDate;
    private String nativePlace;
    private String photoUrl;
    private Long workTypeId;
    private String workTypeName;
    private String payType;
    private BigDecimal dailyWage;
    private BigDecimal monthlyWage;
    private Long currentProjectId;
    private String currentProjectName;
    private Long currentTeamId;
    private String currentTeamName;
    private String emergencyContact;
    private String emergencyPhone;
    private String bankAccount;      // 脱敏展示：****1234
    private String bankName;
    private String defaultPaymentMethod;
    private String status;
    private LocalDate joinDate;
    private LocalDate leaveDate;
    private String remark;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

**Service:**
```java
public interface WorkerService {
    PageResponse<WorkerVO> list(String keyword, Long projectId, Long teamId,
                                 Long workTypeId, String status, int page, int size);
    WorkerDetailVO getById(Long id);
    WorkerDetailVO create(WorkerDTO dto);
    WorkerDetailVO update(Long id, WorkerDTO dto);
    void delete(Long id);                    // 逻辑删除：设状态为 LEFT
    void updateWage(Long id, BigDecimal dailyWage, BigDecimal monthlyWage);
}
```

**Controller 路由设计：**
```
GET    /api/workers                        → list(keyword, projectId, teamId, workTypeId, status, page, size)
POST   /api/workers                        → create(WorkerDTO)
GET    /api/workers/{id}                   → getById(id)
PUT    /api/workers/{id}                   → update(id, WorkerDTO)
DELETE /api/workers/{id}                   → delete(id)
PUT    /api/workers/{id}/wage              → updateWage(id, dailyWage, monthlyWage)
```

#### 7.6 权限控制矩阵

| 接口 | SUPER_ADMIN | PROJECT_MANAGER | FOREMAN | WORKER |
|------|:-----------:|:---------------:|:-------:|:------:|
| 登录/登出/个人信息 | ✅ | ✅ | ✅ | ✅ |
| 工种 查看 | ✅ | ✅ | ✅ | ❌ |
| 工种 增删改 | ✅ | ✅ | ❌ | ❌ |
| 项目 查看 | ✅ | ✅ (本项目) | ✅ (本项目) | ❌ |
| 项目 增删改 | ✅ | ❌ | ❌ | ❌ |
| 班组 查看 | ✅ | ✅ (本项目) | ✅ (本班组) | ❌ |
| 班组 增删改 | ✅ | ✅ | ❌ | ❌ |
| 工人 查看 | ✅ | ✅ (本项目) | ✅ (本班组) | ✅ (仅自己) |
| 工人 增删改 | ✅ | ✅ | ✅ (本班组) | ❌ |
| 工人 日薪修改 | ✅ | ✅ | ❌ | ❌ |

> **Phase 1 实现方式**：使用 `@PreAuthorize` 注解做角色级别控制。数据级别的行级过滤（如"仅看本项目"）在 Service 层通过 `SecurityUtils.getCurrentUserId()` + 查询 user 关联的项目ID 来实现。

---

### Step 8：后端文件结构总览（Phase 1 完成后）

```
com.henrywang.ems/
├── EmsApplication.java
├── common/
│   ├── constant/
│   │   └── Constants.java             # 常量定义（可选）
│   ├── exception/
│   │   ├── BusinessException.java     # [已有]
│   │   └── GlobalExceptionHandler.java # [已有，需更新401/403处理]
│   ├── result/
│   │   └── R.java                     # [已有]
│   └── util/
│       └── PageUtils.java             # [新增] 分页工具
├── config/
│   ├── CorsConfig.java               # [已有]
│   ├── JacksonConfig.java            # [已有]
│   ├── JpaConfig.java                # [已有，需更新 AuditorAware]
│   ├── SecurityConfig.java           # [已有，需重写]
│   └── SwaggerConfig.java            # [已有]
├── controller/
│   ├── AuthController.java           # [新增] 登录/登出/个人信息
│   ├── HealthController.java         # [已有]
│   ├── WorkTypeController.java       # [新增] 工种 CRUD
│   ├── ProjectController.java        # [新增] 项目 CRUD
│   ├── TeamController.java           # [新增] 班组 CRUD
│   └── WorkerController.java         # [新增] 工人 CRUD
├── dto/
│   ├── LoginRequest.java             # [新增]
│   ├── LoginResponse.java            # [新增]
│   ├── PageResponse.java             # [新增]
│   ├── WorkTypeDTO.java              # [新增]
│   ├── ProjectDTO.java               # [新增]
│   ├── TeamDTO.java                  # [新增]
│   └── WorkerDTO.java                # [新增]
├── entity/
│   ├── BaseEntity.java               # [新增]
│   ├── SysUser.java                  # [新增]
│   ├── WorkType.java                 # [新增]
│   ├── Project.java                  # [新增]
│   ├── Worker.java                   # [新增]
│   ├── Team.java                     # [新增]
│   └── enums/
│       ├── UserRole.java             # [新增]
│       ├── ProjectStatus.java        # [新增]
│       ├── WorkerStatus.java         # [新增]
│       ├── PayType.java              # [新增]
│       └── Gender.java               # [新增]
├── repository/
│   ├── SysUserRepository.java        # [新增]
│   ├── WorkTypeRepository.java       # [新增]
│   ├── ProjectRepository.java        # [新增]
│   ├── TeamRepository.java           # [新增]
│   └── WorkerRepository.java         # [新增]
├── security/
│   ├── JwtUtils.java                 # [新增]
│   ├── JwtAuthenticationFilter.java  # [新增]
│   └── SecurityUtils.java            # [新增]
├── service/
│   ├── AuthService.java
│   ├── WorkTypeService.java
│   ├── ProjectService.java
│   ├── TeamService.java
│   ├── WorkerService.java
│   └── impl/
│       ├── AuthServiceImpl.java
│       ├── WorkTypeServiceImpl.java
│       ├── ProjectServiceImpl.java
│       ├── TeamServiceImpl.java
│       └── WorkerServiceImpl.java
└── vo/
    ├── WorkTypeVO.java
    ├── ProjectVO.java
    ├── TeamVO.java
    ├── WorkerVO.java
    └── WorkerDetailVO.java
```

---

## 三、前端实现（uni-app）

### Step 9：前端登录对接

#### 9.1 创建 Auth API 模块

```javascript
// frontend/src/api/modules/auth.js
import { post, get } from '../request'

/**
 * 用户名密码登录
 */
export const login = (data) => post('/auth/login', data)

/**
 * 获取当前用户信息
 */
export const getProfile = () => get('/auth/profile')

/**
 * 登出
 */
export const logout = () => post('/auth/logout')
```

#### 9.2 更新 Login 页面

```vue
<!-- frontend/src/pages/login/index.vue -->
<script setup>
import { ref } from 'vue'
import { login } from '@/api/modules/auth'
import { useUserStore } from '@/store/modules/user'

const userStore = useUserStore()
const username = ref('')
const password = ref('')
const loading = ref(false)

const handleLogin = async () => {
  if (!username.value || !password.value) {
    uni.showToast({ title: '请输入用户名和密码', icon: 'none' })
    return
  }

  loading.value = true
  try {
    const res = await login({
      username: username.value,
      password: password.value
    })
    // res = { code: 200, message: 'success', data: { token, tokenType, expiresIn, user } }
    const { token, user } = res.data
    userStore.setToken(token)
    userStore.setUserInfo(user)

    uni.showToast({ title: '登录成功', icon: 'success' })
    setTimeout(() => {
      uni.switchTab({ url: '/pages/home/index' })
    }, 500)
  } catch (err) {
    // 错误已由 request.js 统一处理并弹 Toast
    console.error('登录失败:', err)
  } finally {
    loading.value = false
  }
}
</script>
```

#### 9.3 更新 User Store

```javascript
// frontend/src/store/modules/user.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { getProfile } from '@/api/modules/auth'

export const useUserStore = defineStore('user', () => {
  const token = ref(uni.getStorageSync('token') || '')
  const userInfo = ref(uni.getStorageSync('userInfo') ? JSON.parse(uni.getStorageSync('userInfo')) : null)

  const isLoggedIn = computed(() => !!token.value)
  const userRole = computed(() => userInfo.value?.role || '')

  function setToken(newToken) {
    token.value = newToken
    uni.setStorageSync('token', newToken)
  }

  function setUserInfo(info) {
    userInfo.value = info
    uni.setStorageSync('userInfo', JSON.stringify(info))
  }

  async function fetchProfile() {
    try {
      const res = await getProfile()
      setUserInfo(res.data)
    } catch (e) {
      console.error('获取用户信息失败', e)
    }
  }

  function logout() {
    token.value = ''
    userInfo.value = null
    uni.removeStorageSync('token')
    uni.removeStorageSync('userInfo')
    uni.reLaunch({ url: '/pages/login/index' })
  }

  // 检查是否有特定角色
  function hasRole(...roles) {
    return roles.includes(userRole.value)
  }

  return {
    token, userInfo, isLoggedIn, userRole,
    setToken, setUserInfo, fetchProfile, logout, hasRole
  }
})
```

#### 9.4 导航守卫 — 登录状态检查

```javascript
// frontend/src/utils/auth.js
/**
 * 页面拦截器：未登录时跳转登录页
 * 在 App.vue 的 onLaunch 中调用，或在各页面 onShow 中调用
 */
export function checkAuth() {
  const token = uni.getStorageSync('token')
  const currentPages = getCurrentPages()
  const currentPath = currentPages[currentPages.length - 1]?.route || ''

  // 白名单页面（无需登录）
  const whiteList = ['pages/login/index']

  if (!token && !whiteList.includes(currentPath)) {
    uni.reLaunch({ url: '/pages/login/index' })
    return false
  }
  return true
}
```

在 `App.vue` 中使用：

```javascript
// frontend/src/App.vue
import { checkAuth } from '@/utils/auth'

export default {
  onLaunch() {
    console.log('App Launch')
  },
  onShow() {
    checkAuth()
  }
}
```

---

### Step 10：前端管理页面

#### 10.1 新增页面清单

| # | 页面路径 | 功能 | 条件编译 |
|---|---------|------|---------|
| 1 | `pages/work-type/index` | 工种列表 + 新增/编辑弹窗 | `#ifdef H5` |
| 2 | `pages/project/index` | 项目列表 | `#ifdef H5` |
| 3 | `pages/project/detail` | 项目详情/编辑 | `#ifdef H5` |
| 4 | `pages/team/index` | 班组列表 | `#ifdef H5` |
| 5 | `pages/team/detail` | 班组详情（含成员管理） | `#ifdef H5` |
| 6 | `pages/worker/index` | 工人列表（分页 + 搜索） | `#ifdef H5` |
| 7 | `pages/worker/detail` | 工人详情/编辑 | — |
| 8 | `pages/worker/add` | 工人新增表单 | `#ifdef H5` |

#### 10.2 更新 pages.json

```json
{
  "pages": [
    { "path": "pages/home/index", "style": { "navigationBarTitleText": "首页" } },
    { "path": "pages/login/index", "style": { "navigationBarTitleText": "登录" } },
    { "path": "pages/profile/index", "style": { "navigationBarTitleText": "我的" } },
    { "path": "pages/work-type/index", "style": { "navigationBarTitleText": "工种管理" } },
    { "path": "pages/project/index", "style": { "navigationBarTitleText": "项目管理" } },
    { "path": "pages/project/detail", "style": { "navigationBarTitleText": "项目详情" } },
    { "path": "pages/team/index", "style": { "navigationBarTitleText": "班组管理" } },
    { "path": "pages/team/detail", "style": { "navigationBarTitleText": "班组详情" } },
    { "path": "pages/worker/index", "style": { "navigationBarTitleText": "工人管理" } },
    { "path": "pages/worker/detail", "style": { "navigationBarTitleText": "工人详情" } },
    { "path": "pages/worker/add", "style": { "navigationBarTitleText": "添加工人" } }
  ],
  "tabBar": {
    "color": "#999999",
    "selectedColor": "#1890FF",
    "backgroundColor": "#FFFFFF",
    "borderStyle": "black",
    "list": [
      { "pagePath": "pages/home/index", "text": "首页" },
      { "pagePath": "pages/profile/index", "text": "我的" }
    ]
  }
}
```

> **注意**：管理页面（工种/项目/班组/工人）不放入 tabBar，通过首页菜单跳转。

#### 10.3 首页改造 — 添加功能入口卡片

```
┌──────────────────────────────────────┐
│  农民工管理系统                        │
│  欢迎回来，管理员                      │
├──────────────────────────────────────┤
│                                      │
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │ 👷     │  │ 🏗️    │  │ 👥     │ │
│  │ 工人管理 │  │ 项目管理│  │ 班组管理│ │
│  └────────┘  └────────┘  └────────┘ │
│                                      │
│  ┌────────┐  ┌────────┐             │
│  │ 🔧     │  │ 📋     │             │
│  │ 工种管理 │  │ 考勤管理│ (Phase 2)   │
│  └────────┘  └────────┘             │
│                                      │
│  ── 快捷统计 ──                       │
│  在岗工人: 0    活跃项目: 0            │
│  班组数量: 0    工种数量: 0            │
│                                      │
└──────────────────────────────────────┘
```

每个卡片点击后通过 `uni.navigateTo({ url: '/pages/xxx/index' })` 跳转。

#### 10.4 API 模块文件

```javascript
// frontend/src/api/modules/workType.js
import { get, post, put, del } from '../request'

export const listWorkTypes = () => get('/work-types')
export const getWorkType = (id) => get(`/work-types/${id}`)
export const createWorkType = (data) => post('/work-types', data)
export const updateWorkType = (id, data) => put(`/work-types/${id}`, data)
export const deleteWorkType = (id) => del(`/work-types/${id}`)
```

```javascript
// frontend/src/api/modules/project.js
import { get, post, put, del } from '../request'

export const listProjects = (params) => get('/projects', params)
export const getProject = (id) => get(`/projects/${id}`)
export const createProject = (data) => post('/projects', data)
export const updateProject = (id, data) => put(`/projects/${id}`, data)
export const deleteProject = (id) => del(`/projects/${id}`)
export const getProjectWorkers = (id) => get(`/projects/${id}/workers`)
export const getProjectTeams = (id) => get(`/projects/${id}/teams`)
```

```javascript
// frontend/src/api/modules/team.js
import { get, post, put, del } from '../request'

export const listTeams = (params) => get('/teams', params)
export const getTeam = (id) => get(`/teams/${id}`)
export const createTeam = (data) => post('/teams', data)
export const updateTeam = (id, data) => put(`/teams/${id}`, data)
export const deleteTeam = (id) => del(`/teams/${id}`)
export const getTeamMembers = (id) => get(`/teams/${id}/members`)
export const addTeamMember = (id, data) => post(`/teams/${id}/members`, data)
export const removeTeamMember = (teamId, workerId) => del(`/teams/${teamId}/members/${workerId}`)
```

```javascript
// frontend/src/api/modules/worker.js
import { get, post, put, del } from '../request'

export const listWorkers = (params) => get('/workers', params)
export const getWorker = (id) => get(`/workers/${id}`)
export const createWorker = (data) => post('/workers', data)
export const updateWorker = (id, data) => put(`/workers/${id}`, data)
export const deleteWorker = (id) => del(`/workers/${id}`)
export const updateWorkerWage = (id, data) => put(`/workers/${id}/wage`, data)
```

#### 10.5 前端页面设计规范

遵循项目计划书中的**适老化 UI 设计原则**（尽管 Phase 1 的管理员用户年龄层可能较广，但保持全局一致性）：

| 规则 | 具体值 |
|------|-------|
| 基础字号 | ≥ 16px（参照 `uni.scss` 已配置） |
| 按钮最小高度 | 48px，重要操作 ≥ 56px |
| 列表行高 | ≥ 52px |
| 对比度 | 前景/背景 ≥ 4.5:1 |
| 主色调 | `#1890FF`（蓝色）— 已在 tabBar 中使用 |
| 成功色 | `#52C41A` |
| 警告色 | `#FAAD14` |
| 错误色 | `#FF4D4F` |

**关键交互原则：**
- 列表页使用**下拉刷新 + 上拉加载更多**（`onPullDownRefresh` + `onReachBottom`）
- 删除操作使用 `uni.showModal` 二次确认
- 表单提交后使用 `uni.showToast` 反馈
- 使用 `uni-icons` 或 Iconify 图标辅助文字说明
- 输入框使用 `uni-easyinput` 或原生 `input`，确保大触控区域

#### 10.6 工人列表页面示例结构

```vue
<!-- pages/worker/index.vue -->
<template>
  <view class="page">
    <!-- 搜索栏 -->
    <view class="search-bar">
      <uni-search-bar
        v-model="keyword"
        placeholder="搜索姓名/手机号"
        @confirm="onSearch"
        @cancel="onSearchCancel"
      />
    </view>

    <!-- 筛选条件 -->
    <view class="filter-bar">
      <picker :range="projectOptions" range-key="name" @change="onProjectChange">
        <text>{{ selectedProject?.name || '全部项目' }}</text>
      </picker>
      <picker :range="workTypeOptions" range-key="name" @change="onWorkTypeChange">
        <text>{{ selectedWorkType?.name || '全部工种' }}</text>
      </picker>
    </view>

    <!-- 工人列表 -->
    <view class="worker-list">
      <view
        v-for="worker in workers"
        :key="worker.id"
        class="worker-card"
        @click="goDetail(worker.id)"
      >
        <view class="worker-info">
          <text class="worker-name">{{ worker.name }}</text>
          <text class="worker-type">{{ worker.workTypeName }}</text>
        </view>
        <view class="worker-meta">
          <text>日薪: ¥{{ worker.dailyWage }}</text>
          <text>{{ worker.currentProjectName || '未分配' }}</text>
        </view>
        <view class="worker-status" :class="worker.status.toLowerCase()">
          {{ statusText(worker.status) }}
        </view>
      </view>
    </view>

    <!-- 无数据提示 -->
    <view v-if="workers.length === 0 && !loading" class="empty">
      <text>暂无工人数据</text>
    </view>

    <!-- 新增按钮 (FAB) -->
    <view class="fab" @click="goAdd">
      <text class="fab-icon">+</text>
    </view>
  </view>
</template>
```

---

### Step 11：Profile 页面改造

将 Profile 页面从静态占位改为展示真实用户信息：

```
┌──────────────────────────────────────┐
│  我的                                │
├──────────────────────────────────────┤
│                                      │
│  ┌──────────────────────────────┐    │
│  │  👤 超级管理员                │    │
│  │  用户名: admin              │    │
│  │  角色: 超级管理员            │    │
│  └──────────────────────────────┘    │
│                                      │
│  ┌──────────────────────────────┐    │
│  │  📋 个人信息                  │ >  │
│  ├──────────────────────────────┤    │
│  │  🔒 修改密码                  │ >  │
│  ├──────────────────────────────┤    │
│  │  ℹ️ 关于系统                  │ >  │
│  └──────────────────────────────┘    │
│                                      │
│  ┌──────────────────────────────┐    │
│  │         退 出 登 录           │    │
│  └──────────────────────────────┘    │
│                                      │
└──────────────────────────────────────┘
```

- 用户信息从 `useUserStore().userInfo` 读取
- 退出登录调用 `useUserStore().logout()`
- "修改密码" 功能可在 Phase 1 做一个简单页面或标记为 TODO

---

## 四、数据库变更

### Phase 1 不需要新增数据库迁移

所有表已在 `V1__create_tables.sql` 中创建完毕，种子数据已在 `V2__seed_data.sql` 中。

### 可选：追加更多种子数据 V3__more_seed_data.sql

如果需要开发测试数据，可以增加 `V3__add_test_data.sql`：

```sql
-- V3: 测试数据（仅在开发环境使用）

-- 项目经理用户（密码: manager123）
INSERT INTO ems_sys_user (username, password, real_name, phone, role, status)
VALUES ('manager', '$2a$10$...BCrypt哈希...', '项目经理王五', '13800000001', 'PROJECT_MANAGER', 1);

-- 工长用户（密码: foreman123）
INSERT INTO ems_sys_user (username, password, real_name, phone, role, status)
VALUES ('foreman', '$2a$10$...BCrypt哈希...', '工长赵六', '13800000002', 'FOREMAN', 1);

-- 测试项目
INSERT INTO ems_project (name, district, location, general_contractor, subcontractor, project_manager_name, start_date, status)
VALUES
('XX花园小区项目', '江阳区', '泸州市江阳区XXX路', '泸州建工集团', '泸州XX劳务公司', '王五', '2026-01-01', 'ACTIVE'),
('YY商业广场项目', '龙马潭区', '泸州市龙马潭区YYY路', '四川建工集团', '泸州YY劳务公司', '王五', '2026-02-01', 'ACTIVE');

-- 测试班组
INSERT INTO ems_team (name, project_id, description, status)
VALUES
('钢筋班', 1, '钢筋绑扎施工班组', 1),
('木工班', 1, '木模板施工班组', 1),
('泥瓦班', 2, '砌墙抹灰施工班组', 1);

-- 测试工人
INSERT INTO ems_worker (name, phone, gender, work_type_id, daily_wage, current_project_id, current_team_id, status, join_date)
VALUES
('张三', '13900000001', 1, 2, 340.00, 1, 1, 'ACTIVE', '2026-01-15'),
('李四', '13900000002', 1, 1, 350.00, 1, 2, 'ACTIVE', '2026-01-15'),
('王五', '13900000003', 1, 3, 320.00, 2, 3, 'ACTIVE', '2026-02-01'),
('赵六', '13900000004', 2, 8, 300.00, 1, 2, 'ACTIVE', '2026-01-20'),
('钱七', '13900000005', 1, 6, 380.00, 2, 3, 'ACTIVE', '2026-02-05');
```

> 注意：由于 `application-dev.yml` 中 `flyway.enabled: false`，需要手动执行这些SQL，或临时启用 Flyway。建议在开发阶段直接使用 Navicat/DBeaver 手动导入。

---

## 五、测试要求

### 5.1 后端测试

| # | 测试项 | 方法 | 预期结果 |
|---|-------|------|---------|
| 1 | 登录成功 | POST `/api/auth/login` `{"username":"admin","password":"admin123"}` | 200, 返回 token |
| 2 | 登录失败-密码错误 | POST `/api/auth/login` `{"username":"admin","password":"wrong"}` | 401, "用户名或密码错误" |
| 3 | 登录失败-用户不存在 | POST `/api/auth/login` `{"username":"nobody","password":"123"}` | 401, "用户名或密码错误" |
| 4 | 未登录访问受保护接口 | GET `/api/workers` (无 Authorization 头) | 401 或 403 |
| 5 | 使用 Token 访问 | GET `/api/workers` (带 Bearer Token) | 200 |
| 6 | Token 过期 | 使用过期 Token 访问 | 401 |
| 7 | 工种 CRUD | POST/GET/PUT/DELETE `/api/work-types` | 正常响应 |
| 8 | 项目 CRUD | POST/GET/PUT/DELETE `/api/projects` | 正常响应 |
| 9 | 班组 CRUD | POST/GET/PUT/DELETE `/api/teams` | 正常响应 |
| 10 | 工人 CRUD | POST/GET/PUT/DELETE `/api/workers` | 正常响应，列表支持分页搜索 |
| 11 | 工种名称唯一 | POST `/api/work-types` 重复名称 | 400, "工种名称已存在" |
| 12 | 工人手机号校验 | POST `/api/workers` 手机号格式错误 | 400, 校验错误消息 |
| 13 | 权限控制 | FOREMAN 角色尝试删除项目 | 403 |

可使用 **Swagger UI** (`http://localhost:8080/swagger-ui.html`) 或 **Postman** 测试。

### 5.2 前端测试

| # | 测试项 | 操作 | 预期结果 |
|---|-------|------|---------|
| 1 | 登录跳转 | 输入 admin/admin123 点击登录 | Toast "登录成功"，跳转首页 |
| 2 | 登录失败 | 输入错误密码 | Toast "用户名或密码错误" |
| 3 | 未登录拦截 | 直接访问首页 | 自动跳转登录页 |
| 4 | 退出登录 | 点击"退出登录" | 跳转登录页，Token 清除 |
| 5 | 工种管理 | 进入工种管理，新增/编辑/删除 | 操作成功，列表刷新 |
| 6 | 项目管理 | 创建项目 → 查看详情 → 编辑 | 数据正确保存和展示 |
| 7 | 工人管理 | 搜索工人 → 查看详情 → 编辑日薪 | 搜索结果正确，日薪更新成功 |
| 8 | 班组管理 | 创建班组 → 添加成员 → 移除成员 | 成员正确关联 |

---

## 六、关键决策与注意事项

### 6.1 响应格式统一

| 维度 | 决策 |
|------|------|
| 统一使用 `R<T>` | `{ code: 200, message: "success", data: T }` |
| 废弃 OpenAPI 生成模型 | 不再使用 `LoginResponse`、`EmployeeResponse` 等生成类 |
| Controller 手写 | 不再 implements `DefaultApi`，自行编写路由和注解 |
| Swagger 文档 | 通过 `@Tag`、`@Operation`、`@Schema` 注解自动生成 |

### 6.2 命名约定

| 维度 | 约定 |
|------|------|
| 实体名 → 表名 | `SysUser` → `ems_sys_user` (通过 `@Table` 注解) |
| URL 路径 | 复数名词 kebab-case: `/api/work-types`, `/api/workers` |
| Java 包命名 | `controller/service/repository/entity/dto/vo` |
| DTO 后缀 | `XxxDTO` — 请求入参 |
| VO 后缀 | `XxxVO` — 响应出参 |
| Service 接口+实现 | `XxxService` + `XxxServiceImpl` |

### 6.3 EmployeeController 处理

Phase 0 的 `EmployeeController.java`（实现 `DefaultApi`）在 Phase 1 中应该 **删除**。原因：
1. 它使用的是 OpenAPI 生成的 Response 模型（`LoginResponse`、`EmployeeResponse`），与 `R<T>` 格式不一致
2. 业务模型名称是 `Worker`（工人），不是 `Employee`（员工）
3. 所有方法都是 stub，无实际逻辑

替代方案：新建 `AuthController`（处理登录）+ `WorkerController`（处理工人 CRUD），使用 `R<T>` 统一响应。

### 6.4 OpenAPI 代码生成器处理

Phase 1 有两个选择：

**选项 A（推荐）：保留代码生成器但不使用生成的接口**
- `pom.xml` 中保留 `openapi-generator-maven-plugin`
- 生成的代码仍在 `target/generated-sources/` 中，但 Controller 不再 implements 生成的接口
- `swagger-input.yml` 作为 API 文档参考，可更新也可不更新

**选项 B：完全移除代码生成器**
- 从 `pom.xml` 中移除 `openapi-generator-maven-plugin` 和 `build-helper-maven-plugin`
- 删除 `swagger-input.yml` 或保留为纯文档
- 完全依赖 SpringDoc 注解自动生成 Swagger 文档

> 建议选择 **选项 A**，减少 pom.xml 改动风险。生成的代码在 target/ 中不影响项目。

### 6.5 Gender 字段映射

数据库中 `gender` 是 `TINYINT (1=男, 2=女)`，不是 ENUM。需要：
- Entity 中使用 `Integer gender`（简单直接）
- 或定义 `Gender` 枚举 + `AttributeConverter`（更规范）

建议 Phase 1 先用 `Integer gender`，保持简单。

### 6.6 前端 request.js 适配

当前 `request.js` 在 `res.data.code !== undefined` 时按 `R<T>` 处理，否则直接返回。这个逻辑在统一使用 `R<T>` 后无需修改，但需要注意：

- 后端所有接口返回 `R<T>` 格式
- 当后端返回 `401` HTTP 状态码时，`res.statusCode` 不是 200，需要在 `request.js` 的非 200 分支中处理

建议更新 `request.js` 增加 401 状态码处理：

```javascript
// 在 success 回调中
if (res.statusCode === 401) {
    uni.removeStorageSync('token')
    uni.reLaunch({ url: '/pages/login/index' })
    reject({ code: 401, message: '登录已过期' })
    return
}
if (res.statusCode === 403) {
    uni.showToast({ title: '无操作权限', icon: 'none' })
    reject({ code: 403, message: '无操作权限' })
    return
}
```

---

## 七、Phase 1 完成的验收标准

### ✅ 后端验收

- [ ] 使用 admin/admin123 登录返回有效 JWT Token
- [ ] Token 可解析出 userId、username、role
- [ ] 无 Token 访问受保护接口返回 401
- [ ] 工种 CRUD 5 个接口全部正常工作
- [ ] 项目 CRUD 7 个接口全部正常工作 (含关联查询)
- [ ] 班组 CRUD 8 个接口全部正常工作 (含成员管理)
- [ ] 工人 CRUD 7 个接口全部正常工作 (含日薪修改)
- [ ] 工人列表支持按姓名/手机号搜索
- [ ] 工人列表支持按项目/班组/工种/状态过滤
- [ ] 工人列表支持分页
- [ ] Swagger UI (`/swagger-ui.html`) 可正常访问并展示所有接口
- [ ] `@PreAuthorize` 角色权限生效

### ✅ 前端验收

- [ ] 登录页可输入用户名密码并成功登录
- [ ] 登录后 Token 和用户信息保存到本地存储
- [ ] 首页展示功能入口卡片
- [ ] 点击卡片可跳转到对应管理页面
- [ ] 工种管理页面：列表展示、新增、编辑、删除
- [ ] 项目管理页面：列表展示、新增、编辑、详情
- [ ] 班组管理页面：列表展示、新增、编辑、成员管理
- [ ] 工人管理页面：列表搜索、新增表单、详情查看、编辑
- [ ] "我的"页面展示真实用户信息
- [ ] 退出登录后跳转登录页
- [ ] 未登录时自动跳转登录页

---

## 八、与 Phase 2 的衔接

Phase 1 完成后，以下内容为 Phase 2（工天管理）做好准备：

| 准备项 | 状态 |
|--------|------|
| `ems_attendance` 表 | ✅ 已创建 |
| Worker、Project、Team 实体 | ✅ Phase 1 创建 |
| 工人日薪字段 | ✅ Worker.dailyWage 已就绪 |
| JWT 认证体系 | ✅ Phase 1 完成 |
| 角色权限控制 | ✅ Phase 1 完成 |
| Attendance Entity | ❌ Phase 2 创建 |
| 考勤 CRUD 接口 | ❌ Phase 2 实现 |
| 适老化考勤录入页 | ❌ Phase 2 实现 |

---

> **执行提示**：建议按照 Step 1 → Step 13 的顺序逐步实施，每完成一个 Step 即运行后端 / 前端确认无编译报错。尤其是 Step 2（Entity）和 Step 5（SecurityConfig）改动后，务必 `mvn clean compile` 验证 JPA 实体与数据库表的字段映射正确。
