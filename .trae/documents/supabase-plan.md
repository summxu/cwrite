# Supabase 数据库设计与实现方案

## 1. 项目目标与约束提取

### 关键项目目标
- **用户管理**: 支持管理员、老师和学生角色。管理员创建账号，无自主注册。
- **班级管理**: 管理员创建和管理班级，学生分组到班级。
- **课程管理**: 老师创建课程，包括大纲、评估量规、提交设置和关联班级。
- **日志编写**: 学生按PAVE步骤编写多个日志，支持草稿和提交。
- **评估功能**: AI根据量规评估日志，老师参考给出最终评估。
- **AI集成**: 使用指定大模型进行评估和提示。

### 约束条件
- **数据库**: Supabase PostgreSQL，使用supabase-js客户端进行所有CURD操作。
- **认证**: Supabase RLS（Row Level Security）系统，无其他认证。
- **操作方式**: 通过Supabase MCP服务进行数据库结构变更、RLS配置等，无本地CLI或迁移。
- **安全**: 严格RLS策略，确保数据隔离（如学生只能访问自己的日志）。

## 2. 数据库表设计

### 表结构

#### users
- id: uuid (primary key, auto-generated)
- email: text (unique)
- password: text (hashed)
- role: text (admin, teacher, student)
- class_id: uuid (foreign key to classes, only for students)
- created_at: timestamptz

#### classes
- id: uuid (primary key)
- name: text (unique)
- created_at: timestamptz

#### courses
- id: uuid (primary key)
- name: text
- outline: text
- teacher_id: uuid (foreign key to users)
- created_at: timestamptz

#### course_classes
- course_id: uuid (foreign key to courses)
- class_id: uuid (foreign key to classes)
- primary key (course_id, class_id)

#### course_steps
- id: uuid (primary key)
- course_id: uuid (foreign key to courses)
- step_name: text (e.g., Plan, Analysis, Value, Evaluation)
- rubric: text (评估量规)
- submission_limit: integer
- deadline: timestamptz

#### journals
- id: uuid (primary key)
- student_id: uuid (foreign key to users)
- course_id: uuid (foreign key to courses)
- step_id: uuid (foreign key to course_steps)
- content: text
- status: text (draft, submitted)
- submission_number: integer
- created_at: timestamptz
- submitted_at: timestamptz

#### evaluations
- id: uuid (primary key)
- journal_id: uuid (foreign key to journals)
- ai_evaluation: text
- teacher_evaluation: text
- final_score: integer
- created_at: timestamptz

### 关系设计
- users 一对多 classes (学生属于一个班级)
- users (老师) 一对多 courses
- courses 多对多 classes (通过course_classes)
- courses 一对多 course_steps
- journals 多对一 users (student), courses, course_steps
- evaluations 一对一 journals

## 3. RLS 策略设计

- **users**: 仅管理员可插入/更新，行级：用户只能读取/更新自己的记录。
- **classes**: 仅管理员可操作。
- **courses**: 老师可插入/更新自己的课程，学生可读取所属班级的课程。
- **course_steps**: 与courses类似。
- **journals**: 学生可插入/更新自己的草稿，提交后只读；老师可读取所属课程的学生日志。
- **evaluations**: 老师可插入/更新，学生不可见AI评估。

具体RLS通过Supabase MCP服务配置。