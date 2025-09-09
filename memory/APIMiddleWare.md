# Skill Matrix Power BI API - Complete Integration Guide

**Purpose:** Isolated API middleware for Power BI integration with Skill Matrix MVP4 database  
**Status:** ✅ Production Ready with Dynamic Data Endpoint  
**Last Updated:** 08AUG25

## 🎯 **API Overview**

This API provides a secure, isolated middleware layer between Power BI and the Skill Matrix database. It features both fixed endpoints for specific data types and a powerful dynamic endpoint that allows flexible table and column selection through JSON configuration.

### **Key Features**
- ✅ **Dynamic Data Endpoint** - Single endpoint for all table access
- ✅ **Time-based Authentication** - 1-hour token expiry with auto-renewal
- ✅ **Flexible Column Selection** - Get all columns or specify exactly what you need
- ✅ **Table Security** - Allowlist prevents unauthorized table access
- ✅ **Power BI Optimized** - Ready-to-use M-code templates
- ✅ **Isolated Deployment** - Independent Docker container

## 🔧 **Architecture & Isolation**

This API is **completely isolated** from the main Skill Matrix project:

- **Source Code:** Located in `MVP4/API/` folder
- **Docker Setup:** Separate `docker-compose.api.yml`
- **Environment:** Isolated `.env.api` configuration  
- **Container:** Independent `skillmatrix-powerbi-api` container
- **Network:** Bridges to existing Supabase network for database access
- **Deployment:** Separate build and deployment process

## 🚀 **Quick Start**

### **Prerequisites**
- Docker and Docker Compose installed
- Access to existing Supabase network (`supabase2_default`)
- Power BI Desktop or Power BI Service access

### **Setup Steps**

1. **Configure Environment**
   ```bash
   # Ensure .env.api file exists with correct values
   SUPABASE_URL=http://supabase-kong:8000
   SUPABASE_SERVICE_ROLE_KEY=your_service_role_jwt_token
   TOKEN_STATIC_PART=1aacc6d7c6180bef97d69489f584bd40
   TOKEN_EXPIRY_HOURS=1
   ALLOWED_IPS=172.16.0.0/12,10.135.12.166,10.135.3.144,127.0.0.1,::1
   ```

2. **Deploy to NAS (Production)**
   ```bash
   # Use deployment script
   C:\Users\mes13a03\DockerBuild\SkillMatrixMVP\MVP4\deploy\deploy-api-to-nas.bat
   ```

3. **Start API (Local Development)**
   ```bash
   docker-compose -f docker-compose.api.yml up -d
   ```

4. **Verify Health**
   ```bash
   curl http://10.135.3.144:3001/api/v1/health
   ```

## 📊 **API Endpoints**

### **Base URL:** `http://10.135.3.144:3001/api/v1/`

### **Authentication**
- **Header:** `Authorization: Bearer {token}`
- **Token Format:** `1aacc6d7c6180bef97d69489f584bd40` + `yyMMddHH`
- **Example:** `Authorization: Bearer 1aacc6d7c6180bef97d69489f584bd4025080812`
- **Expiry:** 1 hour (auto-renewable in Power BI)

### **Available Endpoints**

| Endpoint | Method | Description | Authentication |
|----------|--------|-------------|----------------|
| `/health` | GET | API health check | No |
| `/health/database` | GET | Database connectivity check | No |
| `/skill-hierarchy` | GET | Complete skill tree structure | Required |
| `/skill-assignments` | GET | Skill assignments with 6-level org hierarchy | Required |
| `/employee-matrix` | GET | Employee skill matrix for pivot tables | Required |
| `/organizational-summary` | GET | Aggregated organizational data | Required |
| `/priority-skills` | GET | Priority skills only | Required |
| **`/data`** | **POST** | **🔥 Dynamic data endpoint - specify table & columns** | **Required** |
| `/documentation` | GET | API documentation and examples | No |

## 🔥 **Dynamic Data Endpoint (Recommended)**

### **Why Use the Dynamic Endpoint?**
The `/data` endpoint is the most flexible and powerful way to access your data:
- **Single Endpoint** - No need for multiple fixed endpoints
- **Flexible Column Selection** - Get all columns or specify exactly what you need
- **Table Agnostic** - Works with any allowed table
- **Future Proof** - Easy to extend for new tables and features

### **URL:** `POST http://10.135.3.144:3001/api/v1/data`

### **⚠️ Important Design Note: POST vs GET**
**Why HTTP POST with JSON "method": "get"?**

- **HTTP Level**: Uses `POST /api/v1/data` because we need to send JSON request body
- **JSON Level**: Contains `"method": "get"` to specify database operation type
- **Reasoning**: 
  - Power BI handles POST with JSON body more reliably than GET with query parameters
  - Follows existing API pattern from working templates
  - Allows future expansion for other operations ("count", "filter", "insert", etc.)
  - GET requests traditionally don't have request bodies, making POST more appropriate

**This is intentional design:**
- HTTP POST = "I'm sending data in the request body"
- JSON "method": "get" = "I want to retrieve/SELECT data from the database"

### **Supported Tables**
- `skill_base` - Core skill hierarchy data
- `skill_detail` - Detailed skill information  
- `skill_assignments` - Employee skill assignments with organizational data
- `employees` - Employee master data
- `emp_skill_assessments` - Skill assessment history
- `skill_levels` - Skill proficiency level definitions

### **JSON Request Format**

**Get All Columns:**
```json
{
    "para": {
        "method": "get",
        "table": "skill_base",
        "schema": "SkillMatrix"
    }
}
```

**Get Specific Columns:**
```json
{
    "para": {
        "method": "get",
        "table": "skill_base", 
        "schema": "SkillMatrix",
        "columns": ["id", "skill_name", "parent_id", "hlvl", "skill_type"]
    }
}
```

## 🔧 **Power BI Integration**

### **Template 1: Get All Columns from Table**

```m
let
    // ===== CONFIGURATION =====
    StaticPart = "1aacc6d7c6180bef97d69489f584bd40",
    BaseUrl = "http://10.135.3.144:3001/api/v1/data",
    
    // JSON Body: All columns from skill_base table (readable format)
    #"Body" = "{""para"": {""method"": ""get"", ""table"": ""skill_base"", ""schema"": ""SkillMatrix""}}",
    
    // ===== TOKEN GENERATION =====
    CurrentDateTime = DateTime.LocalNow(),
    FormattedDateTime = DateTime.ToText(CurrentDateTime, "yyMMddHH"),
    FullToken = "Bearer " & StaticPart & FormattedDateTime,
    
    #"Headers" = [#"Authorization"=FullToken, #"Content-Type"="application/json"],
    
    // ===== API CALL =====
    Source = Json.Document(Web.Contents(BaseUrl, [Headers=#"Headers", Content=Text.ToBinary(#"Body")])),
    #"Converted to Table" = Table.FromRecords({Source}),
    #"Expanded data" = Table.ExpandListColumn(#"Converted to Table", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", Record.FieldNames(#"Expanded data"{0}[data]{0}))
in
    #"Expanded data1"
```

### **Template 2: Get Specific Columns from Table**

```m
let
    // ===== CONFIGURATION =====
    StaticPart = "1aacc6d7c6180bef97d69489f584bd40",
    BaseUrl = "http://10.135.3.144:3001/api/v1/data",
    
    // JSON Body: Specific columns from skill_base table (readable format)
    #"Body" = "{""para"": {""method"": ""get"", ""table"": ""skill_base"", ""schema"": ""SkillMatrix"", ""columns"": [""id"", ""skill_name"", ""parent_id"", ""hlvl"", ""skill_type""]}}",
    
    // ===== TOKEN GENERATION =====
    CurrentDateTime = DateTime.LocalNow(),
    FormattedDateTime = DateTime.ToText(CurrentDateTime, "yyMMddHH"),
    FullToken = "Bearer " & StaticPart & FormattedDateTime,
    
    #"Headers" = [#"Authorization"=FullToken, #"Content-Type"="application/json"],
    
    // ===== API CALL =====
    Source = Json.Document(Web.Contents(BaseUrl, [Headers=#"Headers", Content=Text.ToBinary(#"Body")])),
    #"Converted to Table" = Table.FromRecords({Source}),
    #"Expanded data" = Table.ExpandListColumn(#"Converted to Table", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"id", "skill_name", "parent_id", "hlvl", "skill_type"}, {"data.id", "data.skill_name", "data.parent_id", "data.hlvl", "data.skill_type"}),
    #"Changed Type" = Table.TransformColumnTypes(#"Expanded data1",{{"success", type logical}, {"data.id", type text}, {"data.skill_name", type text}, {"data.parent_id", type text}, {"data.hlvl", Int64.Type}, {"data.skill_type", type text}, {"table", type text}, {"schema", type text}, {"totalRecords", Int64.Type}, {"timestamp", type datetime}})
in
    #"Changed Type"
```

### **Template 3: Skill Assignments (All Columns)**

```m
let
    // ===== CONFIGURATION =====
    StaticPart = "1aacc6d7c6180bef97d69489f584bd40",
    BaseUrl = "http://10.135.3.144:3001/api/v1/data",
    
    // JSON Body: All columns from skill_assignments table (readable format)
    #"Body" = "{""para"": {""method"": ""get"", ""table"": ""skill_assignments"", ""schema"": ""SkillMatrix""}}",
    
    // ===== TOKEN GENERATION =====
    CurrentDateTime = DateTime.LocalNow(),
    FormattedDateTime = DateTime.ToText(CurrentDateTime, "yyMMddHH"),
    FullToken = "Bearer " & StaticPart & FormattedDateTime,
    
    #"Headers" = [#"Authorization"=FullToken, #"Content-Type"="application/json"],
    
    // ===== API CALL =====
    Source = Json.Document(Web.Contents(BaseUrl, [Headers=#"Headers", Content=Text.ToBinary(#"Body")])),
    #"Converted to Table" = Table.FromRecords({Source}),
    #"Expanded data" = Table.ExpandListColumn(#"Converted to Table", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", Record.FieldNames(#"Expanded data"{0}[data]{0}))
in
    #"Expanded data1"
```

### **Template 4: Employees with Specific Columns**

```m
let
    // ===== CONFIGURATION =====
    StaticPart = "1aacc6d7c6180bef97d69489f584bd40",
    BaseUrl = "http://10.135.3.144:3001/api/v1/data",
    
    // JSON Body: Specific employee columns (readable format)
    #"Body" = "{""para"": {""method"": ""get"", ""table"": ""employees"", ""schema"": ""SkillMatrix"", ""columns"": [""employee_id"", ""employee_name"", ""org_l1"", ""org_l2"", ""org_l3"", ""job_grade""]}}",
    
    // ===== TOKEN GENERATION =====
    CurrentDateTime = DateTime.LocalNow(),
    FormattedDateTime = DateTime.ToText(CurrentDateTime, "yyMMddHH"),
    FullToken = "Bearer " & StaticPart & FormattedDateTime,
    
    #"Headers" = [#"Authorization"=FullToken, #"Content-Type"="application/json"],
    
    // ===== API CALL =====
    Source = Json.Document(Web.Contents(BaseUrl, [Headers=#"Headers", Content=Text.ToBinary(#"Body")])),
    #"Converted to Table" = Table.FromRecords({Source}),
    #"Expanded data" = Table.ExpandListColumn(#"Converted to Table", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"employee_id", "employee_name", "org_l1", "org_l2", "org_l3", "job_grade"}, {"data.employee_id", "data.employee_name", "data.org_l1", "data.org_l2", "data.org_l3", "data.job_grade"}),
    #"Changed Type" = Table.TransformColumnTypes(#"Expanded data1",{{"success", type logical}, {"data.employee_id", type text}, {"data.employee_name", type text}, {"data.org_l1", type text}, {"data.org_l2", type text}, {"data.org_l3", type text}, {"data.job_grade", type text}, {"table", type text}, {"schema", type text}, {"totalRecords", Int64.Type}, {"timestamp", type datetime}})
in
    #"Changed Type"
```

## 📝 **JSON Body Format Guide**

### **Readable Single-Line Format (Recommended)**
Use this format to prevent Power BI from adding `#(cr)#(lf)` line breaks:

```m
// All columns
#"Body" = "{""para"": {""method"": ""get"", ""table"": ""skill_assignments"", ""schema"": ""SkillMatrix""}}"

// Specific columns  
#"Body" = "{""para"": {""method"": ""get"", ""table"": ""skill_detail"", ""schema"": ""SkillMatrix"", ""columns"": [""id"", ""skill_name"", ""description""]}}"
```

### **What NOT to Use**
❌ **Avoid multi-line JSON** - Power BI converts it to messy format with `#(cr)#(lf)` insertions

### **Quick Configuration Examples**

```m
// All columns from skill_base
#"Body" = "{""para"": {""method"": ""get"", ""table"": ""skill_base"", ""schema"": ""SkillMatrix""}}"

// Specific columns from employees
#"Body" = "{""para"": {""method"": ""get"", ""table"": ""employees"", ""schema"": ""SkillMatrix"", ""columns"": [""employee_id"", ""employee_name"", ""org_l1""]}}"

// All skill assignments
#"Body" = "{""para"": {""method"": ""get"", ""table"": ""skill_assignments"", ""schema"": ""SkillMatrix""}}"
```

## 🗂️ **Available Tables & Common Columns**

### **skill_base**
Core skill hierarchy data
- **Common columns**: `id`, `skill_name`, `parent_id`, `hlvl`, `description`, `skill_type`, `selection1`, `created_at`, `updated_at`

### **skill_detail**  
Detailed skill information
- **Common columns**: `id`, `skill_id`, `skill_name`, `skill_description`, `created_at`, `updated_at`

### **skill_assignments**
Employee skill assignments with organizational hierarchy
- **Common columns**: `id`, `employee_id`, `skill_id`, `proficiency_level`, `org_l1`, `org_l2`, `org_l3`, `org_l4`, `org_l5`, `org_l6`, `created_at`, `updated_at`

### **employees**
Employee master data with organizational structure
- **Common columns**: `employee_id`, `employee_name`, `org_l1`, `org_l2`, `org_l3`, `org_l4`, `org_l5`, `org_l6`, `job_grade`, `created_at`, `updated_at`

### **emp_skill_assessments**
Skill assessment history and tracking
- **Common columns**: `id`, `employee_id`, `skill_id`, `assessment_date`, `assessor`, `proficiency_level`, `created_at`

### **skill_levels**
Skill proficiency level definitions
- **Common columns**: `id`, `level_name`, `level_description`, `level_order`

## 🔐 **Security Features**

### **Authentication & Authorization**
- **Time-based Token Validation** - 1-hour expiration with automatic renewal
- **Service Role Access** - Bypasses RLS policies for trusted middleware
- **Table Allowlist** - Only permitted tables can be accessed
- **Column Name Validation** - SQL injection protection through regex validation

### **Network Security**
- **IP Whitelisting** - Configurable IP restrictions in `.env.api`
- **Docker Network Isolation** - Runs in isolated container environment
- **Rate Limiting** - Prevents API abuse (100 requests per 15-minute window)

### **Data Protection**
- **Read-Only Access** - No data modification capabilities
- **Request Logging** - Complete audit trail of all API calls
- **Error Message Sanitization** - No internal system details exposed

## 📈 **Usage Notes**

1. **All Columns**: Omit the `columns` array to get all available columns from the table
2. **Specific Columns**: Include the `columns` array with desired column names
3. **Dynamic Expansion**: Templates 1 & 3 use `Record.FieldNames()` for automatic column detection
4. **Fixed Expansion**: Templates 2 & 4 use predefined column lists for better type control
5. **Token Expiry**: Tokens expire every hour (format: yyMMddHH) - Power BI handles renewal automatically
6. **Error Handling**: API returns detailed error messages for invalid tables/columns

## 🧪 **Testing**

### **Postman Testing**

**Test Dynamic Endpoint:**
```bash
# URL
POST http://10.135.3.144:3001/api/v1/data

# Headers
Authorization: Bearer 1aacc6d7c6180bef97d69489f584bd4025080812
Content-Type: application/json

# Body (All Columns)
{
    "para": {
        "method": "get",
        "table": "skill_base",
        "schema": "SkillMatrix"
    }
}

# Body (Specific Columns)
{
    "para": {
        "method": "get",
        "table": "skill_base",
        "schema": "SkillMatrix",
        "columns": ["id", "skill_name", "parent_id"]
    }
}
```

**Expected Response:**
```json
{
    "success": true,
    "data": [
        {
            "id": "uuid",
            "skill_name": "string",
            "parent_id": "uuid",
            "hlvl": 0
        }
    ],
    "table": "skill_base",
    "schema": "SkillMatrix",
    "columns": ["id", "skill_name", "parent_id"],
    "totalRecords": 150,
    "timestamp": "2025-08-08T12:30:00.000Z"
}
```

### **Health Check**
```bash
# Check API health
curl http://10.135.3.144:3001/api/v1/health

# Expected response:
{
  "status": "healthy",
  "timestamp": "2025-08-08T10:30:00.000Z",
  "version": "1.0.0"
}
```

## 🐳 **Docker Commands**

### **Production Deployment (NAS)**
```bash
# Deploy to NAS
C:\Users\mes13a03\DockerBuild\SkillMatrixMVP\MVP4\deploy\deploy-api-to-nas.bat

# SSH to NAS and check status
ssh WONGSF@10.135.3.144
cd /volume8/docker/skillmatrixMVP/api
docker-compose -f docker-compose.api.yml ps
```

### **Local Development**
```bash
# Build container
docker-compose -f docker-compose.api.yml build

# Start services
docker-compose -f docker-compose.api.yml up -d

# View logs
docker-compose -f docker-compose.api.yml logs -f

# Stop services
docker-compose -f docker-compose.api.yml down

# Restart services
docker-compose -f docker-compose.api.yml restart
```

### **Container Management**
```bash
# Check status
docker-compose -f docker-compose.api.yml ps

# View container logs
docker logs skillmatrix-powerbi-api --tail 100 --follow

# Execute commands in container
docker exec -it skillmatrix-powerbi-api bash

# Check container stats
docker stats skillmatrix-powerbi-api
```

## 🔧 **Development**

### **Local Development Setup**
```bash
# Install dependencies
npm install

# Start in development mode (if not using Docker)
npm run dev

# Environment variables needed in .env.api:
SUPABASE_URL=http://supabase-kong:8000
SUPABASE_SERVICE_ROLE_KEY=your_service_role_jwt
TOKEN_STATIC_PART=1aacc6d7c6180bef97d69489f584bd40
TOKEN_EXPIRY_HOURS=1
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_WINDOW_MS=900000
ALLOWED_IPS=172.16.0.0/12,10.135.12.166,10.135.3.144,127.0.0.1,::1
```

### **Key Files**
- `src/app.js` - Main Express application with all endpoints
- `src/services/supabaseClient.js` - Database connection and query methods
- `src/middleware/auth.js` - Time-based token authentication
- `docker-compose.api.yml` - Container orchestration
- `.env.api` - Environment configuration

## 🚨 **Troubleshooting**

### **Common Issues**

1. **404 Error on /api/v1/data**
   - **Cause**: API not deployed or container not restarted
   - **Solution**: Run deployment script and verify container is running

2. **Authentication Failed**
   - **Cause**: Token format incorrect or expired
   - **Solution**: Verify token generation matches `StaticPart + yyMMddHH` format

3. **Container Won't Start**
   - **Cause**: Network connectivity or environment variables
   - **Solution**: Check `docker network ls | grep supabase` and verify `.env.api`

4. **Database Connection Issues**
   - **Cause**: Supabase not running or network bridge missing
   - **Solution**: Verify Supabase containers and network bridge: `docker network connect supabase2_default <api-container-id>`

5. **Power BI Connection Fails**
   - **Cause**: URL, headers, or JSON format issues
   - **Solution**: Test with Postman first, then use exact working format in Power BI

### **Debugging Steps**
1. **Check API Health**: `curl http://10.135.3.144:3001/api/v1/health`
2. **View Container Logs**: `docker logs skillmatrix-powerbi-api --tail 50`
3. **Test with Postman**: Verify endpoint works outside Power BI
4. **Check Network**: `docker exec -it skillmatrix-powerbi-api ping supabase-kong`
5. **Verify Environment**: `docker exec -it skillmatrix-powerbi-api env | grep SUPABASE`

## 📋 **Maintenance**

### **Updates & Redeployment**
```bash
# Full redeployment to NAS
C:\Users\mes13a03\DockerBuild\SkillMatrixMVP\MVP4\deploy\deploy-api-to-nas.bat

# Manual container rebuild (on NAS)
docker-compose -f docker-compose.api.yml down
docker-compose -f docker-compose.api.yml build --no-cache
docker-compose -f docker-compose.api.yml up -d
```

### **Monitoring**
```bash
# Check container health
docker exec -it skillmatrix-powerbi-api curl http://localhost:3001/api/v1/health

# Monitor logs
docker logs skillmatrix-powerbi-api --tail 100 --follow

# Check network connectivity
docker exec -it skillmatrix-powerbi-api curl http://supabase-kong:8000/rest/v1/
```

### **Cleanup**
```bash
# Remove container and volumes
docker-compose -f docker-compose.api.yml down --volumes

# Remove unused images
docker image prune -f
```

## 🎯 **Business Use Cases**

### **Skill Hierarchy Analysis**
Use `skill_base` table for:
- Organizational skill structure visualization
- Priority skill identification
- Skill category analysis

### **Employee Skills Matrix**
Use `skill_assignments` table for:
- Skills gap analysis by organization
- Employee competency tracking
- Training needs identification

### **Organizational Reporting**
Use `employees` table for:
- Headcount by organizational level
- Job grade distribution
- Organizational structure analysis

### **Assessment Tracking**
Use `emp_skill_assessments` table for:
- Skills assessment history
- Assessor performance tracking
- Competency development over time

---

## 📞 **Support & Documentation**

- **API Health Check**: `http://10.135.3.144:3001/api/v1/health`
- **API Documentation**: `http://10.135.3.144:3001/api/v1/documentation`
- **Container Logs**: Available via Docker commands above
- **Network Status**: Verify Supabase connectivity through health endpoints

**Note:** This API is completely isolated from the main Skill Matrix project and can be developed, deployed, and maintained independently.

---

**🎉 Ready for Production Use! The dynamic Power BI API provides flexible, secure access to your Skill Matrix data with comprehensive Power BI integration support.**