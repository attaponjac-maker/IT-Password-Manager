# 🚀 Supabase Setup Guide - IT Password Manager

## ขั้นตอนที่ 1: สร้าง Supabase Project

### 1.1 สมัครบัญชี Supabase
1. ไปที่ https://supabase.com
2. คลิก **"Start your project"**
3. Sign up ด้วย GitHub account (แนะนำ) หรือ Email
4. ยืนยัน Email

### 1.2 สร้าง Project ใหม่
1. คลิก **"New Project"**
2. กรอกข้อมูล:
   - **Name**: `it-password-manager`
   - **Database Password**: สร้างรหัสผ่านที่แข็งแรง (เก็บไว้ดี!)
   - **Region**: `Southeast Asia (Singapore)` (ใกล้ที่สุด)
   - **Pricing Plan**: `Free` (เพียงพอสำหรับเริ่มต้น)
3. คลิก **"Create new project"**
4. รอ 1-2 นาที ให้ระบบสร้าง Database

---

## ขั้นตอนที่ 2: สร้าง Database Tables

### 2.1 เปิด SQL Editor
1. ไปที่ Sidebar → คลิก **"SQL Editor"**
2. คลิก **"New query"**

### 2.2 รัน SQL Script นี้

คัดลอก SQL ด้านล่างนี้ทั้งหมด แล้ววางใน SQL Editor:

```sql
-- ===================================
-- IT Password Manager Database Schema
-- ===================================

-- 1. Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 2. Create Teams/Workspaces table
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID REFERENCES auth.users(id)
);

-- 3. Create Team Members table (User-Team relationship)
CREATE TABLE team_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    role TEXT NOT NULL CHECK (role IN ('admin', 'user', 'viewer')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(team_id, user_id)
);

-- 4. Create Branches table
CREATE TABLE branches (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(team_id, name)
);

-- 5. Create Credentials table (Main data)
CREATE TABLE credentials (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    branch_id UUID REFERENCES branches(id) ON DELETE SET NULL,
    category TEXT NOT NULL CHECK (category IN ('google-workspace', 'website', 'cctv', 'network', 'server', 'etc')),
    title TEXT NOT NULL,
    url TEXT,
    username TEXT,
    password_encrypted TEXT, -- AES-256 encrypted
    notes TEXT,
    expiry_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID REFERENCES auth.users(id),
    updated_by UUID REFERENCES auth.users(id)
);

-- 6. Create Activity Log table
CREATE TABLE activity_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    action TEXT NOT NULL, -- 'create', 'update', 'delete', 'view'
    resource_type TEXT NOT NULL, -- 'credential', 'branch', 'team'
    resource_id UUID,
    details JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 7. Create User Profiles table (Extended user info)
CREATE TABLE user_profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    full_name TEXT,
    avatar_url TEXT,
    department TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ===================================
-- Indexes for Performance
-- ===================================

CREATE INDEX idx_credentials_team_id ON credentials(team_id);
CREATE INDEX idx_credentials_branch_id ON credentials(branch_id);
CREATE INDEX idx_credentials_category ON credentials(category);
CREATE INDEX idx_credentials_expiry_date ON credentials(expiry_date);
CREATE INDEX idx_team_members_team_id ON team_members(team_id);
CREATE INDEX idx_team_members_user_id ON team_members(user_id);
CREATE INDEX idx_activity_logs_team_id ON activity_logs(team_id);
CREATE INDEX idx_activity_logs_user_id ON activity_logs(user_id);
CREATE INDEX idx_branches_team_id ON branches(team_id);

-- ===================================
-- Row Level Security (RLS) Policies
-- ===================================

-- Enable RLS on all tables
ALTER TABLE teams ENABLE ROW LEVEL SECURITY;
ALTER TABLE team_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE branches ENABLE ROW LEVEL SECURITY;
ALTER TABLE credentials ENABLE ROW LEVEL SECURITY;
ALTER TABLE activity_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;

-- Teams: Users can only see teams they're members of
CREATE POLICY "Users can view their teams"
    ON teams FOR SELECT
    USING (
        id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid()
        )
    );

CREATE POLICY "Users can create teams"
    ON teams FOR INSERT
    WITH CHECK (auth.uid() = created_by);

CREATE POLICY "Team admins can update teams"
    ON teams FOR UPDATE
    USING (
        id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid() AND role = 'admin'
        )
    );

-- Team Members: Users can see members of their teams
CREATE POLICY "Users can view team members"
    ON team_members FOR SELECT
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid()
        )
    );

CREATE POLICY "Team admins can manage members"
    ON team_members FOR ALL
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid() AND role = 'admin'
        )
    );

-- Branches: Team members can view, admins can manage
CREATE POLICY "Team members can view branches"
    ON branches FOR SELECT
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid()
        )
    );

CREATE POLICY "Team admins can manage branches"
    ON branches FOR ALL
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid() AND role = 'admin'
        )
    );

-- Credentials: Team members can view, users+ can create/update
CREATE POLICY "Team members can view credentials"
    ON credentials FOR SELECT
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid()
        )
    );

CREATE POLICY "Team users can create credentials"
    ON credentials FOR INSERT
    WITH CHECK (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid() AND role IN ('admin', 'user')
        )
    );

CREATE POLICY "Team users can update credentials"
    ON credentials FOR UPDATE
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid() AND role IN ('admin', 'user')
        )
    );

CREATE POLICY "Team admins can delete credentials"
    ON credentials FOR DELETE
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid() AND role = 'admin'
        )
    );

-- Activity Logs: Team members can view
CREATE POLICY "Team members can view activity logs"
    ON activity_logs FOR SELECT
    USING (
        team_id IN (
            SELECT team_id FROM team_members 
            WHERE user_id = auth.uid()
        )
    );

CREATE POLICY "Users can create activity logs"
    ON activity_logs FOR INSERT
    WITH CHECK (auth.uid() = user_id);

-- User Profiles: Users can manage their own profile
CREATE POLICY "Users can view all profiles"
    ON user_profiles FOR SELECT
    USING (true);

CREATE POLICY "Users can update own profile"
    ON user_profiles FOR ALL
    USING (auth.uid() = id);

-- ===================================
-- Functions & Triggers
-- ===================================

-- Function to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Triggers for updated_at
CREATE TRIGGER update_teams_updated_at
    BEFORE UPDATE ON teams
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_branches_updated_at
    BEFORE UPDATE ON branches
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_credentials_updated_at
    BEFORE UPDATE ON credentials
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_user_profiles_updated_at
    BEFORE UPDATE ON user_profiles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Function to automatically create user profile on signup
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_profiles (id, full_name)
    VALUES (NEW.id, NEW.raw_user_meta_data->>'full_name');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Trigger to create profile on user signup
CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW
    EXECUTE FUNCTION handle_new_user();

-- ===================================
-- Insert Default Data (Optional)
-- ===================================

-- สร้าง Team ตัวอย่าง (จะทำหลังจาก User สร้างแล้ว)
-- INSERT INTO teams (name, description) 
-- VALUES ('Rubber Business IT', 'IT Department - Southern Thailand');

```

3. คลิก **"Run"** เพื่อสร้าง Tables ทั้งหมด
4. ถ้าสำเร็จ จะเห็นข้อความ "Success. No rows returned"

---

## ขั้นตอนที่ 3: ตั้งค่า Authentication

### 3.1 เปิด Email Authentication
1. ไปที่ **Authentication** → **Providers**
2. เปิด **Email** (เปิดอยู่แล้วโดย default)
3. (Optional) เปิด **Google OAuth** ถ้าต้องการ Login ด้วย Google

### 3.2 ตั้งค่า Email Templates (Optional)
1. ไปที่ **Authentication** → **Email Templates**
2. ปรับแต่ง Email สำหรับ:
   - Confirm signup
   - Reset password
   - Magic Link

---

## ขั้นตอนที่ 4: ดึง API Keys

### 4.1 หา Project URL และ API Keys
1. ไปที่ **Settings** → **API**
2. คัดลอกข้อมูลเหล่านี้:

```
Project URL: https://xxxxxxxxxx.supabase.co
anon/public key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

3. **เก็บข้อมูลนี้ไว้ดี!** จะใช้ใน Code

---

## ขั้นตอนที่ 5: ตั้งค่า Storage (สำหรับ Backup)

### 5.1 สร้าง Storage Bucket
1. ไปที่ **Storage**
2. คลิก **"New bucket"**
3. กรอก:
   - **Name**: `backups`
   - **Public**: ปิด (Private)
4. คลิก **"Create bucket"**

### 5.2 ตั้งค่า Policies สำหรับ Bucket
1. คลิกที่ bucket `backups`
2. ไปที่ **Policies**
3. เพิ่ม Policy:

```sql
-- Allow authenticated users to upload their own backups
CREATE POLICY "Users can upload backups"
ON storage.objects FOR INSERT
WITH CHECK (
    bucket_id = 'backups' 
    AND auth.uid()::text = (storage.foldername(name))[1]
);

-- Allow users to read their own backups
CREATE POLICY "Users can read own backups"
ON storage.objects FOR SELECT
USING (
    bucket_id = 'backups' 
    AND auth.uid()::text = (storage.foldername(name))[1]
);
```

---

## ขั้นตอนที่ 6: Test การเชื่อมต่อ

### 6.1 ทดสอบใน SQL Editor
รัน Query นี้เพื่อดูว่า Tables ถูกสร้างแล้ว:

```sql
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public';
```

คุณควรเห็น Tables:
- teams
- team_members
- branches
- credentials
- activity_logs
- user_profiles

---

## 📝 สรุปข้อมูลที่ได้

หลัง Setup เสร็จ คุณจะได้:

✅ **Project URL**: `https://xxxxxxxxxx.supabase.co`
✅ **API Key (anon)**: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`
✅ **Database Password**: (ที่คุณตั้งไว้)

**เก็บข้อมูลนี้ไว้ใช้ในขั้นตอนถัดไป!**

---

## 🔐 ตรวจสอบ Security

### Row Level Security (RLS) ใช้งานแล้ว:
- ✅ User แต่ละคนเห็นเฉพาะ Team ของตัวเอง
- ✅ Viewer อ่านได้อย่างเดียว
- ✅ User สร้าง/แก้ไขได้
- ✅ Admin ทำได้ทุกอย่าง รวมถึงลบ

### ทดสอบ Policies:
1. สร้าง User ทดสอบ 2 คน
2. ให้แต่ละคนเข้า Team คนละ Team
3. ตรวจสอบว่าเห็นข้อมูลคนละชุด

---

## 🎯 ขั้นตอนถัดไป

1. ✅ Setup Supabase เสร็จแล้ว
2. ⏭️ ต่อไป: ใช้ Code ที่ผมเขียนให้
3. 🔧 Config: ใส่ Project URL + API Key
4. 🚀 Deploy: Host บน GitHub Pages หรือ Vercel

---

## 🆘 Troubleshooting

### ปัญหา: Tables ไม่ถูกสร้าง
**แก้ไข**: ตรวจสอบว่ารัน SQL Script ครบทั้งหมดหรือไม่

### ปัญหา: RLS ไม่ทำงาน
**แก้ไข**: ตรวจสอบว่า `ALTER TABLE ... ENABLE ROW LEVEL SECURITY;` รันแล้ว

### ปัญหา: User ไม่สามารถเข้าถึงข้อมูล
**แก้ไข**: ตรวจสอบว่า User ถูกเพิ่มเข้า `team_members` table แล้ว

---

## 💡 Tips

- 📊 **Dashboard**: ดู Database ได้ที่ Table Editor
- 🔍 **Query**: ใช้ SQL Editor สำหรับ Query ข้อมูล
- 📈 **Monitor**: ดู Usage ที่ Settings → Usage
- 💾 **Backup**: Supabase ทำ Auto backup ให้ (Daily)

---

**เสร็จแล้ว!** 🎉

ต่อไปจะเป็น Code ใหม่ที่ใช้กับ Supabase ที่เพิ่ง Setup นี้
