# AI-Pave 数据库设计文档

## 设计概览

**项目**: AI-Pave 智能教学平台  
**数据库**: PostgreSQL (Supabase 托管)  
**设计版本**: 1.0  
**设计日期**: 2025-08-08  

### 核心设计理念

1. **教育场景优先**: 围绕 PAVE 教学法设计，支持计划(Plan)、分析(Analysis)、价值(Value)、评估(Evaluation)四个步骤
2. **角色权限分离**: 严格区分管理员、教师、学生三种角色的数据访问权限
3. **数据完整性**: 通过约束、触发器、外键确保数据一致性和业务规则
4. **性能优化**: 合理的索引设计支持高频查询场景
5. **安全第一**: 行级安全(RLS)策略确保数据访问安全

## 数据库架构

### 架构图

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     classes     │    │      users      │    │    courses      │
│                 │    │                 │    │                 │
│ • id (PK)       │◄──┤ • class_id (FK) │    │ • teacher_id(FK)├──┐
│ • name          │    │ • role          │    │ • name          │  │
│ • description   │    │ • email         │    │ • description   │  │
└─────────────────┘    └─────────────────┘    └─────────────────┘  │
         ▲                       ▲                       ▲          │
         │                       │                       │          │
         │              ┌─────────────────┐              │          │
         │              │ course_classes  │              │          │
         │              │                 │              │          │
         └──────────────┤ • class_id (FK) │              │          │
                        │ • course_id(FK) ├──────────────┘          │
                        └─────────────────┘                         │
                                 ▲                                  │
                                 │                                  │
                        ┌─────────────────┐                         │
                        │  course_steps   │                         │
                        │                 │                         │
                        │ • course_id(FK) ├─────────────────────────┘
                        │ • step_name     │
                        │ • step_order    │
                        └─────────────────┘
                                 ▲
                                 │
                        ┌─────────────────┐    ┌─────────────────┐
                        │    journals     │    │   evaluations   │
                        │                 │    │                 │
                        │ • student_id(FK)├────┤ • journal_id(FK)│
                        │ • step_id (FK)  ├────┤ • ai_evaluation │
                        │ • status        │    │ • teacher_eval  │
                        │ • content       │    │ • final_score   │
                        └─────────────────┘    └─────────────────┘
```

### 核心实体关系

1. **用户-班级关系**: 学生属于班级，教师和管理员不属于班级
2. **课程-教师关系**: 每个课程由一个教师创建和管理
3. **课程-班级关系**: 多对多关系，一个课程可以分配给多个班级
4. **课程-步骤关系**: 一对多关系，每个课程包含多个 PAVE 步骤
5. **学生-日志关系**: 一对多关系，学生可以为每个步骤提交多次
6. **日志-评价关系**: 一对一关系，每个日志对应一个评价记录

## 数据表设计

### 1. 枚举类型定义

#### user_role 枚举
```sql
CREATE TYPE user_role AS ENUM ('admin', 'teacher', 'student');
```
**用途**: 定义用户角色类型  
**值域**: 管理员、教师、学生  

#### journal_status 枚举
```sql
CREATE TYPE journal_status AS ENUM ('draft', 'submitted');
```
**用途**: 定义日志提交状态  
**值域**: 草稿、已提交  

#### pave_step 枚举
```sql
CREATE TYPE pave_step AS ENUM ('plan', 'analysis', 'value', 'evaluation');
```
**用途**: 定义 PAVE 教学步骤  
**值域**: 计划、分析、价值、评估  

### 2. 核心数据表

#### classes 表 - 班级信息

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 班级唯一标识 |
| name | TEXT | NOT NULL, UNIQUE | 班级名称 |
| description | TEXT | NULL | 班级描述 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 更新时间 |

**索引**:
- `idx_classes_name`: 班级名称索引，支持快速查找

**业务规则**:
- 班级名称必须唯一
- 支持软删除（通过应用层控制）

#### users 表 - 用户信息

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 用户唯一标识 |
| email | TEXT | NOT NULL, UNIQUE | 用户邮箱 |
| password_hash | TEXT | NOT NULL | 密码哈希 |
| role | user_role | NOT NULL, DEFAULT 'student' | 用户角色 |
| class_id | UUID | FK → classes(id) | 所属班级（学生专用） |
| first_name | TEXT | NULL | 名 |
| last_name | TEXT | NULL | 姓 |
| is_active | BOOLEAN | DEFAULT TRUE | 账户状态 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 更新时间 |

**约束**:
```sql
CONSTRAINT check_student_class_id CHECK (
    (role = 'student' AND class_id IS NOT NULL) OR 
    (role IN ('admin', 'teacher') AND class_id IS NULL)
)
```

**索引**:
- `idx_users_email`: 邮箱索引，支持登录查询
- `idx_users_role`: 角色索引，支持角色筛选
- `idx_users_class_id`: 班级索引，支持班级成员查询
- `idx_users_is_active`: 状态索引，支持活跃用户查询

**业务规则**:
- 学生必须属于一个班级，教师和管理员不属于班级
- 邮箱作为登录凭证，必须唯一
- 支持账户禁用功能

#### courses 表 - 课程信息

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 课程唯一标识 |
| name | TEXT | NOT NULL | 课程名称 |
| description | TEXT | NULL | 课程描述 |
| outline | TEXT | NULL | 课程大纲 |
| teacher_id | UUID | NOT NULL, FK → users(id) | 授课教师 |
| is_active | BOOLEAN | DEFAULT TRUE | 课程状态 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 更新时间 |

**索引**:
- `idx_courses_teacher_id`: 教师索引，支持教师课程查询
- `idx_courses_is_active`: 状态索引，支持活跃课程查询
- `idx_courses_name`: 名称索引，支持课程搜索

**业务规则**:
- 每个课程由一个教师负责
- 支持课程禁用功能
- 删除教师时级联删除其课程

#### course_classes 表 - 课程班级关联

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| course_id | UUID | PK, FK → courses(id) | 课程ID |
| class_id | UUID | PK, FK → classes(id) | 班级ID |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 关联创建时间 |

**主键**: (course_id, class_id) 复合主键

**索引**:
- `idx_course_classes_course_id`: 课程索引
- `idx_course_classes_class_id`: 班级索引

**业务规则**:
- 多对多关系，一个课程可以分配给多个班级
- 删除课程或班级时级联删除关联记录

#### course_steps 表 - 课程步骤

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 步骤唯一标识 |
| course_id | UUID | NOT NULL, FK → courses(id) | 所属课程 |
| step_name | pave_step | NOT NULL | PAVE步骤名称 |
| step_order | INTEGER | NOT NULL | 步骤顺序 |
| title | TEXT | NOT NULL | 步骤标题 |
| description | TEXT | NULL | 步骤描述 |
| rubric | TEXT | NOT NULL | 评分标准 |
| submission_limit | INTEGER | NOT NULL, DEFAULT 1, CHECK > 0 | 提交次数限制 |
| deadline | TIMESTAMPTZ | NULL | 截止时间 |
| is_active | BOOLEAN | DEFAULT TRUE | 步骤状态 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 更新时间 |

**唯一约束**:
- `UNIQUE(course_id, step_name)`: 同一课程中步骤名称唯一
- `UNIQUE(course_id, step_order)`: 同一课程中步骤顺序唯一

**索引**:
- `idx_course_steps_course_id`: 课程索引
- `idx_course_steps_step_name`: 步骤名称索引
- `idx_course_steps_step_order`: 步骤顺序索引
- `idx_course_steps_is_active`: 状态索引
- `idx_course_steps_deadline`: 截止时间索引

**业务规则**:
- 每个课程的 PAVE 四个步骤都必须定义
- 步骤顺序决定学生完成的先后顺序
- 支持多次提交，但有次数限制
- 支持设置截止时间

#### journals 表 - 学习日志

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 日志唯一标识 |
| student_id | UUID | NOT NULL, FK → users(id) | 提交学生 |
| course_id | UUID | NOT NULL, FK → courses(id) | 所属课程 |
| step_id | UUID | NOT NULL, FK → course_steps(id) | 对应步骤 |
| title | TEXT | NULL | 日志标题 |
| content | TEXT | NOT NULL | 日志内容 |
| status | journal_status | NOT NULL, DEFAULT 'draft' | 提交状态 |
| submission_number | INTEGER | NOT NULL, DEFAULT 1, CHECK > 0 | 提交次数 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 更新时间 |
| submitted_at | TIMESTAMPTZ | NULL | 提交时间 |

**约束**:
```sql
-- 提交时间约束
CONSTRAINT check_submission_time CHECK (
    (status = 'draft' AND submitted_at IS NULL) OR
    (status = 'submitted' AND submitted_at IS NOT NULL)
)

-- 提交次数约束
CONSTRAINT check_submission_number CHECK (submission_number > 0)
```

**唯一约束**:
- `UNIQUE(student_id, course_id, step_id, submission_number)`: 防止重复提交

**索引**:
- `idx_journals_student_id`: 学生索引
- `idx_journals_course_id`: 课程索引
- `idx_journals_step_id`: 步骤索引
- `idx_journals_status`: 状态索引
- `idx_journals_created_at`: 创建时间索引
- `idx_journals_submitted_at`: 提交时间索引
- `idx_journals_composite`: 复合索引 (student_id, course_id, step_id)

**业务规则**:
- 学生可以为每个步骤提交多次，但受步骤限制
- 草稿状态可以修改，提交后不可修改
- 提交时自动设置提交时间
- 支持版本控制（通过提交次数）

#### evaluations 表 - 评价记录

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 评价唯一标识 |
| journal_id | UUID | NOT NULL, UNIQUE, FK → journals(id) | 对应日志 |
| ai_evaluation | JSONB | NULL | AI评价结果 |
| ai_score | INTEGER | NULL, CHECK 0-100 | AI评分 |
| teacher_evaluation | TEXT | NULL | 教师评价 |
| teacher_score | INTEGER | NULL, CHECK 0-100 | 教师评分 |
| final_score | INTEGER | NULL, CHECK 0-100 | 最终得分 |
| is_final | BOOLEAN | DEFAULT FALSE | 是否最终评价 |
| created_at | TIMESTAMPTZ | DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() | 更新时间 |

**约束**:
```sql
CONSTRAINT check_final_evaluation CHECK (
    (is_final = FALSE) OR 
    (is_final = TRUE AND teacher_evaluation IS NOT NULL AND final_score IS NOT NULL)
)
```

**索引**:
- `idx_evaluations_journal_id`: 日志索引
- `idx_evaluations_is_final`: 最终评价索引
- `idx_evaluations_created_at`: 创建时间索引

**业务规则**:
- 每个日志对应一个评价记录
- AI评价自动生成，教师评价手动输入
- 最终评价必须包含教师评价和最终得分
- 支持评价修订（通过更新时间跟踪）

## 业务逻辑实现

### 触发器系统

#### 1. 自动更新时间戳

**函数**: `update_updated_at_column()`
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';
```

**应用表**: 所有核心表  
**触发时机**: BEFORE UPDATE  
**作用**: 自动更新 updated_at 字段

#### 2. 日志提交时间管理

**函数**: `set_journal_submitted_at()`
```sql
CREATE OR REPLACE FUNCTION set_journal_submitted_at()
RETURNS TRIGGER AS $$
BEGIN
    -- 状态从 draft 变为 submitted 时设置提交时间
    IF OLD.status = 'draft' AND NEW.status = 'submitted' THEN
        NEW.submitted_at = NOW();
    -- 状态从 submitted 变为 draft 时清除提交时间
    ELSIF OLD.status = 'submitted' AND NEW.status = 'draft' THEN
        NEW.submitted_at = NULL;
    END IF;
    
    RETURN NEW;
END;
$$ language 'plpgsql';
```

**应用表**: journals  
**触发时机**: BEFORE UPDATE  
**作用**: 根据状态变化自动管理提交时间

### 权限控制系统 (RLS)

#### 权限辅助函数

1. **get_current_user_role()**: 获取当前用户角色
2. **get_current_user_class_id()**: 获取当前用户班级ID
3. **is_course_teacher(course_uuid)**: 检查是否为课程教师
4. **can_student_access_course(course_uuid)**: 检查学生课程访问权限

#### 权限策略矩阵

| 表名 | 管理员 | 教师 | 学生 |
|------|--------|------|------|
| classes | 全部访问 | 查看自己班级 | 查看自己班级 |
| users | 全部访问 | 查看班级学生+自己 | 查看自己 |
| courses | 全部访问 | 管理自己课程 | 查看分配课程 |
| course_classes | 全部访问 | 管理自己课程关联 | 查看自己班级关联 |
| course_steps | 全部访问 | 管理自己课程步骤 | 查看分配步骤 |
| journals | 全部访问 | 查看自己课程日志 | 管理自己日志 |
| evaluations | 全部访问 | 管理自己课程评价 | 查看自己评价 |

## 性能优化策略

### 索引设计原则

1. **主键索引**: 所有表使用 UUID 主键，自动创建唯一索引
2. **外键索引**: 所有外键字段创建索引，优化关联查询
3. **查询索引**: 根据常用查询模式创建单列索引
4. **复合索引**: 为复杂查询创建多列复合索引
5. **状态索引**: 为状态字段创建索引，支持筛选查询

### 查询优化

#### 高频查询场景

1. **学生课程列表**: 通过班级关联查询分配的课程
2. **教师课程管理**: 查询教师创建的所有课程
3. **日志列表**: 按学生、课程、步骤查询日志
4. **评价统计**: 按课程、班级统计评价结果

#### 索引使用策略

```sql
-- 学生查询分配课程（使用复合索引）
SELECT c.* FROM courses c
JOIN course_classes cc ON c.id = cc.course_id
WHERE cc.class_id = get_current_user_class_id()
AND c.is_active = true;
-- 使用索引: idx_course_classes_class_id, idx_courses_is_active

-- 教师查询课程日志（使用复合索引）
SELECT j.* FROM journals j
JOIN courses c ON j.course_id = c.id
WHERE c.teacher_id = auth.uid()
AND j.status = 'submitted'
ORDER BY j.submitted_at DESC;
-- 使用索引: idx_courses_teacher_id, idx_journals_status, idx_journals_submitted_at
```

### 数据分区策略

**未来扩展考虑**:
- 按时间分区 journals 表（按学期或年度）
- 按课程分区 evaluations 表
- 归档历史数据到单独分区

## 数据安全设计

### 行级安全 (RLS) 策略

#### 设计原则
1. **最小权限原则**: 用户只能访问必要的数据
2. **角色隔离**: 不同角色严格隔离数据访问
3. **业务逻辑**: 权限策略反映真实业务场景
4. **性能考虑**: 权限检查不影响查询性能

#### 安全函数设计
- 使用 `SECURITY DEFINER` 确保函数执行权限
- 函数内部使用 `auth.uid()` 获取当前用户
- 避免在权限函数中进行复杂计算

### 数据完整性保障

#### 约束层次
1. **数据类型约束**: 使用枚举类型限制值域
2. **NOT NULL 约束**: 确保关键字段不为空
3. **唯一约束**: 防止重复数据
4. **检查约束**: 实现复杂业务规则
5. **外键约束**: 维护引用完整性

#### 业务规则实现
- 学生班级归属检查
- 日志提交状态一致性
- 评价完整性验证
- 提交次数限制

## 扩展性设计

### 水平扩展
- UUID 主键支持分布式环境
- 无自增序列依赖
- 支持读写分离

### 垂直扩展
- JSONB 字段支持灵活数据结构
- 枚举类型支持值域扩展
- 触发器系统支持业务逻辑扩展

### 功能扩展点
1. **多媒体支持**: 在 journals 表中添加附件字段
2. **协作功能**: 添加小组表和成员关系
3. **通知系统**: 添加消息表和推送机制
4. **统计分析**: 添加报表视图和聚合表
5. **版本控制**: 扩展日志版本管理

## 维护和监控

### 性能监控
- 查询执行计划分析
- 索引使用率统计
- 慢查询日志监控
- 数据库连接池监控

### 数据维护
- 定期更新表统计信息
- 清理过期数据
- 重建碎片化索引
- 备份策略执行

### 安全审计
- RLS 策略有效性检查
- 权限函数安全审查
- 数据访问日志分析
- 异常访问模式检测

## 部署和迁移

### 部署环境
- **生产环境**: Supabase 托管 PostgreSQL
- **开发环境**: 本地 PostgreSQL 或 Supabase 开发项目
- **测试环境**: 独立的 Supabase 测试项目

### 迁移策略
- 使用 Supabase MCP 服务进行结构变更
- 分步骤执行复杂迁移
- 每步验证数据完整性
- 保留迁移执行记录

### 回滚计划
- 结构变更回滚脚本
- 数据备份恢复流程
- 应用程序兼容性检查
- 紧急修复流程

---

**文档版本**: 1.0  
**创建日期**: 2025-08-08  
**维护团队**: AI-Pave 开发团队  
**审查周期**: 每季度或重大变更时  
**相关文档**: 
- [Supabase SQL 执行记录](./supabase-sql-execution.md)
- [API 设计文档](./api-design.md)
- [前端架构文档](./frontend-architecture.md)