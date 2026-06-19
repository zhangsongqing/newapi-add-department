# newapi-add-department

# New API 二次开发新增功能说明（README-add）

> 本文档记录在 New API 原版基础上**二次开发新增的功能**，便于版本对比、维护与后续升级时的冲突排查。
>
> 核心主题：**部门（Department）能力**——以「模式 A：部门仅作统计维度」为设计原则。

---

## 一、功能概览

围绕「部门」共新增两块能力：

| 模块                       | 能力                                                         | 状态   |
| -------------------------- | ------------------------------------------------------------ | ------ |
| **部门统计维度（模式 A）** | 用户可归属一个「部门」，部门随消费日志落库，可按部门做消费统计与日志筛选 | 已完成 |
| **部门字典管理**           | 后台新增「部门管理」页面（增删改查），用户编辑/新增时**下拉选择**部门，替代手输文本 | 已完成 |

> **设计原则（模式 A）**：部门只是用户身上的一个文本标签，**仅用于消费统计与日志筛选，不参与鉴权、不参与计费、不参与限流**。这样改动面最小、风险最低。

---

## 二、核心设计原则：模式 A

- 部门 = 用户的一个属性（`users.department`，varchar）。
- 部门值在**写消费日志时被快照记录**到 `logs.department`，因此统计基于「消费发生时」的部门归属。
- 部门字典表（`department`）只为前端提供**下拉候选项**与统一管理，**不与用户表做外键强约束**——用户的 `department` 仍是自由文本，删除字典里的部门不会影响历史用户/日志数据（向后兼容、容错）。
- 不触碰鉴权、计费、令牌、分组（group）等既有核心逻辑。

---

## 三、后端改动

### 1. 新增文件

| 文件                       | 作用                                       |
| -------------------------- | ------------------------------------------ |
| `model/department.go`      | 部门字典表模型 `Department` 及其 CRUD 方法 |
| `controller/department.go` | 部门管理 HTTP 控制器（增删改查）           |

#### `model/department.go` — 部门字典表

```go
type Department struct {
    Id          int
    Name        string // size:64, 软删除范围内唯一（uk_department_name）
    Description string // varchar(255)
    Sort        int    // 排序权重，越大越靠前，带索引
    CreatedTime int64
    UpdatedTime int64
    DeletedAt   gorm.DeletedAt // 软删除
}
```

提供方法：

- `Insert()` / `Update()` / `DeleteDepartmentByID(id)`
- `IsDepartmentNameDuplicated(id, name)` — 重名校验（排除自身）
- `GetAllDepartments()` — 按 `sort DESC, updated_time DESC` 排序返回全量
- `GetDepartmentById(id)`

#### `controller/department.go` — 部门管理控制器

- `GetDepartments` — 返回全部部门
- `CreateDepartment` — 名称非空 + 重名校验后创建
- `UpdateDepartment` — 校验 ID/名称/重名后更新
- `DeleteDepartment` — 按 ID 删除（软删除）

### 2. 修改的文件

| 文件                   | 改动内容                                                     |
| ---------------------- | ------------------------------------------------------------ |
| `router/api-router.go` | 新增受管理员保护的 `/api/department` 路由组                  |
| `model/main.go`        | `migrateDB()` 与 `migrateDBFast()` **两处**均注册 `&Department{}`，启动时自动建表（适配 SQLite/MySQL/PG） |
| `model/user.go`        | `User` 新增 `Department` 字段（`varchar(64)`，带索引，仅作统计维度）；新增 `GetUserDepartmentById(id)`（优先走用户缓存，回退数据库） |
| `model/user_cache.go`  | 用户缓存结构体增加 `Department` 字段，写缓存时一并填充       |
| `model/log.go`         | `Log` 新增 `Department` 字段（带索引）；写日志时通过 `GetUserDepartmentById` 快照部门；`GetAllLogs` / `SumUsedQuota` 增加 `department` 筛选参数 |
| `controller/user.go`   | `CreateUser` 创建用户时写入 `Department`（此前缺失，导致新增用户无法保存部门） |
| `controller/log.go`    | 日志查询、消费统计接口读取 `department` query 参数并下传     |

#### 新增 API 路由（`router/api-router.go`）

```text
路由组：/api/department  （middleware.AdminAuth() 仅管理员可访问）
  GET    /api/department/        获取部门列表
  POST   /api/department/        创建部门
  PUT    /api/department/        更新部门
  DELETE /api/department/:id     删除部门
```

#### 日志与统计的部门维度

- `logs` 表新增 `department` 列（带索引）。
- 写日志时调用 `GetUserDepartmentById(userId)`，把**当时**的用户部门快照进日志。
- `GetAllLogs(... department)` 支持按部门过滤日志。
- `SumUsedQuota(... department)` 支持按部门统计消费额度。

---

## 四、前端改动（默认主题 `web/default`）

### 1. 新增 feature：部门管理

目录 `web/default/src/features/departments/`：

| 文件           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| `types.ts`     | `Department` 类型、zod schema、表单数据类型                  |
| `api.ts`       | `getDepartments` / `createDepartment` / `updateDepartment` / `deleteDepartment` |
| `constants.ts` | 表单校验 schema、默认值、成功提示文案                        |
| `index.tsx`    | 部门管理页主组件（Provider + 页面布局 + 表格 + 弹窗）        |
| `components/`  | Provider、Table、Columns、Dialogs、MutateDrawer（表单抽屉）、DeleteDialog、PrimaryButtons、RowActions |

页面能力：部门列表（id/名称/描述/排序/创建时间）、新建/编辑（名称、描述、排序权重）、删除（二次确认）、名称重复校验。

### 2. 新增路由 + 菜单

| 文件                                          | 改动                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| `routes/_authenticated/departments/index.tsx` | 新增 `/departments` 文件路由，`beforeLoad` 守卫：非管理员重定向 `/403` |
| `hooks/use-sidebar-data.ts`                   | 后台管理分组「用户」与「兑换码」之间新增「部门管理」菜单项（图标 `Building2`） |
| `routeTree.gen.ts`                            | TanStack Router 自动生成的路由树（含 `/departments`）        |

### 3. 用户管理表单：部门改为下拉选择

`web/default/src/features/users/components/users-mutate-drawer.tsx`：

- 拉取部门字典（`getDepartments`）作为下拉候选。
- **新增用户**与**编辑用户**均提供「部门」下拉选择（含「无」选项；历史不在字典中的旧值做兜底保留为可选项）。
- 相关数据转换 `lib/user-form.ts`：创建与更新两条路径都会提交 `department`。

### 4. 用户列表 / 日志：部门列与筛选

- 用户列表、日志表格新增「部门」列展示。
- 日志页支持按部门筛选。

### 5. 国际化

`web/default/src/i18n/locales/{zh,en}.json` 补齐部门相关全部文案（部门管理、增删改成功提示、校验提示、占位符、空态文案等）。

---

## 五、数据库变更

启动时由 GORM `AutoMigrate` 自动执行，**无需手动建表**，且自动适配 SQLite / MySQL / PostgreSQL：

| 表                   | 变更                                                         |
| -------------------- | ------------------------------------------------------------ |
| `department`（新表） | 部门字典：`id` / `name`(唯一) / `description` / `sort` / `created_time` / `updated_time` / `deleted_at` |
| `users`              | 新增 `department` 列（varchar(64)，带索引）                  |
| `logs`               | 新增 `department` 列（带索引）                               |

> ⚠️ **数据迁移提醒**：`AutoMigrate` 只同步**表结构**，不迁移**数据**。从本地 SQLite 切换到生产 MySQL/PG 时，表结构会自动在目标库创建，但本地已录入的部门字典与用户部门值**不会自动迁移**，需在生产环境重新录入或单独做数据迁移。

---

## 六、升级 / 部署注意事项

1. **必须用包含本改动的源码重新构建镜像/二进制**，旧镜像无部门功能、也不会建表。
2. 后端编译前需先构建前端（`cd web/default && bun run build`），因 `//go:embed` 需要 `dist`。
3. 首次启动会自动创建/变更上述表结构。
4. 部门字典数据需在「后台管理 → 部门管理」中录入；用户部门在用户管理中下拉指定。
5. 部门为统计维度，不影响既有鉴权与计费逻辑，可安全回滚（回滚后历史 `department` 列数据保留但不再使用）。

---

## 七、改动文件清单速查

**后端（Go）**

- 新增：`model/department.go`、`controller/department.go`
- 修改：`router/api-router.go`、`model/main.go`、`model/user.go`、`model/user_cache.go`、`model/log.go`、`controller/user.go`、`controller/log.go`

**前端（`web/default`）**

- 新增：`src/features/departments/**`、`src/routes/_authenticated/departments/index.tsx`
- 修改：`src/hooks/use-sidebar-data.ts`、`src/features/users/components/users-mutate-drawer.tsx`、`src/features/users/lib/user-form.ts`、用户列表/日志相关组件、`src/i18n/locales/{zh,en}.json`、`src/routeTree.gen.ts`（自动生成）

---

> 设计原则：模式 A —— 部门仅作统计维度，不参与鉴权与计费。如需「部门参与权限/配额管控」的模式 B，需另行设计。
