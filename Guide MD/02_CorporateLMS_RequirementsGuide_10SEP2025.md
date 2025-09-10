# Corporate LMS Requirements & Migration Guide
**Date:** September 10, 2025  
**Status:** Requirements Documentation Complete  
**Target:** School LMS → Corporate Learning Management System Migration

---

## 📋 **PROJECT OVERVIEW**

### **Current State**
- **Source Application:** School Management Dashboard (Next.js 14 + Prisma + Clerk)
- **Target Context:** Corporate Learning Management System
- **Database Migration:** Docker PostgreSQL → Supabase (shared with SkillMatrix)
- **Architecture Retention:** Keep UI/UX patterns, technology stack, and development workflow

### **Business Context Transformation**
**FROM:** School-focused entities (Students, Teachers, Classes, Grades)  
**TO:** Corporate learning entities (Employees, Trainers, Courses, Batches)

---

## 🔄 **REVISED TABLE TRANSFORMATION MAP**

| Current School Entity | Target Corporate Entity | Strategy | Implementation |
|----------------------|------------------------|----------|----------------|
| **Admin** | **allowed_user** | **REUSE** from SkillMatrix | Existing table structure |
| **Student** | **employee** | **REUSE** from SkillMatrix | Existing table structure |
| **Teacher** | **trainer_list** | **NEW TABLE** | Custom trainer assignment system |
| **Class** | **course** | **NEW TABLE** | Course catalog management |
| **Parent** | ~~Dropped~~ | **REMOVE** | Not needed in corporate context |
| **Grade** | **employee** | **REUSE** from SkillMatrix | Link to existing employee levels |
| **Lesson + Event** | **batch** | **MERGE & NEW** | Training session management |
| **Assignment** | **OJT** (On Job Training) | **NEW TABLE** | Workplace training assignments |
| **Exam** | **attendance** | **NEW TABLE** | Training attendance tracking |
| **Result** | **OJT_list** | **NEW TABLE** | OJT catalog and management |

---

## 🏗️ **DATABASE SCHEMA DESIGN**

### **Schema Organization**
```sql
-- SUPABASE MULTI-SCHEMA SETUP
├── SkillMatrix (existing schema)
│   ├── allowed_user        -- Admin functionality (REUSE)
│   ├── employee           -- Employee data (REUSE)
│   └── [other existing tables]
│
└── LMS (new schema) 
    ├── trainer_list       -- Trainer assignments
    ├── course            -- Course catalog
    ├── batch             -- Training sessions  
    ├── batch_reminder    -- Reminder scheduling
    ├── attendance        -- Training attendance
    ├── OJT              -- On-job training assignments
    └── OJT_list         -- OJT catalog
```

### **Core Tables Structure**

#### **trainer_list** (replaces Teacher)
```sql
CREATE TABLE LMS.trainer_list (
  id SERIAL PRIMARY KEY,
  empId VARCHAR REFERENCES SkillMatrix.employee(id),
  courseId VARCHAR REFERENCES LMS.course(id),
  courseName VARCHAR(255),
  effectiveDate DATE,
  currentActiveStatus BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### **course** (replaces Class)
```sql
CREATE TABLE LMS.course (
  id VARCHAR PRIMARY KEY,
  courseName VARCHAR(255) NOT NULL,
  courseDescription TEXT,
  duration_hours INTEGER,
  max_participants INTEGER,
  course_type VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### **batch** (merges Lesson + Event)
```sql
CREATE TABLE LMS.batch (
  id SERIAL PRIMARY KEY,
  batchId VARCHAR(50) UNIQUE NOT NULL,
  courseId VARCHAR REFERENCES LMS.course(id),
  trainedDate DATE,
  empId VARCHAR REFERENCES SkillMatrix.employee(id),
  trainer_empId VARCHAR REFERENCES SkillMatrix.employee(id),
  status VARCHAR(50) DEFAULT 'scheduled',
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### **batch_reminder** (extension for batch scheduling)
```sql
CREATE TABLE LMS.batch_reminder (
  id SERIAL PRIMARY KEY,
  batchId VARCHAR REFERENCES LMS.batch(batchId),
  reminderID VARCHAR(50) UNIQUE,
  reminderDate DATE,
  reminderMessage VARCHAR(500),
  reminderSentDate DATE,
  reminderSentStatus BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### **attendance** (replaces Exam for tracking)
```sql
CREATE TABLE LMS.attendance (
  id SERIAL PRIMARY KEY,
  courseId VARCHAR REFERENCES LMS.course(id),
  batchNum VARCHAR,
  batchId VARCHAR REFERENCES LMS.batch(batchId),
  empId VARCHAR REFERENCES SkillMatrix.employee(id),
  scheduledDate DATE,
  empConfirmationDate DATE,
  empConfirmationStatus VARCHAR(50),
  trainedDate DATE,
  trainedStatus VARCHAR(100),
  nominatedBy VARCHAR REFERENCES SkillMatrix.employee(id),
  nominationDate DATE,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### **OJT** (replaces Assignment)
```sql
CREATE TABLE LMS.OJT (
  Id SERIAL PRIMARY KEY,
  OJT_Code VARCHAR(50) UNIQUE NOT NULL,
  title VARCHAR(255) NOT NULL,
  startDate DATE,
  endDate DATE,
  finalResult VARCHAR(100),
  empId VARCHAR REFERENCES SkillMatrix.employee(id),
  supervisor_empId VARCHAR REFERENCES SkillMatrix.employee(id),
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### **OJT_list** (OJT catalog management)
```sql
CREATE TABLE LMS.OJT_list (
  UUID UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  OJT_code VARCHAR(50) UNIQUE NOT NULL,
  OJT_name VARCHAR(255) NOT NULL,
  section_code VARCHAR(50),
  sectionName VARCHAR(255), -- Reference from organization table
  empId VARCHAR REFERENCES SkillMatrix.employee(id),
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 🎨 **UI/UX ADAPTATION STRATEGY**

### **Design Approach**
- **Color & Branding:** No changes for now - keep existing school theme
- **Terminology:** Update based on database context changes
- **Navigation:** Similar structure - copy and create new pages for safety
- **Page Strategy:** Copy original pages → modify copies → keep originals as reference

### **Page Copy Strategy**
```
src/app/(dashboard)/
├── admin/           (original - keep as backup)
├── admin-corporate/ (new - corporate version)
├── teacher/         (original - keep as backup)  
├── trainer/         (new - corporate version)
├── student/         (original - keep as backup)
├── employee/        (new - corporate version)
└── list/
    ├── classes/     (original - keep as backup)
    ├── courses/     (new - corporate version)
    └── [other entities]
```

### **Terminology Mapping**
| School Term | Corporate Term | UI Impact |
|------------|----------------|-----------|
| Students | Employees | All labels, headings, forms |
| Teachers | Trainers | Role references, assignments |
| Classes | Courses | Navigation, listings |
| Grades | Levels/Positions | Employee hierarchy |
| Lessons | Training Sessions | Calendar, scheduling |
| Assignments | OJT Tasks | Task management |
| Exams | Assessments | Evaluation forms |
| Parents | ~~Removed~~ | Remove all parent-related UI |

---

## 🔗 **ENTITY RELATIONSHIPS**

### **Cross-Schema Relationships**
```
SkillMatrix.employee (1) ←→ (M) LMS.trainer_list
SkillMatrix.employee (1) ←→ (M) LMS.batch  
SkillMatrix.employee (1) ←→ (M) LMS.attendance
SkillMatrix.employee (1) ←→ (M) LMS.OJT
SkillMatrix.allowed_user (1) ←→ (M) LMS.course
```

### **LMS Schema Internal Relationships**
```
LMS.course (1) ←→ (M) LMS.trainer_list
LMS.course (1) ←→ (M) LMS.batch
LMS.batch (1) ←→ (M) LMS.batch_reminder
LMS.batch (1) ←→ (M) LMS.attendance
LMS.OJT_list (1) ←→ (M) LMS.OJT
```

---

## 🔧 **INTEGRATION REQUIREMENTS**

### **Supabase Multi-Schema Setup**
- **New Schema Name:** `LMS`
- **Clerk Integration:** Verify table compatibility with existing Clerk auth
- **Cross-Schema Queries:** Enable proper foreign key relationships
- **Row Level Security:** TBD based on security requirements

### **External Systems Integration**
- **Current Status:** No linkage planned
- **Future Consideration:** HR systems, performance management
- **Authentication:** Continue using Clerk for all user management

---

## 📋 **USER ROLE TRANSFORMATIONS**

### **Employee Portal** (was Student Portal)
- **Dashboard:** Personal learning path, upcoming trainings
- **Course Catalog:** Browse available courses by department/skill
- **My Training:** Attendance history, completion status
- **OJT Assignments:** Active workplace training tasks
- **Reminders:** Training schedule notifications

### **Trainer Portal** (was Teacher Portal)
- **Course Assignment:** Manage assigned courses via `trainer_list`
- **Batch Management:** Schedule and conduct training sessions
- **Attendance Tracking:** Mark attendance and completion status
- **OJT Supervision:** Oversee workplace training assignments
- **Resource Management:** Upload training materials

### **Admin Portal** (was Admin Portal)
- **Course Catalog:** Create and manage course offerings
- **Trainer Assignment:** Assign trainers to courses
- **Batch Scheduling:** Plan training sessions and reminders
- **Attendance Reports:** Track completion and compliance
- **OJT Program:** Manage on-job training catalog and assignments

---

## 🎯 **DEVELOPMENT APPROACH**

### **Phase 1: Database Setup**
1. **Create LMS schema** in existing Supabase instance
2. **Implement table structure** with proper relationships
3. **Verify Clerk compatibility** with new schema
4. **Test cross-schema queries** between SkillMatrix and LMS

### **Phase 2: Page Duplication**
1. **Copy existing dashboard pages** to new corporate versions
2. **Maintain original pages** as working reference
3. **Update navigation** to point to new corporate pages
4. **Implement terminology changes** in copied pages

### **Phase 3: Database Integration**
1. **Update Prisma schema** to include LMS tables
2. **Modify server actions** for new table structure
3. **Update form validation schemas** for new entities
4. **Test CRUD operations** across all new tables

### **Phase 4: Feature Implementation**
1. **Implement reminder system** for batch scheduling
2. **Build attendance tracking** with confirmation workflow
3. **Create OJT management** system with catalog
4. **Add reporting features** for training metrics

---

## ⚠️ **MIGRATION CONSIDERATIONS**

### **Database Migration**
- **Schema Isolation:** LMS tables separate from existing SkillMatrix
- **Foreign Key Constraints:** Proper relationships across schemas
- **Data Migration:** No existing data to migrate (fresh start)
- **Backup Strategy:** Keep original Docker setup until migration confirmed

### **Code Migration**
- **Prisma Schema Updates:** Add LMS tables to existing schema file
- **Server Actions:** Create new actions for LMS-specific operations
- **Form Components:** Duplicate and modify for corporate context
- **Authentication:** No changes required - continue using Clerk

### **Testing Strategy**
- **Database Connectivity:** Verify Supabase connection from Next.js
- **CRUD Operations:** Test all create/read/update/delete functions
- **Cross-Schema Queries:** Ensure proper foreign key relationships
- **User Authentication:** Verify Clerk integration with new schema
- **Page Navigation:** Test all corporate dashboard pages

---

## 📊 **SUCCESS METRICS**

### **Technical Metrics**
- [ ] All LMS tables created and accessible
- [ ] Cross-schema relationships working
- [ ] Clerk authentication integrated
- [ ] All CRUD operations functional
- [ ] Corporate pages fully navigable

### **Functional Metrics**
- [ ] Employee portal displays personal training data
- [ ] Trainer portal manages courses and batches
- [ ] Admin portal controls full system
- [ ] Reminder system sends notifications
- [ ] Attendance tracking captures participation
- [ ] OJT system manages workplace training

---

## 🚀 **NEXT STEPS**

### **Immediate Actions**
1. **Provide Supabase connection details** for LMS schema creation
2. **Confirm Clerk integration approach** for new schema
3. **Review and approve database table structures**
4. **Start with Phase 1: Database Setup**

### **Documentation Updates**
- Update CLAUDE.md with new development commands
- Document corporate terminology mapping
- Create troubleshooting guide for cross-schema queries
- Update README with corporate LMS context

---

**🎯 Ready to proceed with corporate LMS migration using this comprehensive requirements guide.**

**Next Steps:** Begin Phase 1 - Database Setup with Supabase LMS schema creation.