# Phase 3：薪资管理 — 详细实施指南

> 编制日期：2026-02-19  
> 前置条件：Phase 2 已完成（考勤 CRUD + 批量录入 + 考勤审核就绪）  
> 周期：第 6–7 周（约 10 个工作日）  
> 目标：实现完整的薪资管理体系，包括薪资自动生成 + 调整项管理 + 审批流程 + 发放标记 + 适老化前端页面

---

## 一、Phase 3 总览

### 1.1 交付目标

| # | 交付物 | 说明 |
|---|--------|------|
| 1 | OpenAPI 契约 (swagger-input.yml) | 薪资模块 15 个 API 端点定义 + 20+ Schema |
| 2 | MonthlySalary Entity + SalaryAdjustment Entity | 月度薪资 + 薪资调整实体类 |
| 3 | SalaryStatus + AdjustmentType 枚举 | 薪资审批状态、调整类型枚举 |
| 4 | 薪资 Repository | 多条件分页查询 + 汇总统计 |
| 5 | 薪资自动生成接口 | 根据项目+月份已确认考勤自动核算薪资 |
| 6 | 薪资 CRUD 接口 | 查询、详情、更新、删除 |
| 7 | 薪资调整项管理接口 | 添加/删除奖金、扣款、预支、代缴等调整项 |
| 8 | 薪资审批流程接口 | 提交、批量提交、审批、批量审批、驳回 |
| 9 | 薪资发放接口 | 标记已发放、批量标记已发放 |
| 10 | 薪资月度汇总统计接口 | 按项目+年月汇总薪资数据 |
| 11 | SalaryController (契约优先) | 实现 OpenAPI Generator 生成的 SalaryApi 接口 |
| 12 | 前端薪资列表页 | 筛选、汇总卡片、分页列表 |
| 13 | 前端薪资详情页 | 薪资明细、调整项管理、审批操作 |
| 14 | 前端薪资生成页 | 选择项目+月份，一键批量生成薪资单 |
| 15 | 首页薪资管理入口 | 在首页菜单添加薪资管理卡片 |

### 1.2 不在本阶段范围

- ❌ 工资条 PDF/Excel 导出（Phase 4 或后续迭代）
- ❌ 多级审批签名链路（Phase 3 仅实现单级审批流转，签名字段已预留）
- ❌ 工人端确认工资（Phase 5 小程序端）
- ❌ 银行代发接口对接（后续迭代）
- ❌ 合同管理（推迟到 Phase 4）
- ❌ 工人调动功能（推迟到 Phase 4）

### 1.3 开发顺序（建议严格遵循）

```
Week 1 (第 6 周):
  Step 1:  OpenAPI 契约 — swagger-input.yml 新增 salary tag + 15 个端点 + 20+ Schema
  Step 2:  薪资枚举类 — SalaryStatus、AdjustmentType
  Step 3:  MonthlySalary + SalaryAdjustment Entity — 实体类
  Step 4:  Repository — MonthlySalaryRepository + SalaryAdjustmentRepository
  Step 5:  DTO / VO — 请求/响应对象定义
  Step 6:  SalaryService — 薪资业务逻辑（生成 + CRUD + 审批流程 + 汇总）
  Step 7:  mvn generate-sources — 生成 SalaryApi 接口
  Step 8:  SalaryController — 契约优先 implements SalaryApi
  Step 9:  后端编译验证 — mvn compile

Week 2 (第 7 周):
  Step 10: 前端 API 模块 — salary.js 接口封装
  Step 11: 薪资列表页面 — 筛选/汇总/分页列表
  Step 12: 薪资详情页面 — 薪资明细/调整项管理/审批操作
  Step 13: 薪资生成页面 — 选择项目+月份批量生成
  Step 14: pages.json 路由更新 + 首页入口
```

### 1.4 关键技术决策：契约优先（Contract-First）

Phase 3 采用 **完整契约优先** 开发模式，与 Phase 1/2 手写 Controller 的方式不同：

1. **先在 `swagger-input.yml` 中定义全部 API 路径和 Schema**
2. **运行 `mvn generate-sources`** — OpenAPI Generator Maven Plugin 自动生成 `SalaryApi` 接口
3. **`SalaryController implements SalaryApi`** — Controller 只做业务委派，路由注解由生成接口提供

**为什么选择契约优先？**
- API 契约即文档，所有端点的请求参数、响应格式一目了然
- 编译期强保证：Controller 必须实现接口中全部方法，缺一不可
- 前后端可并行开发：前端根据 YAML 即可开始编码，无需等待后端完成

**raw-type 桥接模式：** 由于 OpenAPI 生成的接口返回 `ResponseEntity<XxxResponse>`，而实际使用 `R<T>` 封装，采用 raw-type 强转：

```java
@SuppressWarnings({"unchecked", "rawtypes"})
public ResponseEntity<SalaryGenerateResponse> generateSalaries(SalaryGenerateRequest req) {
    return (ResponseEntity) ResponseEntity.ok(R.ok(salaryService.generate(dto)));
}
```

运行时泛型擦除，Jackson 序列化的是实际 `R<T>` 对象，JSON 输出与 YAML Schema 定义的 `{code, message, data}` 结构一致。

---

## 二、OpenAPI 契约定义

### Step 1：swagger-input.yml 薪资模块

在 `backend/src/main/resources/openapi/swagger-input.yml` 中新增以下内容：

#### 1.1 新增 Tag

在 `tags` 节点中添加：

```yaml
  - name: salary
    description: 薪资管理
```

> **命名约定**：使用英文 tag `salary` 而非中文 `薪资管理`，以生成干净的 `SalaryApi.java` 类名。

#### 1.2 新增 15 个 API 端点

```yaml
  # ===================== 薪资管理 =====================

  /api/salaries/generate:
    post:
      tags: [salary]
      summary: 批量生成薪资单
      description: 根据项目+年月已确认考勤，自动为所有工人生成月度薪资单
      operationId: generateSalaries
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SalaryGenerateRequest'
      responses:
        '200':
          description: 生成成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryGenerateResponse'

  /api/salaries:
    get:
      tags: [salary]
      summary: 薪资列表
      description: 分页查询薪资记录，支持多条件筛选
      operationId: listSalaries
      security:
        - bearerAuth: []
      parameters:
        - name: projectId
          in: query
          required: false
          schema:
            type: integer
            format: int64
        - name: workerId
          in: query
          required: false
          schema:
            type: integer
            format: int64
        - name: year
          in: query
          required: false
          schema:
            type: integer
        - name: month
          in: query
          required: false
          schema:
            type: integer
        - name: status
          in: query
          required: false
          schema:
            type: string
            enum: [DRAFT, PENDING_APPROVAL, APPROVED, PAID]
        - name: page
          in: query
          required: false
          schema:
            type: integer
            default: 0
        - name: size
          in: query
          required: false
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: 查询成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryPageResponse'

  /api/salaries/{id}:
    get:
      tags: [salary]
      summary: 薪资详情
      description: 获取薪资单详情，包含调整明细
      operationId: getSalaryById
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: 查询成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryDetailResponse'

    put:
      tags: [salary]
      summary: 更新薪资单
      description: 修改薪资单信息（仅DRAFT状态可修改）
      operationId: updateSalary
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SalaryUpdateRequest'
      responses:
        '200':
          description: 更新成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryDetailResponse'

    delete:
      tags: [salary]
      summary: 删除薪资单
      description: 删除薪资单（仅DRAFT状态可删除）
      operationId: deleteSalary
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: 删除成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/{id}/adjustments:
    post:
      tags: [salary]
      summary: 添加薪资调整项
      description: 为薪资单添加奖金/扣款/预支等调整项
      operationId: addAdjustment
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SalaryAdjustmentRequest'
      responses:
        '200':
          description: 添加成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryDetailResponse'

  /api/salaries/{id}/adjustments/{adjId}:
    delete:
      tags: [salary]
      summary: 删除薪资调整项
      operationId: removeAdjustment
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
        - name: adjId
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: 删除成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryDetailResponse'

  /api/salaries/{id}/submit:
    post:
      tags: [salary]
      summary: 提交审批
      description: 将薪资单从DRAFT提交为PENDING_APPROVAL
      operationId: submitSalary
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: 提交成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/batch-submit:
    post:
      tags: [salary]
      summary: 批量提交审批
      operationId: batchSubmitSalaries
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: array
              items:
                type: integer
                format: int64
      responses:
        '200':
          description: 批量提交成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/{id}/approve:
    post:
      tags: [salary]
      summary: 审批通过
      operationId: approveSalary
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: 审批通过
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/batch-approve:
    post:
      tags: [salary]
      summary: 批量审批通过
      operationId: batchApproveSalaries
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: array
              items:
                type: integer
                format: int64
      responses:
        '200':
          description: 批量审批通过
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/{id}/reject:
    post:
      tags: [salary]
      summary: 驳回薪资单
      operationId: rejectSalary
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SalaryRejectRequest'
      responses:
        '200':
          description: 驳回成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/{id}/pay:
    post:
      tags: [salary]
      summary: 标记已发放
      operationId: paySalary
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: 标记成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/batch-pay:
    post:
      tags: [salary]
      summary: 批量标记已发放
      operationId: batchPaySalaries
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: array
              items:
                type: integer
                format: int64
      responses:
        '200':
          description: 批量发放成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalaryVoidResponse'

  /api/salaries/summary:
    get:
      tags: [salary]
      summary: 薪资月度汇总
      description: 按项目+年月统计薪资汇总数据
      operationId: getSalarySummary
      security:
        - bearerAuth: []
      parameters:
        - name: projectId
          in: query
          required: true
          schema:
            type: integer
            format: int64
        - name: year
          in: query
          required: true
          schema:
            type: integer
        - name: month
          in: query
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 查询成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SalarySummaryResponse'
```

#### 1.3 新增 Schema 定义

在 `components.schemas` 中添加以下 Schema（所有响应体采用 `R<T>` 兼容格式 `{code, message, data}`）：

**请求体 Schema：**

```yaml
    SalaryGenerateRequest:
      type: object
      required: [projectId, year, month]
      properties:
        projectId:
          type: integer
          format: int64
        year:
          type: integer
        month:
          type: integer

    SalaryUpdateRequest:
      type: object
      properties:
        overtimePay:
          type: number
        paymentMethod:
          type: string
        remark:
          type: string

    SalaryAdjustmentRequest:
      type: object
      required: [type, amount]
      properties:
        type:
          type: string
          enum: [BONUS, DEDUCTION, ADVANCE, WITHHOLDING, OTHER]
        amount:
          type: number
        reason:
          type: string

    SalaryRejectRequest:
      type: object
      properties:
        reason:
          type: string
```

**响应体 Schema：**

```yaml
    SalaryVoidResponse:
      type: object
      properties:
        code:
          type: integer
        message:
          type: string
        data:
          type: object

    SalaryGenerateResponse:
      type: object
      properties:
        code:
          type: integer
        message:
          type: string
        data:
          $ref: '#/components/schemas/SalaryGenerateResult'

    SalaryPageResponse:
      type: object
      properties:
        code:
          type: integer
        message:
          type: string
        data:
          $ref: '#/components/schemas/SalaryPageData'

    SalaryDetailResponse:
      type: object
      properties:
        code:
          type: integer
        message:
          type: string
        data:
          $ref: '#/components/schemas/MonthlySalaryDetail'

    SalarySummaryResponse:
      type: object
      properties:
        code:
          type: integer
        message:
          type: string
        data:
          $ref: '#/components/schemas/SalarySummaryData'
```

**数据模型 Schema：**

```yaml
    SalaryGenerateResult:
      type: object
      properties:
        generated:
          type: integer
        skipped:
          type: integer
        total:
          type: integer

    SalaryPageData:
      type: object
      properties:
        content:
          type: array
          items:
            $ref: '#/components/schemas/MonthlySalaryItem'
        totalElements:
          type: integer
          format: int64
        totalPages:
          type: integer
        number:
          type: integer
        size:
          type: integer
        last:
          type: boolean

    MonthlySalaryItem:
      type: object
      properties:
        id:
          type: integer
          format: int64
        workerId:
          type: integer
          format: int64
        workerName:
          type: string
        workerNo:
          type: string
        projectId:
          type: integer
          format: int64
        projectName:
          type: string
        year:
          type: integer
        month:
          type: integer
        totalWorkDays:
          type: number
        dailyWage:
          type: number
        grossSalary:
          type: number
        actualSalary:
          type: number
        status:
          type: string
        statusLabel:
          type: string
        createdAt:
          type: string
          format: date-time

    MonthlySalaryDetail:
      # 同 MonthlySalaryItem 但增加更多字段 + 调整项列表
      type: object
      properties:
        id:
          type: integer
          format: int64
        # ... (完整字段见 swagger-input.yml)
        adjustments:
          type: array
          items:
            $ref: '#/components/schemas/SalaryAdjustmentItem'

    SalaryAdjustmentItem:
      type: object
      properties:
        id:
          type: integer
          format: int64
        type:
          type: string
        typeLabel:
          type: string
        amount:
          type: number
        reason:
          type: string
        createdAt:
          type: string
          format: date-time

    SalarySummaryData:
      type: object
      properties:
        projectId:
          type: integer
          format: int64
        projectName:
          type: string
        year:
          type: integer
        month:
          type: integer
        totalWorkers:
          type: integer
        totalWorkDays:
          type: number
        totalGrossSalary:
          type: number
        totalActualSalary:
          type: number
        totalBonus:
          type: number
        totalDeduction:
          type: number
        draftCount:
          type: integer
        pendingCount:
          type: integer
        approvedCount:
          type: integer
        paidCount:
          type: integer
```

#### 关键注意事项

1. **响应体统一使用 `{code, message, data}` 信封格式**，与 `R<T>` 兼容
2. **Schema 命名规则**：`Xxx Response` 用于信封，`XxxData/XxxItem/XxxResult` 用于实际数据模型
3. **operationId 命名**：驼峰式，对应生成接口方法名（如 `generateSalaries` → `SalaryApi.generateSalaries()`）

---

## 三、后端实现（Spring Boot）

### Step 2：薪资枚举类

在 `com.henrywang.ems.entity.enums` 包下新增两个枚举类。

```java
// com.henrywang.ems.entity.enums.SalaryStatus
package com.henrywang.ems.entity.enums;

public enum SalaryStatus {
    DRAFT,              // 草稿
    PENDING_APPROVAL,   // 待审批
    APPROVED,           // 已审批
    PAID                // 已发放
}
```

```java
// com.henrywang.ems.entity.enums.AdjustmentType
package com.henrywang.ems.entity.enums;

public enum AdjustmentType {
    BONUS,          // 奖金
    DEDUCTION,      // 扣款
    ADVANCE,        // 预支/借支
    WITHHOLDING,    // 代缴
    OTHER           // 其他
}
```

---

### Step 3：MonthlySalary + SalaryAdjustment Entity

在 `com.henrywang.ems.entity` 包下创建两个实体，分别映射 `ems_monthly_salary` 和 `ems_salary_adjustment` 表。

#### 3.1 MonthlySalary 实体

```java
package com.henrywang.ems.entity;

import com.henrywang.ems.entity.enums.SalaryStatus;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "ems_monthly_salary",
       uniqueConstraints = @UniqueConstraint(name = "uk_worker_month",
           columnNames = {"worker_id", "project_id", "year", "month"}))
@Data
@EqualsAndHashCode(callSuper = true)
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class MonthlySalary extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "worker_id", nullable = false)
    private Worker worker;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "project_id", nullable = false)
    private Project project;

    @Column(nullable = false)
    private Integer year;

    @Column(nullable = false)
    private Integer month;

    @Column(name = "total_work_days", nullable = false, precision = 5, scale = 1)
    private BigDecimal totalWorkDays;

    @Column(name = "actual_work_output", length = 200)
    private String actualWorkOutput;

    @Column(name = "daily_wage", precision = 10, scale = 2)
    private BigDecimal dailyWage;

    @Column(name = "monthly_wage", precision = 12, scale = 2)
    private BigDecimal monthlyWage;

    @Column(name = "base_salary", nullable = false, precision = 12, scale = 2)
    private BigDecimal baseSalary;

    @Column(name = "overtime_pay", precision = 10, scale = 2)
    private BigDecimal overtimePay;

    @Column(name = "gross_salary", nullable = false, precision = 12, scale = 2)
    private BigDecimal grossSalary;

    @Column(name = "previous_unpaid", precision = 10, scale = 2)
    private BigDecimal previousUnpaid;

    @Column(precision = 10, scale = 2)
    private BigDecimal bonus;

    @Column(precision = 10, scale = 2)
    private BigDecimal withholding;

    @Column(precision = 10, scale = 2)
    private BigDecimal deduction;

    @Column(name = "advance_payment", precision = 10, scale = 2)
    private BigDecimal advancePayment;

    @Column(name = "actual_salary", nullable = false, precision = 12, scale = 2)
    private BigDecimal actualSalary;

    @Column(name = "payment_method", length = 30)
    private String paymentMethod;

    @Column(name = "worker_confirmed", columnDefinition = "TINYINT DEFAULT 0")
    private Integer workerConfirmed;

    @Column(name = "worker_confirmed_at")
    private LocalDateTime workerConfirmedAt;

    // 多级审批签名（Phase 3 预留字段，暂不使用）
    @Column(name = "team_leader_id")
    private Long teamLeaderId;

    @Column(name = "team_leader_signed_at")
    private LocalDateTime teamLeaderSignedAt;

    @Column(name = "subcontractor_signer", length = 50)
    private String subcontractorSigner;

    @Column(name = "subcontractor_signed_at")
    private LocalDateTime subcontractorSignedAt;

    @Column(name = "labor_officer_id")
    private Long laborOfficerId;

    @Column(name = "labor_officer_signed_at")
    private LocalDateTime laborOfficerSignedAt;

    @Column(name = "finance_signer_id")
    private Long financeSignerId;

    @Column(name = "finance_signed_at")
    private LocalDateTime financeSignedAt;

    @Column(name = "business_mgr_id")
    private Long businessMgrId;

    @Column(name = "business_mgr_signed_at")
    private LocalDateTime businessMgrSignedAt;

    @Column(name = "project_mgr_id")
    private Long projectMgrId;

    @Column(name = "project_mgr_signed_at")
    private LocalDateTime projectMgrSignedAt;

    @Enumerated(EnumType.STRING)
    @Column(columnDefinition = "ENUM('DRAFT','PENDING_APPROVAL','APPROVED','PAID') DEFAULT 'DRAFT'")
    private SalaryStatus status;

    @Column(name = "generated_at")
    private LocalDateTime generatedAt;

    @Column(name = "paid_at")
    private LocalDateTime paidAt;

    @Column(length = 500)
    private String remark;

    @OneToMany(mappedBy = "monthlySalary", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @Builder.Default
    @ToString.Exclude
    @EqualsAndHashCode.Exclude
    private List<SalaryAdjustment> adjustments = new ArrayList<>();

    /**
     * 根据调整项汇总重算 actual_salary
     */
    public void recalculate() {
        BigDecimal totalBonus = BigDecimal.ZERO;
        BigDecimal totalDeduction = BigDecimal.ZERO;
        BigDecimal totalAdvance = BigDecimal.ZERO;
        BigDecimal totalWithholding = BigDecimal.ZERO;

        if (adjustments != null) {
            for (SalaryAdjustment adj : adjustments) {
                switch (adj.getType()) {
                    case BONUS -> totalBonus = totalBonus.add(adj.getAmount());
                    case DEDUCTION -> totalDeduction = totalDeduction.add(adj.getAmount());
                    case ADVANCE -> totalAdvance = totalAdvance.add(adj.getAmount());
                    case WITHHOLDING -> totalWithholding = totalWithholding.add(adj.getAmount());
                    case OTHER -> totalDeduction = totalDeduction.add(adj.getAmount());
                }
            }
        }

        this.bonus = totalBonus;
        this.deduction = totalDeduction;
        this.advancePayment = totalAdvance;
        this.withholding = totalWithholding;

        BigDecimal ot = this.overtimePay != null ? this.overtimePay : BigDecimal.ZERO;
        this.grossSalary = this.baseSalary.add(ot);

        BigDecimal prev = this.previousUnpaid != null ? this.previousUnpaid : BigDecimal.ZERO;
        this.actualSalary = this.grossSalary
                .add(prev)
                .add(totalBonus)
                .subtract(totalWithholding)
                .subtract(totalDeduction)
                .subtract(totalAdvance);
    }
}
```

#### 关键注意事项

1. **唯一约束 `uk_worker_month`**：同一工人在同一项目的同一年月只能有一条薪资记录
2. **`@OneToMany(cascade = ALL, orphanRemoval = true)`**：调整项随薪资单级联保存/删除
3. **`@Builder.Default`**：确保 Builder 模式下 `adjustments` 初始化为空列表
4. **`recalculate()` 方法**：每次添加/删除调整项后调用，自动汇总并重算实发金额
5. **薪资计算公式**：`actualSalary = grossSalary + previousUnpaid + bonus - withholding - deduction - advancePayment`
6. **多级审批签名字段**：已在数据库表中预留 6 级签名（班组长 → 分包 → 劳务专管员 → 财务 → 经营部 → 项目经理），Phase 3 仅实现单级状态流转

#### 3.2 SalaryAdjustment 实体

```java
package com.henrywang.ems.entity;

import com.henrywang.ems.entity.enums.AdjustmentType;
import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "ems_salary_adjustment")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@EntityListeners(AuditingEntityListener.class)
public class SalaryAdjustment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "monthly_salary_id", nullable = false)
    @ToString.Exclude
    @EqualsAndHashCode.Exclude
    private MonthlySalary monthlySalary;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, columnDefinition = "ENUM('BONUS','DEDUCTION','ADVANCE','WITHHOLDING','OTHER')")
    private AdjustmentType type;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal amount;

    @Column(length = 200)
    private String reason;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
}
```

> **注意**：SalaryAdjustment 不继承 BaseEntity，使用独立的 `@Id` + `@CreatedDate`。

---

### Step 4：Repository 层

#### 4.1 MonthlySalaryRepository

```java
package com.henrywang.ems.repository;

import com.henrywang.ems.entity.MonthlySalary;
import com.henrywang.ems.entity.enums.SalaryStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

public interface MonthlySalaryRepository extends JpaRepository<MonthlySalary, Long> {

    /**
     * 检查某工人某项目某月是否已有薪资单
     */
    boolean existsByWorkerIdAndProjectIdAndYearAndMonth(
            Long workerId, Long projectId, Integer year, Integer month);

    /**
     * 详情查询（含所有关联对象）
     */
    @Query("SELECT s FROM MonthlySalary s " +
           "LEFT JOIN FETCH s.worker w " +
           "LEFT JOIN FETCH w.workType " +
           "LEFT JOIN FETCH s.project " +
           "LEFT JOIN FETCH s.adjustments " +
           "WHERE s.id = :id")
    Optional<MonthlySalary> findDetailById(@Param("id") Long id);

    /**
     * 多条件分页查询
     */
    @Query("SELECT s FROM MonthlySalary s " +
           "LEFT JOIN FETCH s.worker w " +
           "LEFT JOIN FETCH w.workType " +
           "LEFT JOIN FETCH s.project " +
           "WHERE (:projectId IS NULL OR s.project.id = :projectId) " +
           "AND (:workerId IS NULL OR s.worker.id = :workerId) " +
           "AND (:year IS NULL OR s.year = :year) " +
           "AND (:month IS NULL OR s.month = :month) " +
           "AND (:status IS NULL OR s.status = :status)")
    Page<MonthlySalary> findByFilters(
            @Param("projectId") Long projectId,
            @Param("workerId") Long workerId,
            @Param("year") Integer year,
            @Param("month") Integer month,
            @Param("status") SalaryStatus status,
            Pageable pageable);

    /**
     * 按项目+年月统计各状态数量
     */
    @Query("SELECT s.status, COUNT(s) FROM MonthlySalary s " +
           "WHERE s.project.id = :projectId AND s.year = :year AND s.month = :month " +
           "GROUP BY s.status")
    List<Object[]> countByStatusGrouped(
            @Param("projectId") Long projectId,
            @Param("year") Integer year,
            @Param("month") Integer month);

    /**
     * 按项目+年月汇总薪资数据
     */
    @Query("SELECT COUNT(s), SUM(s.totalWorkDays), SUM(s.grossSalary), SUM(s.actualSalary), " +
           "SUM(s.bonus), SUM(s.deduction) FROM MonthlySalary s " +
           "WHERE s.project.id = :projectId AND s.year = :year AND s.month = :month")
    Object[] sumByProjectAndMonth(
            @Param("projectId") Long projectId,
            @Param("year") Integer year,
            @Param("month") Integer month);
}
```

#### 4.2 SalaryAdjustmentRepository

```java
package com.henrywang.ems.repository;

import com.henrywang.ems.entity.SalaryAdjustment;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface SalaryAdjustmentRepository extends JpaRepository<SalaryAdjustment, Long> {

    List<SalaryAdjustment> findByMonthlySalaryId(Long monthlySalaryId);

    void deleteByMonthlySalaryId(Long monthlySalaryId);
}
```

---

### Step 5：DTO / VO

#### 5.1 请求 DTO

```java
// SalaryGenerateDTO.java
package com.henrywang.ems.dto;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class SalaryGenerateDTO {

    @NotNull(message = "项目ID不能为空")
    private Long projectId;

    @NotNull(message = "年份不能为空")
    @Min(value = 2020, message = "年份不合法")
    private Integer year;

    @NotNull(message = "月份不能为空")
    @Min(value = 1, message = "月份范围1-12")
    @Max(value = 12, message = "月份范围1-12")
    private Integer month;
}
```

```java
// SalaryUpdateDTO.java
package com.henrywang.ems.dto;

import lombok.Data;
import java.math.BigDecimal;

@Data
public class SalaryUpdateDTO {
    private BigDecimal overtimePay;
    private String paymentMethod;
    private String remark;
}
```

```java
// SalaryAdjustmentDTO.java
package com.henrywang.ems.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;
import java.math.BigDecimal;

@Data
public class SalaryAdjustmentDTO {

    @NotBlank(message = "调整类型不能为空")
    private String type;

    @NotNull(message = "金额不能为空")
    @DecimalMin(value = "0.01", message = "金额必须大于0")
    private BigDecimal amount;

    private String reason;
}
```

#### 5.2 响应 VO

```java
// MonthlySalaryVO.java — 列表项
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@Builder
public class MonthlySalaryVO {
    private Long id;
    private Long workerId;
    private String workerName;
    private String workerNo;
    private Long projectId;
    private String projectName;
    private Integer year;
    private Integer month;
    private BigDecimal totalWorkDays;
    private BigDecimal dailyWage;
    private BigDecimal monthlyWage;
    private BigDecimal baseSalary;
    private BigDecimal overtimePay;
    private BigDecimal grossSalary;
    private BigDecimal bonus;
    private BigDecimal deduction;
    private BigDecimal advancePayment;
    private BigDecimal withholding;
    private BigDecimal previousUnpaid;
    private BigDecimal actualSalary;
    private String paymentMethod;
    private String status;
    private String statusLabel;
    private LocalDateTime createdAt;
}
```

```java
// MonthlySalaryDetailVO.java — 详情（含调整项列表）
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
public class MonthlySalaryDetailVO {
    private Long id;
    private Long workerId;
    private String workerName;
    private String workerNo;
    private String workerPhone;
    private String workTypeName;
    private String payType;
    private Long projectId;
    private String projectName;
    private Integer year;
    private Integer month;
    private BigDecimal totalWorkDays;
    private BigDecimal dailyWage;
    private BigDecimal monthlyWage;
    private BigDecimal baseSalary;
    private BigDecimal overtimePay;
    private BigDecimal grossSalary;
    private BigDecimal bonus;
    private BigDecimal deduction;
    private BigDecimal advancePayment;
    private BigDecimal withholding;
    private BigDecimal previousUnpaid;
    private BigDecimal actualSalary;
    private String paymentMethod;
    private String bankAccount;
    private String bankName;
    private Boolean workerConfirmed;
    private String status;
    private String statusLabel;
    private String remark;
    private List<SalaryAdjustmentVO> adjustments;
    private LocalDateTime generatedAt;
    private LocalDateTime paidAt;
    private LocalDateTime createdAt;
}
```

```java
// SalaryAdjustmentVO.java
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@Builder
public class SalaryAdjustmentVO {
    private Long id;
    private String type;
    private String typeLabel;
    private BigDecimal amount;
    private String reason;
    private LocalDateTime createdAt;
}
```

```java
// SalarySummaryVO.java — 月度汇总统计
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;

@Data
@Builder
public class SalarySummaryVO {
    private Long projectId;
    private String projectName;
    private Integer year;
    private Integer month;
    private Integer totalWorkers;
    private BigDecimal totalWorkDays;
    private BigDecimal totalGrossSalary;
    private BigDecimal totalActualSalary;
    private BigDecimal totalBonus;
    private BigDecimal totalDeduction;
    private Integer draftCount;
    private Integer pendingCount;
    private Integer approvedCount;
    private Integer paidCount;
}
```

```java
// SalaryGenerateResultVO.java — 生成结果
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class SalaryGenerateResultVO {
    private int generated;
    private int skipped;
    private int total;
}
```

---

### Step 6：SalaryService

在 `com.henrywang.ems.service` 包下创建 Service 接口和实现类。

#### 6.1 Service 接口

```java
package com.henrywang.ems.service;

import com.henrywang.ems.dto.*;
import com.henrywang.ems.vo.*;

import java.util.List;

public interface SalaryService {

    /** 批量生成薪资单（根据项目+年月，从已确认考勤自动计算） */
    SalaryGenerateResultVO generate(SalaryGenerateDTO dto);

    /** 分页查询薪资列表 */
    PageResponse<MonthlySalaryVO> list(Long projectId, Long workerId,
                                        Integer year, Integer month,
                                        String status, int page, int size);

    /** 薪资详情（含调整明细） */
    MonthlySalaryDetailVO getById(Long id);

    /** 更新薪资单 */
    MonthlySalaryDetailVO update(Long id, SalaryUpdateDTO dto);

    /** 删除薪资单（仅DRAFT状态） */
    void delete(Long id);

    /** 添加调整项 */
    MonthlySalaryDetailVO addAdjustment(Long salaryId, SalaryAdjustmentDTO dto);

    /** 删除调整项 */
    MonthlySalaryDetailVO removeAdjustment(Long salaryId, Long adjustmentId);

    /** 提交审批 (DRAFT → PENDING_APPROVAL) */
    void submit(Long id);

    /** 批量提交审批 */
    void batchSubmit(List<Long> ids);

    /** 审批通过 (PENDING_APPROVAL → APPROVED) */
    void approve(Long id);

    /** 批量审批通过 */
    void batchApprove(List<Long> ids);

    /** 驳回 (PENDING_APPROVAL → DRAFT) */
    void reject(Long id, String reason);

    /** 标记已发放 (APPROVED → PAID) */
    void pay(Long id);

    /** 批量标记已发放 */
    void batchPay(List<Long> ids);

    /** 薪资月度汇总统计 */
    SalarySummaryVO getSummary(Long projectId, Integer year, Integer month);
}
```

#### 6.2 Service 实现

```java
package com.henrywang.ems.service.impl;

import com.henrywang.ems.common.exception.BusinessException;
import com.henrywang.ems.common.util.PageUtils;
import com.henrywang.ems.dto.*;
import com.henrywang.ems.entity.*;
import com.henrywang.ems.entity.enums.*;
import com.henrywang.ems.repository.*;
import com.henrywang.ems.service.SalaryService;
import com.henrywang.ems.vo.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class SalaryServiceImpl implements SalaryService {

    private final MonthlySalaryRepository salaryRepository;
    private final SalaryAdjustmentRepository adjustmentRepository;
    private final WorkerRepository workerRepository;
    private final ProjectRepository projectRepository;
    private final AttendanceRepository attendanceRepository;

    // ===================== 批量生成 =====================

    @Override
    @Transactional
    public SalaryGenerateResultVO generate(SalaryGenerateDTO dto) {
        Project project = projectRepository.findById(dto.getProjectId())
                .orElseThrow(() -> new BusinessException(404, "项目不存在"));

        List<Worker> workers = workerRepository.findByCurrentProjectId(dto.getProjectId());
        if (workers.isEmpty()) {
            throw new BusinessException(400, "该项目下没有工人");
        }

        LocalDate startDate = LocalDate.of(dto.getYear(), dto.getMonth(), 1);
        LocalDate endDate = startDate.withDayOfMonth(startDate.lengthOfMonth());

        int generated = 0;
        int skipped = 0;

        for (Worker worker : workers) {
            // 跳过已存在的薪资单
            if (salaryRepository.existsByWorkerIdAndProjectIdAndYearAndMonth(
                    worker.getId(), dto.getProjectId(), dto.getYear(), dto.getMonth())) {
                skipped++;
                continue;
            }

            // 从考勤计算工天数（仅已确认的考勤）
            BigDecimal totalWorkDays = attendanceRepository.sumWorkDaysByWorkerAndMonth(
                    worker.getId(), startDate, endDate, AttendanceStatus.CONFIRMED);
            if (totalWorkDays == null) {
                totalWorkDays = BigDecimal.ZERO;
            }

            // 计算基本薪资
            BigDecimal baseSalary;
            BigDecimal dailyWage = worker.getDailyWage();
            BigDecimal monthlyWage = worker.getMonthlyWage();

            if (worker.getPayType() == PayType.MONTHLY && monthlyWage != null) {
                // 月薪制：基本薪资 = 月薪
                baseSalary = monthlyWage;
            } else {
                // 日薪制：基本薪资 = 工天数 × 日薪
                if (dailyWage == null) {
                    dailyWage = worker.getWorkType() != null
                            && worker.getWorkType().getDefaultDailyWage() != null
                            ? worker.getWorkType().getDefaultDailyWage()
                            : BigDecimal.ZERO;
                }
                baseSalary = totalWorkDays.multiply(dailyWage)
                        .setScale(2, RoundingMode.HALF_UP);
            }

            BigDecimal grossSalary = baseSalary;
            BigDecimal actualSalary = grossSalary;

            MonthlySalary salary = MonthlySalary.builder()
                    .worker(worker)
                    .project(project)
                    .year(dto.getYear())
                    .month(dto.getMonth())
                    .totalWorkDays(totalWorkDays)
                    .dailyWage(dailyWage)
                    .monthlyWage(monthlyWage)
                    .baseSalary(baseSalary)
                    .overtimePay(BigDecimal.ZERO)
                    .grossSalary(grossSalary)
                    .previousUnpaid(BigDecimal.ZERO)
                    .bonus(BigDecimal.ZERO)
                    .withholding(BigDecimal.ZERO)
                    .deduction(BigDecimal.ZERO)
                    .advancePayment(BigDecimal.ZERO)
                    .actualSalary(actualSalary)
                    .paymentMethod(worker.getDefaultPaymentMethod() != null
                            ? worker.getDefaultPaymentMethod() : "总包代发")
                    .workerConfirmed(0)
                    .status(SalaryStatus.DRAFT)
                    .generatedAt(LocalDateTime.now())
                    .build();

            salaryRepository.save(salary);
            generated++;
        }

        return SalaryGenerateResultVO.builder()
                .generated(generated)
                .skipped(skipped)
                .total(workers.size())
                .build();
    }

    // ===================== 列表查询 =====================

    @Override
    @Transactional(readOnly = true)
    public PageResponse<MonthlySalaryVO> list(Long projectId, Long workerId,
                                               Integer year, Integer month,
                                               String status, int page, int size) {
        SalaryStatus salaryStatus = null;
        if (StringUtils.hasText(status)) {
            salaryStatus = SalaryStatus.valueOf(status);
        }

        Page<MonthlySalary> pageResult = salaryRepository.findByFilters(
                projectId, workerId, year, month, salaryStatus,
                PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "createdAt")));

        List<MonthlySalaryVO> voList = pageResult.getContent().stream()
                .map(this::toVO)
                .collect(Collectors.toList());

        return PageUtils.toPageResponse(pageResult, voList);
    }

    // ===================== 详情 =====================

    @Override
    @Transactional(readOnly = true)
    public MonthlySalaryDetailVO getById(Long id) {
        MonthlySalary salary = salaryRepository.findDetailById(id)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));
        return toDetailVO(salary);
    }

    // ===================== 更新 =====================

    @Override
    @Transactional
    public MonthlySalaryDetailVO update(Long id, SalaryUpdateDTO dto) {
        MonthlySalary salary = salaryRepository.findDetailById(id)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));

        if (salary.getStatus() != SalaryStatus.DRAFT) {
            throw new BusinessException(400, "仅草稿状态的薪资单可以修改");
        }

        if (dto.getOvertimePay() != null) salary.setOvertimePay(dto.getOvertimePay());
        if (dto.getPaymentMethod() != null) salary.setPaymentMethod(dto.getPaymentMethod());
        if (dto.getRemark() != null) salary.setRemark(dto.getRemark());

        salary.recalculate();
        salaryRepository.save(salary);
        return toDetailVO(salary);
    }

    // ===================== 删除 =====================

    @Override
    @Transactional
    public void delete(Long id) {
        MonthlySalary salary = salaryRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));

        if (salary.getStatus() != SalaryStatus.DRAFT) {
            throw new BusinessException(400, "仅草稿状态的薪资单可以删除");
        }

        salaryRepository.delete(salary);
    }

    // ===================== 调整项 =====================

    @Override
    @Transactional
    public MonthlySalaryDetailVO addAdjustment(Long salaryId, SalaryAdjustmentDTO dto) {
        MonthlySalary salary = salaryRepository.findDetailById(salaryId)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));

        if (salary.getStatus() != SalaryStatus.DRAFT) {
            throw new BusinessException(400, "仅草稿状态的薪资单可以添加调整项");
        }

        AdjustmentType type;
        try {
            type = AdjustmentType.valueOf(dto.getType());
        } catch (IllegalArgumentException e) {
            throw new BusinessException(400, "无效的调整类型: " + dto.getType());
        }

        SalaryAdjustment adjustment = SalaryAdjustment.builder()
                .monthlySalary(salary)
                .type(type)
                .amount(dto.getAmount())
                .reason(dto.getReason())
                .build();

        salary.getAdjustments().add(adjustment);
        salary.recalculate();
        salaryRepository.save(salary);

        return toDetailVO(salary);
    }

    @Override
    @Transactional
    public MonthlySalaryDetailVO removeAdjustment(Long salaryId, Long adjustmentId) {
        MonthlySalary salary = salaryRepository.findDetailById(salaryId)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));

        if (salary.getStatus() != SalaryStatus.DRAFT) {
            throw new BusinessException(400, "仅草稿状态的薪资单可以删除调整项");
        }

        boolean removed = salary.getAdjustments()
                .removeIf(adj -> adj.getId().equals(adjustmentId));
        if (!removed) {
            throw new BusinessException(404, "调整项不存在");
        }

        salary.recalculate();
        salaryRepository.save(salary);

        return toDetailVO(salary);
    }

    // ===================== 审批流程 =====================

    @Override
    @Transactional
    public void submit(Long id) {
        MonthlySalary salary = salaryRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));
        if (salary.getStatus() != SalaryStatus.DRAFT) {
            throw new BusinessException(400, "仅草稿状态可提交审批");
        }
        salary.setStatus(SalaryStatus.PENDING_APPROVAL);
        salaryRepository.save(salary);
    }

    @Override
    @Transactional
    public void batchSubmit(List<Long> ids) {
        List<MonthlySalary> salaries = salaryRepository.findAllById(ids);
        for (MonthlySalary salary : salaries) {
            if (salary.getStatus() == SalaryStatus.DRAFT) {
                salary.setStatus(SalaryStatus.PENDING_APPROVAL);
            }
        }
        salaryRepository.saveAll(salaries);
    }

    @Override
    @Transactional
    public void approve(Long id) {
        MonthlySalary salary = salaryRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));
        if (salary.getStatus() != SalaryStatus.PENDING_APPROVAL) {
            throw new BusinessException(400, "仅待审批状态可审批通过");
        }
        salary.setStatus(SalaryStatus.APPROVED);
        salaryRepository.save(salary);
    }

    @Override
    @Transactional
    public void batchApprove(List<Long> ids) {
        List<MonthlySalary> salaries = salaryRepository.findAllById(ids);
        for (MonthlySalary salary : salaries) {
            if (salary.getStatus() == SalaryStatus.PENDING_APPROVAL) {
                salary.setStatus(SalaryStatus.APPROVED);
            }
        }
        salaryRepository.saveAll(salaries);
    }

    @Override
    @Transactional
    public void reject(Long id, String reason) {
        MonthlySalary salary = salaryRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));
        if (salary.getStatus() != SalaryStatus.PENDING_APPROVAL) {
            throw new BusinessException(400, "仅待审批状态可驳回");
        }
        salary.setStatus(SalaryStatus.DRAFT);
        if (StringUtils.hasText(reason)) {
            salary.setRemark(reason);
        }
        salaryRepository.save(salary);
    }

    @Override
    @Transactional
    public void pay(Long id) {
        MonthlySalary salary = salaryRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "薪资单不存在"));
        if (salary.getStatus() != SalaryStatus.APPROVED) {
            throw new BusinessException(400, "仅已审批状态可标记发放");
        }
        salary.setStatus(SalaryStatus.PAID);
        salary.setPaidAt(LocalDateTime.now());
        salaryRepository.save(salary);
    }

    @Override
    @Transactional
    public void batchPay(List<Long> ids) {
        List<MonthlySalary> salaries = salaryRepository.findAllById(ids);
        LocalDateTime now = LocalDateTime.now();
        for (MonthlySalary salary : salaries) {
            if (salary.getStatus() == SalaryStatus.APPROVED) {
                salary.setStatus(SalaryStatus.PAID);
                salary.setPaidAt(now);
            }
        }
        salaryRepository.saveAll(salaries);
    }

    // ===================== 汇总统计 =====================

    @Override
    @Transactional(readOnly = true)
    public SalarySummaryVO getSummary(Long projectId, Integer year, Integer month) {
        Project project = projectRepository.findById(projectId)
                .orElseThrow(() -> new BusinessException(404, "项目不存在"));

        Object[] sums = salaryRepository.sumByProjectAndMonth(projectId, year, month);
        List<Object[]> statusCounts = salaryRepository.countByStatusGrouped(projectId, year, month);

        int totalWorkers = sums[0] != null ? ((Number) sums[0]).intValue() : 0;
        BigDecimal totalWorkDays = sums[1] != null ? (BigDecimal) sums[1] : BigDecimal.ZERO;
        BigDecimal totalGrossSalary = sums[2] != null ? (BigDecimal) sums[2] : BigDecimal.ZERO;
        BigDecimal totalActualSalary = sums[3] != null ? (BigDecimal) sums[3] : BigDecimal.ZERO;
        BigDecimal totalBonus = sums[4] != null ? (BigDecimal) sums[4] : BigDecimal.ZERO;
        BigDecimal totalDeduction = sums[5] != null ? (BigDecimal) sums[5] : BigDecimal.ZERO;

        int draftCount = 0, pendingCount = 0, approvedCount = 0, paidCount = 0;
        for (Object[] row : statusCounts) {
            SalaryStatus s = (SalaryStatus) row[0];
            int count = ((Number) row[1]).intValue();
            switch (s) {
                case DRAFT -> draftCount = count;
                case PENDING_APPROVAL -> pendingCount = count;
                case APPROVED -> approvedCount = count;
                case PAID -> paidCount = count;
            }
        }

        return SalarySummaryVO.builder()
                .projectId(projectId)
                .projectName(project.getName())
                .year(year)
                .month(month)
                .totalWorkers(totalWorkers)
                .totalWorkDays(totalWorkDays)
                .totalGrossSalary(totalGrossSalary)
                .totalActualSalary(totalActualSalary)
                .totalBonus(totalBonus)
                .totalDeduction(totalDeduction)
                .draftCount(draftCount)
                .pendingCount(pendingCount)
                .approvedCount(approvedCount)
                .paidCount(paidCount)
                .build();
    }

    // ===================== VO 映射（private） =====================

    private MonthlySalaryVO toVO(MonthlySalary salary) {
        return MonthlySalaryVO.builder()
                .id(salary.getId())
                .workerId(salary.getWorker().getId())
                .workerName(salary.getWorker().getName())
                .workerNo(salary.getWorker().getWorkerNo())
                .projectId(salary.getProject().getId())
                .projectName(salary.getProject().getName())
                .year(salary.getYear())
                .month(salary.getMonth())
                .totalWorkDays(salary.getTotalWorkDays())
                .dailyWage(salary.getDailyWage())
                .monthlyWage(salary.getMonthlyWage())
                .baseSalary(salary.getBaseSalary())
                .overtimePay(salary.getOvertimePay())
                .grossSalary(salary.getGrossSalary())
                .bonus(salary.getBonus())
                .deduction(salary.getDeduction())
                .advancePayment(salary.getAdvancePayment())
                .withholding(salary.getWithholding())
                .previousUnpaid(salary.getPreviousUnpaid())
                .actualSalary(salary.getActualSalary())
                .paymentMethod(salary.getPaymentMethod())
                .status(salary.getStatus().name())
                .statusLabel(getStatusLabel(salary.getStatus()))
                .createdAt(salary.getCreatedAt())
                .build();
    }

    private MonthlySalaryDetailVO toDetailVO(MonthlySalary salary) {
        Worker worker = salary.getWorker();
        List<SalaryAdjustmentVO> adjVOs = salary.getAdjustments().stream()
                .map(this::toAdjustmentVO)
                .collect(Collectors.toList());

        return MonthlySalaryDetailVO.builder()
                .id(salary.getId())
                .workerId(worker.getId())
                .workerName(worker.getName())
                .workerNo(worker.getWorkerNo())
                .workerPhone(worker.getPhone())
                .workTypeName(worker.getWorkType() != null ? worker.getWorkType().getName() : null)
                .payType(worker.getPayType() != null ? worker.getPayType().name() : null)
                .projectId(salary.getProject().getId())
                .projectName(salary.getProject().getName())
                .year(salary.getYear())
                .month(salary.getMonth())
                .totalWorkDays(salary.getTotalWorkDays())
                .dailyWage(salary.getDailyWage())
                .monthlyWage(salary.getMonthlyWage())
                .baseSalary(salary.getBaseSalary())
                .overtimePay(salary.getOvertimePay())
                .grossSalary(salary.getGrossSalary())
                .bonus(salary.getBonus())
                .deduction(salary.getDeduction())
                .advancePayment(salary.getAdvancePayment())
                .withholding(salary.getWithholding())
                .previousUnpaid(salary.getPreviousUnpaid())
                .actualSalary(salary.getActualSalary())
                .paymentMethod(salary.getPaymentMethod())
                .bankAccount(maskBankAccount(worker.getBankAccount()))
                .bankName(worker.getBankName())
                .workerConfirmed(salary.getWorkerConfirmed() != null
                        && salary.getWorkerConfirmed() == 1)
                .status(salary.getStatus().name())
                .statusLabel(getStatusLabel(salary.getStatus()))
                .remark(salary.getRemark())
                .adjustments(adjVOs)
                .generatedAt(salary.getGeneratedAt())
                .paidAt(salary.getPaidAt())
                .createdAt(salary.getCreatedAt())
                .build();
    }

    private SalaryAdjustmentVO toAdjustmentVO(SalaryAdjustment adj) {
        return SalaryAdjustmentVO.builder()
                .id(adj.getId())
                .type(adj.getType().name())
                .typeLabel(getAdjustmentTypeLabel(adj.getType()))
                .amount(adj.getAmount())
                .reason(adj.getReason())
                .createdAt(adj.getCreatedAt())
                .build();
    }

    private String getStatusLabel(SalaryStatus status) {
        return switch (status) {
            case DRAFT -> "草稿";
            case PENDING_APPROVAL -> "待审批";
            case APPROVED -> "已审批";
            case PAID -> "已发放";
        };
    }

    private String getAdjustmentTypeLabel(AdjustmentType type) {
        return switch (type) {
            case BONUS -> "奖金";
            case DEDUCTION -> "扣款";
            case ADVANCE -> "预支/借支";
            case WITHHOLDING -> "代缴";
            case OTHER -> "其他";
        };
    }

    private String maskBankAccount(String bankAccount) {
        if (bankAccount == null || bankAccount.length() < 4) return bankAccount;
        return "****" + bankAccount.substring(bankAccount.length() - 4);
    }
}
```

#### 核心业务逻辑说明

**薪资生成 `generate()`：**

1. 查找项目下所有工人
2. 逐个工人检查是否已有本月薪资单（有则跳过）
3. 调用 `attendanceRepository.sumWorkDaysByWorkerAndMonth()` 从已确认考勤汇总工天数
4. 根据计薪方式计算基本薪资：
   - **月薪制**：`baseSalary = monthlyWage`
   - **日薪制**：`baseSalary = totalWorkDays × dailyWage`
5. 日薪使用优先级：工人个人日薪 > 工种默认日薪 > 0
6. 初始生成时 `grossSalary = baseSalary`，`actualSalary = grossSalary`
7. 状态为 `DRAFT`，后续可手动添加调整项并 `recalculate()`

**审批状态流转：**

```
DRAFT → PENDING_APPROVAL → APPROVED → PAID
             ↑                  |
             └── (reject) ──────┘
```

| 操作 | 前置状态 | 目标状态 |
|------|---------|---------|
| submit | DRAFT | PENDING_APPROVAL |
| approve | PENDING_APPROVAL | APPROVED |
| reject | PENDING_APPROVAL | DRAFT |
| pay | APPROVED | PAID |

---

### Step 7：生成 SalaryApi 接口

```bash
cd backend
mvn generate-sources
```

这将调用 OpenAPI Generator Maven Plugin，根据 `swagger-input.yml` 生成 `SalaryApi.java` 接口到 `target/generated-sources/openapi/src/main/java/com/henrywang/ems/api/SalaryApi.java`。

验证生成结果（共 15 个方法）：
- `generateSalaries(SalaryGenerateRequest)`
- `listSalaries(Long, Long, Integer, Integer, String, Integer, Integer)`
- `getSalaryById(Long)`
- `updateSalary(Long, SalaryUpdateRequest)`
- `deleteSalary(Long)`
- `addAdjustment(Long, SalaryAdjustmentRequest)`
- `removeAdjustment(Long, Long)`
- `submitSalary(Long)`
- `batchSubmitSalaries(List<Long>)`
- `approveSalary(Long)`
- `batchApproveSalaries(List<Long>)`
- `rejectSalary(Long, SalaryRejectRequest)`
- `paySalary(Long)`
- `batchPaySalaries(List<Long>)`
- `getSalarySummary(Long, Integer, Integer)`

---

### Step 8：SalaryController（契约优先 implements SalaryApi）

```java
package com.henrywang.ems.controller;

import com.henrywang.ems.api.SalaryApi;
import com.henrywang.ems.api.model.SalaryAdjustmentRequest;
import com.henrywang.ems.api.model.SalaryDetailResponse;
import com.henrywang.ems.api.model.SalaryGenerateRequest;
import com.henrywang.ems.api.model.SalaryGenerateResponse;
import com.henrywang.ems.api.model.SalaryPageResponse;
import com.henrywang.ems.api.model.SalaryRejectRequest;
import com.henrywang.ems.api.model.SalarySummaryResponse;
import com.henrywang.ems.api.model.SalaryUpdateRequest;
import com.henrywang.ems.api.model.SalaryVoidResponse;
import com.henrywang.ems.common.result.R;
import com.henrywang.ems.dto.SalaryAdjustmentDTO;
import com.henrywang.ems.dto.SalaryGenerateDTO;
import com.henrywang.ems.dto.SalaryUpdateDTO;
import com.henrywang.ems.service.SalaryService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.RestController;

import java.math.BigDecimal;
import java.util.List;

/**
 * 薪资管理控制器 — 契约优先：implements 由 OpenAPI Generator 生成的 SalaryApi 接口。
 * <p>
 * 所有路由注解（@RequestMapping）由 SalaryApi 提供，此处仅做业务委派。
 * 返回值统一使用 R<T> 封装，通过 raw-type 强转兼容生成的 ResponseEntity<XxxResponse> 签名。
 */
@RestController
@RequiredArgsConstructor
public class SalaryController implements SalaryApi {

    private final SalaryService salaryService;

    // ==================== 生成 ====================

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryGenerateResponse> generateSalaries(SalaryGenerateRequest req) {
        SalaryGenerateDTO dto = new SalaryGenerateDTO();
        dto.setProjectId(req.getProjectId());
        dto.setYear(req.getYear());
        dto.setMonth(req.getMonth());
        return (ResponseEntity) ResponseEntity.ok(R.ok(salaryService.generate(dto)));
    }

    // ==================== 查询 ====================

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryPageResponse> listSalaries(Long projectId, Long workerId,
                                                            Integer year, Integer month,
                                                            String status, Integer page,
                                                            Integer size) {
        return (ResponseEntity) ResponseEntity.ok(
                R.ok(salaryService.list(projectId, workerId, year, month, status, page, size)));
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryDetailResponse> getSalaryById(Long id) {
        return (ResponseEntity) ResponseEntity.ok(R.ok(salaryService.getById(id)));
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalarySummaryResponse> getSalarySummary(Long projectId,
                                                                   Integer year,
                                                                   Integer month) {
        return (ResponseEntity) ResponseEntity.ok(
                R.ok(salaryService.getSummary(projectId, year, month)));
    }

    // ==================== 更新/删除 ====================

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryDetailResponse> updateSalary(Long id, SalaryUpdateRequest req) {
        SalaryUpdateDTO dto = new SalaryUpdateDTO();
        if (req.getOvertimePay() != null) {
            dto.setOvertimePay(req.getOvertimePay());
        }
        dto.setPaymentMethod(req.getPaymentMethod());
        dto.setRemark(req.getRemark());
        return (ResponseEntity) ResponseEntity.ok(R.ok(salaryService.update(id, dto)));
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> deleteSalary(Long id) {
        salaryService.delete(id);
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }

    // ==================== 调整项 ====================

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryDetailResponse> addAdjustment(Long id,
                                                               SalaryAdjustmentRequest req) {
        SalaryAdjustmentDTO dto = new SalaryAdjustmentDTO();
        dto.setType(req.getType().getValue());
        dto.setAmount(req.getAmount());
        dto.setReason(req.getReason());
        return (ResponseEntity) ResponseEntity.ok(R.ok(salaryService.addAdjustment(id, dto)));
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryDetailResponse> removeAdjustment(Long id, Long adjId) {
        return (ResponseEntity) ResponseEntity.ok(
                R.ok(salaryService.removeAdjustment(id, adjId)));
    }

    // ==================== 审批流程 ====================

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> submitSalary(Long id) {
        salaryService.submit(id);
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> batchSubmitSalaries(List<Long> ids) {
        salaryService.batchSubmit(ids);
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> approveSalary(Long id) {
        salaryService.approve(id);
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> batchApproveSalaries(List<Long> ids) {
        salaryService.batchApprove(ids);
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> rejectSalary(Long id, SalaryRejectRequest req) {
        salaryService.reject(id, req.getReason());
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> paySalary(Long id) {
        salaryService.pay(id);
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }

    @Override
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    @SuppressWarnings({"unchecked", "rawtypes"})
    public ResponseEntity<SalaryVoidResponse> batchPaySalaries(List<Long> ids) {
        salaryService.batchPay(ids);
        return (ResponseEntity) ResponseEntity.ok(R.ok());
    }
}
```

#### Controller 关键实现细节

1. **`implements SalaryApi`**：不写任何 `@RequestMapping`，路由注解全部由生成接口提供
2. **raw-type 桥接**：每个方法使用 `@SuppressWarnings({"unchecked", "rawtypes"})` + `(ResponseEntity) ResponseEntity.ok(R.ok(result))`
3. **请求类型映射**：OpenAPI 生成的 Request 类型 → 手写 DTO
   - `SalaryGenerateRequest` → `SalaryGenerateDTO`
   - `SalaryUpdateRequest` → `SalaryUpdateDTO`
   - `SalaryAdjustmentRequest` → `SalaryAdjustmentDTO`（注意 `req.getType().getValue()` 获取枚举字符串值）
   - `SalaryRejectRequest` → 直接取 `req.getReason()`
4. **权限控制**：写操作使用 `@PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")`，只读操作无限制

---

### Step 9：后端编译验证

```bash
cd backend
mvn compile
```

预期输出：`BUILD SUCCESS`，128+ 源文件编译通过。

---

## 四、前端实现（uni-app）

### Step 10：前端 API 模块

创建 `frontend/src/api/modules/salary.js`：

```javascript
import { get, post, put, del } from '../request'

/** 批量生成薪资单 */
export const generateSalaries = (data) => post('/salaries/generate', data)

/** 薪资列表（分页） */
export const listSalaries = (params) => get('/salaries', params)

/** 薪资详情 */
export const getSalary = (id) => get(`/salaries/${id}`)

/** 更新薪资单 */
export const updateSalary = (id, data) => put(`/salaries/${id}`, data)

/** 删除薪资单（仅DRAFT） */
export const deleteSalary = (id) => del(`/salaries/${id}`)

/** 添加薪资调整项 */
export const addAdjustment = (salaryId, data) => post(`/salaries/${salaryId}/adjustments`, data)

/** 删除薪资调整项 */
export const removeAdjustment = (salaryId, adjId) => del(`/salaries/${salaryId}/adjustments/${adjId}`)

/** 提交审批 */
export const submitSalary = (id) => post(`/salaries/${id}/submit`)

/** 批量提交审批 */
export const batchSubmitSalaries = (ids) => post('/salaries/batch-submit', ids)

/** 审批通过 */
export const approveSalary = (id) => post(`/salaries/${id}/approve`)

/** 批量审批通过 */
export const batchApproveSalaries = (ids) => post('/salaries/batch-approve', ids)

/** 驳回薪资单 */
export const rejectSalary = (id, reason) => post(`/salaries/${id}/reject`, { reason })

/** 标记已发放 */
export const paySalary = (id) => post(`/salaries/${id}/pay`)

/** 批量标记已发放 */
export const batchPaySalaries = (ids) => post('/salaries/batch-pay', ids)

/** 薪资月度汇总 */
export const getSalarySummary = (params) => get('/salaries/summary', params)
```

---

### Step 11：薪资列表页面

**文件路径**：`frontend/src/pages/salary/index.vue`

**功能要点**：
- 筛选栏：项目选择器、月份选择器、状态选择器、清除按钮
- 汇总卡片：显示工人数、总工天、实发总额、各状态数量（仅选中项目时显示）
- 分页薪资列表：卡片式布局，每卡显示工人姓名、工号、工天、日薪、应发、实发、状态标签
- 底部 FAB 按钮：点击跳转"生成薪资"页面
- 下拉刷新 + 触底加载

**核心逻辑**：
- `onShow` 时加载列表 + 汇总 + 项目列表
- `onReachBottom` 分页加载
- `onPullDownRefresh` 刷新
- 筛选变更时 `loadList(reset=true)` 重新加载

---

### Step 12：薪资详情页面

**文件路径**：`frontend/src/pages/salary/detail.vue`

**功能要点**：
- **基本信息区**：工人、工种、项目、月份、计薪方式、发放方式 + 状态标签
- **薪资明细区**：出勤工天、日薪/月薪标准、基本薪资、加班工资、应发金额、上月未发、奖金、代缴、扣款、预支借支、实发金额
- **调整明细区**：调整项列表（类型、金额、原因），DRAFT 状态下可删除
- **银行信息区**：开户行、银行卡号（脱敏显示）
- **操作按钮栏**（底部固定，根据状态显示不同按钮）：
  - `DRAFT`：+调整项、提交审批、删除
  - `PENDING_APPROVAL`：审批通过、驳回
  - `APPROVED`：标记已发放

**添加调整项交互流程**：
1. 点击"+调整项" → ActionSheet 选择类型（奖金/扣款/预支/代缴/其他）
2. 选择后弹出 Modal 输入金额
3. 调用 `addAdjustment` API → 返回更新后的详情数据

---

### Step 13：薪资生成页面

**文件路径**：`frontend/src/pages/salary/generate.vue`

**功能要点**：
- 表单：选择项目（底部弹窗列表）、选择月份（ActionSheet 选择最近12个月）
- 提示文案：`将根据所选项目与月份的考勤数据自动计算薪资。已有薪资记录的工人将被跳过。`
- 生成按钮 + 加载状态
- 生成结果展示：工人总数、成功生成、跳过三个指标
- 返回薪资列表按钮
- 默认选中上个月

---

### Step 14：pages.json 路由更新 + 首页入口

#### 14.1 pages.json 新增路由

```json
{
  "path": "pages/salary/index",
  "style": {
    "navigationBarTitleText": "薪资管理",
    "enablePullDownRefresh": true
  }
},
{
  "path": "pages/salary/detail",
  "style": {
    "navigationBarTitleText": "薪资详情"
  }
},
{
  "path": "pages/salary/generate",
  "style": {
    "navigationBarTitleText": "生成薪资"
  }
}
```

#### 14.2 首页添加薪资管理入口

在 `pages/home/index.vue` 的 `menu-grid` 中新增卡片：

```html
<view class="menu-card" @click="navigateTo('/pages/salary/index')">
  <view class="menu-icon" style="background: rgba(250,140,22,0.1);">
    <Icon icon="ant-design:money-collect-outlined" :width="32" :height="32" color="#FA8C16" />
  </view>
  <text class="menu-label">薪资管理</text>
</view>
```

---

## 五、数据库变更

### Phase 3 不需要新增数据库迁移

薪资相关表已在 `V1__create_tables.sql` 中创建：

- `ems_monthly_salary` — 月度薪资主表（含唯一约束 `uk_worker_month`）
- `ems_salary_adjustment` — 薪资调整项子表（外键关联 `monthly_salary_id`）

Flyway 迁移 V1-V5 均已就绪，无需新增迁移文件。

---

## 六、测试要求

### 6.1 后端测试

**核心测试用例：**

| # | 场景 | 预期 |
|---|------|------|
| 1 | 生成薪资 — 正常流程 | 返回 generated > 0，每人一条薪资单 |
| 2 | 生成薪资 — 重复生成 | skipped 数增加，不会产生重复记录 |
| 3 | 生成薪资 — 项目无工人 | 抛出 BusinessException(400) |
| 4 | 更新 DRAFT 薪资单 | 成功更新，recalculate 后实发金额正确 |
| 5 | 更新非 DRAFT 薪资单 | 抛出 BusinessException(400) |
| 6 | 添加奖金调整项 | actualSalary 增加 |
| 7 | 添加扣款调整项 | actualSalary 减少 |
| 8 | 删除调整项 | actualSalary 重新计算 |
| 9 | DRAFT → submit → PENDING | 状态正确流转 |
| 10 | PENDING → approve → APPROVED | 状态正确流转 |
| 11 | PENDING → reject → DRAFT | 状态回退，remark 记录驳回原因 |
| 12 | APPROVED → pay → PAID | paidAt 时间被记录 |
| 13 | 非法状态流转 | 抛出 BusinessException(400) |
| 14 | 批量提交/审批/发放 | 跳过不符合条件的记录 |
| 15 | 汇总统计 | totalWorkers/totalWorkDays/totalActualSalary 正确 |

### 6.2 前端测试

| # | 场景 | 验证点 |
|---|------|--------|
| 1 | 打开薪资列表页 | 列表正常加载，分页正常 |
| 2 | 筛选项目/月份/状态 | 列表和汇总数据联动刷新 |
| 3 | 点击薪资卡片 | 跳转详情页，数据完整 |
| 4 | DRAFT 详情页 | 显示"提交审批""删除""调整项"按钮 |
| 5 | 添加调整项 | 选择类型 → 输入金额 → 实发金额实时更新 |
| 6 | 提交审批 | 状态变为"待审批"，按钮区域更新 |
| 7 | 审批通过 | 状态变为"已审批" |
| 8 | 驳回 | 弹窗输入原因，状态回退到"草稿" |
| 9 | 标记已发放 | 状态变为"已发放"，无更多操作按钮 |
| 10 | 生成薪资 | 选择项目+月份，点击生成，显示结果 |

---

## 七、关键决策与注意事项

### 7.1 薪资计算公式

```
baseSalary = 日薪制 ? totalWorkDays × dailyWage : monthlyWage
grossSalary = baseSalary + overtimePay
actualSalary = grossSalary + previousUnpaid + bonus - withholding - deduction - advancePayment
```

### 7.2 调整项类型与正负方向

| 类型 | 方向 | 说明 |
|------|------|------|
| BONUS | 增加 | 奖金（加到 actualSalary） |
| DEDUCTION | 减少 | 扣款 |
| ADVANCE | 减少 | 预支/借支 |
| WITHHOLDING | 减少 | 代缴（社保、工会费等） |
| OTHER | 减少 | 其他扣款（归入 deduction） |

### 7.3 审批状态约束

| 操作 | 仅限状态 | 权限 |
|------|---------|------|
| 修改/删除/添加调整项 | DRAFT | SUPER_ADMIN, PROJECT_MANAGER |
| 提交审批 | DRAFT | SUPER_ADMIN, PROJECT_MANAGER |
| 审批通过/驳回 | PENDING_APPROVAL | SUPER_ADMIN, PROJECT_MANAGER |
| 标记已发放 | APPROVED | SUPER_ADMIN, PROJECT_MANAGER |

### 7.4 契约优先 vs 手写 Controller 对比

| 维度 | Phase 1/2（手写） | Phase 3（契约优先） |
|------|-------------------|-------------------|
| Controller 声明 | `@RestController` + 手写 `@RequestMapping` | `@RestController implements SalaryApi` |
| 路由注解 | 手动 `@GetMapping/@PostMapping` | 由 SalaryApi 生成接口提供 |
| Swagger 文档 | `@Tag/@Operation` 手写注解 | YAML 自动生成 |
| 编译期检查 | 依赖开发者自觉 | 必须实现接口全部方法 |
| 请求参数 | 直接使用内部 DTO | 生成 Request → 手动转换 → 内部 DTO |

### 7.5 文件结构（Phase 3 新增）

**后端新增文件：**
```
com.henrywang.ems/
├── entity/
│   ├── MonthlySalary.java                 [新增]
│   └── SalaryAdjustment.java              [新增]
├── entity/enums/
│   ├── SalaryStatus.java                  [新增]
│   └── AdjustmentType.java                [新增]
├── repository/
│   ├── MonthlySalaryRepository.java       [新增]
│   └── SalaryAdjustmentRepository.java    [新增]
├── dto/
│   ├── SalaryGenerateDTO.java             [新增]
│   ├── SalaryUpdateDTO.java               [新增]
│   └── SalaryAdjustmentDTO.java           [新增]
├── vo/
│   ├── MonthlySalaryVO.java               [新增]
│   ├── MonthlySalaryDetailVO.java         [新增]
│   ├── SalaryAdjustmentVO.java            [新增]
│   ├── SalarySummaryVO.java               [新增]
│   └── SalaryGenerateResultVO.java        [新增]
├── service/
│   ├── SalaryService.java                 [新增]
│   └── impl/
│       └── SalaryServiceImpl.java         [新增]
└── controller/
    └── SalaryController.java              [新增]
```

**自动生成文件（target/generated-sources/）：**
```
com.henrywang.ems.api/
│   └── SalaryApi.java                     [自动生成]
com.henrywang.ems.api.model/
│   ├── SalaryGenerateRequest.java         [自动生成]
│   ├── SalaryUpdateRequest.java           [自动生成]
│   ├── SalaryAdjustmentRequest.java       [自动生成]
│   ├── SalaryRejectRequest.java           [自动生成]
│   ├── SalaryVoidResponse.java            [自动生成]
│   ├── SalaryGenerateResponse.java        [自动生成]
│   ├── SalaryPageResponse.java            [自动生成]
│   ├── SalaryDetailResponse.java          [自动生成]
│   ├── SalarySummaryResponse.java         [自动生成]
│   └── ... (其他数据模型)                  [自动生成]
```

**前端新增文件：**
```
frontend/src/
├── api/modules/
│   └── salary.js                          [新增]
└── pages/salary/
    ├── index.vue                          [新增] 薪资管理列表
    ├── detail.vue                         [新增] 薪资详情
    └── generate.vue                       [新增] 薪资生成
```

**修改的现有文件：**
```
backend/src/main/resources/openapi/
    swagger-input.yml                      [修改] 新增 salary tag + 15 端点 + 20 Schema
frontend/src/pages/home/index.vue          [修改] 添加薪资管理功能入口卡片
frontend/src/pages.json                    [修改] 注册 3 个薪资页面路由
```

---

## 八、与 Phase 4 的衔接

Phase 3 完成后，以下内容为后续阶段做好准备：

| 准备项 | 状态 |
|--------|------|
| `ems_monthly_salary` 表 | ✅ Phase 0 创建 |
| MonthlySalary + SalaryAdjustment Entity | ✅ Phase 3 创建 |
| 薪资自动生成（从考勤核算） | ✅ Phase 3 完成 |
| 调整项管理 | ✅ Phase 3 完成 |
| 审批流程（DRAFT → PENDING → APPROVED → PAID） | ✅ Phase 3 完成 |
| 薪资月度汇总 | ✅ Phase 3 完成 |
| 多级审批签名链路 | ❌ 字段已预留，业务逻辑待后续实现 |
| 工资条 PDF/Excel 导出 | ❌ Phase 4 实现 |
| 工人端确认工资（小程序） | ❌ Phase 5 实现 |
| 合同管理 | ❌ Phase 4 实现 |
| 工人调动功能 | ❌ Phase 4 实现 |

---

## 九、API 接口汇总（Phase 3 新增）

```
# ===== 薪资管理 =====
POST   /api/salaries/generate                    # 批量生成薪资单
GET    /api/salaries                              # 薪资列表（分页+筛选）
GET    /api/salaries/{id}                         # 薪资详情（含调整明细）
PUT    /api/salaries/{id}                         # 更新薪资单（仅DRAFT）
DELETE /api/salaries/{id}                         # 删除薪资单（仅DRAFT）
POST   /api/salaries/{id}/adjustments             # 添加调整项
DELETE /api/salaries/{id}/adjustments/{adjId}      # 删除调整项
POST   /api/salaries/{id}/submit                  # 提交审批
POST   /api/salaries/batch-submit                 # 批量提交审批
POST   /api/salaries/{id}/approve                 # 审批通过
POST   /api/salaries/batch-approve                # 批量审批通过
POST   /api/salaries/{id}/reject                  # 驳回薪资单
POST   /api/salaries/{id}/pay                     # 标记已发放
POST   /api/salaries/batch-pay                    # 批量标记已发放
GET    /api/salaries/summary                      # 薪资月度汇总
```

---

> **执行提示**：建议按照 Step 1 → Step 14 的顺序逐步实施，每完成一个 Step 即进行编译验证。特别注意：
> 1. Step 1（YAML 契约）完成后务必 `mvn generate-sources` 验证 SalaryApi.java 生成正确
> 2. Step 6（Service）是核心业务逻辑，需要确保 `recalculate()` 计算正确
> 3. Step 8（Controller）的 raw-type 桥接模式是 Phase 3 的关键决策，需理解泛型擦除原理
> 4. 所有写操作需使用 `@PreAuthorize` 限制权限
