# 🔥 Quick Action Checklist - Masters Backend & Cleanup

## 🚀 Day 1 - Immediate Actions

### Backend Setup (2 hours)
```bash
# 1. Create backend project
mkdir erp-backend && cd erp-backend
python -m venv venv
source venv/bin/activate

# 2. Install core dependencies
pip install fastapi sqlalchemy asyncpg uvicorn alembic

# 3. Create folder structure
mkdir -p app/{api/v1,core,models,schemas,services}

# 4. Create main.py
touch app/main.py
```

### PostgreSQL Setup (1 hour)
```sql
-- Create database and schemas
CREATE DATABASE erp_db;
\c erp_db;

CREATE SCHEMA core;     -- Companies, Users
CREATE SCHEMA masters;  -- All master data
CREATE SCHEMA audit;    -- Audit logs

CREATE EXTENSION "uuid-ossp";
```

---

## 📋 Frontend Files to Clean (Priority Order)

### 🔴 HIGH PRIORITY - Delete These Files:
```
❌ src/data/mockData.ts
❌ src/data/sampleData.ts
❌ src/data/glAccounts.ts
❌ src/data/customers.ts
❌ src/data/suppliers.ts
❌ src/data/items.ts
❌ src/utils/testDataGenerator.ts
```

### 🟡 MEDIUM PRIORITY - Refactor These:
```
⚠️ src/pages/Login.tsx → Remove hardcoded companies
⚠️ src/hooks/useMasterData.ts → Convert to API calls
⚠️ src/context/AppContext.tsx → Remove localStorage
⚠️ src/components/selects/* → Update to use API
```

### 🟢 LOW PRIORITY - Update Later:
```
📝 src/pages/masters/* → Add Create/Update forms
📝 src/utils/validation.ts → Keep, still useful
```

---

## 🗑️ localStorage Keys to Remove

Search and remove all references to:
```javascript
// Remove these localStorage calls:
localStorage.setItem('customers', ...)
localStorage.setItem('suppliers', ...)
localStorage.setItem('glAccounts', ...)
localStorage.setItem('items', ...)
localStorage.setItem('taxGroups', ...)
localStorage.setItem('salesTypes', ...)
localStorage.setItem('paymentTerms', ...)
localStorage.setItem('masterData', ...)

// Keep only:
localStorage.getItem('access_token')  // ✅ Keep for auth
localStorage.getItem('user')          // ✅ Keep for user session
localStorage.getItem('theme')         // ✅ Keep for UI preferences
```

---

## 🎯 Masters Implementation Order

Implement in this exact order for dependencies:

1. **Companies** (Everything depends on this)
2. **Users & Auth** (Need auth for all APIs)
3. **GL Accounts** (Referenced by many masters)
4. **Tax Groups** (Referenced by items/customers)
5. **Customers** (Core master)
6. **Suppliers** (Core master)
7. **Items** (Depends on GL, Tax)
8. **Sales Types** (Depends on GL accounts)
9. **Payment Terms** (Simple master)
10. **Cost Centers** (Optional master)

---

## 🔍 Find & Replace Patterns

### Pattern 1: Mock Data Arrays
```javascript
// FIND:
const customers = [
  { id: 1, name: "Test Customer", ... },
  { id: 2, name: "Demo Customer", ... }
];

// REPLACE WITH:
const { data: customers, isLoading } = useQuery({
  queryKey: ['customers'],
  queryFn: () => apiService.getCustomers()
});
```

### Pattern 2: localStorage Usage
```javascript
// FIND:
const data = JSON.parse(localStorage.getItem('customers') || '[]');

// REPLACE WITH:
const { data } = await apiService.getCustomers();
```

### Pattern 3: Save Functions
```javascript
// FIND:
const saveCustomer = (customer) => {
  const customers = JSON.parse(localStorage.getItem('customers') || '[]');
  customers.push(customer);
  localStorage.setItem('customers', JSON.stringify(customers));
};

// REPLACE WITH:
const saveCustomer = async (customer) => {
  return await apiService.createCustomer(customer);
};
```

---

## 📁 First Backend Endpoint (GL Accounts)

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="ERP API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# app/api/v1/gl_accounts.py
from fastapi import APIRouter, Depends
from typing import List

router = APIRouter(prefix="/api/v1/masters")

@router.get("/gl-accounts")
async def get_gl_accounts():
    # Start with static data, then connect to DB
    return {
        "data": [
            {"id": "1", "code": "1000", "name": "Cash", "type": "Asset"},
            {"id": "2", "code": "1200", "name": "Accounts Receivable", "type": "Asset"},
            {"id": "3", "code": "4000", "name": "Sales Revenue", "type": "Revenue"},
        ]
    }

# app/main.py (add this)
from app.api.v1 import gl_accounts
app.include_router(gl_accounts.router)
```

**Run it:**
```bash
uvicorn app.main:app --reload --port 8000
```

**Test it:**
```bash
curl http://localhost:8000/api/v1/masters/gl-accounts
```

---

## 🧪 Test Your Cleanup

### Step 1: Search for Hardcoded Data
```bash
# Run these commands in frontend folder:
grep -r "const.*=.*\[.*{.*id:" src/
grep -r "localStorage.setItem" src/
grep -r "localStorage.getItem" src/ | grep -v "token\|user\|theme"
grep -r "TEST\|DEMO\|Sample\|Mock" src/
```

### Step 2: Verify API Integration
```javascript
// Add to your test file:
console.log('Checking for hardcoded data...');
if (window.localStorage.length > 3) {  // token, user, theme
  console.error('Too many localStorage items!');
}
```

---

## 📊 Progress Tracker

### Week 1 Goals:
- [ ] Backend project created
- [ ] PostgreSQL database setup
- [ ] Authentication working
- [ ] Companies endpoint ready
- [ ] GL Accounts endpoint ready

### Week 2 Goals:
- [ ] All major masters have APIs
- [ ] Frontend login using backend
- [ ] GL Account dropdown using API
- [ ] Customer dropdown using API

### Week 3 Goals:
- [ ] All hardcoded data removed
- [ ] All dropdowns use APIs
- [ ] localStorage cleaned
- [ ] Error handling added

---

## ⚡ Quick Commands

### Backend Development:
```bash
# Start backend server
uvicorn app.main:app --reload

# Create new migration
alembic revision --autogenerate -m "Add customers table"

# Run migrations
alembic upgrade head

# Run tests
pytest tests/
```

### Frontend Cleanup:
```bash
# Find all localStorage usage
grep -r "localStorage" src/

# Find all hardcoded arrays
grep -r "= \[" src/

# Find mock data imports
grep -r "import.*mock\|sample\|test" src/
```

---

## 🚨 DO NOT FORGET!

1. **Backup everything** before deleting
2. **Test each API** before removing localStorage
3. **Keep auth tokens** in localStorage (they're OK!)
4. **Add loading states** when switching to API
5. **Handle API errors** gracefully
6. **Version your API** (/v1/) from start

---

## 📞 When You're Stuck

1. **Can't remove localStorage yet?** → Add a feature flag
2. **API not ready?** → Use mock API responses temporarily
3. **Breaking changes?** → Keep both versions temporarily
4. **Performance issues?** → Add caching layer
5. **Complex dependencies?** → Start with simple masters first

---

**Start NOW with:** Create backend → Add auth → Add GL Accounts API → Update frontend GL dropdown

This gets you immediate value and proves the concept! 🚀