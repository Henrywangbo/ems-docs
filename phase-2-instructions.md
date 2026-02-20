# Phase 2：工天管理 — 详细实施指南

> 编制日期：2026-02-18  
> 前置条件：Phase 1 已完成（JWT 认证、工种/项目/班组/工人 CRUD 就绪）  
> 周期：第 4–5 周（约 10 个工作日）  
> 目标：实现完整的考勤/工天管理体系，包括后端 CRUD + 批量录入 + 考勤审核 + 适老化前端页面

---

## 一、Phase 2 总览

### 1.1 交付目标

| # | 交付物 | 说明 |
|---|--------|------|
| 1 | Attendance Entity + Repository | 考勤实体类与数据访问层 |
| 2 | 考勤 CRUD 接口 | 单条录入、查询、修改、删除 |
| 3 | 批量考勤录入接口 | 按班组一次性提交多人当日考勤 |
| 4 | 考勤审核接口 | 工长确认 / 驳回考勤记录 |
| 5 | 考勤统计查询接口 | 按日期/班组/项目/工人查询考勤 |
| 6 | 考勤日历数据接口 | 返回指定工人/月份的逐日考勤信息 |
| 7 | 适老化工天批量录入页面 | 大按钮、大字体、一行一人的极简交互（核心高频页面） |
| 8 | 考勤管理列表页面 | 按日期/项目/班组筛选查看考勤记录 |
| 9 | 考勤日历视图页面 | 日历形式展示个人月度出勤 |
| 10 | 首页考勤快捷入口 | 在首页添加"考勤录入"功能卡片 |

### 1.2 不在本阶段范围

- ❌ 小程序端 GPS 定位 + 拍照打卡（Phase 5）
- ❌ 补卡申请流程（P2 优先级，推迟到 Phase 5 或后续迭代）
- ❌ 考勤与薪资的自动关联核算（Phase 3）
- ❌ 工人调动功能（推迟到 Phase 3）
- ❌ 打卡围栏校验（Phase 5 小程序端）

### 1.3 开发顺序（建议严格遵循）

```
Week 1 (第 4 周):
  Step 1:  考勤枚举类 — WorkCategory、AttendanceStatus
  Step 2:  Attendance Entity — 考勤实体类
  Step 3:  AttendanceRepository — 数据访问层（含自定义查询）
  Step 4:  考勤 DTO/VO — 请求/响应对象定义
  Step 5:  AttendanceService — 考勤业务逻辑（CRUD + 批量 + 审核）
  Step 6:  AttendanceController — RESTful 接口
  Step 7:  后端接口联调测试 — Swagger UI 验证全部接口

Week 2 (第 5 周):
  Step 8:  前端 API 模块 — attendance.js 接口封装
  Step 9:  工天批量录入页面 — 适老化核心页面（大按钮、一行一人）
  Step 10: 考勤管理列表页面 — 筛选、查看、修改、审核操作
  Step 11: 考勤日历视图页面 — 日历展示个人月度出勤
  Step 12: 首页改造 — 添加考勤入口 + 今日考勤统计
  Step 13: pages.json 路由更新 — 注册新页面
  Step 14: 适老化组件封装 — elderly-ui 目录下的可复用组件
```

---

## 二、后端实现（Spring Boot）

### Step 1：考勤枚举类

在 `com.henrywang.ems.entity.enums` 包下新增两个枚举类。

```java
// com.henrywang.ems.entity.enums.WorkCategory
package com.henrywang.ems.entity.enums;

/**
 * 出勤类型
 */
public enum WorkCategory {
    FULL_DAY,    // 全天 (1工)
    HALF_DAY,    // 半天 (0.5工)
    OVERTIME,    // 加班 (1.5工 或 2工)
    LEAVE        // 请假 (0工)
}
```

```java
// com.henrywang.ems.entity.enums.AttendanceStatus
package com.henrywang.ems.entity.enums;

/**
 * 考勤审核状态
 */
public enum AttendanceStatus {
    PENDING,     // 待确认
    CONFIRMED,   // 已确认
    REJECTED     // 已驳回
}
```

---

### Step 2：Attendance Entity

在 `com.henrywang.ems.entity` 包下创建 `Attendance` 实体，映射 `ems_attendance` 表。

```java
package com.henrywang.ems.entity;

import com.henrywang.ems.entity.enums.AttendanceStatus;
import com.henrywang.ems.entity.enums.WorkCategory;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "ems_attendance")
@Data
@EqualsAndHashCode(callSuper = true)
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Attendance extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "worker_id", nullable = false)
    private Worker worker;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "project_id", nullable = false)
    private Project project;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    @Column(name = "work_date", nullable = false)
    private LocalDate workDate;

    @Enumerated(EnumType.STRING)
    @Column(name = "work_category", nullable = false,
            columnDefinition = "ENUM('FULL_DAY','HALF_DAY','OVERTIME','LEAVE')")
    private WorkCategory workCategory;

    @Column(name = "work_days", nullable = false, precision = 3, scale = 1)
    private BigDecimal workDays;

    @Column(name = "daily_wage_snapshot", nullable = false, precision = 10, scale = 2)
    private BigDecimal dailyWageSnapshot;

    @Column(name = "is_overtime", columnDefinition = "TINYINT DEFAULT 0")
    private Integer isOvertime;

    @Column(name = "overtime_hours", precision = 4, scale = 1)
    private BigDecimal overtimeHours;

    @Column(name = "actual_work_output", length = 200)
    private String actualWorkOutput;

    // ===== 打卡信息（Phase 5 小程序端使用，Phase 2 暂不填充） =====

    @Column(name = "check_in_time")
    private LocalDateTime checkInTime;

    @Column(name = "check_in_lng", precision = 10, scale = 7)
    private BigDecimal checkInLng;

    @Column(name = "check_in_lat", precision = 10, scale = 7)
    private BigDecimal checkInLat;

    @Column(name = "check_in_photo", length = 500)
    private String checkInPhoto;

    @Column(name = "check_out_time")
    private LocalDateTime checkOutTime;

    @Column(name = "check_out_lng", precision = 10, scale = 7)
    private BigDecimal checkOutLng;

    @Column(name = "check_out_lat", precision = 10, scale = 7)
    private BigDecimal checkOutLat;

    @Column(name = "check_out_photo", length = 500)
    private String checkOutPhoto;

    // ===== 审核信息 =====

    @Enumerated(EnumType.STRING)
    @Column(columnDefinition = "ENUM('PENDING','CONFIRMED','REJECTED') DEFAULT 'PENDING'")
    private AttendanceStatus status;

    @Column(name = "confirmed_by")
    private Long confirmedBy;

    @Column(name = "confirmed_at")
    private LocalDateTime confirmedAt;

    @Column(length = 500)
    private String remark;
}
```

#### 关键注意事项

1. **`@Table(name = "ems_attendance")`**：表名必须带 `ems_` 前缀
2. **唯一约束**：数据库中有 `UNIQUE KEY uk_worker_date (worker_id, work_date)`，同一工人同一天只能有一条考勤记录。业务逻辑层需要校验
3. **`daily_wage_snapshot`**：录入考勤时快照当日工人日薪，后续工人调薪不影响历史考勤的薪资计算
4. **打卡字段**：Phase 2 暂不使用 `check_in_*` 和 `check_out_*` 字段，但实体中需要定义，方便 Hibernate validate 通过
5. **`workDays` 计算规则**：

| 出勤类型 | workDays 默认值 | 说明 |
|---------|----------------|------|
| FULL_DAY | 1.0 | 全天出勤 |
| HALF_DAY | 0.5 | 半天出勤 |
| OVERTIME | 1.5 或 2.0 | 加班，由录入时指定 |
| LEAVE | 0.0 | 请假无工天 |

---

### Step 3：AttendanceRepository

在 `com.henrywang.ems.repository` 包下创建。

```java
package com.henrywang.ems.repository;

import com.henrywang.ems.entity.Attendance;
import com.henrywang.ems.entity.enums.AttendanceStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

public interface AttendanceRepository extends JpaRepository<Attendance, Long> {

    /**
     * 查询某工人某天的考勤记录（用于唯一性校验）
     */
    Optional<Attendance> findByWorkerIdAndWorkDate(Long workerId, LocalDate workDate);

    /**
     * 判断某工人某天是否已有考勤
     */
    boolean existsByWorkerIdAndWorkDate(Long workerId, LocalDate workDate);

    /**
     * 查询某个班组某天的全部考勤
     */
    List<Attendance> findByTeamIdAndWorkDate(Long teamId, LocalDate workDate);

    /**
     * 查询某个项目某天的全部考勤
     */
    List<Attendance> findByProjectIdAndWorkDate(Long projectId, LocalDate workDate);

    /**
     * 查询某工人某月的全部考勤（日历视图）
     */
    @Query("SELECT a FROM Attendance a " +
           "WHERE a.worker.id = :workerId " +
           "AND a.workDate BETWEEN :startDate AND :endDate " +
           "ORDER BY a.workDate ASC")
    List<Attendance> findByWorkerIdAndMonth(
            @Param("workerId") Long workerId,
            @Param("startDate") LocalDate startDate,
            @Param("endDate") LocalDate endDate);

    /**
     * 多条件分页查询（考勤管理列表用）
     */
    @Query("SELECT a FROM Attendance a " +
           "LEFT JOIN FETCH a.worker w " +
           "LEFT JOIN FETCH a.project p " +
           "LEFT JOIN FETCH a.team t " +
           "WHERE (:projectId IS NULL OR a.project.id = :projectId) " +
           "AND (:teamId IS NULL OR a.team.id = :teamId) " +
           "AND (:workerId IS NULL OR a.worker.id = :workerId) " +
           "AND (:workDate IS NULL OR a.workDate = :workDate) " +
           "AND (:status IS NULL OR a.status = :status) " +
           "AND (:startDate IS NULL OR a.workDate >= :startDate) " +
           "AND (:endDate IS NULL OR a.workDate <= :endDate)")
    Page<Attendance> findByFilters(
            @Param("projectId") Long projectId,
            @Param("teamId") Long teamId,
            @Param("workerId") Long workerId,
            @Param("workDate") LocalDate workDate,
            @Param("status") AttendanceStatus status,
            @Param("startDate") LocalDate startDate,
            @Param("endDate") LocalDate endDate,
            Pageable pageable);

    /**
     * Count 查询（配合分页，避免 FETCH JOIN + COUNT 的问题）
     */
    @Query("SELECT COUNT(a) FROM Attendance a " +
           "WHERE (:projectId IS NULL OR a.project.id = :projectId) " +
           "AND (:teamId IS NULL OR a.team.id = :teamId) " +
           "AND (:workerId IS NULL OR a.worker.id = :workerId) " +
           "AND (:workDate IS NULL OR a.workDate = :workDate) " +
           "AND (:status IS NULL OR a.status = :status) " +
           "AND (:startDate IS NULL OR a.workDate >= :startDate) " +
           "AND (:endDate IS NULL OR a.workDate <= :endDate)")
    long countByFilters(
            @Param("projectId") Long projectId,
            @Param("teamId") Long teamId,
            @Param("workerId") Long workerId,
            @Param("workDate") LocalDate workDate,
            @Param("status") AttendanceStatus status,
            @Param("startDate") LocalDate startDate,
            @Param("endDate") LocalDate endDate);

    /**
     * 按项目+日期范围统计各出勤类型的数量
     */
    @Query("SELECT a.workCategory, COUNT(a), SUM(a.workDays) FROM Attendance a " +
           "WHERE a.project.id = :projectId " +
           "AND a.workDate BETWEEN :startDate AND :endDate " +
           "GROUP BY a.workCategory")
    List<Object[]> countByProjectAndDateRange(
            @Param("projectId") Long projectId,
            @Param("startDate") LocalDate startDate,
            @Param("endDate") LocalDate endDate);

    /**
     * 查询某班组某天已录入考勤的工人ID列表
     */
    @Query("SELECT a.worker.id FROM Attendance a " +
           "WHERE a.team.id = :teamId AND a.workDate = :workDate")
    List<Long> findWorkerIdsByTeamIdAndWorkDate(
            @Param("teamId") Long teamId,
            @Param("workDate") LocalDate workDate);

    /**
     * 按工人+月份统计总工天数（薪资核算预备）
     */
    @Query("SELECT SUM(a.workDays) FROM Attendance a " +
           "WHERE a.worker.id = :workerId " +
           "AND a.workDate BETWEEN :startDate AND :endDate " +
           "AND a.status = 'CONFIRMED'")
    BigDecimal sumWorkDaysByWorkerAndMonth(
            @Param("workerId") Long workerId,
            @Param("startDate") LocalDate startDate,
            @Param("endDate") LocalDate endDate);
}
```

> **注意 FETCH JOIN + 分页的问题**：当使用 `LEFT JOIN FETCH` 同时配合 `Pageable` 时，Hibernate 会在内存中分页（warn: `HHH90003004`）。解决方案：
> - 方案 A（推荐）：分页查询不使用 FETCH JOIN，查出 ID 列表后再批量 FETCH 关联实体
> - 方案 B：使用 `@EntityGraph` 代替 FETCH JOIN
> - 方案 C：接受 warn，数据量小（< 100 工人 × 30 天 ≈ 3000 条/月）时无性能问题
>
> 本项目数据量极小，建议先用方案 C，后续有性能问题再优化。

---

### Step 4：考勤 DTO / VO

#### 4.1 AttendanceDTO — 单条考勤录入

```java
package com.henrywang.ems.dto;

import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDate;

/**
 * 单条考勤录入请求
 */
@Data
public class AttendanceDTO {

    @NotNull(message = "工人ID不能为空")
    private Long workerId;

    @NotNull(message = "项目ID不能为空")
    private Long projectId;

    private Long teamId;

    @NotNull(message = "工作日期不能为空")
    private LocalDate workDate;

    @NotBlank(message = "出勤类型不能为空")
    private String workCategory;  // FULL_DAY / HALF_DAY / OVERTIME / LEAVE

    /**
     * 折算工天数（可选，不传则根据 workCategory 自动计算）
     * 加班时需要手动指定：1.5 或 2.0
     */
    private BigDecimal workDays;

    /**
     * 加班时长（小时），仅当 workCategory = OVERTIME 时有效
     */
    private BigDecimal overtimeHours;

    /**
     * 当日实际工作量描述（可选）
     */
    private String actualWorkOutput;

    /**
     * 备注（可选）
     */
    private String remark;
}
```

#### 4.2 AttendanceBatchDTO — 批量考勤录入

```java
package com.henrywang.ems.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import lombok.Data;

import java.time.LocalDate;
import java.util.List;

/**
 * 批量考勤录入请求
 * 对应前端"工天批量录入页面"一次性提交整个班组的考勤
 */
@Data
public class AttendanceBatchDTO {

    @NotNull(message = "项目ID不能为空")
    private Long projectId;

    @NotNull(message = "班组ID不能为空")
    private Long teamId;

    @NotNull(message = "工作日期不能为空")
    private LocalDate workDate;

    @NotEmpty(message = "考勤记录不能为空")
    @Valid
    private List<AttendanceBatchItem> items;

    /**
     * 批量录入的单条记录
     */
    @Data
    public static class AttendanceBatchItem {

        @NotNull(message = "工人ID不能为空")
        private Long workerId;

        @NotBlank(message = "出勤类型不能为空")
        private String workCategory;  // FULL_DAY / HALF_DAY / OVERTIME / LEAVE

        /**
         * 折算工天数（可选，不传则自动计算）
         */
        private java.math.BigDecimal workDays;

        /**
         * 加班时长（仅 OVERTIME 时填）
         */
        private java.math.BigDecimal overtimeHours;

        /**
         * 备注
         */
        private String remark;
    }
}
```

#### 4.3 AttendanceUpdateDTO — 修改考勤

```java
package com.henrywang.ems.dto;

import lombok.Data;
import java.math.BigDecimal;

/**
 * 考勤修改请求（仅允许修改出勤类型、工天数、备注等）
 */
@Data
public class AttendanceUpdateDTO {

    private String workCategory;    // FULL_DAY / HALF_DAY / OVERTIME / LEAVE

    private BigDecimal workDays;

    private BigDecimal overtimeHours;

    private String actualWorkOutput;

    private String remark;
}
```

#### 4.4 AttendanceVO — 考勤列表展示

```java
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Data
@Builder
public class AttendanceVO {
    private Long id;
    private Long workerId;
    private String workerName;
    private String workerWorkTypeName;  // 工种名称
    private Long projectId;
    private String projectName;
    private Long teamId;
    private String teamName;
    private LocalDate workDate;
    private String workCategory;       // FULL_DAY / HALF_DAY / OVERTIME / LEAVE
    private String workCategoryLabel;  // 全天 / 半天 / 加班 / 请假（中文展示）
    private BigDecimal workDays;       // 折算工天
    private BigDecimal dailyWageSnapshot;
    private BigDecimal dailyEarning;   // 当日收入 = workDays × dailyWageSnapshot
    private Integer isOvertime;
    private BigDecimal overtimeHours;
    private String actualWorkOutput;
    private String status;             // PENDING / CONFIRMED / REJECTED
    private String statusLabel;        // 待确认 / 已确认 / 已驳回
    private Long confirmedBy;
    private LocalDateTime confirmedAt;
    private String remark;
    private LocalDateTime createdAt;
}
```

#### 4.5 AttendanceCalendarVO — 日历视图数据

```java
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;

/**
 * 考勤日历视图 — 返回某工人某月的逐日考勤信息
 */
@Data
@Builder
public class AttendanceCalendarVO {

    private Long workerId;
    private String workerName;
    private int year;
    private int month;

    /** 当月各日考勤详情 */
    private List<DayRecord> days;

    /** 当月汇总 */
    private BigDecimal totalWorkDays;     // 总工天
    private int fullDayCount;             // 全天出勤天数
    private int halfDayCount;             // 半天出勤天数
    private int overtimeCount;            // 加班天数
    private int leaveCount;              // 请假天数
    private int absentCount;             // 未记录天数（工作日无考勤）

    @Data
    @Builder
    public static class DayRecord {
        private LocalDate date;
        private String workCategory;      // FULL_DAY / HALF_DAY / OVERTIME / LEAVE / null(无记录)
        private String workCategoryLabel;
        private BigDecimal workDays;
        private String status;            // PENDING / CONFIRMED / REJECTED / null
        private String remark;
    }
}
```

#### 4.6 AttendanceSummaryVO — 考勤统计

```java
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;

import java.math.BigDecimal;

/**
 * 考勤统计 — 用于日统计、月统计展示
 */
@Data
@Builder
public class AttendanceSummaryVO {

    private int totalWorkers;         // 总人数（班组/项目内）
    private int recordedWorkers;      // 已录入人数
    private int unrecordedWorkers;    // 未录入人数
    private int fullDayCount;         // 全天人数
    private int halfDayCount;         // 半天人数
    private int overtimeCount;        // 加班人数
    private int leaveCount;          // 请假人数
    private BigDecimal totalWorkDays; // 总工天数
    private int confirmedCount;       // 已确认数
    private int pendingCount;         // 待确认数
    private int rejectedCount;        // 已驳回数
}
```

#### 4.7 AttendanceBatchResultVO — 批量录入结果

```java
package com.henrywang.ems.vo;

import lombok.Builder;
import lombok.Data;

import java.util.List;

/**
 * 批量考勤录入结果
 */
@Data
@Builder
public class AttendanceBatchResultVO {

    private int totalCount;           // 提交总数
    private int successCount;         // 成功数
    private int failCount;            // 失败数
    private List<FailItem> failItems; // 失败详情

    @Data
    @Builder
    public static class FailItem {
        private Long workerId;
        private String workerName;
        private String reason;        // 失败原因（如"该日期已有考勤记录"）
    }
}
```

---

### Step 5：AttendanceService

#### 5.1 接口定义

```java
package com.henrywang.ems.service;

import com.henrywang.ems.dto.*;
import com.henrywang.ems.vo.*;

import java.time.LocalDate;

public interface AttendanceService {

    /**
     * 分页查询考勤记录（管理列表）
     */
    PageResponse<AttendanceVO> list(Long projectId, Long teamId, Long workerId,
                                     LocalDate workDate, String status,
                                     LocalDate startDate, LocalDate endDate,
                                     int page, int size);

    /**
     * 查询考勤详情
     */
    AttendanceVO getById(Long id);

    /**
     * 单条录入考勤
     */
    AttendanceVO create(AttendanceDTO dto);

    /**
     * 批量录入考勤
     */
    AttendanceBatchResultVO batchCreate(AttendanceBatchDTO dto);

    /**
     * 修改考勤记录（仅 PENDING 状态可修改）
     */
    AttendanceVO update(Long id, AttendanceUpdateDTO dto);

    /**
     * 删除考勤记录（仅 PENDING 状态可删除）
     */
    void delete(Long id);

    /**
     * 确认考勤
     */
    void confirm(Long id);

    /**
     * 批量确认考勤
     */
    void batchConfirm(java.util.List<Long> ids);

    /**
     * 驳回考勤
     */
    void reject(Long id, String reason);

    /**
     * 获取考勤日历数据（某工人某月）
     */
    AttendanceCalendarVO getCalendar(Long workerId, int year, int month);

    /**
     * 获取某天某班组/项目的考勤统计
     */
    AttendanceSummaryVO getDailySummary(Long projectId, Long teamId, LocalDate workDate);
}
```

#### 5.2 实现类

```java
package com.henrywang.ems.service.impl;

import com.henrywang.ems.common.exception.BusinessException;
import com.henrywang.ems.common.util.PageUtils;
import com.henrywang.ems.dto.*;
import com.henrywang.ems.entity.*;
import com.henrywang.ems.entity.enums.AttendanceStatus;
import com.henrywang.ems.entity.enums.WorkCategory;
import com.henrywang.ems.entity.enums.WorkerStatus;
import com.henrywang.ems.repository.*;
import com.henrywang.ems.security.SecurityUtils;
import com.henrywang.ems.service.AttendanceService;
import com.henrywang.ems.vo.*;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.YearMonth;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class AttendanceServiceImpl implements AttendanceService {

    private final AttendanceRepository attendanceRepository;
    private final WorkerRepository workerRepository;
    private final ProjectRepository projectRepository;
    private final TeamRepository teamRepository;

    // ===================== 查询 =====================

    @Override
    public PageResponse<AttendanceVO> list(Long projectId, Long teamId, Long workerId,
                                            LocalDate workDate, String status,
                                            LocalDate startDate, LocalDate endDate,
                                            int page, int size) {
        AttendanceStatus attStatus = null;
        if (StringUtils.hasText(status)) {
            attStatus = AttendanceStatus.valueOf(status);
        }

        Page<Attendance> pageResult = attendanceRepository.findByFilters(
                projectId, teamId, workerId, workDate, attStatus, startDate, endDate,
                PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "workDate", "createdAt")));

        List<AttendanceVO> voList = pageResult.getContent().stream()
                .map(this::toVO)
                .collect(Collectors.toList());

        return PageUtils.toPageResponse(pageResult, voList);
    }

    @Override
    public AttendanceVO getById(Long id) {
        Attendance attendance = attendanceRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "考勤记录不存在"));
        return toVO(attendance);
    }

    // ===================== 单条录入 =====================

    @Override
    @Transactional
    public AttendanceVO create(AttendanceDTO dto) {
        // 1. 校验工人是否存在
        Worker worker = workerRepository.findById(dto.getWorkerId())
                .orElseThrow(() -> new BusinessException(404, "工人不存在"));

        // 2. 校验该工人当天是否已有考勤
        if (attendanceRepository.existsByWorkerIdAndWorkDate(dto.getWorkerId(), dto.getWorkDate())) {
            throw new BusinessException(400, "该工人在 " + dto.getWorkDate() + " 已有考勤记录");
        }

        // 3. 校验项目
        Project project = projectRepository.findById(dto.getProjectId())
                .orElseThrow(() -> new BusinessException(404, "项目不存在"));

        // 4. 校验班组（可选）
        Team team = null;
        if (dto.getTeamId() != null) {
            team = teamRepository.findById(dto.getTeamId())
                    .orElseThrow(() -> new BusinessException(404, "班组不存在"));
        }

        // 5. 解析出勤类型并计算工天数
        WorkCategory category = parseWorkCategory(dto.getWorkCategory());
        BigDecimal workDays = calculateWorkDays(category, dto.getWorkDays(), dto.getOvertimeHours());

        // 6. 快照日薪
        BigDecimal dailyWage = worker.getDailyWage();
        if (dailyWage == null && worker.getWorkType() != null) {
            dailyWage = worker.getWorkType().getDefaultDailyWage();
        }
        if (dailyWage == null) {
            dailyWage = BigDecimal.ZERO;
        }

        // 7. 创建考勤记录
        Attendance attendance = Attendance.builder()
                .worker(worker)
                .project(project)
                .team(team)
                .workDate(dto.getWorkDate())
                .workCategory(category)
                .workDays(workDays)
                .dailyWageSnapshot(dailyWage)
                .isOvertime(category == WorkCategory.OVERTIME ? 1 : 0)
                .overtimeHours(dto.getOvertimeHours())
                .actualWorkOutput(dto.getActualWorkOutput())
                .status(AttendanceStatus.PENDING)
                .remark(dto.getRemark())
                .build();

        return toVO(attendanceRepository.save(attendance));
    }

    // ===================== 批量录入 =====================

    @Override
    @Transactional
    public AttendanceBatchResultVO batchCreate(AttendanceBatchDTO dto) {
        // 1. 校验项目和班组
        Project project = projectRepository.findById(dto.getProjectId())
                .orElseThrow(() -> new BusinessException(404, "项目不存在"));
        Team team = teamRepository.findById(dto.getTeamId())
                .orElseThrow(() -> new BusinessException(404, "班组不存在"));

        // 2. 查询该班组该天已有考勤的工人
        Set<Long> existingWorkerIds = new HashSet<>(
                attendanceRepository.findWorkerIdsByTeamIdAndWorkDate(dto.getTeamId(), dto.getWorkDate()));

        int successCount = 0;
        List<AttendanceBatchResultVO.FailItem> failItems = new ArrayList<>();

        for (AttendanceBatchDTO.AttendanceBatchItem item : dto.getItems()) {
            try {
                // 跳过已有考勤的工人
                if (existingWorkerIds.contains(item.getWorkerId())) {
                    failItems.add(AttendanceBatchResultVO.FailItem.builder()
                            .workerId(item.getWorkerId())
                            .reason("该日期已有考勤记录")
                            .build());
                    continue;
                }

                Worker worker = workerRepository.findById(item.getWorkerId())
                        .orElseThrow(() -> new BusinessException(404, "工人不存在"));

                WorkCategory category = parseWorkCategory(item.getWorkCategory());
                BigDecimal workDays = calculateWorkDays(category, item.getWorkDays(), item.getOvertimeHours());

                // 快照日薪
                BigDecimal dailyWage = worker.getDailyWage();
                if (dailyWage == null && worker.getWorkType() != null) {
                    dailyWage = worker.getWorkType().getDefaultDailyWage();
                }
                if (dailyWage == null) {
                    dailyWage = BigDecimal.ZERO;
                }

                Attendance attendance = Attendance.builder()
                        .worker(worker)
                        .project(project)
                        .team(team)
                        .workDate(dto.getWorkDate())
                        .workCategory(category)
                        .workDays(workDays)
                        .dailyWageSnapshot(dailyWage)
                        .isOvertime(category == WorkCategory.OVERTIME ? 1 : 0)
                        .overtimeHours(item.getOvertimeHours())
                        .status(AttendanceStatus.PENDING)
                        .remark(item.getRemark())
                        .build();

                attendanceRepository.save(attendance);
                existingWorkerIds.add(item.getWorkerId()); // 防止同批次重复
                successCount++;

            } catch (Exception e) {
                failItems.add(AttendanceBatchResultVO.FailItem.builder()
                        .workerId(item.getWorkerId())
                        .reason(e.getMessage())
                        .build());
            }
        }

        return AttendanceBatchResultVO.builder()
                .totalCount(dto.getItems().size())
                .successCount(successCount)
                .failCount(failItems.size())
                .failItems(failItems)
                .build();
    }

    // ===================== 修改 =====================

    @Override
    @Transactional
    public AttendanceVO update(Long id, AttendanceUpdateDTO dto) {
        Attendance attendance = attendanceRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "考勤记录不存在"));

        // 仅 PENDING 状态可修改
        if (attendance.getStatus() != AttendanceStatus.PENDING) {
            throw new BusinessException(400, "已确认或已驳回的考勤不可修改");
        }

        if (StringUtils.hasText(dto.getWorkCategory())) {
            WorkCategory category = parseWorkCategory(dto.getWorkCategory());
            attendance.setWorkCategory(category);
            attendance.setIsOvertime(category == WorkCategory.OVERTIME ? 1 : 0);

            BigDecimal workDays = calculateWorkDays(category, dto.getWorkDays(), dto.getOvertimeHours());
            attendance.setWorkDays(workDays);
        } else if (dto.getWorkDays() != null) {
            attendance.setWorkDays(dto.getWorkDays());
        }

        if (dto.getOvertimeHours() != null) {
            attendance.setOvertimeHours(dto.getOvertimeHours());
        }
        if (dto.getActualWorkOutput() != null) {
            attendance.setActualWorkOutput(dto.getActualWorkOutput());
        }
        if (dto.getRemark() != null) {
            attendance.setRemark(dto.getRemark());
        }

        return toVO(attendanceRepository.save(attendance));
    }

    // ===================== 删除 =====================

    @Override
    @Transactional
    public void delete(Long id) {
        Attendance attendance = attendanceRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "考勤记录不存在"));

        if (attendance.getStatus() != AttendanceStatus.PENDING) {
            throw new BusinessException(400, "已确认或已驳回的考勤不可删除");
        }

        attendanceRepository.delete(attendance);
    }

    // ===================== 审核 =====================

    @Override
    @Transactional
    public void confirm(Long id) {
        Attendance attendance = attendanceRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "考勤记录不存在"));

        if (attendance.getStatus() != AttendanceStatus.PENDING) {
            throw new BusinessException(400, "该考勤记录状态不允许确认");
        }

        attendance.setStatus(AttendanceStatus.CONFIRMED);
        attendance.setConfirmedBy(SecurityUtils.getCurrentUserId());
        attendance.setConfirmedAt(LocalDateTime.now());
        attendanceRepository.save(attendance);
    }

    @Override
    @Transactional
    public void batchConfirm(List<Long> ids) {
        List<Attendance> attendances = attendanceRepository.findAllById(ids);

        for (Attendance attendance : attendances) {
            if (attendance.getStatus() == AttendanceStatus.PENDING) {
                attendance.setStatus(AttendanceStatus.CONFIRMED);
                attendance.setConfirmedBy(SecurityUtils.getCurrentUserId());
                attendance.setConfirmedAt(LocalDateTime.now());
            }
        }

        attendanceRepository.saveAll(attendances);
    }

    @Override
    @Transactional
    public void reject(Long id, String reason) {
        Attendance attendance = attendanceRepository.findById(id)
                .orElseThrow(() -> new BusinessException(404, "考勤记录不存在"));

        if (attendance.getStatus() != AttendanceStatus.PENDING) {
            throw new BusinessException(400, "该考勤记录状态不允许驳回");
        }

        attendance.setStatus(AttendanceStatus.REJECTED);
        attendance.setConfirmedBy(SecurityUtils.getCurrentUserId());
        attendance.setConfirmedAt(LocalDateTime.now());
        attendance.setRemark(StringUtils.hasText(reason) ? reason : attendance.getRemark());
        attendanceRepository.save(attendance);
    }

    // ===================== 日历视图 =====================

    @Override
    public AttendanceCalendarVO getCalendar(Long workerId, int year, int month) {
        Worker worker = workerRepository.findById(workerId)
                .orElseThrow(() -> new BusinessException(404, "工人不存在"));

        YearMonth yearMonth = YearMonth.of(year, month);
        LocalDate startDate = yearMonth.atDay(1);
        LocalDate endDate = yearMonth.atEndOfMonth();

        List<Attendance> records = attendanceRepository.findByWorkerIdAndMonth(workerId, startDate, endDate);

        // 将考勤记录按日期建索引
        Map<LocalDate, Attendance> dateMap = records.stream()
                .collect(Collectors.toMap(Attendance::getWorkDate, a -> a));

        // 构建每日记录
        List<AttendanceCalendarVO.DayRecord> days = new ArrayList<>();
        int fullDayCount = 0, halfDayCount = 0, overtimeCount = 0, leaveCount = 0, absentCount = 0;
        BigDecimal totalWorkDays = BigDecimal.ZERO;

        for (int day = 1; day <= yearMonth.lengthOfMonth(); day++) {
            LocalDate date = yearMonth.atDay(day);
            Attendance att = dateMap.get(date);

            AttendanceCalendarVO.DayRecord record;
            if (att != null) {
                record = AttendanceCalendarVO.DayRecord.builder()
                        .date(date)
                        .workCategory(att.getWorkCategory().name())
                        .workCategoryLabel(getCategoryLabel(att.getWorkCategory()))
                        .workDays(att.getWorkDays())
                        .status(att.getStatus().name())
                        .remark(att.getRemark())
                        .build();

                totalWorkDays = totalWorkDays.add(att.getWorkDays());
                switch (att.getWorkCategory()) {
                    case FULL_DAY -> fullDayCount++;
                    case HALF_DAY -> halfDayCount++;
                    case OVERTIME -> overtimeCount++;
                    case LEAVE -> leaveCount++;
                }
            } else {
                record = AttendanceCalendarVO.DayRecord.builder()
                        .date(date)
                        .build();
                // 仅统计已过去的日期（不含未来）且为工作日
                if (!date.isAfter(LocalDate.now())) {
                    absentCount++;
                }
            }
            days.add(record);
        }

        return AttendanceCalendarVO.builder()
                .workerId(workerId)
                .workerName(worker.getName())
                .year(year)
                .month(month)
                .days(days)
                .totalWorkDays(totalWorkDays)
                .fullDayCount(fullDayCount)
                .halfDayCount(halfDayCount)
                .overtimeCount(overtimeCount)
                .leaveCount(leaveCount)
                .absentCount(absentCount)
                .build();
    }

    // ===================== 日统计 =====================

    @Override
    public AttendanceSummaryVO getDailySummary(Long projectId, Long teamId, LocalDate workDate) {
        // 统计班组/项目内总人数
        int totalWorkers;
        if (teamId != null) {
            totalWorkers = workerRepository.findByCurrentTeamId(teamId).size();
        } else if (projectId != null) {
            totalWorkers = workerRepository.findByCurrentProjectId(projectId).size();
        } else {
            totalWorkers = 0;
        }

        // 查询当天考勤
        List<Attendance> records;
        if (teamId != null) {
            records = attendanceRepository.findByTeamIdAndWorkDate(teamId, workDate);
        } else if (projectId != null) {
            records = attendanceRepository.findByProjectIdAndWorkDate(projectId, workDate);
        } else {
            records = List.of();
        }

        int fullDayCount = 0, halfDayCount = 0, overtimeCount = 0, leaveCount = 0;
        int confirmedCount = 0, pendingCount = 0, rejectedCount = 0;
        BigDecimal totalWorkDays = BigDecimal.ZERO;

        for (Attendance att : records) {
            switch (att.getWorkCategory()) {
                case FULL_DAY -> fullDayCount++;
                case HALF_DAY -> halfDayCount++;
                case OVERTIME -> overtimeCount++;
                case LEAVE -> leaveCount++;
            }
            totalWorkDays = totalWorkDays.add(att.getWorkDays());

            switch (att.getStatus()) {
                case CONFIRMED -> confirmedCount++;
                case PENDING -> pendingCount++;
                case REJECTED -> rejectedCount++;
            }
        }

        return AttendanceSummaryVO.builder()
                .totalWorkers(totalWorkers)
                .recordedWorkers(records.size())
                .unrecordedWorkers(totalWorkers - records.size())
                .fullDayCount(fullDayCount)
                .halfDayCount(halfDayCount)
                .overtimeCount(overtimeCount)
                .leaveCount(leaveCount)
                .totalWorkDays(totalWorkDays)
                .confirmedCount(confirmedCount)
                .pendingCount(pendingCount)
                .rejectedCount(rejectedCount)
                .build();
    }

    // ===================== 工具方法 =====================

    /**
     * 解析出勤类型字符串
     */
    private WorkCategory parseWorkCategory(String category) {
        try {
            return WorkCategory.valueOf(category);
        } catch (IllegalArgumentException e) {
            throw new BusinessException(400,
                    "无效的出勤类型: " + category + "，可选值: FULL_DAY, HALF_DAY, OVERTIME, LEAVE");
        }
    }

    /**
     * 计算折算工天数
     * @param category  出勤类型
     * @param inputDays 前端传入的工天数（可选）
     * @param overtimeHours 加班时长（可选）
     * @return 折算工天数
     */
    private BigDecimal calculateWorkDays(WorkCategory category, BigDecimal inputDays, BigDecimal overtimeHours) {
        // 如果前端显式传入了工天数，直接使用
        if (inputDays != null) {
            return inputDays;
        }
        // 否则按类型默认计算
        return switch (category) {
            case FULL_DAY -> new BigDecimal("1.0");
            case HALF_DAY -> new BigDecimal("0.5");
            case OVERTIME -> new BigDecimal("1.5");  // 默认加班 1.5 工天，可被前端覆盖
            case LEAVE -> BigDecimal.ZERO;
        };
    }

    /**
     * Entity → VO 转换
     */
    private AttendanceVO toVO(Attendance attendance) {
        BigDecimal dailyEarning = attendance.getWorkDays()
                .multiply(attendance.getDailyWageSnapshot());

        return AttendanceVO.builder()
                .id(attendance.getId())
                .workerId(attendance.getWorker().getId())
                .workerName(attendance.getWorker().getName())
                .workerWorkTypeName(attendance.getWorker().getWorkType() != null
                        ? attendance.getWorker().getWorkType().getName() : null)
                .projectId(attendance.getProject().getId())
                .projectName(attendance.getProject().getName())
                .teamId(attendance.getTeam() != null ? attendance.getTeam().getId() : null)
                .teamName(attendance.getTeam() != null ? attendance.getTeam().getName() : null)
                .workDate(attendance.getWorkDate())
                .workCategory(attendance.getWorkCategory().name())
                .workCategoryLabel(getCategoryLabel(attendance.getWorkCategory()))
                .workDays(attendance.getWorkDays())
                .dailyWageSnapshot(attendance.getDailyWageSnapshot())
                .dailyEarning(dailyEarning)
                .isOvertime(attendance.getIsOvertime())
                .overtimeHours(attendance.getOvertimeHours())
                .actualWorkOutput(attendance.getActualWorkOutput())
                .status(attendance.getStatus().name())
                .statusLabel(getStatusLabel(attendance.getStatus()))
                .confirmedBy(attendance.getConfirmedBy())
                .confirmedAt(attendance.getConfirmedAt())
                .remark(attendance.getRemark())
                .createdAt(attendance.getCreatedAt())
                .build();
    }

    /**
     * 出勤类型 → 中文标签
     */
    private String getCategoryLabel(WorkCategory category) {
        return switch (category) {
            case FULL_DAY -> "全天";
            case HALF_DAY -> "半天";
            case OVERTIME -> "加班";
            case LEAVE -> "请假";
        };
    }

    /**
     * 审核状态 → 中文标签
     */
    private String getStatusLabel(AttendanceStatus status) {
        return switch (status) {
            case PENDING -> "待确认";
            case CONFIRMED -> "已确认";
            case REJECTED -> "已驳回";
        };
    }
}
```

---

### Step 6：AttendanceController

```java
package com.henrywang.ems.controller;

import com.henrywang.ems.common.result.R;
import com.henrywang.ems.dto.*;
import com.henrywang.ems.service.AttendanceService;
import com.henrywang.ems.vo.*;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

@Tag(name = "考勤/工天管理", description = "考勤录入、批量录入、审核、日历查询")
@RestController
@RequestMapping("/api/attendances")
@RequiredArgsConstructor
public class AttendanceController {

    private final AttendanceService attendanceService;

    // ==================== 查询接口 ====================

    @Operation(summary = "考勤记录列表（分页）")
    @GetMapping
    public R<PageResponse<AttendanceVO>> list(
            @RequestParam(required = false) Long projectId,
            @RequestParam(required = false) Long teamId,
            @RequestParam(required = false) Long workerId,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate workDate,
            @RequestParam(required = false) String status,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate startDate,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate endDate,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return R.ok(attendanceService.list(projectId, teamId, workerId,
                workDate, status, startDate, endDate, page, size));
    }

    @Operation(summary = "考勤详情")
    @GetMapping("/{id}")
    public R<AttendanceVO> getById(@PathVariable Long id) {
        return R.ok(attendanceService.getById(id));
    }

    @Operation(summary = "考勤日历数据（某工人某月）")
    @GetMapping("/calendar")
    public R<AttendanceCalendarVO> getCalendar(
            @RequestParam Long workerId,
            @RequestParam int year,
            @RequestParam int month) {
        return R.ok(attendanceService.getCalendar(workerId, year, month));
    }

    @Operation(summary = "考勤日统计（某天某班组/项目）")
    @GetMapping("/daily-summary")
    public R<AttendanceSummaryVO> getDailySummary(
            @RequestParam(required = false) Long projectId,
            @RequestParam(required = false) Long teamId,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate workDate) {
        return R.ok(attendanceService.getDailySummary(projectId, teamId, workDate));
    }

    // ==================== 录入接口 ====================

    @Operation(summary = "录入考勤（单条）")
    @PostMapping
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN')")
    public R<AttendanceVO> create(@RequestBody @Valid AttendanceDTO dto) {
        return R.ok(attendanceService.create(dto));
    }

    @Operation(summary = "批量录入考勤")
    @PostMapping("/batch")
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN')")
    public R<AttendanceBatchResultVO> batchCreate(@RequestBody @Valid AttendanceBatchDTO dto) {
        return R.ok(attendanceService.batchCreate(dto));
    }

    // ==================== 修改/删除接口 ====================

    @Operation(summary = "修改考勤记录")
    @PutMapping("/{id}")
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN')")
    public R<AttendanceVO> update(@PathVariable Long id, @RequestBody @Valid AttendanceUpdateDTO dto) {
        return R.ok(attendanceService.update(id, dto));
    }

    @Operation(summary = "删除考勤记录")
    @DeleteMapping("/{id}")
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    public R<Void> delete(@PathVariable Long id) {
        attendanceService.delete(id);
        return R.ok();
    }

    // ==================== 审核接口 ====================

    @Operation(summary = "确认考勤")
    @PostMapping("/{id}/confirm")
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN')")
    public R<Void> confirm(@PathVariable Long id) {
        attendanceService.confirm(id);
        return R.ok();
    }

    @Operation(summary = "批量确认考勤")
    @PostMapping("/batch-confirm")
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN')")
    public R<Void> batchConfirm(@RequestBody List<Long> ids) {
        attendanceService.batchConfirm(ids);
        return R.ok();
    }

    @Operation(summary = "驳回考勤")
    @PostMapping("/{id}/reject")
    @PreAuthorize("hasAnyRole('SUPER_ADMIN', 'PROJECT_MANAGER')")
    public R<Void> reject(@PathVariable Long id, @RequestBody Map<String, String> body) {
        attendanceService.reject(id, body.get("reason"));
        return R.ok();
    }
}
```

#### 权限控制矩阵（Phase 2 新增）

| 接口 | SUPER_ADMIN | PROJECT_MANAGER | FOREMAN | WORKER |
|------|:-----------:|:---------------:|:-------:|:------:|
| 考勤列表查看 | ✅ | ✅ (本项目) | ✅ (本班组) | ✅ (仅自己) |
| 考勤详情 | ✅ | ✅ | ✅ | ✅ (仅自己) |
| 单条录入 | ✅ | ✅ | ✅ | ❌ |
| 批量录入 | ✅ | ✅ | ✅ | ❌ |
| 修改考勤 | ✅ | ✅ | ✅ (仅本班组 PENDING) | ❌ |
| 删除考勤 | ✅ | ✅ | ❌ | ❌ |
| 确认考勤 | ✅ | ✅ | ✅ (仅本班组) | ❌ |
| 批量确认 | ✅ | ✅ | ✅ (仅本班组) | ❌ |
| 驳回考勤 | ✅ | ✅ | ❌ | ❌ |
| 日历视图 | ✅ | ✅ | ✅ | ✅ (仅自己) |
| 日统计 | ✅ | ✅ | ✅ | ❌ |

> **Phase 2 实现方式**：使用 `@PreAuthorize` 做角色级别控制。数据级别的行级过滤（如"工长仅看本班组"）后续在 Service 层增强，本阶段先允许角色级别访问。

---

### Step 7：后端接口联调测试

使用 Swagger UI (`http://localhost:8080/swagger-ui.html`) 进行接口验证。

#### 7.1 测试场景清单

| # | 测试场景 | HTTP 方法 & URL | 请求数据 | 预期结果 |
|---|---------|-----------------|---------|---------|
| 1 | 单条录入-全天 | POST `/api/attendances` | `{"workerId":1, "projectId":1, "teamId":1, "workDate":"2026-02-18", "workCategory":"FULL_DAY"}` | 200，workDays=1.0，dailyWageSnapshot=工人日薪 |
| 2 | 单条录入-半天 | POST `/api/attendances` | `{"workerId":2, "projectId":1, "teamId":1, "workDate":"2026-02-18", "workCategory":"HALF_DAY"}` | 200，workDays=0.5 |
| 3 | 单条录入-加班 | POST `/api/attendances` | `{"workerId":3, "projectId":1, "teamId":1, "workDate":"2026-02-18", "workCategory":"OVERTIME", "workDays":2.0, "overtimeHours":4}` | 200，workDays=2.0 |
| 4 | 单条录入-请假 | POST `/api/attendances` | `{"workerId":4, "projectId":1, "teamId":1, "workDate":"2026-02-18", "workCategory":"LEAVE"}` | 200，workDays=0.0 |
| 5 | 重复录入拦截 | POST `/api/attendances` | 同一 workerId + workDate | 400，"该工人在 2026-02-18 已有考勤记录" |
| 6 | 批量录入 | POST `/api/attendances/batch` | `{"projectId":1, "teamId":1, "workDate":"2026-02-19", "items":[{"workerId":1,"workCategory":"FULL_DAY"},{"workerId":2,"workCategory":"HALF_DAY"}]}` | 200，successCount=2 |
| 7 | 批量录入含冲突 | POST `/api/attendances/batch` | items 中包含已有考勤的工人 | 200，failCount>0，failItems 包含原因 |
| 8 | 查询列表 | GET `/api/attendances?projectId=1&workDate=2026-02-18` | - | 200，返回该项目当天的考勤列表 |
| 9 | 按状态查询 | GET `/api/attendances?status=PENDING` | - | 200，仅返回待确认的记录 |
| 10 | 按日期范围查询 | GET `/api/attendances?startDate=2026-02-01&endDate=2026-02-28` | - | 200，返回月度考勤 |
| 11 | 修改考勤 | PUT `/api/attendances/1` | `{"workCategory":"HALF_DAY"}` | 200，workDays 更新为 0.5 |
| 12 | 修改已确认考勤 | PUT `/api/attendances/1` (status=CONFIRMED) | `{"workCategory":"LEAVE"}` | 400，"已确认或已驳回的考勤不可修改" |
| 13 | 确认考勤 | POST `/api/attendances/1/confirm` | - | 200，status 变为 CONFIRMED |
| 14 | 批量确认 | POST `/api/attendances/batch-confirm` | `[1, 2, 3]` | 200 |
| 15 | 驳回考勤 | POST `/api/attendances/1/reject` | `{"reason":"出勤时间有误"}` | 200，status 变为 REJECTED |
| 16 | 删除考勤 | DELETE `/api/attendances/1` | - | 200（仅 PENDING 可删） |
| 17 | 日历视图 | GET `/api/attendances/calendar?workerId=1&year=2026&month=2` | - | 200，返回2月逐日考勤 |
| 18 | 日统计 | GET `/api/attendances/daily-summary?teamId=1&workDate=2026-02-18` | - | 200，返回统计数据 |
| 19 | 无Token访问 | GET `/api/attendances` (无 Authorization) | - | 401 |
| 20 | WORKER角色录入 | POST `/api/attendances` (用 WORKER Token) | - | 403 |

---

## 三、前端实现（uni-app）

### Step 8：前端 API 模块

```javascript
// frontend/src/api/modules/attendance.js
import { get, post, put, del } from '../request'

/**
 * 考勤记录列表（分页）
 * @param {Object} params - { projectId, teamId, workerId, workDate, status, startDate, endDate, page, size }
 */
export const listAttendances = (params) => get('/attendances', params)

/**
 * 考勤详情
 */
export const getAttendance = (id) => get(`/attendances/${id}`)

/**
 * 单条录入考勤
 */
export const createAttendance = (data) => post('/attendances', data)

/**
 * 批量录入考勤
 * @param {Object} data - { projectId, teamId, workDate, items: [{ workerId, workCategory, workDays?, overtimeHours?, remark? }] }
 */
export const batchCreateAttendance = (data) => post('/attendances/batch', data)

/**
 * 修改考勤
 */
export const updateAttendance = (id, data) => put(`/attendances/${id}`, data)

/**
 * 删除考勤
 */
export const deleteAttendance = (id) => del(`/attendances/${id}`)

/**
 * 确认考勤
 */
export const confirmAttendance = (id) => post(`/attendances/${id}/confirm`)

/**
 * 批量确认考勤
 * @param {Array<number>} ids
 */
export const batchConfirmAttendance = (ids) => post('/attendances/batch-confirm', ids)

/**
 * 驳回考勤
 */
export const rejectAttendance = (id, reason) => post(`/attendances/${id}/reject`, { reason })

/**
 * 考勤日历数据
 * @param {Object} params - { workerId, year, month }
 */
export const getAttendanceCalendar = (params) => get('/attendances/calendar', params)

/**
 * 考勤日统计
 * @param {Object} params - { projectId?, teamId?, workDate }
 */
export const getAttendanceDailySummary = (params) => get('/attendances/daily-summary', params)
```

---

### Step 9：工天批量录入页面（核心高频页面）

> **这是全系统最重要的页面**，工长/班组长每天都会使用。必须严格遵循适老化 UI 设计原则。

#### 9.1 页面路径

`frontend/src/pages/attendance/batch-input.vue`

#### 9.2 交互设计

```
┌────────────────────────────────────────────────┐
│  📅 2026年2月18日(三)    [< 前一天] [后一天 >]    │
│  🏗️ 项目: [XX工地 ▼]   👥 班组: [钢筋班 ▼]      │
├────────────────────────────────────────────────┤
│                                                │
│  ── 今日统计 ──                                 │
│  全天: 3人  半天: 1人  加班: 0人  请假: 1人       │
│  总工天: 3.5    已录入: 5/8人                    │
│                                                │
├────────────────────────────────────────────────┤
│                                                │
│  ┌───────────┬──────────────────────────┐      │
│  │  张 三     │ [全天✓] [半天] [加班] [休] │      │
│  │  钢筋工    │                          │      │
│  │  ¥350/天   │                          │      │
│  ├───────────┼──────────────────────────┤      │
│  │  李 四     │ [全天] [半天✓] [加班] [休] │      │
│  │  木  工    │                          │      │
│  │  ¥300/天   │                          │      │
│  ├───────────┼──────────────────────────┤      │
│  │  王 五     │ [全天] [半天] [加班] [休✓] │      │
│  │  泥瓦工    │                          │      │
│  │  ¥280/天   │                          │      │
│  ├───────────┼──────────────────────────┤      │
│  │  赵 六     │ 已录入：全天 ✅            │      │
│  │  电  工    │ (点击可修改)               │      │
│  │  ¥320/天   │                          │      │
│  └───────────┴──────────────────────────┘      │
│                                                │
│  ┌────────────────────────────────────────┐    │
│  │        ✅ 提 交 考 勤 (大按钮 56px)      │    │
│  └────────────────────────────────────────┘    │
│                                                │
└────────────────────────────────────────────────┘
```

#### 9.3 核心交互逻辑

1. **进入页面**：自动加载当前用户所属班组（或让工长选择），默认日期为今天
2. **加载班组工人列表**：调用 `GET /api/teams/{teamId}/members` 获取班组工人
3. **加载当天已有考勤**：调用 `GET /api/attendances?teamId=xxx&workDate=yyyy-mm-dd` 获取已录入数据
4. **合并展示**：
   - 未录入的工人：展示 4 个按钮（全天/半天/加班/休），默认预选"全天"
   - 已录入的工人：展示已录入的类型，提供"修改"入口
5. **选择出勤类型**：点击按钮即选中，无需额外输入。每人只需点击一次
6. **提交考勤**：点击底部大按钮，调用 `POST /api/attendances/batch`，将所有新选择的考勤一次性提交
7. **提交结果**：
   - 全部成功：Toast "考勤提交成功"
   - 部分失败：弹窗展示失败详情
8. **切换日期**：前一天/后一天按钮，不允许选择未来日期
9. **切换班组/项目**：顶部下拉选择器

#### 9.4 页面代码骨架

```vue
<!-- frontend/src/pages/attendance/batch-input.vue -->
<template>
  <view class="page batch-input-page">
    <!-- 顶部：日期选择 -->
    <view class="date-bar">
      <view class="date-nav" @click="prevDay">
        <text class="nav-btn">‹ 前一天</text>
      </view>
      <view class="date-display" @click="showDatePicker">
        <text class="date-text">📅 {{ formatDate(currentDate) }}</text>
      </view>
      <view class="date-nav" @click="nextDay">
        <text class="nav-btn" :class="{ disabled: isToday }">后一天 ›</text>
      </view>
    </view>

    <!-- 项目 + 班组选择 -->
    <view class="selector-bar">
      <picker :range="projectOptions" range-key="name" @change="onProjectChange">
        <view class="selector-item">
          <text>🏗️ {{ selectedProject?.name || '选择项目' }}</text>
        </view>
      </picker>
      <picker :range="teamOptions" range-key="name" @change="onTeamChange">
        <view class="selector-item">
          <text>👥 {{ selectedTeam?.name || '选择班组' }}</text>
        </view>
      </picker>
    </view>

    <!-- 今日统计 -->
    <view class="summary-bar" v-if="workers.length > 0">
      <view class="summary-item">
        <text class="summary-label">全天</text>
        <text class="summary-value full">{{ summary.fullDay }}人</text>
      </view>
      <view class="summary-item">
        <text class="summary-label">半天</text>
        <text class="summary-value half">{{ summary.halfDay }}人</text>
      </view>
      <view class="summary-item">
        <text class="summary-label">加班</text>
        <text class="summary-value overtime">{{ summary.overtime }}人</text>
      </view>
      <view class="summary-item">
        <text class="summary-label">请假</text>
        <text class="summary-value leave">{{ summary.leave }}人</text>
      </view>
      <view class="summary-item">
        <text class="summary-label">总工天</text>
        <text class="summary-value total">{{ summary.totalWorkDays }}</text>
      </view>
    </view>

    <!-- 工人列表 -->
    <view class="worker-list">
      <view
        v-for="item in workers"
        :key="item.worker.id"
        class="worker-row"
        :class="{ recorded: item.recorded }"
      >
        <!-- 左侧：工人信息 -->
        <view class="worker-info">
          <text class="worker-name">{{ item.worker.name }}</text>
          <text class="worker-type">{{ item.worker.workTypeName || '' }}</text>
          <text class="worker-wage">¥{{ item.worker.dailyWage || 0 }}/天</text>
        </view>

        <!-- 右侧：出勤按钮组 -->
        <view class="category-buttons" v-if="!item.recorded">
          <view
            class="cat-btn full"
            :class="{ selected: item.selected === 'FULL_DAY' }"
            @click="selectCategory(item, 'FULL_DAY')"
          >
            <text>全天</text>
          </view>
          <view
            class="cat-btn half"
            :class="{ selected: item.selected === 'HALF_DAY' }"
            @click="selectCategory(item, 'HALF_DAY')"
          >
            <text>半天</text>
          </view>
          <view
            class="cat-btn overtime"
            :class="{ selected: item.selected === 'OVERTIME' }"
            @click="selectCategory(item, 'OVERTIME')"
          >
            <text>加班</text>
          </view>
          <view
            class="cat-btn leave"
            :class="{ selected: item.selected === 'LEAVE' }"
            @click="selectCategory(item, 'LEAVE')"
          >
            <text>休</text>
          </view>
        </view>

        <!-- 已录入的显示 -->
        <view class="recorded-info" v-else>
          <text class="recorded-label">
            {{ getCategoryLabel(item.existingCategory) }} ✅
          </text>
          <text class="recorded-edit" @click="editExisting(item)">修改</text>
        </view>
      </view>
    </view>

    <!-- 空状态 -->
    <view v-if="workers.length === 0 && !loading" class="empty">
      <text class="empty-text">{{ selectedTeam ? '当前班组暂无工人' : '请选择项目和班组' }}</text>
    </view>

    <!-- 提交按钮 -->
    <view class="submit-bar" v-if="hasNewSelections">
      <view class="submit-btn" :class="{ disabled: submitting }" @click="submitBatch">
        <text class="submit-text">✅ 提 交 考 勤</text>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { onLoad, onShow } from '@dcloudio/uni-app'
import { listProjects } from '@/api/modules/project'
import { listTeams, getTeamMembers } from '@/api/modules/team'
import { listAttendances, batchCreateAttendance, updateAttendance } from '@/api/modules/attendance'

// ===== 响应式数据 =====
const currentDate = ref(new Date().toISOString().split('T')[0])  // 'yyyy-MM-dd'
const projectOptions = ref([])
const teamOptions = ref([])
const selectedProject = ref(null)
const selectedTeam = ref(null)
const workers = ref([])  // [{ worker, selected, recorded, existingCategory, existingId }]
const loading = ref(false)
const submitting = ref(false)

// ===== 日期操作 =====
const isToday = computed(() => currentDate.value === new Date().toISOString().split('T')[0])

const formatDate = (dateStr) => {
  const d = new Date(dateStr)
  const weekDays = ['日', '一', '二', '三', '四', '五', '六']
  return `${d.getFullYear()}年${d.getMonth() + 1}月${d.getDate()}日(${weekDays[d.getDay()]})`
}

const prevDay = () => {
  const d = new Date(currentDate.value)
  d.setDate(d.getDate() - 1)
  currentDate.value = d.toISOString().split('T')[0]
}

const nextDay = () => {
  if (isToday.value) return
  const d = new Date(currentDate.value)
  d.setDate(d.getDate() + 1)
  currentDate.value = d.toISOString().split('T')[0]
}

// ===== 统计 =====
const summary = computed(() => {
  let fullDay = 0, halfDay = 0, overtime = 0, leave = 0, totalWorkDays = 0
  for (const item of workers.value) {
    const cat = item.recorded ? item.existingCategory : item.selected
    if (!cat) continue
    switch (cat) {
      case 'FULL_DAY': fullDay++; totalWorkDays += 1; break
      case 'HALF_DAY': halfDay++; totalWorkDays += 0.5; break
      case 'OVERTIME': overtime++; totalWorkDays += 1.5; break
      case 'LEAVE': leave++; break
    }
  }
  return { fullDay, halfDay, overtime, leave, totalWorkDays }
})

const hasNewSelections = computed(() =>
  workers.value.some(item => !item.recorded && item.selected)
)

// ===== 数据加载 =====
const loadProjects = async () => {
  try {
    const res = await listProjects({ status: 'ACTIVE' })
    projectOptions.value = res.data?.content || res.data || []
  } catch (e) { console.error('加载项目失败', e) }
}

const loadTeams = async () => {
  if (!selectedProject.value) { teamOptions.value = []; return }
  try {
    const res = await listTeams({ projectId: selectedProject.value.id })
    teamOptions.value = res.data || []
  } catch (e) { console.error('加载班组失败', e) }
}

const loadWorkers = async () => {
  if (!selectedTeam.value) { workers.value = []; return }
  loading.value = true
  try {
    // 1. 获取班组成员
    const membersRes = await getTeamMembers(selectedTeam.value.id)
    const memberList = membersRes.data || []

    // 2. 获取已有考勤
    const attRes = await listAttendances({
      teamId: selectedTeam.value.id,
      workDate: currentDate.value,
      size: 100
    })
    const existingRecords = attRes.data?.content || []
    const existingMap = {}
    for (const rec of existingRecords) {
      existingMap[rec.workerId] = rec
    }

    // 3. 合并
    workers.value = memberList.map(w => {
      const existing = existingMap[w.id]
      return {
        worker: w,
        selected: existing ? null : 'FULL_DAY',   // 默认预选"全天"
        recorded: !!existing,
        existingCategory: existing?.workCategory || null,
        existingId: existing?.id || null
      }
    })
  } catch (e) {
    console.error('加载数据失败', e)
  } finally {
    loading.value = false
  }
}

// ===== 交互操作 =====
const selectCategory = (item, category) => {
  item.selected = category
}

const getCategoryLabel = (cat) => {
  const labels = { FULL_DAY: '全天', HALF_DAY: '半天', OVERTIME: '加班', LEAVE: '请假' }
  return labels[cat] || cat
}

const editExisting = (item) => {
  // 将已录入的记录切换为可编辑模式
  item.recorded = false
  item.selected = item.existingCategory
}

// ===== 提交 =====
const submitBatch = async () => {
  if (submitting.value) return

  const newItems = workers.value
    .filter(item => !item.recorded && item.selected)
    .map(item => ({
      workerId: item.worker.id,
      workCategory: item.selected
    }))

  // 分离：需要更新的（existingId 存在）vs 新建的
  const toCreate = []
  const toUpdate = []
  for (const item of workers.value) {
    if (item.recorded || !item.selected) continue
    if (item.existingId) {
      toUpdate.push({ id: item.existingId, workCategory: item.selected })
    } else {
      toCreate.push({ workerId: item.worker.id, workCategory: item.selected })
    }
  }

  submitting.value = true
  try {
    let success = true

    // 批量新建
    if (toCreate.length > 0) {
      const res = await batchCreateAttendance({
        projectId: selectedProject.value.id,
        teamId: selectedTeam.value.id,
        workDate: currentDate.value,
        items: toCreate
      })
      if (res.data?.failCount > 0) {
        const failNames = res.data.failItems.map(f => f.workerName || f.workerId).join('、')
        uni.showModal({
          title: '部分提交失败',
          content: `失败 ${res.data.failCount} 人：${failNames}`,
          showCancel: false
        })
        success = false
      }
    }

    // 逐条更新
    for (const item of toUpdate) {
      await updateAttendance(item.id, { workCategory: item.workCategory })
    }

    if (success) {
      uni.showToast({ title: '考勤提交成功', icon: 'success' })
    }

    // 重新加载
    await loadWorkers()
  } catch (e) {
    uni.showToast({ title: '提交失败', icon: 'none' })
  } finally {
    submitting.value = false
  }
}

// ===== 选择器回调 =====
const onProjectChange = (e) => {
  selectedProject.value = projectOptions.value[e.detail.value]
  selectedTeam.value = null
  loadTeams()
}

const onTeamChange = (e) => {
  selectedTeam.value = teamOptions.value[e.detail.value]
  loadWorkers()
}

// ===== 日期变化时重新加载 =====
watch(currentDate, () => { if (selectedTeam.value) loadWorkers() })

// ===== 生命周期 =====
onLoad(() => { loadProjects() })
onShow(() => { if (selectedTeam.value) loadWorkers() })
</script>

<style lang="scss" scoped>
/* ===== 适老化样式：大字体、大按钮、高对比度 ===== */

.batch-input-page {
  padding-bottom: 120rpx;
  background: #F5F6FA;
}

/* 日期栏 */
.date-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 24rpx 32rpx;
  background: #fff;
}
.date-text {
  font-size: 36rpx;
  font-weight: bold;
  color: #333;
}
.nav-btn {
  font-size: 32rpx;
  color: #1890FF;
  padding: 16rpx 24rpx;
  &.disabled { color: #ccc; }
}

/* 选择器栏 */
.selector-bar {
  display: flex;
  gap: 24rpx;
  padding: 16rpx 32rpx;
  background: #fff;
  border-top: 1rpx solid #eee;
}
.selector-item {
  flex: 1;
  padding: 16rpx 24rpx;
  background: #F0F2F5;
  border-radius: 8rpx;
  text-align: center;
  font-size: 30rpx;
  color: #333;
}

/* 统计栏 */
.summary-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 16rpx;
  padding: 20rpx 32rpx;
  background: #fff;
  margin-top: 16rpx;
}
.summary-item {
  display: flex;
  align-items: center;
  gap: 8rpx;
}
.summary-label { font-size: 28rpx; color: #666; }
.summary-value { font-size: 30rpx; font-weight: bold; }
.summary-value.full { color: #52C41A; }
.summary-value.half { color: #1890FF; }
.summary-value.overtime { color: #FA8C16; }
.summary-value.leave { color: #999; }
.summary-value.total { color: #333; }

/* 工人行 */
.worker-list {
  margin-top: 16rpx;
}
.worker-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 24rpx 32rpx;
  background: #fff;
  border-bottom: 1rpx solid #F0F2F5;
  min-height: 120rpx;  /* 大触控区域 */
}
.worker-row.recorded {
  background: #FAFAFA;
}

/* 工人信息 */
.worker-info {
  display: flex;
  flex-direction: column;
  gap: 4rpx;
  min-width: 160rpx;
}
.worker-name {
  font-size: 34rpx;  /* 大字体 */
  font-weight: bold;
  color: #333;
}
.worker-type {
  font-size: 26rpx;
  color: #999;
}
.worker-wage {
  font-size: 28rpx;
  color: #FA8C16;
  font-weight: 500;
}

/* 出勤按钮组 */
.category-buttons {
  display: flex;
  gap: 12rpx;
}
.cat-btn {
  padding: 16rpx 28rpx;
  border-radius: 8rpx;
  border: 2rpx solid #D9D9D9;
  background: #fff;
  min-height: 72rpx;  /* 大按钮 ≥ 36px */
  display: flex;
  align-items: center;
  justify-content: center;

  text {
    font-size: 30rpx;
    color: #666;
  }

  &.selected {
    border-color: transparent;
    text { color: #fff; font-weight: bold; }
  }
  &.full.selected { background: #52C41A; }
  &.half.selected { background: #1890FF; }
  &.overtime.selected { background: #FA8C16; }
  &.leave.selected { background: #999; }
}

/* 已录入状态 */
.recorded-info {
  display: flex;
  align-items: center;
  gap: 16rpx;
}
.recorded-label {
  font-size: 30rpx;
  color: #52C41A;
  font-weight: bold;
}
.recorded-edit {
  font-size: 28rpx;
  color: #1890FF;
  padding: 8rpx 16rpx;
}

/* 提交按钮 */
.submit-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 24rpx 32rpx;
  background: #fff;
  box-shadow: 0 -4rpx 12rpx rgba(0,0,0,0.08);
}
.submit-btn {
  background: #1890FF;
  border-radius: 12rpx;
  padding: 28rpx 0;   /* 56px 大按钮 */
  text-align: center;
  &.disabled { opacity: 0.5; }
}
.submit-text {
  font-size: 36rpx;
  color: #fff;
  font-weight: bold;
  letter-spacing: 8rpx;
}

/* 空状态 */
.empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 100rpx 0;
}
.empty-text {
  font-size: 30rpx;
  color: #999;
  margin-top: 16rpx;
}
</style>
```

---

### Step 10：考勤管理列表页面

#### 10.1 页面路径

`frontend/src/pages/attendance/index.vue`

#### 10.2 交互设计

```
┌──────────────────────────────────────────────┐
│  考勤管理                                     │
├──────────────────────────────────────────────┤
│  日期: [2026-02-18 ▼]  项目: [全部 ▼]         │
│  班组: [全部 ▼]  状态: [全部 ▼]  [清除筛选]    │
├──────────────────────────────────────────────┤
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ 张三  钢筋工  |  2月18日  全天  1.0工    │  │
│  │ XX工地 钢筋班  |  ¥350   待确认 ⏳      │  │
│  │                    [确认✓] [驳回✗]      │  │
│  ├────────────────────────────────────────┤  │
│  │ 李四  木工    |  2月18日  半天  0.5工    │  │
│  │ XX工地 木工班  |  ¥150   已确认 ✅      │  │
│  ├────────────────────────────────────────┤  │
│  │ 王五  泥瓦工  |  2月18日  请假  0工     │  │
│  │ YY广场 泥瓦班  |  ¥0    已驳回 ❌       │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ── 批量操作 ──                               │
│  [全选] [批量确认]                             │
│                                              │
└──────────────────────────────────────────────┘
```

#### 10.3 功能要点

- **筛选条件**：日期（默认今天）、项目、班组、状态（待确认/已确认/已驳回）
- **列表展示**：工人姓名、工种、日期、出勤类型、工天数、日薪、收入、状态
- **操作按钮**：
  - 待确认记录：显示"确认"和"驳回"按钮
  - 已确认/已驳回记录：仅显示状态标签
- **批量操作**：多选 + 批量确认
- **下拉刷新 + 上拉加载更多**
- **点击卡片**：跳转考勤详情/编辑
- **FAB 按钮**：跳转到批量录入页

#### 10.4 页面代码骨架

```vue
<!-- frontend/src/pages/attendance/index.vue -->
<template>
  <view class="page attendance-page">
    <!-- 筛选栏 -->
    <view class="filter-bar">
      <picker mode="date" :value="filters.workDate" @change="onDateChange">
        <view class="filter-item date">
          <text>📅 {{ filters.workDate || '选择日期' }}</text>
        </view>
      </picker>
      <picker :range="projectOptions" range-key="name" @change="onProjectFilterChange">
        <view class="filter-item">
          <text>{{ selectedFilterProject || '项目' }}</text>
        </view>
      </picker>
      <picker :range="statusOptions" range-key="label" @change="onStatusFilterChange">
        <view class="filter-item">
          <text>{{ selectedFilterStatus || '状态' }}</text>
        </view>
      </picker>
    </view>

    <!-- 考勤列表 -->
    <view class="att-list">
      <view v-for="att in list" :key="att.id" class="att-card">
        <view class="att-header">
          <view class="worker-info">
            <text class="name">{{ att.workerName }}</text>
            <text class="type">{{ att.workerWorkTypeName }}</text>
          </view>
          <view class="status-tag" :class="att.status.toLowerCase()">
            <text>{{ att.statusLabel }}</text>
          </view>
        </view>
        <view class="att-body">
          <text class="date">{{ att.workDate }}</text>
          <text class="category" :class="att.workCategory.toLowerCase()">
            {{ att.workCategoryLabel }}
          </text>
          <text class="work-days">{{ att.workDays }}工</text>
          <text class="earning">¥{{ att.dailyEarning }}</text>
        </view>
        <view class="att-footer">
          <text class="project-team">{{ att.projectName }} · {{ att.teamName }}</text>
          <!-- 操作按钮 -->
          <view class="actions" v-if="att.status === 'PENDING'">
            <view class="action-btn confirm" @click.stop="onConfirm(att.id)">
              <text>确认</text>
            </view>
            <view class="action-btn reject" @click.stop="onReject(att.id)">
              <text>驳回</text>
            </view>
          </view>
        </view>
      </view>
    </view>

    <!-- 空状态 -->
    <view v-if="list.length === 0 && !loading" class="empty">
      <text>暂无考勤记录</text>
    </view>

    <!-- FAB: 跳转批量录入 -->
    <view class="fab" @click="goBatchInput">
      <text class="fab-text">录入考勤</text>
    </view>
  </view>
</template>

<script setup>
import { ref, computed } from 'vue'
import { onShow, onReachBottom, onPullDownRefresh } from '@dcloudio/uni-app'
import {
  listAttendances,
  confirmAttendance,
  rejectAttendance
} from '@/api/modules/attendance'
import { listProjects } from '@/api/modules/project'

const list = ref([])
const loading = ref(false)
const page = ref(0)
const isLast = ref(false)
const filters = ref({
  workDate: new Date().toISOString().split('T')[0],
  projectId: null,
  status: ''
})

const statusOptions = [
  { label: '全部状态', value: '' },
  { label: '待确认', value: 'PENDING' },
  { label: '已确认', value: 'CONFIRMED' },
  { label: '已驳回', value: 'REJECTED' }
]

const projectOptions = ref([{ id: null, name: '全部项目' }])

// 加载列表...
const loadList = async (reset = false) => {
  if (loading.value) return
  if (reset) { page.value = 0; list.value = []; isLast.value = false }
  loading.value = true
  try {
    const params = { page: page.value, size: 20 }
    if (filters.value.workDate) params.workDate = filters.value.workDate
    if (filters.value.projectId) params.projectId = filters.value.projectId
    if (filters.value.status) params.status = filters.value.status
    const res = await listAttendances(params)
    if (res.code === 200 && res.data) {
      const newItems = res.data.content || []
      list.value = reset ? newItems : [...list.value, ...newItems]
      isLast.value = res.data.last
      page.value = res.data.number + 1
    }
  } catch (e) {
    console.error('加载考勤列表失败', e)
  } finally {
    loading.value = false
  }
}

// 确认考勤
const onConfirm = async (id) => {
  try {
    await confirmAttendance(id)
    uni.showToast({ title: '确认成功', icon: 'success' })
    loadList(true)
  } catch (e) {
    console.error('确认失败', e)
  }
}

// 驳回考勤
const onReject = async (id) => {
  uni.showModal({
    title: '驳回原因',
    editable: true,
    placeholderText: '请输入驳回原因',
    success: async (res) => {
      if (res.confirm) {
        try {
          await rejectAttendance(id, res.content || '')
          uni.showToast({ title: '已驳回', icon: 'success' })
          loadList(true)
        } catch (e) {
          console.error('驳回失败', e)
        }
      }
    }
  })
}

const goBatchInput = () => {
  uni.navigateTo({ url: '/pages/attendance/batch-input' })
}

// 筛选器回调
const onDateChange = (e) => { filters.value.workDate = e.detail.value; loadList(true) }
const onProjectFilterChange = (e) => {
  filters.value.projectId = projectOptions.value[e.detail.value]?.id || null
  loadList(true)
}
const onStatusFilterChange = (e) => {
  filters.value.status = statusOptions[e.detail.value]?.value || ''
  loadList(true)
}

// 生命周期
onShow(() => {
  loadList(true)
  listProjects({ status: 'ACTIVE' }).then(res => {
    projectOptions.value = [{ id: null, name: '全部项目' }, ...(res.data?.content || res.data || [])]
  })
})
onReachBottom(() => loadList())
onPullDownRefresh(() => { loadList(true); uni.stopPullDownRefresh() })
</script>

<style lang="scss" scoped>
.attendance-page {
  padding-bottom: 120rpx;
  background: #F5F6FA;
}

.filter-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 16rpx;
  padding: 20rpx 32rpx;
  background: #fff;
}
.filter-item {
  padding: 12rpx 24rpx;
  background: #F0F2F5;
  border-radius: 8rpx;
  font-size: 28rpx;
  color: #333;
  &.date { font-weight: bold; }
}

.att-list { margin-top: 16rpx; }

.att-card {
  background: #fff;
  padding: 24rpx 32rpx;
  margin-bottom: 2rpx;
}
.att-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.att-header .name {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
}
.att-header .type {
  font-size: 26rpx;
  color: #999;
  margin-left: 12rpx;
}
.status-tag {
  padding: 6rpx 16rpx;
  border-radius: 6rpx;
  font-size: 24rpx;
  &.pending { background: #FFF7E6; color: #FA8C16; }
  &.confirmed { background: #F6FFED; color: #52C41A; }
  &.rejected { background: #FFF1F0; color: #FF4D4F; }
}

.att-body {
  display: flex;
  gap: 20rpx;
  align-items: center;
  margin-top: 12rpx;
  font-size: 28rpx;
  color: #666;
}
.category {
  font-weight: bold;
  &.full_day { color: #52C41A; }
  &.half_day { color: #1890FF; }
  &.overtime { color: #FA8C16; }
  &.leave { color: #999; }
}
.work-days { color: #333; }
.earning { color: #FA8C16; font-weight: bold; }

.att-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 12rpx;
}
.project-team {
  font-size: 24rpx;
  color: #999;
}
.actions {
  display: flex;
  gap: 16rpx;
}
.action-btn {
  padding: 10rpx 24rpx;
  border-radius: 6rpx;
  font-size: 26rpx;
  &.confirm { background: #52C41A; color: #fff; }
  &.reject { background: #FF4D4F; color: #fff; }
}

.fab {
  position: fixed;
  right: 32rpx;
  bottom: 120rpx;
  background: #1890FF;
  border-radius: 48rpx;
  padding: 20rpx 40rpx;
  box-shadow: 0 8rpx 24rpx rgba(24, 144, 255, 0.4);
}
.fab-text {
  font-size: 30rpx;
  color: #fff;
  font-weight: bold;
}

.empty {
  text-align: center;
  padding: 100rpx 0;
  color: #999;
  font-size: 30rpx;
}
</style>
```

---

### Step 11：考勤日历视图页面

#### 11.1 页面路径

`frontend/src/pages/attendance/calendar.vue`

#### 11.2 交互设计

```
┌──────────────────────────────────────────────┐
│  📅 考勤日历 — 张三                           │
├──────────────────────────────────────────────┤
│  [<] 2026年2月 [>]                            │
├──────────────────────────────────────────────┤
│  一   二   三   四   五   六   日              │
│                                              │
│  ●1   ●2   ●3   ●4   ●5   ○6   ○7           │
│  全天  全天  半天  全天  加班  —    —           │
│                                              │
│  ●8   ●9   ●10  ●11  ●12  ○13  ○14          │
│  全天  请假  全天  全天  全天  —    —           │
│                                              │
│  ●15  ●16  ●17  ●18  ○19  ○20  ○21          │
│  全天  半天  全天  今天  —    —    —           │
│                                              │
│  ○22  ○23  ○24  ○25  ○26  ○27  ○28          │
│                                              │
├──────────────────────────────────────────────┤
│  📊 本月汇总                                  │
│  ─────────────────────                       │
│  总工天: 14.0  |  出勤: 12天                   │
│  全天: 9   半天: 2   加班: 1   请假: 1         │
│  未记录: 1天                                  │
│                                              │
│  预估月收入: ¥4,900                            │
└──────────────────────────────────────────────┘
```

#### 11.3 关键实现要点

- 调用 `GET /api/attendances/calendar?workerId=xxx&year=2026&month=2` 获取数据
- 日历渲染：根据 `days[]` 数组，每天显示对应颜色圆点
  - 绿色 ● = 全天  
  - 蓝色 ● = 半天  
  - 橙色 ● = 加班  
  - 灰色 ● = 请假  
  - 空心 ○ = 无记录
- 点击某天可弹出该日考勤详情
- 月份切换：左右箭头切换上月/下月
- 底部汇总区：展示考勤统计 + 预估收入

#### 11.4 页面代码骨架

```vue
<!-- frontend/src/pages/attendance/calendar.vue -->
<template>
  <view class="page calendar-page">
    <!-- 工人信息 -->
    <view class="worker-header">
      <text class="worker-name">📅 {{ calendarData?.workerName || '考勤日历' }}</text>
    </view>

    <!-- 月份选择 -->
    <view class="month-nav">
      <view @click="prevMonth"><text class="nav-arrow">‹</text></view>
      <text class="month-label">{{ year }}年{{ month }}月</text>
      <view @click="nextMonth"><text class="nav-arrow">›</text></view>
    </view>

    <!-- 星期表头 -->
    <view class="week-header">
      <text v-for="d in ['一','二','三','四','五','六','日']" :key="d" class="week-day">{{ d }}</text>
    </view>

    <!-- 日历网格 -->
    <view class="calendar-grid">
      <!-- 前置空白（对齐星期） -->
      <view v-for="n in firstDayOffset" :key="'empty-' + n" class="day-cell empty"></view>

      <!-- 每日单元格 -->
      <view
        v-for="day in calendarData?.days || []"
        :key="day.date"
        class="day-cell"
        :class="getDayCellClass(day)"
        @click="showDayDetail(day)"
      >
        <text class="day-num">{{ getDayNum(day.date) }}</text>
        <text class="day-label" v-if="day.workCategoryLabel">{{ day.workCategoryLabel }}</text>
      </view>
    </view>

    <!-- 月度汇总 -->
    <view class="summary-card" v-if="calendarData">
      <text class="summary-title">📊 本月汇总</text>
      <view class="summary-row">
        <text>总工天: <text class="bold">{{ calendarData.totalWorkDays }}</text></text>
        <text>全天: <text class="bold green">{{ calendarData.fullDayCount }}</text></text>
        <text>半天: <text class="bold blue">{{ calendarData.halfDayCount }}</text></text>
      </view>
      <view class="summary-row">
        <text>加班: <text class="bold orange">{{ calendarData.overtimeCount }}</text></text>
        <text>请假: <text class="bold gray">{{ calendarData.leaveCount }}</text></text>
        <text>未记录: <text class="bold red">{{ calendarData.absentCount }}</text></text>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed } from 'vue'
import { onLoad } from '@dcloudio/uni-app'
import { getAttendanceCalendar } from '@/api/modules/attendance'

const workerId = ref(null)
const year = ref(new Date().getFullYear())
const month = ref(new Date().getMonth() + 1)
const calendarData = ref(null)

// 计算月初是星期几（用于日历对齐）
const firstDayOffset = computed(() => {
  const d = new Date(year.value, month.value - 1, 1)
  const day = d.getDay()
  return day === 0 ? 6 : day - 1  // 周一为0，周日为6
})

const loadCalendar = async () => {
  if (!workerId.value) return
  try {
    const res = await getAttendanceCalendar({
      workerId: workerId.value,
      year: year.value,
      month: month.value
    })
    calendarData.value = res.data
  } catch (e) {
    console.error('加载日历失败', e)
  }
}

const prevMonth = () => {
  if (month.value === 1) { year.value--; month.value = 12 }
  else { month.value-- }
  loadCalendar()
}

const nextMonth = () => {
  const now = new Date()
  if (year.value >= now.getFullYear() && month.value >= now.getMonth() + 1) return
  if (month.value === 12) { year.value++; month.value = 1 }
  else { month.value++ }
  loadCalendar()
}

const getDayNum = (dateStr) => {
  if (!dateStr) return ''
  return new Date(dateStr).getDate()
}

const getDayCellClass = (day) => {
  if (!day.workCategory) return 'no-record'
  return day.workCategory.toLowerCase().replace('_', '-')
}

const showDayDetail = (day) => {
  if (!day.workCategory) return
  uni.showModal({
    title: `${day.date}`,
    content: `出勤: ${day.workCategoryLabel || '无'}\n工天: ${day.workDays || 0}\n状态: ${day.status || '无记录'}`,
    showCancel: false
  })
}

onLoad((options) => {
  workerId.value = Number(options.workerId)
  if (options.year) year.value = Number(options.year)
  if (options.month) month.value = Number(options.month)
  loadCalendar()
})
</script>

<style lang="scss" scoped>
.calendar-page { background: #fff; }

.worker-header {
  padding: 24rpx 32rpx;
  background: #1890FF;
}
.worker-name { font-size: 36rpx; color: #fff; font-weight: bold; }

.month-nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24rpx 48rpx;
}
.month-label { font-size: 34rpx; font-weight: bold; color: #333; }
.nav-arrow { font-size: 40rpx; color: #1890FF; padding: 12rpx 24rpx; }

.week-header {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  text-align: center;
  padding: 16rpx 0;
  border-bottom: 1rpx solid #eee;
}
.week-day { font-size: 26rpx; color: #999; }

.calendar-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 4rpx;
  padding: 8rpx;
}

.day-cell {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 12rpx 0;
  min-height: 96rpx;
  border-radius: 8rpx;

  &.empty { }

  &.full-day { background: #F6FFED; }
  &.half-day { background: #E6F7FF; }
  &.overtime { background: #FFF7E6; }
  &.leave { background: #F5F5F5; }
  &.no-record { }
}

.day-num { font-size: 30rpx; color: #333; font-weight: 500; }
.day-label { font-size: 22rpx; margin-top: 4rpx; color: #666; }

.full-day .day-num { color: #52C41A; }
.half-day .day-num { color: #1890FF; }
.overtime .day-num { color: #FA8C16; }
.leave .day-num { color: #999; }

.summary-card {
  margin: 32rpx;
  padding: 24rpx;
  background: #FAFAFA;
  border-radius: 12rpx;
}
.summary-title { font-size: 32rpx; font-weight: bold; color: #333; }
.summary-row {
  display: flex;
  gap: 32rpx;
  margin-top: 16rpx;
  font-size: 28rpx;
  color: #666;
}
.bold { font-weight: bold; }
.green { color: #52C41A; }
.blue { color: #1890FF; }
.orange { color: #FA8C16; }
.gray { color: #999; }
.red { color: #FF4D4F; }
</style>
```

---

### Step 12：首页改造

在首页 `pages/home/index.vue` 中添加考勤相关入口和统计。

#### 12.1 新增功能入口卡片

在现有的功能卡片网格中添加：

```javascript
// 功能菜单项新增
{ icon: '📋', label: '考勤录入', url: '/pages/attendance/batch-input', roles: ['SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN'] },
{ icon: '📊', label: '考勤管理', url: '/pages/attendance/index', roles: ['SUPER_ADMIN', 'PROJECT_MANAGER', 'FOREMAN'] },
```

#### 12.2 首页今日考勤概览（可选）

在快捷统计区域添加今日考勤数据：

```
── 今日考勤 ──
出勤: 45人  请假: 3人  未录入: 12人
```

调用 `GET /api/attendances/daily-summary?projectId=xxx&workDate=today` 获取数据。

---

### Step 13：pages.json 路由更新

在 `frontend/src/pages.json` 的 `pages` 数组中新增以下页面路由：

```json
{
  "path": "pages/attendance/index",
  "style": {
    "navigationBarTitleText": "考勤管理",
    "enablePullDownRefresh": true
  }
},
{
  "path": "pages/attendance/batch-input",
  "style": {
    "navigationBarTitleText": "考勤录入",
    "enablePullDownRefresh": false
  }
},
{
  "path": "pages/attendance/calendar",
  "style": {
    "navigationBarTitleText": "考勤日历",
    "enablePullDownRefresh": false
  }
}
```

---

### Step 14：适老化组件封装

在 `frontend/src/components/elderly-ui/` 目录下创建可复用的适老化组件。

#### 14.1 ElderlyButton — 大按钮组件

```vue
<!-- frontend/src/components/elderly-ui/ElderlyButton.vue -->
<template>
  <view
    class="elderly-btn"
    :class="[type, size, { disabled, selected }]"
    @click="onClick"
  >
    <text class="btn-text"><slot /></text>
  </view>
</template>

<script setup>
const props = defineProps({
  type: { type: String, default: 'default' },     // default / primary / success / warning / danger
  size: { type: String, default: 'normal' },       // normal / large / small
  disabled: { type: Boolean, default: false },
  selected: { type: Boolean, default: false }
})

const emit = defineEmits(['click'])

const onClick = () => {
  if (props.disabled) return
  emit('click')
}
</script>

<style lang="scss" scoped>
.elderly-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 8rpx;
  border: 2rpx solid #D9D9D9;
  background: #fff;
  transition: all 0.2s;

  // 尺寸
  &.normal { min-height: 96rpx; padding: 16rpx 32rpx; }       // 48px
  &.large { min-height: 112rpx; padding: 24rpx 48rpx; }       // 56px
  &.small { min-height: 72rpx; padding: 12rpx 24rpx; }        // 36px

  // 类型
  &.primary { background: #1890FF; border-color: #1890FF; .btn-text { color: #fff; } }
  &.success { background: #52C41A; border-color: #52C41A; .btn-text { color: #fff; } }
  &.warning { background: #FA8C16; border-color: #FA8C16; .btn-text { color: #fff; } }
  &.danger { background: #FF4D4F; border-color: #FF4D4F; .btn-text { color: #fff; } }

  // 状态
  &.disabled { opacity: 0.5; }
  &.selected { border-width: 4rpx; transform: scale(1.05); }
}

.btn-text {
  font-size: 32rpx;
  font-weight: 500;
  color: #333;
}
</style>
```

#### 14.2 ElderlyCard — 大卡片组件

```vue
<!-- frontend/src/components/elderly-ui/ElderlyCard.vue -->
<template>
  <view class="elderly-card" @click="$emit('click')">
    <slot />
  </view>
</template>

<script setup>
defineEmits(['click'])
</script>

<style lang="scss" scoped>
.elderly-card {
  background: #fff;
  border-radius: 12rpx;
  padding: 24rpx 32rpx;
  margin: 16rpx 0;
  min-height: 104rpx;  // 行高 ≥ 52px
  display: flex;
  align-items: center;
  box-shadow: 0 2rpx 8rpx rgba(0,0,0,0.05);
}
</style>
```

#### 14.3 ElderlyToggleGroup — 单选按钮组（考勤类型选择器）

```vue
<!-- frontend/src/components/elderly-ui/ElderlyToggleGroup.vue -->
<template>
  <view class="toggle-group">
    <view
      v-for="option in options"
      :key="option.value"
      class="toggle-item"
      :class="{ selected: modelValue === option.value, [option.color || 'default']: true }"
      @click="$emit('update:modelValue', option.value)"
    >
      <text class="toggle-text">{{ option.label }}</text>
    </view>
  </view>
</template>

<script setup>
defineProps({
  options: {
    type: Array,
    default: () => []
    // [{ label: '全天', value: 'FULL_DAY', color: 'green' }]
  },
  modelValue: { type: String, default: '' }
})

defineEmits(['update:modelValue'])
</script>

<style lang="scss" scoped>
.toggle-group {
  display: flex;
  gap: 12rpx;
}
.toggle-item {
  padding: 16rpx 28rpx;
  border: 2rpx solid #D9D9D9;
  border-radius: 8rpx;
  background: #fff;
  min-height: 72rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s;

  &.selected {
    border-color: transparent;
    .toggle-text { color: #fff; font-weight: bold; }
  }

  &.green.selected { background: #52C41A; }
  &.blue.selected { background: #1890FF; }
  &.orange.selected { background: #FA8C16; }
  &.gray.selected { background: #999; }
  &.default.selected { background: #1890FF; }
}
.toggle-text {
  font-size: 30rpx;
  color: #666;
}
</style>
```

> **使用示例**（在批量录入页中）：
> ```vue
> <ElderlyToggleGroup
>   v-model="item.selected"
>   :options="categoryOptions"
> />
>
> const categoryOptions = [
>   { label: '全天', value: 'FULL_DAY', color: 'green' },
>   { label: '半天', value: 'HALF_DAY', color: 'blue' },
>   { label: '加班', value: 'OVERTIME', color: 'orange' },
>   { label: '休', value: 'LEAVE', color: 'gray' }
> ]
> ```

---

## 四、数据库变更

### Phase 2 不需要新增数据库迁移

`ems_attendance` 表已在 `V1__create_tables.sql` 中创建完毕，所有字段齐全。

### 可选：新增测试考勤数据 (V4)

如果需要开发/演示用的测试考勤数据，可新增 Flyway 迁移文件：

```sql
-- V4__seed_attendance_data.sql
-- 测试用考勤数据（需先有工人和项目数据）

-- 假设 worker_id=1 (张三), project_id=1 (XX工地), team_id=1 (钢筋班)
-- 2月份前两周考勤
INSERT INTO ems_attendance (worker_id, project_id, team_id, work_date, work_category, work_days, daily_wage_snapshot, is_overtime, status)
VALUES
(1, 1, 1, '2026-02-02', 'FULL_DAY', 1.0, 340.00, 0, 'CONFIRMED'),
(1, 1, 1, '2026-02-03', 'FULL_DAY', 1.0, 340.00, 0, 'CONFIRMED'),
(1, 1, 1, '2026-02-04', 'HALF_DAY', 0.5, 340.00, 0, 'CONFIRMED'),
(1, 1, 1, '2026-02-05', 'FULL_DAY', 1.0, 340.00, 0, 'CONFIRMED'),
(1, 1, 1, '2026-02-06', 'OVERTIME', 1.5, 340.00, 1, 'CONFIRMED'),
(1, 1, 1, '2026-02-09', 'FULL_DAY', 1.0, 340.00, 0, 'CONFIRMED'),
(1, 1, 1, '2026-02-10', 'LEAVE', 0.0, 340.00, 0, 'CONFIRMED'),
(1, 1, 1, '2026-02-11', 'FULL_DAY', 1.0, 340.00, 0, 'CONFIRMED'),
(1, 1, 1, '2026-02-12', 'FULL_DAY', 1.0, 340.00, 0, 'PENDING'),
(1, 1, 1, '2026-02-13', 'FULL_DAY', 1.0, 340.00, 0, 'PENDING');
```

---

## 五、测试要求

### 5.1 后端测试

| # | 测试项 | 方法 | 预期结果 |
|---|--------|------|---------|
| 1 | 单条录入-全天 | POST `/api/attendances` | workDays=1.0，status=PENDING |
| 2 | 单条录入-半天 | POST `/api/attendances` | workDays=0.5 |
| 3 | 单条录入-加班 | POST `/api/attendances` (workDays=2.0) | workDays=2.0 |
| 4 | 单条录入-请假 | POST `/api/attendances` | workDays=0.0 |
| 5 | 重复录入拦截 | 同一 workerId + workDate 提交两次 | 400，提示已有记录 |
| 6 | 批量录入成功 | POST `/api/attendances/batch` 3人 | successCount=3 |
| 7 | 批量录入含冲突 | 含已有记录的工人 | 部分失败，failItems 有原因 |
| 8 | 修改 PENDING 考勤 | PUT `/api/attendances/{id}` | 修改成功 |
| 9 | 修改 CONFIRMED 考勤 | PUT `/api/attendances/{id}` | 400，不可修改 |
| 10 | 确认考勤 | POST `/api/attendances/{id}/confirm` | status=CONFIRMED |
| 11 | 批量确认 | POST `/api/attendances/batch-confirm` | 全部变为 CONFIRMED |
| 12 | 驳回考勤 | POST `/api/attendances/{id}/reject` | status=REJECTED |
| 13 | 删除 PENDING 考勤 | DELETE `/api/attendances/{id}` | 200 |
| 14 | 删除 CONFIRMED 考勤 | DELETE `/api/attendances/{id}` | 400，不可删除 |
| 15 | 日历视图 | GET `/api/attendances/calendar` | 返回完整月度日历 |
| 16 | 日统计 | GET `/api/attendances/daily-summary` | 统计数据正确 |
| 17 | 分页查询 | GET `/api/attendances?page=0&size=5` | 分页数据正确 |
| 18 | 按日期范围查询 | GET `/api/attendances?startDate=...&endDate=...` | 仅返回范围内数据 |
| 19 | 按状态筛选 | GET `/api/attendances?status=PENDING` | 仅返回待确认记录 |
| 20 | 权限-WORKER录入 | POST `/api/attendances` (WORKER Token) | 403 |

### 5.2 前端测试

| # | 测试项 | 操作 | 预期结果 |
|---|--------|------|---------|
| 1 | 批量录入页加载 | 选择项目+班组 | 展示班组全部工人，默认预选"全天" |
| 2 | 选择出勤类型 | 点击按钮切换 | 按钮高亮切换，同时只能选一个 |
| 3 | 提交考勤 | 点击"提交考勤" | Toast"考勤提交成功"，列表刷新 |
| 4 | 已录入工人显示 | 查看已有考勤 | 显示"全天 ✅"和"修改"链接 |
| 5 | 修改已录入考勤 | 点击"修改" | 切换为按钮选择模式 |
| 6 | 日期切换 | 点击"前一天"/"后一天" | 日期更新，重新加载数据 |
| 7 | 不可选择未来 | 点击"后一天"（当天已是今天） | 按钮禁用/无反应 |
| 8 | 统计实时更新 | 选择出勤类型 | 顶部统计数字实时变化 |
| 9 | 考勤管理列表 | 进入考勤管理页 | 展示当日考勤列表 |
| 10 | 确认考勤 | 点击"确认"按钮 | Toast"确认成功"，状态更新 |
| 11 | 驳回考勤 | 点击"驳回"按钮 | 弹出原因输入框，驳回成功 |
| 12 | 筛选条件 | 切换日期/项目/状态 | 列表正确筛选 |
| 13 | 日历视图 | 进入考勤日历页 | 日历展示颜色正确，汇总数据准确 |
| 14 | 月份切换 | 点击箭头切换月份 | 日历数据更新 |
| 15 | 首页入口 | 点击"考勤录入"卡片 | 跳转到批量录入页 |

---

## 六、关键决策与注意事项

### 6.1 dailyWageSnapshot 快照机制

考勤录入时自动快照工人当时的日薪到 `daily_wage_snapshot` 字段，这是设计关键：
- 后续工人调薪不影响已录入的历史考勤
- 薪资核算（Phase 3）基于 `daily_wage_snapshot` 而非 Worker 表当前日薪
- 快照取值优先级：`worker.dailyWage` > `workType.defaultDailyWage` > `0`

### 6.2 唯一约束与批量录入

- 数据库层面：`UNIQUE KEY uk_worker_date (worker_id, work_date)` 保证同一工人同一天最多一条
- 业务层面：批量录入前先查询已有记录，跳过已录入的工人，避免数据库报错
- 前端层面：已录入的工人显示为"已录入"状态，提供"修改"入口（调用 PUT 接口）

### 6.3 审核状态流转

```
PENDING → CONFIRMED  (工长/项目经理确认)
PENDING → REJECTED   (项目经理驳回)
仅 PENDING 状态可修改/删除
CONFIRMED/REJECTED 状态不可再次变更（除非超级管理员直接操作数据库）
```

### 6.4 适老化设计检查清单

实现前端页面时，逐项检查：

- [ ] 全局基础字号 ≥ 16px (32rpx)
- [ ] 按钮最小高度 ≥ 48px (96rpx)，提交按钮 ≥ 56px (112rpx)
- [ ] 列表行高 ≥ 52px (104rpx)
- [ ] 前景/背景对比度 ≥ 4.5:1
- [ ] 出勤类型同时用颜色 + 文字标识（不依赖颜色传达信息）
- [ ] 工天录入：一屏展示班组全部工人，每人点击一次即完成
- [ ] 日薪自动展示，无需手动输入
- [ ] 日期默认当天
- [ ] 默认预选"全天"，只需修改例外情况
- [ ] 操作后有明确的成功/失败 Toast 提示
- [ ] 删除/驳回需二次确认

### 6.5 文件结构（Phase 2 新增）

**后端新增文件：**
```
com.henrywang.ems/
├── entity/
│   └── Attendance.java                    [新增]
├── entity/enums/
│   ├── WorkCategory.java                  [新增]
│   └── AttendanceStatus.java              [新增]
├── repository/
│   └── AttendanceRepository.java          [新增]
├── dto/
│   ├── AttendanceDTO.java                 [新增]
│   ├── AttendanceBatchDTO.java            [新增]
│   └── AttendanceUpdateDTO.java           [新增]
├── vo/
│   ├── AttendanceVO.java                  [新增]
│   ├── AttendanceCalendarVO.java          [新增]
│   ├── AttendanceSummaryVO.java           [新增]
│   └── AttendanceBatchResultVO.java       [新增]
├── service/
│   ├── AttendanceService.java             [新增]
│   └── impl/
│       └── AttendanceServiceImpl.java     [新增]
└── controller/
    └── AttendanceController.java          [新增]
```

**前端新增文件：**
```
frontend/src/
├── api/modules/
│   └── attendance.js                      [新增]
├── pages/attendance/
│   ├── index.vue                          [新增] 考勤管理列表
│   ├── batch-input.vue                    [新增] 工天批量录入（核心页面）
│   └── calendar.vue                       [新增] 考勤日历视图
├── components/elderly-ui/
│   ├── ElderlyButton.vue                  [新增]
│   ├── ElderlyCard.vue                    [新增]
│   └── ElderlyToggleGroup.vue             [新增]
└── pages.json                             [修改] 新增 3 个页面路由
```

**修改的现有文件：**
```
frontend/src/pages/home/index.vue          [修改] 添加考勤功能入口卡片
frontend/src/pages.json                    [修改] 注册考勤页面路由
```

---

## 七、与 Phase 3 的衔接

Phase 2 完成后，以下内容为 Phase 3（薪资管理）做好准备：

| 准备项 | 状态 |
|--------|------|
| `ems_attendance` 表 | ✅ Phase 0 创建 |
| Attendance Entity + Repository | ✅ Phase 2 创建 |
| 考勤 CRUD + 批量录入 | ✅ Phase 2 完成 |
| 考勤审核（CONFIRMED 状态） | ✅ Phase 2 完成 |
| `dailyWageSnapshot` 快照 | ✅ Phase 2 已实现 |
| `sumWorkDaysByWorkerAndMonth` 统计 | ✅ Repository 已预置 |
| MonthlySalary Entity | ❌ Phase 3 创建 |
| 月度薪资核算逻辑 | ❌ Phase 3 实现 |
| 工人调动功能 | ❌ Phase 3 实现（P1 优先级） |
| 合同管理 | ❌ Phase 3 实现（P1 优先级） |

---

## 八、API 接口汇总（Phase 2 新增）

```
# ===== 考勤/工天管理 =====
GET    /api/attendances                    # 考勤记录列表（分页+筛选）
GET    /api/attendances/{id}               # 考勤详情
POST   /api/attendances                    # 录入考勤（单条）
POST   /api/attendances/batch              # 批量录入考勤
PUT    /api/attendances/{id}               # 修改考勤（仅 PENDING）
DELETE /api/attendances/{id}               # 删除考勤（仅 PENDING）
POST   /api/attendances/{id}/confirm       # 确认考勤
POST   /api/attendances/batch-confirm      # 批量确认考勤
POST   /api/attendances/{id}/reject        # 驳回考勤
GET    /api/attendances/calendar           # 考勤日历数据
GET    /api/attendances/daily-summary      # 考勤日统计
```

---

> **执行提示**：建议按照 Step 1 → Step 14 的顺序逐步实施，每完成一个 Step 即进行编译验证。特别注意：
> 1. Step 2（Attendance Entity）完成后务必 `mvn clean compile` 验证 Hibernate validate 通过
> 2. Step 5（Service）是核心业务逻辑，建议通过 Swagger UI 逐个接口测试后再进入前端开发
> 3. Step 9（批量录入页）是系统最重要的页面，需要重点打磨适老化体验
