# yudao-module-bill — 记账模块

> 记账 App 的核心业务模块，承载用户账单、分类、预算等子域。本文档先落地「类别管理」子模块的设计方案，后续子模块以本 README 为模板继续追加章节。

## 1. 模块定位

| 项 | 说明 |
| --- | --- |
| Maven 模块名 | `yudao-module-bill`（聚合）<br>`yudao-module-bill-api`（RPC 接口 + DTO）<br>`yudao-module-bill-server`（实现 + DB） |
| 用户体系 | MemberUser（C 端用户 App），API 前缀 `/app-api/bill/**` |
| 多租户 | 通过 `yudao-spring-boot-starter-biz-tenant` 自动注入 `tenant_id`；DDL 中 `tenant_id` 必填 |
| 依赖 starter | `yudao-spring-boot-starter-web`、`-mybatis`、`-redis`、`-security`、`-biz-tenant` |

```
yudao-module-bill/
├── pom.xml
├── yudao-module-bill-api/
│   └── src/main/java/cn/iocoder/yudao/module/bill/
│       ├── api/             # 给其他模块用的 RPC Feign 接口
│       ├── dto/             # RespVO / ReqVO
│       └── enums/           # CategoryTypeEnum、IconTypeEnum
└── yudao-module-bill-server/
    └── src/main/java/cn/iocoder/yudao/module/bill/
        ├── controller/app/  # /app-api/bill/category/**
        ├── service/
        ├── dal/dataobject/  # CategoryDO
        ├── dal/mysql/       # CategoryMapper
        └── dal/migrate/     # 系统预置数据 seed 逻辑
```

SQL 脚本位于 [`sql/mysql/`](../../sql/mysql/) 下的 `bill_category.sql`、`bill_category_seed.sql`。

---

## 2. 类别管理（Category）子模块

### 2.1 功能描述

管理系统内置及用户自定义的收支分类。

### 2.2 功能要求

- 支持按「支出」和「收入」分类展示。
- 支持自定义分类名称、图标（Emoji / Icon）及排序。
- 系统提供一套默认的常用分类（如餐饮、交通、工资等），用户可进行编辑或禁用。

### 2.3 设计要点

1. **单表 + per-user 副本**： 不分 system / user 两张表。新用户注册时，把系统预置分类批量插入到该用户的命名空间下，每行带 `user_id`。这样「编辑/禁用系统预置分类」直接 in-place 更新，不会污染其他用户，`builtin=1` 标记仅用于「恢复默认」等场景识别。
2. **完全对齐 yudao 约定**： 审计字段（`creator / create_time / updater / update_time / deleted`）、逻辑删除值 0/1、`map-underscore-to-camel-case=true`、`tenant_id` 走 `yudao-spring-boot-starter-biz-tenant`、`id-type=NONE`（MySQL 8 自适应）。
3. **图标策略**： 用 `icon_type` 区分 emoji 字符 vs icon 名 vs 图片 URL，避免后续还要解析字符串。Emoji 字符定长 1-4 字节，`utf8mb4` 已支持。
4. **不做树形**： v1 扁平，`sort ASC` 即可排序；如未来要二级分类，再加 `parent_id`。
5. **业务字段冗余设计**： 唯一索引只放在 `(user_id, type, name, deleted)`，确保同一用户下「支出/餐饮」唯一，软删后可重建同名。

### 2.4 表结构 DDL

文件：[`sql/mysql/bill_category.sql`](../../sql/mysql/bill_category.sql)

```sql
-- 索引与唯一约束延后到性能优化阶段（见 §6 索引设计）
CREATE TABLE `bill_category` (
  `id`            BIGINT       NOT NULL AUTO_INCREMENT COMMENT '分类编号',
  `tenant_id`     BIGINT       NOT NULL                COMMENT '租户编号（多租户隔离）',
  `user_id`       BIGINT       NOT NULL                COMMENT '所属用户编号（每用户一份副本）',
  `type`          TINYINT      NOT NULL                COMMENT '分类类型：1=支出 2=收入',
  `name`          VARCHAR(30)  NOT NULL                COMMENT '分类名称',
  `icon_type`     TINYINT      NOT NULL DEFAULT 1      COMMENT '图标类型：1=Emoji 2=Icon 名称 3=图片 URL',
  `icon`          VARCHAR(128) NOT NULL                COMMENT '图标内容（emoji 字符 / icon 名 / 图片地址）',
  `color`         VARCHAR(16)           DEFAULT NULL   COMMENT '主题色 HEX，例如 #FF6B6B',
  `sort`          INT          NOT NULL DEFAULT 0      COMMENT '排序，越小越靠前',
  `status`        TINYINT      NOT NULL DEFAULT 1      COMMENT '状态：0=禁用 1=启用',
  `builtin`       TINYINT      NOT NULL DEFAULT 0      COMMENT '是否系统预置：0=用户新增 1=系统预置',
  `description`   VARCHAR(255)          DEFAULT NULL   COMMENT '备注/描述',
  `creator`       VARCHAR(64)           DEFAULT ''     COMMENT '创建者',
  `create_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP             COMMENT '创建时间',
  `updater`       VARCHAR(64)           DEFAULT ''     COMMENT '更新者',
  `update_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`       TINYINT      NOT NULL DEFAULT 0      COMMENT '是否删除：0=否 1=是',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='记账分类表';
```

### 2.5 系统预置数据

文件：[`sql/mysql/bill_category_seed.sql`](../../sql/mysql/bill_category_seed.sql)

> 该 SQL 是**模板**，实际由后端 `CategorySeedService` 在用户首次创建分类列表或注册时按用户 ID 批量插入，避免并发与历史分类污染。

```sql
-- 支出 (type = 1)
INSERT INTO bill_category (tenant_id, user_id, type, name, icon_type, icon, sort, builtin) VALUES
(?, ?, 1, '餐饮',     1, '🍔',  10, 1),
(?, ?, 1, '交通',     1, '🚗',  20, 1),
(?, ?, 1, '购物',     1, '🛍️', 30, 1),
(?, ?, 1, '居家',     1, '🏠',  40, 1),
(?, ?, 1, '娱乐',     1, '🎮',  50, 1),
(?, ?, 1, '医疗',     1, '💊',  60, 1),
(?, ?, 1, '教育',     1, '📚',  70, 1),
(?, ?, 1, '通讯',     1, '📱',  80, 1),
(?, ?, 1, '其他支出', 1, '💸', 99, 1);

-- 收入 (type = 2)
INSERT INTO bill_category (tenant_id, user_id, type, name, icon_type, icon, sort, builtin) VALUES
(?, ?, 2, '工资',     1, '💰', 10, 1),
(?, ?, 2, '奖金',     1, '🎁', 20, 1),
(?, ?, 2, '理财',     1, '📈', 30, 1),
(?, ?, 2, '兼职',     1, '💼', 40, 1),
(?, ?, 2, '退款',     1, '↩️', 50, 1),
(?, ?, 2, '其他收入', 1, '🪙', 99, 1);
```

### 2.6 关键决策与扩展点

| 决策 | 原因 |
| --- | --- |
| 不分 system / user 两表 | 「用户可编辑系统预置」语义在单表 + `user_id` 隔离下天然成立；少一次 JOIN、少一份同步逻辑 |
| 软删 + 唯一索引带 `deleted` | 用户禁用/删除后可重建同名；硬删的话 `name` 唯一约束会冲突 |
| `tenant_id` 必填 | 即使将来想做家庭账本共享，租户维度天然隔离 |
| `icon_type` 字段 | 前端不用猜字符串含义；emoji 一个字符、icon 一段 class、图片一串 URL 都好处理 |
| `color` 预留 | UI 列表分类色块必备，挪到 v2 再加要写迁移 |
| `description` 预留 | 部分类目（如「礼金」）需要备注说明 |
| 唯一约束只放 `(user_id,type,name,deleted)` | 跨用户同名互不影响；不同类型可同名（支出/餐饮 与 收入/餐饮 不冲突，需求上也合理） |

### 2.7 常见查询（验证索引够用）

```sql
-- 用户拉取自己「支出」列表，按 sort 排序
SELECT * FROM bill_category
WHERE user_id = ? AND type = 1 AND status = 1 AND deleted = 0
ORDER BY sort ASC;

-- 重命名校验
SELECT 1 FROM bill_category
WHERE user_id = ? AND type = 1 AND name = ? AND deleted = 0 LIMIT 1;
```

### 2.8 后续 TODO

- [ ] `CategoryDO` + `CategoryMapper` + `CategoryService` 代码骨架
- [ ] `CategoryTypeEnum`、`IconTypeEnum`
- [ ] 新用户注册时自动 seed 默认分类（监听 `MemberUserCreateEvent`）
- [ ] 应用版本升级时增量补默认分类（比对 seed 文件 vs 用户已有 `builtin=1` 的差异）
- [ ] 「恢复默认」接口：按 `builtin=1` 删除用户自定义副本，重新 seed

---

## 3. 账本管理（Book）子模块

> ⚠️ 本节定义**账本**与**账本成员**两张基础表，以及「创建/修改/删除/归档/预算/列表」等核心 CRUD；**邀请、户主鉴权、成员进出、协作权限**等家庭账本协作能力统一在 §4 模块详细设计。

### 3.1 功能描述

支持用户创建和管理多个独立账本，实现资金隔离与预算控制。

### 3.2 功能要求

- **个人账本**：用户可创建多个个人专属账本（如：日常开销、旅游基金、装修备用金）。
- **家庭账本**：支持创建共享账本，用于记录家庭共同收支（如：家庭生活费、房贷车贷）。
- **账本设置**：支持为账本设置月度预算额度。

### 3.3 设计要点

1. **单账本表 + 类型字段**： 个人账本与家庭账本共用 `bill_book` 一张表，通过 `type` 字段（1=个人 / 2=家庭）区分，避免拆表带来的双倍维护成本。
2. **成员表统一存在**： 即使是个人账本，也要在 `bill_book_member` 中写入一条 `role=1`（户主）的记录。这样「我在哪些账本里」「该账本有谁」的查询走同一张表、同一套权限模型。
3. **owner 字段冗余到账本表**： `bill_book.owner_user_id` 与 `bill_book_member(role=1)` 信息冗余。列表查询「我作为户主的账本」走 `bill_book` 主键索引即可，不必每次 JOIN。
4. **预算作为普通字段**： v1 `monthly_budget` 单字段即可，**不做预算变更历史**（YAGNI），后续要追溯再拆 `bill_book_budget_history` 表。
5. **type 创建后不可改**： 个人 ↔ 家庭 的升级/降级不在 v1 支持，避免成员关系与账本类型不一致的边界情况。需要时直接新建账本。
6. **删除做软删 + 业务校验**： 账本软删时必须先校验「无关联账单」，否则阻止删除并提示「请先清空账本内的账单」。
7. **默认账本 seed**： 新用户注册时自动建一本「默认账本」（type=1, name='日常开销'），用户首次进入即可记账，无需手动创建。

### 3.4 表结构 DDL

文件：[`sql/mysql/bill_book.sql`](../../sql/mysql/bill_book.sql)

```sql
-- 索引与唯一约束延后到性能优化阶段（见 §6 索引设计）
-- 账本表
CREATE TABLE `bill_book` (
  `id`              BIGINT         NOT NULL AUTO_INCREMENT COMMENT '账本编号',
  `tenant_id`       BIGINT         NOT NULL                COMMENT '租户编号',
  `owner_user_id`   BIGINT         NOT NULL                COMMENT '创建人/户主用户编号',
  `type`            TINYINT        NOT NULL                COMMENT '账本类型：1=个人 2=家庭',
  `name`            VARCHAR(50)    NOT NULL                COMMENT '账本名称',
  `description`     VARCHAR(255)             DEFAULT NULL   COMMENT '账本描述/备注',
  `icon`            VARCHAR(128)             DEFAULT NULL   COMMENT '图标（emoji 或 icon）',
  `color`           VARCHAR(16)              DEFAULT NULL   COMMENT '主题色 HEX',
  `currency`        VARCHAR(8)     NOT NULL DEFAULT 'CNY'   COMMENT '币种',
  `monthly_budget`  DECIMAL(18,2)           DEFAULT NULL   COMMENT '月度预算金额，NULL 表示未设置',
  `sort`            INT            NOT NULL DEFAULT 0      COMMENT '排序，越小越靠前',
  `status`          TINYINT        NOT NULL DEFAULT 1      COMMENT '状态：0=归档 1=启用',
  `creator`         VARCHAR(64)              DEFAULT ''     COMMENT '创建者',
  `create_time`     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP             COMMENT '创建时间',
  `updater`         VARCHAR(64)              DEFAULT ''     COMMENT '更新者',
  `update_time`     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`         TINYINT        NOT NULL DEFAULT 0      COMMENT '是否删除：0=否 1=是',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='账本表';

-- 账本成员表（家庭协作能力的基础；个人账本也有一行 owner=户主）
CREATE TABLE `bill_book_member` (
  `id`              BIGINT         NOT NULL AUTO_INCREMENT COMMENT '成员编号',
  `tenant_id`       BIGINT         NOT NULL                COMMENT '租户编号',
  `book_id`         BIGINT         NOT NULL                COMMENT '账本编号',
  `user_id`         BIGINT         NOT NULL                COMMENT '用户编号',
  `role`            TINYINT        NOT NULL                COMMENT '角色：1=户主 2=普通成员',
  `join_time`       DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '加入时间',
  `inviter_user_id` BIGINT                  DEFAULT NULL   COMMENT '邀请人用户编号（户主邀请时为 owner_user_id）',
  `status`          TINYINT        NOT NULL DEFAULT 1      COMMENT '状态：0=退出 1=在册',
  `creator`         VARCHAR(64)              DEFAULT ''     COMMENT '创建者',
  `create_time`     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP             COMMENT '创建时间',
  `updater`         VARCHAR(64)              DEFAULT ''     COMMENT '更新者',
  `update_time`     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`         TINYINT        NOT NULL DEFAULT 0      COMMENT '是否删除：0=否 1=是',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='账本成员表';
```

### 3.5 默认账本 seed（用户注册时）

Java 常量 + 事件监听器方式（与 §2.5 类别 seed 同模式，**不直接执行 SQL**）：

```java
public final class BookSeedData {
    public static final String DEFAULT_PERSONAL_BOOK_NAME = "默认账本";
    public static final String DEFAULT_BOOK_ICON = "📒";
    public static final String DEFAULT_BOOK_COLOR = "#4F8AF7";
}

@Component
public class BookSeedListener {
    @Resource private BookMapper bookMapper;
    @Resource private BookMemberMapper bookMemberMapper;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onMemberUserCreated(MemberUserCreateEvent event) {
        // 幂等
        if (bookMapper.countOwnedBy(event.getUserId()) > 0) return;

        BookDO book = new BookDO()
            .setTenantId(event.getTenantId())
            .setOwnerUserId(event.getUserId())
            .setType(1)                              // 1=个人
            .setName(BookSeedData.DEFAULT_PERSONAL_BOOK_NAME)
            .setIcon(BookSeedData.DEFAULT_BOOK_ICON)
            .setColor(BookSeedData.DEFAULT_BOOK_COLOR)
            .setCurrency("CNY")
            .setStatus(1);
        bookMapper.insert(book);

        BookMemberDO owner = new BookMemberDO()
            .setTenantId(event.getTenantId())
            .setBookId(book.getId())
            .setUserId(event.getUserId())
            .setRole(1)                              // 1=户主
            .setStatus(1)
            .setInviterUserId(event.getUserId());
        bookMemberMapper.insert(owner);
    }
}
```

老用户补种同 §2.5 方式二，写一份 `sql/mysql/bill_book_backfill.sql`（运维一次性）。

### 3.6 API 概览（本模块负责）

所有 API 前缀 `/app-api/bill/book/**`，登录态校验走 `yudao-spring-boot-starter-security` 的 `@PreAuthorize`。

| 方法 + 路径 | 说明 | 权限 |
| --- | --- | --- |
| `POST /create` | 创建账本 | 登录用户 |
| `PUT /update` | 修改账本名称/图标/描述 | 仅户主 |
| `PUT /update-budget` | 设置月度预算 | 仅户主 |
| `PUT /archive` | 归档账本（status=0） | 仅户主 |
| `PUT /restore` | 恢复归档账本 | 仅户主 |
| `DELETE /delete` | 软删账本 | 仅户本，且账本无账单 |
| `GET /get?id=` | 账本详情 | 必须是成员 |
| `GET /list` | 我的账本列表（户主 + 成员） | 登录用户 |
| `GET /page` | 我的账本分页 | 登录用户 |

`§4 家庭账本协作` 会补：`POST /invite`、`GET /invite-link`、`POST /accept-invite`、`DELETE /member/remove`、`GET /member/list` 等成员管理接口。

### 3.7 关键决策与扩展点

| 决策 | 原因 |
| --- | --- |
| 单表 + `type` 字段 | 个人/家庭账本 95% 字段相同，拆表收益小、双倍维护成本 |
| `owner_user_id` 冗余到账本表 | 「我作为户主的账本」是高频查询，避免每次 JOIN `bill_book_member` |
| 成员表无 type 区分 | 个人账本也存一行户主记录，权限模型统一 |
| `monthly_budget` 单字段 | 不做变更历史；预算是「当前生效值」，不是审计字段 |
| 删除前校验账单数 | 防误删导致账单变孤儿（孤儿账单需重建账本才能恢复） |
| `currency` 字段预留 | 多币种是记账 App 的常见后续需求，初始值固定 `CNY` |
| 默认账本 seed | 降低首次使用门槛；用户可改名/归档 |

### 3.8 常见查询（验证索引够用）

```sql
-- 「我作为户主的账本」
SELECT * FROM bill_book
WHERE owner_user_id = ? AND status = 1 AND deleted = 0
ORDER BY sort ASC, id DESC;

-- 「我参与的所有账本」（户主 OR 成员）
SELECT DISTINCT b.*
FROM bill_book b
LEFT JOIN bill_book_member m
  ON m.book_id = b.id AND m.deleted = 0 AND m.status = 1
WHERE (b.owner_user_id = ? OR m.user_id = ?)
  AND b.status = 1 AND b.deleted = 0
ORDER BY b.sort ASC, b.id DESC;

-- 「我作为户主的家庭账本」
SELECT * FROM bill_book
WHERE owner_user_id = ? AND type = 2 AND status = 1 AND deleted = 0;
```

### 3.9 与后续模块的边界

| 关注点 | 模块 2（账本）| 模块 3（账单）| 模块 4（家庭协作）|
| --- | --- | --- | --- |
| `bill_book` 表 | ✅ 定义 | ✅ 只读引用 | ✅ 只读引用 |
| `bill_book_member` 表 | ✅ 定义 | ✅ 只读引用 | ✅ 增删改主力 |
| 户主鉴权 | ✅ 仅 owner 可改 | ✅ 记账时不区分 | ✅ 完整 RBAC |
| 邀请流程 | ❌ | ❌ | ✅ 设计 |
| 记账人字段 | ❌ | ✅ 写入 `creator_user_id` | ✅ 列表展示 |
| 协作权限矩阵 | ❌ | ❌ | ✅ 设计 |

### 3.10 后续 TODO

- [ ] `BookDO` + `BookMapper` + `BookService` 代码骨架
- [ ] `BookTypeEnum`、`BookMemberRoleEnum`、`BookStatusEnum`
- [ ] `BookReqVO` / `BookRespVO`（含 owner 信息冗余展示）
- [ ] 新用户注册时自动 seed 默认账本（监听 `MemberUserCreateEvent`）
- [ ] 删除账本前置校验（关联账单数查询）
- [ ] 老用户账本补种脚本 `sql/mysql/bill_book_backfill.sql`
- [ ] 模块 4：成员邀请、户主/成员权限矩阵、协作权限校验

---

## 4. 账单流水（Record）子模块

> ⚠️ 本节定义**账单流水表**与**核心 CRUD/列表/汇总**接口；账单上的「记账人字段」由本模块写入，**展示与协作权限矩阵**统一在 §5 家庭账本协作模块落地。

### 4.1 功能描述

记录每一笔具体的收支明细，是整个 App 数据量最大、查询最频繁的表。

### 4.2 功能要求

- **快捷记账**：支持选择账本、分类、输入金额、备注和记账时间。
- **账单列表**：按时间倒序展示账单明细，支持按日、按月折叠展示。
- **账单筛选**：支持按账本、分类、时间段、收支类型进行组合筛选。
- **数据编辑**：支持对历史账单进行修改和删除操作。

### 4.3 设计要点

1. **`record_time` ≠ `create_time`**： `record_time` 是用户指定的**业务时间**（例如"昨天那笔餐饮"），是排序、筛选、聚合（本月支出）的唯一基准；`create_time` 仅作系统审计，**不参与业务查询**。
2. **`amount` 始终为正数**： 收支方向由 `type` 字段（1=支出 / 2=收入）区分，**禁止存负数**。理由：`SUM(amount) WHERE type=1` 即为支出总额，`SUM(amount) WHERE type=2` 即为收入总额，**预算剩余 = budget − SUM(支出)** 一行 SQL 搞定；如果存负数，所有聚合都要加 `ABS()`，逻辑极易出错。
3. **`type` 冗余自分类**： 分类的 `type` 可能因运营调整而变化，但**账单是历史事实，必须冻结当时的收支方向**。从分类写入时把 `type` 复制到账单，避免历史报表被未来的分类变更污染。
4. **`user_id` 记账人字段**： 即使是个人账本也必须有「记账人」字段（=当前登录用户）。家庭账本中用于 §5 的「张三 记了一笔餐饮」溯源展示；个人账本中冗余无副作用。
5. **4 个核心索引对齐高频查询**： `(book_id,deleted,status,record_time)` / `(book_id,type,...)` / `(book_id,category_id,...)` / `(user_id,deleted,status,record_time)`。**冗余但必要**，覆盖 95% 的查询模式。
6. **软删 + 编辑覆盖**： 历史账单可改可删，**不做编辑历史**（YAGNI，`update_time` 已经表达了"最后修改时间"）。删除走软删，列表查询一律带 `deleted=0`。
7. **权限规则（v1 简化版）**：
    - 记账：必须是账本成员
    - 查看：必须是账本成员
    - 修改/删除：账单创建者本人 **或** 账本户主
    
    §5 会把"户主可删他人账单"扩成完整 RBAC 矩阵；本模块只先把这三条规则落地。

### 4.4 表结构 DDL

文件：[`sql/mysql/bill_record.sql`](../../sql/mysql/bill_record.sql)

```sql
-- 索引延后到性能优化阶段（见 §6 索引设计）
CREATE TABLE `bill_record` (
  `id`              BIGINT         NOT NULL AUTO_INCREMENT COMMENT '账单编号',
  `tenant_id`       BIGINT         NOT NULL                COMMENT '租户编号',
  `book_id`         BIGINT         NOT NULL                COMMENT '账本编号',
  `user_id`         BIGINT         NOT NULL                COMMENT '记账人用户编号',
  `category_id`     BIGINT         NOT NULL                COMMENT '分类编号',
  `type`            TINYINT        NOT NULL                COMMENT '收支类型：1=支出 2=收入（从分类写入时冗余冻结，历史报表不被分类变更影响）',
  `amount`          DECIMAL(18,2)  NOT NULL                COMMENT '金额，始终为正数，配合 type 区分收支',
  `description`     VARCHAR(255)             DEFAULT NULL   COMMENT '备注',
  `record_time`     DATETIME       NOT NULL                COMMENT '业务时间（用户指定的记账时间，用于排序/筛选/聚合）',
  `source`          TINYINT        NOT NULL DEFAULT 1      COMMENT '来源：1=手动记账，预留导入/同步等场景',
  `status`          TINYINT        NOT NULL DEFAULT 1      COMMENT '状态：0=作废 1=正常',
  `creator`         VARCHAR(64)              DEFAULT ''     COMMENT '创建者（用户名，yudao 自动填充）',
  `create_time`     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP             COMMENT '创建时间（系统时间）',
  `updater`         VARCHAR(64)              DEFAULT ''     COMMENT '更新者',
  `update_time`     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`         TINYINT        NOT NULL DEFAULT 0      COMMENT '是否删除',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='账单流水表';
```

### 4.5 API 概览（本模块负责）

所有 API 前缀 `/app-api/bill/record/**`。

| 方法 + 路径 | 说明 | 权限 |
| --- | --- | --- |
| `POST /create` | 创建账单 | 账本成员 |
| `PUT /update` | 修改账单 | 创建者 或 户主 |
| `DELETE /delete` | 软删账单 | 创建者 或 户主 |
| `GET /get?id=` | 账单详情 | 账本成员 |
| `GET /page` | 分页列表（多条件筛选） | 账本成员 |
| `GET /summary` | 汇总统计（按账本/分类/时间范围聚合） | 账本成员 |

**`/page` 请求参数：**

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `bookId` | 否 | 不传 = 当前用户所有账本聚合 |
| `type` | 否 | 1=支出 / 2=收入 |
| `categoryId` | 否 | 单一分类 |
| `startTime` | 否 | record_time 下界 |
| `endTime` | 否 | record_time 上界 |
| `keyword` | 否 | 模糊匹配 `description` |
| `pageNum` / `pageSize` | 是 | 默认 20 |

> 「按日/按月折叠」是**前端分组**职责，后端只负责 `ORDER BY record_time DESC` 排序返回。MySQL 把同一日期/月份的数据尽量按物理顺序返回即可，不需要 SQL 做 GROUP BY。

**`/summary` 响应字段（建议）：**

```json
{
  "bookId": 1,
  "type": 1,                          // 可选：1 支出 / 2 收入
  "totalAmount": 3250.50,
  "count": 42,
  "byCategory": [                     // 按分类聚合
    { "categoryId": 5, "amount": 1200.00, "count": 18 }
  ],
  "byDay": [                          // 按天聚合（用于日历视图）
    { "date": "2026-07-08", "amount": 85.50 }
  ]
}
```

### 4.6 关键决策与扩展点

| 决策 | 原因 |
| --- | --- |
| `amount` 始终正数 + `type` 区分方向 | SUM/预算剩余聚合逻辑零分支；负数方案会让所有报表 SQL 都要 `ABS()`，极易出错 |
| `type` 冗余冻结 | 账单是历史事实，分类 type 可能改；不冗余则历史报表会被污染 |
| `record_time` 与 `create_time` 分开 | 用户记账业务时间 ≠ 系统时间；今天补录昨天的账很常见 |
| `user_id` 记账人冗余在每行 | §5 协作必用；个人账本冗余无副作用 |
| 不做编辑历史 | `update_time` 表达最后修改；YAGNI，要追溯再拆 `bill_record_edit_log` 表 |
| 日/月折叠走前端分组 | 后端只 ORDER BY，前端做 UI 折叠；后端做 GROUP BY 会丢明细 |
| 软删而非硬删 | 与账本/分类一致；硬删会让月度报表对账时数据缺失 |
| `source` 字段预留 | v2 计划接入「短信/邮箱账单自动解析」时直接用 |

### 4.7 常见查询（验证索引够用）

```sql
-- 账本内全部账单（按时间倒序）
SELECT * FROM bill_record
WHERE book_id = ? AND deleted = 0 AND status = 1
ORDER BY record_time DESC LIMIT 20;

-- 账本 + 类型 + 时间范围
SELECT * FROM bill_record
WHERE book_id = ? AND type = 1
  AND deleted = 0 AND status = 1
  AND record_time BETWEEN ? AND ?
ORDER BY record_time DESC;

-- 账本 + 单分类
SELECT * FROM bill_record
WHERE book_id = ? AND category_id = ?
  AND deleted = 0 AND status = 1
ORDER BY record_time DESC LIMIT 50;

-- 「我的账单」跨账本聚合
SELECT * FROM bill_record
WHERE user_id = ? AND deleted = 0 AND status = 1
ORDER BY record_time DESC LIMIT 20;

-- 月度支出总额（预算进度用）
SELECT COALESCE(SUM(amount), 0) FROM bill_record
WHERE book_id = ? AND type = 1
  AND deleted = 0 AND status = 1
  AND record_time BETWEEN ? AND ?;
```

### 4.8 与后续模块的边界

| 关注点 | 模块 3（账单）| 模块 4（账单）| 模块 5（家庭协作）|
| --- | --- | --- | --- |
| `bill_record` 表 | ✅ 定义 | — | — |
| CRUD + 列表 + 汇总 | ✅ 实现 | — | — |
| 记账人字段写入 | ✅ 写入 | — | ✅ 列表展示「张三 记了一笔餐饮」 |
| 协作权限（户主可改/删他人账单） | ✅ 简化版（创建者或户主） | — | ✅ 完整 RBAC + UI 透出 |
| 跨账本聚合查询 | ✅ `/page?bookId=null` 支持 | — | — |

### 4.9 后续 TODO

- [ ] `RecordDO` + `RecordMapper` + `RecordService` 代码骨架
- [ ] `RecordTypeEnum`（1 支出 / 2 收入）、`RecordSourceEnum`、`RecordStatusEnum`
- [ ] `RecordReqVO`（Create/Update/Page 三种）/ `RecordRespVO`（含 category、book 冗余字段）
- [ ] 创建时校验：账本存在、分类存在、分类的 `type` 与传入一致、`amount > 0`
- [ ] 修改/删除权限：`@PreAuthorize("@ss.hasPermission('bill:record:update')")` + Service 内二次校验「创建者或户主」
- [ ] `/summary` 聚合 SQL（一次查账本/分类/日期三维聚合，建议用 MyBatis-Plus `groupBy` + 内存二次聚合）
- [ ] `bookId=null` 跨账本聚合的性能优化（分页 + ES 索引？v2 再议）
- [ ] §5 协作模块：在列表响应中冗余返回「记账人昵称/头像」字段

---

## 5. 家庭账本协作（Collaboration）子模块

> 本节定义**邀请、权限、记账人溯源**三件事。  
> 基础表 `bill_book` / `bill_book_member` 已在 §3 定义，本节新增 `bill_book_invitation` / `bill_book_invite_link` 两张表，并补全完整 RBAC 矩阵。

### 5.1 功能描述

实现家庭成员间的账单共享与权限控制。

### 5.2 功能要求

- **成员邀请**：账本创建者（户主）可通过系统内搜索用户名或生成邀请链接的方式，邀请其他用户加入家庭账本。
- **权限控制**：
    - 户主：拥有最高权限，可增删成员、修改账本设置、查看所有账单。
    - 普通成员：仅能在该家庭账本内进行记账操作，并查看该账本下的所有账单。
- **账单溯源**：在家庭账本的账单列表中，需明确显示「记账人」信息（如：张三 记了一笔餐饮 -50元）。

### 5.3 设计要点

1. **两种邀请流分开建表**： 「定向邀请（按用户名）」与「链接邀请」的生命周期完全不同（前者有明确的 invitee_user_id，后者是 token 匹配），强行合一张表会让状态字段语义模糊。**两张表**，**两套 Service**。
2. **token 用 UUID v4 存库比对**： 简单可靠；不需要 JWT 解码逻辑。`UNIQUE` 约束保证 token 唯一（索引在 §6 统一加）。
3. **状态流转用 `@Transactional` 保护**： 接受邀请的核心是「INSERT member + UPDATE invitation.status=1」两步原子操作，缺一不可。状态字段校验在事务内做。
4. **权限校验下沉到 Service 层**： 不在 Controller 用 `@PreAuthorize` 字符串硬编码，**统一走 `BookPermissionService.canXxx(userId, bookId, ...)`**，便于后续加规则（例如「户主 30 天免审核」）。
5. **记账人显示走 Feign 批量查询**： §4 已经把 `user_id` 写入账单行。账单列表响应里**多塞一个 `creatorInfo` 字段**，由 Service 调 `MemberUserApi.getUserSummary(List<Long> userIds)` 批量取昵称头像。**禁止跨模块 JOIN**。
6. **金额符号展示约定**： 底层 `amount` 始终为正数；UI 显示符号由前端按 `type` 渲染（type=1 支出 → `-amount`，type=2 收入 → `+amount`）。后端**不返回带符号的字符串**，避免多端解析不一致。
7. **户主不可直接退出**： 户主要么转让（v2）要么删除账本；不允许直接 set `member.status=0`，避免出现"孤儿账本"。

### 5.4 表结构 DDL

文件：[`sql/mysql/bill_book_invitation.sql`](../../sql/mysql/bill_book_invitation.sql)

```sql
-- 索引延后到性能优化阶段（见 §6 索引设计）
-- 定向邀请：户主 → 特定用户
CREATE TABLE `bill_book_invitation` (
  `id`               BIGINT       NOT NULL AUTO_INCREMENT COMMENT '邀请编号',
  `tenant_id`        BIGINT       NOT NULL                COMMENT '租户编号',
  `book_id`          BIGINT       NOT NULL                COMMENT '账本编号',
  `inviter_user_id`  BIGINT       NOT NULL                COMMENT '邀请人（户主）',
  `invitee_user_id`  BIGINT       NOT NULL                COMMENT '被邀请人',
  `status`           TINYINT      NOT NULL DEFAULT 0      COMMENT '状态：0=待接受 1=已接受 2=已拒绝 3=已过期 4=已撤销',
  `expire_time`      DATETIME     NOT NULL                COMMENT '过期时间',
  `message`          VARCHAR(255)         DEFAULT NULL   COMMENT '邀请留言',
  `accept_time`      DATETIME             DEFAULT NULL   COMMENT '接受时间（status=1 时填写）',
  `creator`          VARCHAR(64)          DEFAULT ''     COMMENT '创建者',
  `create_time`      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP             COMMENT '创建时间',
  `updater`          VARCHAR(64)          DEFAULT ''     COMMENT '更新者',
  `update_time`      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`          TINYINT      NOT NULL DEFAULT 0      COMMENT '是否删除',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='账本定向邀请表';
```

文件：[`sql/mysql/bill_book_invite_link.sql`](../../sql/mysql/bill_book_invite_link.sql)

```sql
-- 索引延后到性能优化阶段（见 §6 索引设计）
-- 链接邀请：户主生成 token，任何人凭 token 加入
CREATE TABLE `bill_book_invite_link` (
  `id`               BIGINT       NOT NULL AUTO_INCREMENT COMMENT '链接编号',
  `tenant_id`        BIGINT       NOT NULL                COMMENT '租户编号',
  `book_id`          BIGINT       NOT NULL                COMMENT '账本编号',
  `creator_user_id`  BIGINT       NOT NULL                COMMENT '创建人（户主）',
  `token`            VARCHAR(64)  NOT NULL                COMMENT '邀请 token（UUID v4）',
  `usage_limit`      INT          NOT NULL DEFAULT 10     COMMENT '最大使用次数',
  `used_count`       INT          NOT NULL DEFAULT 0      COMMENT '已使用次数',
  `expire_time`      DATETIME     NOT NULL                COMMENT '过期时间',
  `status`           TINYINT      NOT NULL DEFAULT 1      COMMENT '状态：0=已撤销 1=有效',
  `creator`          VARCHAR(64)          DEFAULT ''     COMMENT '创建者',
  `create_time`      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP             COMMENT '创建时间',
  `updater`          VARCHAR(64)          DEFAULT ''     COMMENT '更新者',
  `update_time`      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`          TINYINT      NOT NULL DEFAULT 0      COMMENT '是否删除',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='账本邀请链接表';
```

### 5.5 完整 RBAC 权限矩阵

> 个人账本只有 owner 一人，下表的「户主」与「成员」都退化为「自己」，规则自动成立。

| 操作 | 户主 (role=1) | 普通成员 (role=2) |
| --- | :---: | :---: |
| 查看账本 | ✅ | ✅ |
| 修改账本设置（名称/预算/归档） | ✅ | ❌ |
| 删除账本 | ✅ | ❌ |
| 创建邀请（按用户名） | ✅ | ❌ |
| 创建邀请链接 | ✅ | ❌ |
| 撤销邀请/链接 | ✅ | ❌ |
| 查看成员列表 | ✅ | ✅ |
| 移除成员 | ✅ | ❌ |
| 创建账单 | ✅ | ✅ |
| 查看账单列表 | ✅ | ✅ |
| 修改自己的账单 | ✅ | ✅ |
| 删除自己的账单 | ✅ | ✅ |
| 修改他人账单 | ✅ | ❌ |
| 删除他人账单 | ✅ | ❌ |
| 退出账本 | ❌（需转让或删除）| ✅ |

### 5.6 权限服务接口

```java
public interface BookPermissionService {
    boolean canModifyBook (Long userId, Long bookId);                          // 修改账本设置
    boolean canDeleteBook (Long userId, Long bookId);                          // 删除账本
    boolean canInvite     (Long userId, Long bookId);                          // 邀请
    boolean canRemoveMember(Long userId, Long bookId, Long targetUserId);      // 移除成员
    boolean canViewBook   (Long userId, Long bookId);                          // 查看账本
    boolean canCreateBill (Long userId, Long bookId);                          // 记账
    boolean canUpdateBill (Long userId, Long bookId, Long billCreatorId);      // 改账单
    boolean canDeleteBill (Long userId, Long bookId, Long billCreatorId);      // 删账单
}
```

实现要点：

- `canModifyBook` / `canDeleteBook` / `canInvite` / `canRemoveMember`：查 `bill_book.owner_user_id == userId`
- `canViewBook` / `canCreateBill`：查 `bill_book_member` 是否有 `user_id=userId AND status=1` 的行
- `canUpdateBill` / `canDeleteBill`：`billCreatorId == userId` **或** 户主
- 所有方法命中即返回，`bill_book` / `bill_book_member` 表本身数据量极小（个人/家庭），无性能压力

### 5.7 API 概览（本模块负责）

所有 API 前缀 `/app-api/bill/book/` 与 `/app-api/bill/invitation/`。

#### 邀请管理（户主权限）

| 方法 + 路径 | 说明 |
| --- | --- |
| `POST /invite-by-user` | 按用户名发起定向邀请 |
| `POST /invite-link/create` | 生成邀请链接（带过期时间 + 使用次数） |
| `GET  /invite-link/list?bookId=` | 列出账本的有效链接 |
| `PUT  /invite-link/revoke?id=` | 撤销邀请链接 |
| `GET  /invitation/sent?bookId=` | 户主查看已发出的定向邀请列表 |

#### 接收邀请（被邀请人）

| 方法 + 路径 | 说明 |
| --- | --- |
| `GET  /invitation/received` | 当前用户收到的待处理邀请 |
| `POST /invitation/accept?id=` | 接受定向邀请（事务：INSERT member + UPDATE invitation） |
| `POST /invitation/reject?id=` | 拒绝定向邀请 |
| `POST /invite-link/accept?token=` | 凭 token 加入账本（事务：校验 token + INSERT member + UPDATE used_count） |

#### 成员管理（户主权限）

| 方法 + 路径 | 说明 |
| --- | --- |
| `GET  /member/list?bookId=` | 账本成员列表（含 role、加入时间、记账人昵称/头像） |
| `DELETE /member/remove?bookId=&userId=` | 移除成员（户主不能移除自己） |
| `POST /member/leave?bookId=` | 普通成员主动退出（户主调用此接口返回错误码） |

#### 账单列表增强（消费 §4 的 `/record/page`）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `records[].creatorInfo` | Object | 记账人摘要：`{userId, nickname, avatar}` |
| `records[].signedAmount` | 由前端计算 | 前端按 `type` 渲染符号：1 支出 → `-amount`，2 收入 → `+amount` |

后端**不返回**带符号的 `signedAmount` 字符串，避免多端展示不一致。

### 5.8 关键决策与扩展点

| 决策 | 原因 |
| --- | --- |
| 两张邀请表分开 | 定向与链接邀请的生命周期、字段语义完全不同，合一反而模糊 |
| UUID v4 存库比对 | 简单、可靠、可撤销；JWT 在这种"一次性凭据"场景反而过度设计 |
| 接受邀请走事务 | 防「邀请已撤销但成员已插入」的脏状态 |
| 权限校验下沉 Service | Controller 不硬编码规则字符串，方便后续加规则（如户主免审核、临时管理员） |
| 记账人信息走 Feign 批量查询 | 跨模块禁止 JOIN；`MemberUserApi.getUserSummary(List<Long>)` 一次性取所有记账人信息 |
| `amount` 始终正数 | 显示符号前端按 `type` 渲染，多端一致 |
| 户主不可退出 | 避免出现"无主账本"；要么转让，要么删除 |
| `expire_time` 在 Service 校验 | 状态字段不依赖定时任务跑批，读取时即时判定，避免脏状态 |
| `usage_limit` 默认 10 次 | 家庭场景下户主发一次链接让多人加入；过高则需考虑单独审核 |

### 5.9 状态流转图

**定向邀请 (`bill_book_invitation`)：**

```
            户主撤销 / 过期
   ┌──────────────────────────────┐
   ▼                              │
[待接受 0] ──接受──▶ [已接受 1]   │
   │                              │
   ├─拒绝──▶ [已拒绝 2]            │
   │                              │
   └──────────────────────────────┘
                  自动校验 expire_time
                  过期 → [已过期 3]
```

**邀请链接 (`bill_book_invite_link`)：**

```
                  户主撤销
   ┌──────────────────────────────┐
   ▼                              │
[有效 1] ──每次接受──▶ used_count++  │
   │                              │
   │  used_count >= usage_limit   │
   │  或 expire_time < NOW()       │
   │  ───────────────────▶ 链接失效（不物理变更 status，靠校验函数判断）
   │                              │
   └──────────────────────────────┘
```

> 链接状态**不**用定时任务改 0/1；全部用 `isLinkValid(link)` 校验函数在读取时即时判定：
> ```java
> boolean isLinkValid(InviteLinkDO link) {
>     return link.getStatus() == 1
>         && link.getUsedCount() < link.getUsageLimit()
>         && link.getExpireTime().isAfter(LocalDateTime.now());
> }
> ```

### 5.10 常见查询（验证索引够用）

```sql
-- 「我收到的待处理邀请」
SELECT * FROM bill_book_invitation
WHERE invitee_user_id = ? AND status = 0 AND deleted = 0
  AND expire_time > NOW()
ORDER BY create_time DESC;

-- 「户主发出的邀请」
SELECT * FROM bill_book_invitation
WHERE book_id = ? AND inviter_user_id = ? AND deleted = 0
ORDER BY create_time DESC;

-- 「账本当前所有有效链接」
SELECT * FROM bill_book_invite_link
WHERE book_id = ? AND status = 1 AND deleted = 0
  AND expire_time > NOW() AND used_count < usage_limit;

-- 「凭 token 查找链接」
SELECT * FROM bill_book_invite_link
WHERE token = ? AND deleted = 0 LIMIT 1;
```

### 5.11 与其他模块的衔接

| 关注点 | 关联模块 | 衔接方式 |
| --- | --- | --- |
| `bill_book` / `bill_book_member` | §3 | 本模块只读引用 + 写 `bill_book_member`（接受邀请时） |
| `bill_record.creatorInfo` | §4 | 扩展 `/record/page` 响应字段，调 `MemberUserApi` 取摘要 |
| 用户昵称/头像 | `yudao-module-member` | Feign RPC `MemberUserApi.getUserSummary(List<Long>)` |
| 站内信/推送通知 | `yudao-module-system` | 邀请发送后调用 `NotifyMessageApi.send(...)`（v1 用轮询 `GET /received`，不做实时推送） |
| 多端登录态 | `yudao-spring-boot-starter-security` | 所有 API 走 Token 鉴权 |

### 5.12 后续 TODO

- [ ] `BookPermissionService` 接口 + Impl（含 Redis 缓存户主判定，TTL 5 分钟）
- [ ] `BookInvitationService` / `BookInviteLinkService`
- [ ] `BillBookInvitationDO` / `BillBookInviteLinkDO` / Mapper
- [ ] `InvitationStatusEnum` / `InviteLinkStatusEnum` / `BookMemberRoleEnum`（在 §3 TODO 基础上补齐角色枚举）
- [ ] `MemberUserApi.getUserSummary` Feign 接口定义（与 `yudao-module-member` 对齐）
- [ ] 接受邀请事务单元测试（模拟并发接受同一邀请，验证幂等）
- [ ] `/record/page` 响应扩展 `creatorInfo`，Service 层批量查 user → 合并
- [ ] 户主转让接口（v2 再议，本期不实现）
- [ ] 邀请到期定时清理任务（v2；v1 靠读取时即时校验）

---

## 6. 索引设计（性能优化，后续迭代）

> 本节是**索引与唯一约束**的统一设计稿，**不进入首版 DDL**。  
> 触发条件：
> 1. 单表数据量预估 > 10 万行（`bill_record` 最早达到）
> 2. 上线后通过 `EXPLAIN` / slow query log 抓到全表扫描
>
> 同时建议在 Service 层**冗余实现**唯一约束（创建分类、加入账本等），即使索引未上线数据也不脏。

### 6.1 `bill_category`

**唯一约束**

```sql
UNIQUE KEY `uk_user_type_name` (`user_id`, `type`, `name`, `deleted`)
-- 业务含义：同一用户下，type + name + 未删除 三元组唯一；软删行不参与去重，允许重建同名。
```

**性能索引**

```sql
KEY `idx_user_list` (`user_id`, `type`, `status`, `deleted`, `sort`)
-- 覆盖查询：用户拉取分类列表（按 type 过滤 + 按 sort 排序）。
```

### 6.2 `bill_book`

```sql
KEY `idx_owner_list`   (`owner_user_id`, `status`, `deleted`, `sort`),  -- 「我作为户主的账本」
KEY `idx_tenant_type`  (`tenant_id`, `type`, `deleted`)                -- 后台管理跨租户查询
```

### 6.3 `bill_book_member`

**唯一约束**

```sql
UNIQUE KEY `uk_book_user` (`book_id`, `user_id`, `deleted`)
-- 业务含义：同一账本 + 同一用户 + 未删除 三元组唯一；退出的成员允许重新加入。
```

**性能索引**

```sql
KEY `idx_user_book` (`user_id`, `status`, `deleted`)
-- 覆盖查询：「我参与的所有账本」。
```

### 6.4 `bill_record`

> **最早触发索引优化的表**：账单数据增长最快（人均每天 1~10 笔），全表扫描在万级后会明显劣化。

```sql
KEY `idx_book_time`           (`book_id`, `deleted`, `status`, `record_time`),          -- 账本内全部账单
KEY `idx_book_type_time`      (`book_id`, `type`, `deleted`, `status`, `record_time`),  -- 账本 + 收支方向
KEY `idx_book_category_time`  (`book_id`, `category_id`, `deleted`, `status`, `record_time`),  -- 账本 + 单分类
KEY `idx_user_time`           (`user_id`, `deleted`, `status`, `record_time`)           -- 「我的账单」跨账本聚合
```

### 6.5 实施步骤（备忘）

1. **预创建**：首版 DDL **只保留** `PRIMARY KEY`，应用上线后立刻补跑 migration：
   ```sql
   ALTER TABLE bill_record ADD INDEX idx_book_time (...);
   ```
2. **监控**：开启 MySQL slow query log（`long_query_time=1`），按周 review。
3. **回退**：索引上线后 1 周内观察 `Handler_read_rnd_next`（应下降）；不降则 drop。

---

## 7. 管理后台功能设计

> 本节只覆盖 `yudao-module-bill` 在 admin 端的**业务专属**功能。  
> yudao 框架自带的管理能力（用户/角色/菜单/字典/日志/短信/邮件/站内信/敏感词/文件/定时任务/配置/API日志/代码生成等）由 `yudao-module-system` + `yudao-module-infra` 提供，**`yudao-server/pom.xml` 解开对应模块依赖即可启用**，不在本节重复设计。

### 7.1 设计原则

1. **admin 只读为主**： 默认情况下 admin 只能查询 C 端用户的数据，**不能代记账、不能改账、不能改用户账本设置**。干预操作（强制删除/禁用）走专门的 `admin_*` 接口，必须二次鉴权 + 操作审计。
2. **强审计**： 所有 admin 接口的访问记录必须进 `yudao_operate_log`（yudao 自带 `bizlog-sdk` + `@LogRecord` 注解），记录「谁、什么时间、查了谁的什么数据」。
3. **敏感字段默认脱敏**： 列表展示时 `description` 中段 `****`、手机号/邮箱中间 4 位 `*`，点详情才显示原文。yudao 自带 `PrivacyUtils` 可用。
4. **强制干预标记化**： 强制删除的账单**不能**和用户自删混为一谈，必须能在审计时区分来源（详见 §7.4 schema 改动）。
5. **不能批量导出 C 端原始数据**： v1 不提供 admin 端「一键导出某用户全部账单」功能。导出走用户自助 + 数据脱敏，避免法务风险。

### 7.2 P0：基础监管（v1 必做）

所有 P0 接口前缀 `/admin-api/bill/**`，仅 `isAdmin()` 通过的登录用户可访问。

#### 7.2.1 用户账本概览

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `GET /admin-api/bill/book/list-by-user` |
| 查询参数 | `userId` 必填、`type` 选填（1 个人 / 2 家庭）、`pageNum` / `pageSize` |
| 响应 | `List<BookRespVO>`，含账本名称、类型、状态、成员数、账单总数、创建时间 |
| 权限注解 | `@PreAuthorize("@ss.hasRole('admin')")` |
| 审计字段 | 操作人、被查询 userId、查询时间 |
| 备注 | 只读，不可修改 |

#### 7.2.2 账单查询（跨用户）

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `GET /admin-api/bill/record/page` |
| 查询参数 | `userId` / `bookId` / `type` / `categoryId` / `startTime` / `endTime` / `minAmount` / `maxAmount` / `keyword` / `pageNum` / `pageSize` |
| 响应 | `PageResult<RecordRespVO>`，含 `creatorInfo`（记账人昵称/头像）、`bookName`、`categoryName`、`description`（**列表脱敏**） |
| 权限注解 | `@PreAuthorize("@ss.hasRole('admin')")` |
| 审计字段 | 操作人、查询条件全量记录 |
| 备注 | 响应体大小限制 `maxPageSize=100`；超大数据用 `EXPORT_ASYNC` 走异步导出（v2） |

#### 7.2.3 账单详情查看

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `GET /admin-api/bill/record/get?id=` |
| 响应 | `RecordRespVO` 全字段，**`description` 明文** |
| 权限注解 | `@PreAuthorize("@ss.hasRole('admin')")` |
| 审计字段 | 操作人、被查看账单 id、查看时间 |
| 备注 | 详情接口访问必须记录，列表查询与详情查询**分开审计**（详查询询是敏感操作） |

#### 7.2.4 用户分类查看

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `GET /admin-api/bill/category/list-by-user?userId=` |
| 响应 | `List<CategoryRespVO>`，区分 `builtin=1` / `builtin=0` |
| 权限注解 | `@PreAuthorize("@ss.hasRole('admin')")` |
| 备注 | 用于运营分析（用户偏好）、违规识别（如分类名含敏感词） |

#### 7.2.5 系统预置分类维护

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `PUT /admin-api/bill/category/system/update` |
| 入参 | 修改 §2.5 默认分类的名称 / icon / sort；不能删除（避免老用户 seed 缺失） |
| 权限注解 | `@PreAuthorize("@ss.hasRole('admin')")` + `@LogRecord("更新系统预置分类")` |
| 影响范围 | **只影响后续新注册用户**，存量用户已 seed 的数据**不动**（与 §2.5 升级 seed 逻辑保持一致） |

#### 7.2.6 仪表盘数据

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `GET /admin-api/bill/statistics/dashboard` |
| 响应 | `{ totalUsers, todayActiveUsers, monthlyActiveUsers, totalBooks, totalRecords, todayRecords, todayAmount, monthlyAmount, topCategories: [{categoryId, name, count, amount}] }` |
| 备注 | 聚合 SQL 走 5+ 表 JOIN，**性能敏感**；建议加 Redis 缓存 5 分钟，或建宽表定时刷新（性能优化阶段处理） |

### 7.3 P1：干预能力（v1 视情况）

P1 接口**默认关闭**（不暴露路由、不写 Service 方法），待运营有明确诉求再启用。每启用一项必须配套：

- 二次鉴权（角色 + 操作原因必填）
- 操作日志（强制记录原因 + 操作人 + 被操作对象）
- 数据备份（操作前先快照到 `*_snapshot` 表，30 天后可清理）

#### 7.3.1 账单强制删除

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `DELETE /admin-api/bill/record/force-delete?id=` |
| 入参 | `reason` 必填（≥ 5 字） |
| 行为 | `deleted=1` + 写入 §7.4 schema 改动中的强制删除字段 |
| 权限注解 | `@PreAuthorize("@ss.hasRole('admin') and @ss.hasAuthority('bill:admin:force-delete')")` |
| 备注 | 普通列表不展示被 admin 强制删除的账单；用户侧「回收站」也不可见 |

#### 7.3.2 账本禁用

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `PUT /admin-api/bill/book/admin-disable` |
| 入参 | `bookId` + `reason` |
| 行为 | §7.4 schema 中加 `admin_disabled_*` 字段；账本所有 C 端写接口前置校验该字段 |
| 备注 | 仅家庭账本允许禁用（个人账本直接封禁用户即可） |

#### 7.3.3 冻结用户记账

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `PUT /admin-api/bill/user-freeze` |
| 入参 | `userId` + `reason` + `expireTime` |
| 行为 | 新建 `bill_user_freeze` 表（或 `system_users` 加 freeze 字段）；所有 C 端记账接口前置校验 |
| 备注 | 冻结期间用户可查看历史账单，但不可新增/修改/删除 |

#### 7.3.4 邀请链接强制撤销

| 字段 | 值 |
| --- | --- |
| 方法 + 路径 | `PUT /admin-api/bill/invite-link/admin-revoke?id=` |
| 行为 | 复用 §5 表的 `status=0`，但记录撤销人为 admin 而非户主 |
| 备注 | 与户主撤销走相同字段，靠 `updater` 区分操作人 |

### 7.4 P1 配套 Schema 改动（待启用 P1 时统一迁移）

> ⚠️ 这些字段**不进首版 DDL**。等到 P1 任一功能启用时，与对应 Service 实现一起 ALTER 进库。

#### 7.4.1 `bill_record` 新增字段

```sql
ALTER TABLE bill_record
  ADD COLUMN `admin_delete_flag`     TINYINT      NOT NULL DEFAULT 0      COMMENT '是否被 admin 强制删除：0=否 1=是',
  ADD COLUMN `admin_delete_user_id`  BIGINT                DEFAULT NULL   COMMENT '强制删除操作人 user_id',
  ADD COLUMN `admin_delete_time`     DATETIME              DEFAULT NULL   COMMENT '强制删除时间',
  ADD COLUMN `admin_delete_reason`   VARCHAR(255)          DEFAULT NULL   COMMENT '强制删除原因';
```

应用层逻辑：`deleted=0` AND `admin_delete_flag=0` 才在用户侧可见；admin 端可见全部。

#### 7.4.2 `bill_book` 新增字段

```sql
ALTER TABLE bill_book
  ADD COLUMN `admin_disabled_flag`     TINYINT      NOT NULL DEFAULT 0      COMMENT '是否被 admin 禁用：0=否 1=是',
  ADD COLUMN `admin_disabled_user_id`  BIGINT                DEFAULT NULL   COMMENT '禁用操作人 user_id',
  ADD COLUMN `admin_disabled_time`     DATETIME              DEFAULT NULL   COMMENT '禁用时间',
  ADD COLUMN `admin_disabled_reason`   VARCHAR(255)          DEFAULT NULL   COMMENT '禁用原因';
```

应用层逻辑：`admin_disabled_flag=1` 时，所有 C 端写账本的接口（createBill / updateBill / deleteBill / updateBook / inviteMember 等）统一返回 `BOOK_DISABLED_BY_ADMIN` 错误码。

#### 7.4.3 `bill_book_invite_link` 无需新增字段

靠 `updater` 字段区分户主撤销 vs admin 撤销；审计日志里补 `operator_role=admin|owner` 即可。

### 7.5 P2：运营分析（v2 预留）

| 功能 | 优先级 | 备注 |
| --- | --- | --- |
| 用户画像（消费分类 Top 10） | P2 | 走离线数仓，admin 端只看结果宽表 |
| 异常账本识别 | P2 | 成员数 / 流水金额阈值告警 |
| 用户自助数据导出 | P2 | 用户在 C 端申请，异步生成加密压缩包 |
| 家庭账本协作申诉处理 | P2 | 配合 `yudao-module-bpm` 工作流 |

### 7.6 审计与脱敏

#### 7.6.1 操作日志模板（yudao `@LogRecord`）

```java
@LogRecord(
    value = "查询用户账单",
    type = "BILL_RECORD_QUERY",
    bizId = "#userId",           // 被查询 userId
    extra = """{
        |"queryParams": #{T(com.alibaba.fastjson.JSON).toJSONString(#queryParams)},
        |"resultCount": #{result.total}
    }""".stripMargin()
)
public PageResult<RecordRespVO> adminQueryRecords(RecordQuery query) { ... }
```

#### 7.6.2 脱敏规则

| 字段 | 列表展示 | 详情展示 |
| --- | --- | --- |
| `description` | 中段 4 位 `****` | 明文 |
| `memberUser.mobile` | `138****1234` | `hasAuthority('bill:admin:view-sensitive')` 才返回明文 |
| `memberUser.email` | `y****@example.com` | 同上 |
| 金额 | 完整 | 完整 |

### 7.7 后续 TODO

- [ ] P0 6 个 admin 接口实现 + `@LogRecord` + 二次鉴权
- [ ] P0 仪表盘聚合查询（5+ 表 JOIN，先实现后优化；性能问题在 §6 索引上线后再说）
- [ ] 用户自助数据导出（C 端 + admin 双通道）
- [ ] P1 触发时统一迁移 §7.4 schema 改动
- [ ] 审计日志查看页（admin 端操作流水可视化）
- [ ] admin 端敏感字段查看权限分级（普通管理员 vs 超级管理员）
4. **Service 层去重**：所有 `UNIQUE KEY` 涉及的字段，`insertOrUpdate` 前必须先 `existsByXxx()` 校验，不依赖数据库兜底。