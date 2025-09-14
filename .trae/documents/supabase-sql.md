# Supabase 数据库 SQL 实现代码

## 1. 数据库表结构创建

### 1.2 创建枚举类型

```sql
-- 用户角色枚举
CREATE TYPE user_role AS ENUM ('admin', 'teacher', 'student');

-- 日志状态枚举
CREATE TYPE journal_status AS ENUM ('draft', 'submitted');

-- PAVE步骤枚举
CREATE TYPE pave_step AS ENUM ('plan', 'analysis', 'value', 'evaluation');
```

### 1.3 创建表结构

#### classes 表
```sql
CREATE TABLE classes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 创建索引
CREATE INDEX idx_classes_name ON classes(name);
CREATE INDEX idx_classes_created_at ON classes(created_at);
```

#### users 表
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role user_role NOT NULL DEFAULT 'student',
    class_id UUID REFERENCES classes(id) ON DELETE SET NULL,
    first_name TEXT,
    last_name TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- 约束：只有学生才能有class_id
    CONSTRAINT check_student_class CHECK (
        (role = 'student' AND class_id IS NOT NULL) OR 
        (role IN ('admin', 'teacher') AND class_id IS NULL)
    )
);

-- 创建索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_class_id ON users(class_id);
CREATE INDEX idx_users_created_at ON users(created_at);
```

#### courses 表
```sql
CREATE TABLE courses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    outline TEXT,
    teacher_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- 确保teacher_id对应的用户是老师
    CONSTRAINT check_teacher_role CHECK (
        teacher_id IN (SELECT id FROM users WHERE role = 'teacher')
    )
);

-- 创建索引
CREATE INDEX idx_courses_teacher_id ON courses(teacher_id);
CREATE INDEX idx_courses_name ON courses(name);
CREATE INDEX idx_courses_created_at ON courses(created_at);
CREATE INDEX idx_courses_active ON courses(is_active);
```

#### course_classes 表（多对多关系）
```sql
CREATE TABLE course_classes (
    course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    class_id UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    PRIMARY KEY (course_id, class_id)
);

-- 创建索引
CREATE INDEX idx_course_classes_course_id ON course_classes(course_id);
CREATE INDEX idx_course_classes_class_id ON course_classes(class_id);
```

#### course_steps 表
```sql
CREATE TABLE course_steps (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    step_name pave_step NOT NULL,
    step_order INTEGER NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    rubric TEXT NOT NULL,
    submission_limit INTEGER NOT NULL DEFAULT 1 CHECK (submission_limit > 0),
    deadline TIMESTAMPTZ,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- 确保每个课程的每个步骤只有一个
    UNIQUE(course_id, step_name),
    -- 确保步骤顺序唯一
    UNIQUE(course_id, step_order)
);

-- 创建索引
CREATE INDEX idx_course_steps_course_id ON course_steps(course_id);
CREATE INDEX idx_course_steps_step_name ON course_steps(step_name);
CREATE INDEX idx_course_steps_deadline ON course_steps(deadline);
CREATE INDEX idx_course_steps_order ON course_steps(course_id, step_order);
```

#### journals 表
```sql
CREATE TABLE journals (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    student_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    step_id UUID NOT NULL REFERENCES course_steps(id) ON DELETE CASCADE,
    title TEXT,
    content TEXT NOT NULL,
    status journal_status NOT NULL DEFAULT 'draft',
    submission_number INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    submitted_at TIMESTAMPTZ,
    
    -- 确保学生角色
    CONSTRAINT check_student_role CHECK (
        student_id IN (SELECT id FROM users WHERE role = 'student')
    ),
    -- 确保提交时间逻辑
    CONSTRAINT check_submission_time CHECK (
        (status = 'draft' AND submitted_at IS NULL) OR
        (status = 'submitted' AND submitted_at IS NOT NULL)
    ),
    -- 确保提交编号合理
    CONSTRAINT check_submission_number CHECK (submission_number > 0),
    -- 确保学生、课程、步骤的组合唯一性（每个步骤的每次提交）
    UNIQUE(student_id, course_id, step_id, submission_number)
);

-- 创建索引
CREATE INDEX idx_journals_student_id ON journals(student_id);
CREATE INDEX idx_journals_course_id ON journals(course_id);
CREATE INDEX idx_journals_step_id ON journals(step_id);
CREATE INDEX idx_journals_status ON journals(status);
CREATE INDEX idx_journals_created_at ON journals(created_at);
CREATE INDEX idx_journals_submitted_at ON journals(submitted_at);
CREATE INDEX idx_journals_composite ON journals(student_id, course_id, step_id);
```

#### evaluations 表
```sql
CREATE TABLE evaluations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    journal_id UUID NOT NULL UNIQUE REFERENCES journals(id) ON DELETE CASCADE,
    ai_evaluation JSONB,
    ai_score INTEGER CHECK (ai_score >= 0 AND ai_score <= 100),
    teacher_evaluation TEXT,
    teacher_score INTEGER CHECK (teacher_score >= 0 AND teacher_score <= 100),
    final_score INTEGER CHECK (final_score >= 0 AND final_score <= 100),
    is_final BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- 确保最终评分逻辑
    CONSTRAINT check_final_evaluation CHECK (
        (is_final = FALSE) OR 
        (is_final = TRUE AND teacher_evaluation IS NOT NULL AND final_score IS NOT NULL)
    )
);

-- 创建索引
CREATE INDEX idx_evaluations_journal_id ON evaluations(journal_id);
CREATE INDEX idx_evaluations_is_final ON evaluations(is_final);
CREATE INDEX idx_evaluations_created_at ON evaluations(created_at);
```

## 2. 触发器和函数

### 2.1 更新时间戳触发器

```sql
-- 创建更新时间戳函数
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 为所有表添加更新时间戳触发器
CREATE TRIGGER update_classes_updated_at BEFORE UPDATE ON classes
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_courses_updated_at BEFORE UPDATE ON courses
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_course_steps_updated_at BEFORE UPDATE ON course_steps
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_journals_updated_at BEFORE UPDATE ON journals
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_evaluations_updated_at BEFORE UPDATE ON evaluations
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 2.2 日志提交触发器

```sql
-- 日志提交时自动设置提交时间
CREATE OR REPLACE FUNCTION set_journal_submitted_at()
RETURNS TRIGGER AS $$
BEGIN
    -- 如果状态从draft变为submitted，设置提交时间
    IF OLD.status = 'draft' AND NEW.status = 'submitted' THEN
        NEW.submitted_at = NOW();
    END IF;
    
    -- 如果状态从submitted变为draft，清除提交时间
    IF OLD.status = 'submitted' AND NEW.status = 'draft' THEN
        NEW.submitted_at = NULL;
    END IF;
    
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER journal_submission_trigger BEFORE UPDATE ON journals
    FOR EACH ROW EXECUTE FUNCTION set_journal_submitted_at();
```

## 3. Row Level Security (RLS) 策略

### 3.1 启用RLS

```sql
-- 为所有表启用RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE classes ENABLE ROW LEVEL SECURITY;
ALTER TABLE courses ENABLE ROW LEVEL SECURITY;
ALTER TABLE course_classes ENABLE ROW LEVEL SECURITY;
ALTER TABLE course_steps ENABLE ROW LEVEL SECURITY;
ALTER TABLE journals ENABLE ROW LEVEL SECURITY;
ALTER TABLE evaluations ENABLE ROW LEVEL SECURITY;
```

### 3.2 辅助函数

```sql
-- 获取当前用户角色
CREATE OR REPLACE FUNCTION get_current_user_role()
RETURNS user_role AS $$
BEGIN
    RETURN (SELECT role FROM users WHERE id = auth.uid());
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 获取当前用户班级ID
CREATE OR REPLACE FUNCTION get_current_user_class_id()
RETURNS UUID AS $$
BEGIN
    RETURN (SELECT class_id FROM users WHERE id = auth.uid());
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 检查用户是否是课程的老师
CREATE OR REPLACE FUNCTION is_course_teacher(course_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1 FROM courses 
        WHERE id = course_uuid AND teacher_id = auth.uid()
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 检查学生是否可以访问课程
CREATE OR REPLACE FUNCTION can_student_access_course(course_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1 FROM course_classes cc
        JOIN users u ON u.class_id = cc.class_id
        WHERE cc.course_id = course_uuid AND u.id = auth.uid()
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 3.3 users 表 RLS 策略

```sql
-- 管理员可以查看所有用户
CREATE POLICY "Admins can view all users" ON users
    FOR SELECT USING (get_current_user_role() = 'admin');

-- 用户可以查看自己的信息
CREATE POLICY "Users can view own profile" ON users
    FOR SELECT USING (auth.uid() = id);

-- 老师可以查看自己课程中的学生
CREATE POLICY "Teachers can view their students" ON users
    FOR SELECT USING (
        get_current_user_role() = 'teacher' AND
        role = 'student' AND
        class_id IN (
            SELECT cc.class_id FROM course_classes cc
            JOIN courses c ON c.id = cc.course_id
            WHERE c.teacher_id = auth.uid()
        )
    );

-- 管理员可以插入用户
CREATE POLICY "Admins can insert users" ON users
    FOR INSERT WITH CHECK (get_current_user_role() = 'admin');

-- 管理员可以更新用户
CREATE POLICY "Admins can update users" ON users
    FOR UPDATE USING (get_current_user_role() = 'admin');

-- 用户可以更新自己的部分信息
CREATE POLICY "Users can update own profile" ON users
    FOR UPDATE USING (auth.uid() = id)
    WITH CHECK (auth.uid() = id AND role = (SELECT role FROM users WHERE id = auth.uid()));
```

### 3.4 classes 表 RLS 策略

```sql
-- 管理员可以管理所有班级
CREATE POLICY "Admins can manage classes" ON classes
    FOR ALL USING (get_current_user_role() = 'admin');

-- 老师可以查看自己课程关联的班级
CREATE POLICY "Teachers can view related classes" ON classes
    FOR SELECT USING (
        get_current_user_role() = 'teacher' AND
        id IN (
            SELECT cc.class_id FROM course_classes cc
            JOIN courses c ON c.id = cc.course_id
            WHERE c.teacher_id = auth.uid()
        )
    );

-- 学生可以查看自己的班级
CREATE POLICY "Students can view own class" ON classes
    FOR SELECT USING (
        get_current_user_role() = 'student' AND
        id = get_current_user_class_id()
    );
```

### 3.5 courses 表 RLS 策略

```sql
-- 管理员可以查看所有课程
CREATE POLICY "Admins can view all courses" ON courses
    FOR SELECT USING (get_current_user_role() = 'admin');

-- 老师可以管理自己的课程
CREATE POLICY "Teachers can manage own courses" ON courses
    FOR ALL USING (teacher_id = auth.uid());

-- 学生可以查看自己班级的课程
CREATE POLICY "Students can view accessible courses" ON courses
    FOR SELECT USING (
        get_current_user_role() = 'student' AND
        can_student_access_course(id)
    );
```

### 3.6 course_classes 表 RLS 策略

```sql
-- 管理员可以管理所有课程-班级关联
CREATE POLICY "Admins can manage course classes" ON course_classes
    FOR ALL USING (get_current_user_role() = 'admin');

-- 老师可以管理自己课程的班级关联
CREATE POLICY "Teachers can manage own course classes" ON course_classes
    FOR ALL USING (is_course_teacher(course_id));

-- 学生可以查看自己相关的课程-班级关联
CREATE POLICY "Students can view related course classes" ON course_classes
    FOR SELECT USING (
        get_current_user_role() = 'student' AND
        class_id = get_current_user_class_id()
    );
```

### 3.7 course_steps 表 RLS 策略

```sql
-- 管理员可以查看所有课程步骤
CREATE POLICY "Admins can view all course steps" ON course_steps
    FOR SELECT USING (get_current_user_role() = 'admin');

-- 老师可以管理自己课程的步骤
CREATE POLICY "Teachers can manage own course steps" ON course_steps
    FOR ALL USING (is_course_teacher(course_id));

-- 学生可以查看可访问课程的步骤
CREATE POLICY "Students can view accessible course steps" ON course_steps
    FOR SELECT USING (
        get_current_user_role() = 'student' AND
        can_student_access_course(course_id)
    );
```

### 3.8 journals 表 RLS 策略

```sql
-- 学生可以管理自己的日志
CREATE POLICY "Students can manage own journals" ON journals
    FOR ALL USING (student_id = auth.uid());

-- 老师可以查看自己课程的学生日志
CREATE POLICY "Teachers can view course journals" ON journals
    FOR SELECT USING (is_course_teacher(course_id));

-- 管理员可以查看所有日志
CREATE POLICY "Admins can view all journals" ON journals
    FOR SELECT USING (get_current_user_role() = 'admin');
```

### 3.9 evaluations 表 RLS 策略

```sql
-- 老师可以管理自己课程的评估
CREATE POLICY "Teachers can manage course evaluations" ON evaluations
    FOR ALL USING (
        journal_id IN (
            SELECT j.id FROM journals j
            WHERE is_course_teacher(j.course_id)
        )
    );

-- 学生可以查看自己日志的最终评估（不包括AI评估）
CREATE POLICY "Students can view own final evaluations" ON evaluations
    FOR SELECT USING (
        get_current_user_role() = 'student' AND
        is_final = true AND
        journal_id IN (
            SELECT id FROM journals WHERE student_id = auth.uid()
        )
    );

-- 管理员可以查看所有评估
CREATE POLICY "Admins can view all evaluations" ON evaluations
    FOR SELECT USING (get_current_user_role() = 'admin');
```