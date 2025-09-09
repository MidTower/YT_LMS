# Database Setup Component

## Function Explanation

The Database Setup component manages all aspects of Supabase PostgreSQL database integration, including schema management, RPC functions, authentication, and data access policies. It provides the backend foundation for the entire Skill Matrix application.

**Primary Functions:**
- PostgreSQL database schema management (`skillmatrix` schema)
- Custom RPC (Remote Procedure Call) functions for complex operations
- Row-Level Security (RLS) policies for data access control
- Custom authentication system with session management
- Audit trail implementation for data changes
- Database connection and query optimization

## Architecture & Logic

### Database Schema Architecture
```
public schema (auth, sessions, RPC functions)
    ↓
"SkillMatrix" schema (core application data - CASE SENSITIVE)
    ↓
Schema Permissions (USAGE + SELECT grants)
    ↓
Application Services (JavaScript interface)
```

**⚠️ CRITICAL: Schema Case Sensitivity**
- PostgreSQL schema name: `"SkillMatrix"` (quoted, case-sensitive)
- JavaScript reference: `{ db: { schema: 'SkillMatrix' } }`
- SQL queries: `"SkillMatrix"."table_name"` (always quoted)

### Core Tables Structure

#### 1. **Authentication Tables (Public Schema)**
- `public.allowed_users` - User whitelist and roles
- `public.user_sessions` - Session management and tokens

#### 2. **Core Data Tables ("SkillMatrix" Schema - Case Sensitive)**
- `"SkillMatrix"."skill_base"` - Hierarchical skill definitions with NextGen status
- `"SkillMatrix"."skill_detail"` - Detailed skill descriptions  
- `"SkillMatrix"."skill_assignments"` - User-skill proficiency mappings
- `"SkillMatrix"."employees"` - Employee information
- `"SkillMatrix"."skill_levels"` - Proficiency level definitions
- `"SkillMatrix"."emp_skill_assessments"` - Assessment records

#### 3. **Database Views ("SkillMatrix" Schema)**
- `"SkillMatrix"."v_skill_hierarchy_ancestors"` - Pre-computed hierarchy with NextGen status
- Provides optimized hierarchy data with all levels (hlvl0-hlvl5)
- Eliminates client-side hierarchy processing

#### 3. **Reference Tables**
- `skillmatrix.skill_standard` - Skill standards and benchmarks

### RPC Function Architecture

#### **Core RPC Patterns**
```sql
-- Data Retrieval Pattern
FUNCTION get_[entity]_[scope]() RETURNS [return_type]

-- Data Modification Pattern  
FUNCTION update_[entity]_[scope](params) RETURNS [status_type]

-- Authentication Pattern
FUNCTION [auth_action]_user(params) RETURNS [auth_result]
```

#### **Critical RPC Functions**
1. **`get_skill_hierarchy()`** - Retrieves complete skill tree
2. **`update_skill_assignments_6level()`** - Bulk assignment updates
3. **`get_skill_assignments_6level()`** - Matrix data with 6-level hierarchy
4. **`validate_session()`** - User session validation
5. **`authenticate_user()`** - Custom login system

### Data Access Control Architecture

#### **Row-Level Security (RLS) Implementation**
```sql
-- Policy Pattern
CREATE POLICY policy_name ON table_name
FOR operation TO role_name
USING (condition_expression);
```

#### **Security Layers**
1. **Schema Level:** `"SkillMatrix"` schema access control with explicit permissions
2. **Table Level:** RLS policies on sensitive tables
3. **Function Level:** Parameter validation and sanitization  
4. **Session Level:** Token-based authentication validation

#### **Schema Permissions (Critical for Supabase Access)**
```sql
-- Required for direct schema access via Supabase client
GRANT USAGE ON SCHEMA "SkillMatrix" TO anon;
GRANT USAGE ON SCHEMA "SkillMatrix" TO authenticated;
GRANT SELECT ON "SkillMatrix"."v_skill_hierarchy_ancestors" TO anon;
GRANT SELECT ON "SkillMatrix"."v_skill_hierarchy_ancestors" TO authenticated;
```

### Connection Management
```javascript
// Hybrid Service Pattern
RPC Functions (Complex Operations) ← Hybrid Router → Direct API (Simple CRUD)
                ↓                                              ↓
        PostgreSQL Stored Procedures                    Supabase Client
```

## Past Mistakes & Solutions

### 1. **PostgreSQL Reserved Keywords**
**Problem:** Using `exists` as column name causing SQL syntax errors
**Solution:** Always quote reserved keywords or use alternative names
**Prevention:** Check PostgreSQL reserved word list before column naming
```sql
-- ❌ WRONG: CREATE TABLE test (exists BOOLEAN);
-- ✅ CORRECT: CREATE TABLE test (record_exists BOOLEAN);
```

### 2. **Schema Case Sensitivity Issues**
**Problem:** Schema name `SkillMatrix` vs `skillmatrix` causing access errors
**Error:** `permission denied for view v_skill_hierarchy_ancestors (42501)`
**Root Cause:** PostgreSQL stores schema names case-sensitively
**Solution:** Always use quoted schema names in SQL and proper case in JavaScript
**Prevention:** Establish consistent casing conventions

```sql
-- ❌ WRONG: SELECT * FROM skillmatrix.skill_base;
-- ✅ CORRECT: SELECT * FROM "SkillMatrix"."skill_base";
```

```javascript
// ❌ WRONG: { db: { schema: 'skillmatrix' } }
// ✅ CORRECT: { db: { schema: 'SkillMatrix' } }
```

**Key Requirements:**
- SQL: Always quote schema name `"SkillMatrix"`
- JavaScript: Match exact case `'SkillMatrix'`
- Permissions: Grant to quoted schema `GRANT USAGE ON SCHEMA "SkillMatrix"`

### 2. **RCP Function Parameter Ambiguity**
**Problem:** Column name ambiguity in complex JOINs (e.g., `id` field conflicts)
**Solution:** Explicit table aliasing and column prefixing
**Files:** Multiple RCP functions in `/sql/rcp_in_operation_by_function/`
**Prevention:** Always use table aliases and prefix column names

### 3. **Session Management Security Issues**
**Problem:** Session tokens not expiring, security vulnerability
**Solution:** Implemented session expiry with cleanup procedures
**Files:** `public.validate_session_05JUN25.sql`, `public.cleanup_expired_sessions_05JUN25.sql`
**Prevention:** Always implement session expiry and cleanup

### 4. **Database Connection Pooling Issues**
**Problem:** Connection exhaustion under high load
**Solution:** Implemented connection pooling in service layer
**Files:** `src/services/core/supabaseClient.js`
**Prevention:** Use singleton pattern for database connections

### 5. **Audit Trail Data Integrity**
**Problem:** Missing audit information on data changes
**Solution:** Added audit columns with automatic timestamping
**Files:** Table creation scripts with `created_at`, `updated_at`, `updated_by`
**Prevention:** Include audit fields in all data modification operations

### 6. **Bulk Update Performance Issues**
**Problem:** Individual updates causing timeout on large datasets
**Solution:** Implemented batch processing in RCP functions
**Files:** `(34)_update_skill_assignments_6level_ABSOLUTELY_FINAL_16JUL25.sql`
**Prevention:** Use bulk operations for large data modifications

## Critically Affected Files

### Table Creation Scripts
- **`sql/table_creation/01_create_skillmatrix_schema_04JUN25.sql`** - Schema creation
- **`sql/table_creation/02_create_skill_base_table_04JUN25.sql`** - Core skill hierarchy
- **`sql/table_creation/03_create_skill_detail_table_04JUN25.sql`** - Skill details
- **`sql/table_creation/skill_assignments_table_09JUL25.sql`** - Assignment mappings
- **`sql/table_creation/employees_table_09JUL25.sql`** - Employee data

### RCP Functions (Latest Working Versions)
- **`(34)_update_skill_assignments_6level_ABSOLUTELY_FINAL_16JUL25.sql`** - Bulk assignments
- **`get_skill_assignments_6level_16JUL25.sql`** - Matrix data retrieval
- **`public.get_all_skill_details_07JUL25_FIXED.sql`** - Skill details with hierarchy
- **`public.validate_session_05JUN25.sql`** - Session validation
- **`public.authenticate_user_07JUN25_FIXED.sql`** - User authentication

### Security Policies
- **`sql/policies/skillmatrix_skill_base_policies_05JUN25.sql`** - Skill base RLS
- **`sql/policies/skillmatrix_skill_detail_policies_05JUN25.sql`** - Skill detail RLS  
- **`sql/policies/public_allowed_users_policies_05JUN25.sql`** - User access control

### Service Layer Integration
- **`src/services/core/supabaseClient.js`** - Database connection singleton
- **`src/services/core/rpcService.js`** - RCP function interface
- **`src/services/core/hybridService.js`** - Smart routing between RCP/API

### Configuration
- **`src/config/supabaseConfig.js`** - Database connection configuration
- **Environment Variables:** `SUPABASE_URL`, `SUPABASE_ANON_KEY`

## Database Rebuild Preparation

### Ready-to-Deploy SQL Scripts
All scripts in these folders are production-ready for database rebuild:

#### **Schema and Tables**
```
sql/table_creation/
├── 01_create_skillmatrix_schema_04JUN25.sql
├── 02_create_skill_base_table_04JUN25.sql  
├── 03_create_skill_detail_table_04JUN25.sql
└── [additional table scripts]
```

#### **RCP Functions**
```
sql/rcp_in_operation_by_function/
├── (34)_update_skill_assignments_6level_ABSOLUTELY_FINAL_16JUL25.sql
├── get_skill_assignments_6level_16JUL25.sql
└── [additional RCP scripts]
```

#### **Security Policies**
```
sql/policies/
├── skillmatrix_skill_base_policies_05JUN25.sql
├── skillmatrix_skill_detail_policies_05JUN25.sql
└── [additional policy scripts]
```

## Debug Configuration

### Logging Modules
```javascript
// Enable database debugging
window.__debugLogger.setModuleFilter('supabase', true);
window.__debugLogger.setModuleFilter('tabulator-db', true);
window.__debugLogger.setLogLevel(window.__debugLogger.CONFIG.LOG_LEVELS.DEBUG);
```

### Testing Scripts
- **`sql/testing/`** - RCP function test scripts
- **`test/dbTest.html`** - Database integration testing tool

## Database Views Implementation

### Skill Progression Analysis View (v_skill_progression_analysis)
**Created:** 18AUG25  
**Location:** `sql/views/40_v_skill_progression_analysis_18AUG25.sql`  
**Purpose:** Comprehensive career progression skill gap analysis

#### **View Capabilities**
- **Career Track Analysis:** Analyzes skill requirements across all progression paths
- **Skill Gap Identification:** Identifies NEW, LEVEL_UP, CONTINUES, REMOVED skill changes
- **Business Intelligence:** Provides human-readable progression labels and severity assessment
- **Hierarchy Integration:** Joins with skill hierarchy for complete context

#### **Supported Career Tracks**
1. **Production Assistant (DA):** DA1 → DA2 → DA3
2. **Technician (T):** DA3 → T1 → T2 → T3  
3. **Operations (O):** DA3 → O1 → O2 → O3
4. **Engineering (S):** T3/O3 → S1 → S2 → S3 → S4 → S5
5. **Management (M):** S5 → M1 → M2 → M3 → M4 → M5
6. **Principal Engineer (P):** S5 → P1 → P2 → P3 → P4

#### **API Integration Challenges & Solutions**
**Problem:** API ordering by `created_at` column not available in views  
**Solution:** Modified `API/src/services/supabaseClient.js` to handle views without timestamp columns
```javascript
const viewsWithoutCreatedAt = ['v_skill_progression_analysis', 'v_skill_hierarchy_ancestors'];
// Skip default ordering for views without created_at
```

#### **Power BI Integration**
- **Endpoint:** `POST /api/v1/views`
- **Data Limit Bypass:** Use `columns` parameter to retrieve all data (bypasses 1000 row limit)
- **Authentication:** Time-based Bearer token system

### Skill Hierarchy Ancestors View (v_skill_hierarchy_ancestors)
**Purpose:** Recursive skill hierarchy with ancestor path information  
**Features:** Provides full skill paths and hierarchy level context

## Current RCP Function Count

**Next RCP Function Number:** (41) *(40 used for v_skill_progression_analysis view)*
**Naming Convention:** `(##)_function_name_DDMMMYY.sql`

## Recent Updates

**Last Updated:** 20AUG25  
**Recent Changes:**   
- ✅ **Skill Progression Analysis View:** Complete implementation with API integration
- ✅ **API Column Reference Fix:** Resolved ordering issues for views without `created_at`
- ✅ **Power BI Integration:** M-code solution for unlimited data retrieval
- ✅ **Documentation:** Complete implementation guide created

**Critical Notes:**   
- PostgreSQL reserved keywords must be avoided or quoted  
- Views require special handling in API for ordering and column references
- Power BI data limits can be bypassed using specific column selection

**Next Planned:** Database performance optimization and materialized view consideration for large datasets