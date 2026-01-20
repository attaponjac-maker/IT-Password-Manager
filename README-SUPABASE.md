# 🚀 IT Password Manager - Supabase Edition

## 📦 ไฟล์ที่ได้

✅ **SUPABASE_SETUP_GUIDE.md** - คู่มือ Setup Supabase ทีละขั้นตอน
✅ **password-manager-supabase.html** - Code เวอร์ชัน Multi-User (ยังไม่สมบูรณ์)

---

## 🎯 สถานะปัจจุบัน

### ✅ Features ที่ทำเสร็จแล้ว:

1. **Authentication System**
   - ✅ Login / Register
   - ✅ Email verification
   - ✅ Logout
   
2. **Team/Workspace Management**
   - ✅ Create new team
   - ✅ List user's teams
   - ✅ Select team to work with
   - ✅ Role-based display (admin/user/viewer)

3. **Dashboard Stats**
   - ✅ Total credentials
   - ✅ Total branches
   - ✅ Expiring items count
   - ✅ Team members count

4. **Database Schema**
   - ✅ Teams table
   - ✅ Team members table
   - ✅ Branches table
   - ✅ Credentials table
   - ✅ Activity logs table
   - ✅ User profiles table
   - ✅ Row Level Security (RLS)

### 🚧 Features ที่ต้องทำต่อ:

1. **Credentials Management**
   - ⏳ Add/Edit/Delete credentials
   - ⏳ List credentials with filters
   - ⏳ Search functionality
   - ⏳ Category/Branch filters
   - ⏳ AES-256 encryption for passwords

2. **Branch Management**
   - ⏳ Add/Edit/Delete branches
   - ⏳ List branches

3. **UI Components**
   - ⏳ Full dashboard view
   - ⏳ Credential cards grid
   - ⏳ Modals for CRUD operations
   - ⏳ Activity log viewer

4. **Advanced Features**
   - ⏳ Real-time sync (Supabase Realtime)
   - ⏳ Activity logging
   - ⏳ Team member management (invite/remove)
   - ⏳ User profile settings
   - ⏳ Export/Import data

5. **Security**
   - ⏳ Client-side password encryption (AES-256)
   - ⏳ Secure password sharing within team
   - ⏳ Audit trail

---

## 🛠️ วิธีใช้งาน (ขั้นตอนที่ 1)

### 1. Setup Supabase
อ่านและทำตาม **SUPABASE_SETUP_GUIDE.md**

คุณจะได้:
- Project URL (เช่น `https://xxxxx.supabase.co`)
- Anon Key (เช่น `eyJhbGciOiJIUzI1...`)

### 2. Config ไฟล์ HTML
เปิด **password-manager-supabase.html** แก้บรรทัดที่ ~240:

```javascript
const SUPABASE_URL = 'YOUR_SUPABASE_URL'; 
const SUPABASE_ANON_KEY = 'YOUR_SUPABASE_ANON_KEY';
```

เปลี่ยนเป็น:

```javascript
const SUPABASE_URL = 'https://xxxxx.supabase.co'; 
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';
```

### 3. ทดสอบ
1. เปิดไฟล์ในเบราว์เซอร์
2. สมัครสมาชิกด้วย Email ของคุณ
3. ตรวจสอบ Email เพื่อยืนยันบัญชี
4. Login เข้าสู่ระบบ
5. สร้าง Team แรก
6. เข้าสู่ Team

คุณจะเห็น Dashboard พร้อมสถิติเบื้องต้น! 🎉

---

## 📋 ขั้นตอนถัดไป (เลือกได้)

### Option A: ให้ผมทำต่อเลย (แนะนำ) ⭐

ผมจะเพิ่ม Features ทั้งหมดที่เหลือให้:
- ✅ Full CRUD for credentials
- ✅ Dashboard แบบสมบูรณ์
- ✅ Activity logging
- ✅ Real-time sync
- ✅ Team member management
- ✅ Export/Import

**บอกว่า "ทำต่อเลย" หรือ "เพิ่ม Full Features"**

---

### Option B: ใช้แบบนี้ก่อน แล้วค่อยพัฒนาเอง

ใช้ Code นี้เป็น Foundation แล้วพัฒนาเพิ่มเติมเอง:

**ตัวอย่าง Features ที่สามารถเพิ่มได้:**

#### เพิ่มฟังก์ชัน CRUD Credentials:

```javascript
// Create Credential
async function createCredential(data) {
    const encryptedPassword = CryptoJS.AES.encrypt(
        data.password, 
        'encryption-key'
    ).toString();

    const { data: credential, error } = await supabase
        .from('credentials')
        .insert({
            team_id: currentTeam.id,
            ...data,
            password_encrypted: encryptedPassword,
            created_by: currentUser.id
        })
        .select()
        .single();

    if (!error) {
        await logActivity('create', 'credential', credential.id);
    }
    return { credential, error };
}

// List Credentials
async function loadCredentials() {
    const { data, error } = await supabase
        .from('credentials')
        .select(`
            *,
            branches (name)
        `)
        .eq('team_id', currentTeam.id)
        .order('updated_at', { ascending: false });

    return { data, error };
}
```

#### เพิ่ม Real-time Sync:

```javascript
// Subscribe to changes
supabase
    .channel('credentials-changes')
    .on('postgres_changes', 
        { 
            event: '*', 
            schema: 'public', 
            table: 'credentials',
            filter: `team_id=eq.${currentTeam.id}`
        }, 
        (payload) => {
            console.log('Change received!', payload);
            loadCredentials(); // Refresh
        }
    )
    .subscribe();
```

---

## 🔐 Security Best Practices

### 1. Password Encryption
```javascript
// Encrypt before save
const encrypted = CryptoJS.AES.encrypt(password, encryptionKey).toString();

// Decrypt when read
const decrypted = CryptoJS.AES.decrypt(encrypted, encryptionKey).toString(CryptoJS.enc.Utf8);
```

### 2. Use Master Key per Team
- แต่ละ Team มี Encryption Key เป็นของตัวเอง
- เก็บ Key ใน localStorage (encrypted with user password)
- ไม่ส่ง Plain password ขึ้น Supabase

### 3. Enable MFA (Optional)
ใน Supabase Dashboard:
- Authentication → Providers → Phone → Enable
- Users can enable 2FA in settings

---

## 📊 Database Structure

```
teams
├── id
├── name
├── description
└── created_by

team_members
├── team_id
├── user_id
└── role (admin/user/viewer)

credentials
├── id
├── team_id
├── branch_id
├── category
├── title
├── url
├── username
├── password_encrypted ← AES-256
├── notes
└── expiry_date

branches
├── id
├── team_id
└── name

activity_logs
├── id
├── team_id
├── user_id
├── action
├── resource_type
└── resource_id
```

---

## 🚀 Deployment Options

### 1. GitHub Pages (ฟรี)
```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/attaponjac-maker/Information.git
git push -u origin main
```

ไปที่ Settings → Pages → เลือก `main` branch → Save

### 2. Vercel (ฟรี + Fast)
1. ไปที่ https://vercel.com
2. Import repository
3. Deploy!

### 3. Netlify (ฟรี)
1. Drag & drop HTML file
2. Done!

---

## 💰 Supabase Free Tier Limits

- ✅ 500 MB Database
- ✅ 1 GB File storage
- ✅ 50,000 Monthly Active Users
- ✅ 2 GB Bandwidth
- ✅ 50 GB Data transfer
- ✅ Unlimited API requests

**เพียงพอสำหรับ:**
- ทีม 10-50 คน
- เก็บข้อมูล 5,000-10,000 credentials
- ใช้งานปกติ

---

## 🆘 Troubleshooting

### ปัญหา: "Invalid API key"
**แก้:** ตรวจสอบว่า `SUPABASE_ANON_KEY` ถูกต้อง

### ปัญหา: "Row Level Security policy violation"
**แก้:** ตรวจสอบว่า User ถูก add เข้า `team_members`

### ปัญหา: "Not receiving signup email"
**แก้:** ตรวจสอบ Spam folder หรือใช้ Gmail

### ปัญหา: Data ไม่ Sync
**แก้:** Refresh หน้าเว็บ หรือตรวจสอบ Network connection

---

## 📞 Support

มีปัญหาหรือต้องการความช่วยเหลือ?

1. ตรวจสอบ Console (F12) ดู Error messages
2. ตรวจสอบ Supabase Dashboard → Logs
3. อ่าน Supabase Docs: https://supabase.com/docs

---

## 🎯 ต้องการให้ทำอะไรต่อ?

**เลือกได้เลย:**

A. **ให้ผมทำ Full Features ต่อ** (Recommended)
   - จะได้ระบบที่สมบูรณ์
   - พร้อมใช้งานจริง
   - มี UI ครบทุก Features

B. **ใช้แบบนี้ก่อน** 
   - เป็น Foundation ดี
   - พัฒนาต่อเองได้
   - เรียนรู้ Supabase ไปด้วย

C. **ต้องการ Feature เฉพาะบางอย่าง**
   - บอกมาได้เลยว่าต้องการอะไร
   - ผมจะเขียนให้เฉพาะส่วนนั้น

---

## 📚 Resources

- [Supabase Documentation](https://supabase.com/docs)
- [Supabase JavaScript Client](https://supabase.com/docs/reference/javascript)
- [Row Level Security Guide](https://supabase.com/docs/guides/auth/row-level-security)
- [Realtime Guide](https://supabase.com/docs/guides/realtime)

---

**Ready to continue?** บอกผมได้เลยครับ! 🚀
