# 🎯 ERP Backend Development Action Plan
## Phase 1: Masters Implementation & Frontend Cleanup

**Objective**: Build backend for all master data, remove hardcoded data from frontend
**Timeline**: 4-6 weeks
**Priority**: Masters only (transactions later)

---

## 📊 Master Data Inventory

### Core Masters to Implement:
1. **Companies** (Multi-tenant support)
2. **Users & Roles** (Authentication/Authorization)
3. **Chart of Accounts** (GL Accounts)
4. **Customers**
5. **Suppliers**
6. **Items/Products**
7. **Tax Groups & Tax Types**
8. **Sales Types**
9. **Payment Terms**
10. **Cost Centers**
11. **Currencies & Exchange Rates**
12. **Units of Measure**
13. **Warehouses/Locations**
14. **Item Categories**
15. **Customer/Supplier Groups**
16. **Price Lists**

---

## 🚀 WEEK 1: Backend Infrastructure Setup

### Day 1-2: Project Setup
```bash
# 1. Create backend project structure
mkdir erp-backend
cd erp-backend

# 2. Initialize Python environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate  # Windows

# 3. Create project structure
mkdir -p app/{api,core,models,schemas,services}
mkdir -p app/api/v1/{masters,auth}
mkdir -p tests migrations scripts
```

**Create requirements.txt:**
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
alembic==1.12.1
pydantic==2.5.0
pydantic-settings==2.1.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
redis==5.0.1
pandas==2.1.3
python-dateutil==2.8.2
httpx==0.25.1
pytest==7.4.3
pytest-asyncio==0.21.1
python-dotenv==1.0.0
```

### Day 2-3: Database Setup & Base Configuration

**1. PostgreSQL Database Schema:**
```sql
-- Create database
CREATE DATABASE erp_db;

-- Create schemas for logical separation
CREATE SCHEMA IF NOT EXISTS core;      -- System tables
CREATE SCHEMA IF NOT EXISTS masters;   -- Master data
CREATE SCHEMA IF NOT EXISTS audit;     -- Audit logs

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
```

**2. Create Base Models (app/models/base.py):**
```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, DateTime, UUID, Boolean
from datetime import datetime
import uuid

Base = declarative_base()

class BaseModel(Base):
    __abstract__ = True

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    is_active = Column(Boolean, default=True)

class CompanyScopedModel(BaseModel):
    __abstract__ = True
    company_id = Column(UUID(as_uuid=True), nullable=False)
```

**3. Database Connection (app/core/database.py):**
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
import os

DATABASE_URL = os.getenv("DATABASE_URL")
# In production, DATABASE_URL must be set as environment variable
# Example format: postgresql+asyncpg://username:password@host:5432/dbname
if not DATABASE_URL:
    raise ValueError("DATABASE_URL environment variable is required")

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

### Day 4-5: Authentication & Company Setup

**Implement core authentication and multi-tenant support**

**Models needed:**
- Company model
- User model
- Role model
- Permission model

**Endpoints needed:**
- POST /api/v1/auth/login
- POST /api/v1/auth/refresh
- GET /api/v1/auth/me
- POST /api/v1/companies (super admin only)

---

## 🗂️ WEEK 2: Core Masters Implementation

### Day 6-7: Chart of Accounts

**Backend Tasks:**
```python
# 1. Create GL Account model
# 2. Implement endpoints:
GET    /api/v1/masters/gl-accounts
GET    /api/v1/masters/gl-accounts/{id}
POST   /api/v1/masters/gl-accounts
PUT    /api/v1/masters/gl-accounts/{id}
DELETE /api/v1/masters/gl-accounts/{id}
GET    /api/v1/masters/gl-accounts/tree  # Hierarchical view

# 3. Add validation:
- Account code uniqueness per company
- Parent-child relationship validation
- Account type validation
```

**Frontend Cleanup:**
```typescript
// REMOVE from frontend:
// ❌ src/data/glAccounts.ts (hardcoded accounts)
// ❌ Any localStorage initialization for GL accounts

// REPLACE with:
// ✅ API service to fetch from backend
const fetchGLAccounts = async () => {
  const response = await api.get('/masters/gl-accounts');
  return response.data;
};
```

### Day 8-9: Customers & Suppliers

**Backend Implementation:**
```python
# Customer endpoints:
GET    /api/v1/masters/customers
GET    /api/v1/masters/customers/{id}
POST   /api/v1/masters/customers
PUT    /api/v1/masters/customers/{id}
DELETE /api/v1/masters/customers/{id}
GET    /api/v1/masters/customers/search?q=

# Supplier endpoints (similar pattern)
```

**Frontend Cleanup Tasks:**
```javascript
// Files to clean:
// 1. Remove getMasterData() mock data
// 2. Remove saveMasterData() localStorage saves
// 3. Remove any hardcoded customer/supplier arrays
// 4. Update all dropdown/select components to use API
```

### Day 10: Tax Groups & Tax Types

**Backend:**
```python
# Tax master endpoints:
GET    /api/v1/masters/tax-groups
GET    /api/v1/masters/tax-types
POST   /api/v1/masters/tax-groups
# Include tax calculation logic in backend
```

**Frontend Cleanup:**
```typescript
// REMOVE:
// ❌ Hardcoded tax rates
// ❌ Local tax calculation functions
// ❌ Tax data in localStorage

// REPLACE with API calls
```

---

## 🧹 WEEK 3: Frontend Deep Cleaning

### Day 11-12: Remove ALL Hardcoded Data

**Step 1: Identify all hardcoded data locations**
```bash
# Search for hardcoded data patterns
grep -r "const.*Data\s*=" src/
grep -r "localStorage.setItem" src/
grep -r "getMasterData" src/
grep -r "TEST\|DEMO\|Sample" src/
```

**Step 2: Create a cleanup checklist**

| File/Component | Hardcoded Data | Action | Status |
|----------------|---------------|---------|---------|
| `src/pages/Login.tsx` | Demo companies | Connect to API | ⏳ |
| `src/components/GLAccountSelect.tsx` | Mock GL accounts | Use API endpoint | ⏳ |
| `src/hooks/useMasterData.ts` | localStorage data | Replace with API | ⏳ |
| `src/data/*.ts` | All mock data files | Delete entirely | ⏳ |

### Day 13-14: Implement API Service Layer

**Create centralized API service (src/services/api.ts):**
```typescript
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:8000/api/v1';

class ApiService {
  private token: string | null = null;

  constructor() {
    this.token = localStorage.getItem('access_token');
  }

  // Auth methods
  async login(username: string, password: string) {
    const response = await axios.post(`${API_BASE_URL}/auth/login`, {
      username,
      password
    });
    this.token = response.data.access_token;
    localStorage.setItem('access_token', this.token);
    return response.data;
  }

  // Master data methods
  async getGLAccounts() {
    return this.get('/masters/gl-accounts');
  }

  async getCustomers(params?: any) {
    return this.get('/masters/customers', params);
  }

  async createCustomer(data: any) {
    return this.post('/masters/customers', data);
  }

  // Generic HTTP methods
  private async get(endpoint: string, params?: any) {
    const response = await axios.get(`${API_BASE_URL}${endpoint}`, {
      params,
      headers: this.getHeaders()
    });
    return response.data;
  }

  private async post(endpoint: string, data: any) {
    const response = await axios.post(`${API_BASE_URL}${endpoint}`, data, {
      headers: this.getHeaders()
    });
    return response.data;
  }

  private getHeaders() {
    return {
      'Authorization': `Bearer ${this.token}`,
      'Content-Type': 'application/json'
    };
  }
}

export default new ApiService();
```

### Day 15: Update React Hooks

**Transform existing hooks to use API:**
```typescript
// src/hooks/useGLAccounts.ts
import { useQuery } from '@tanstack/react-query';
import ApiService from '../services/api';

export const useGLAccounts = () => {
  return useQuery({
    queryKey: ['glAccounts'],
    queryFn: () => ApiService.getGLAccounts(),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};

// src/hooks/useCustomers.ts
export const useCustomers = (searchTerm?: string) => {
  return useQuery({
    queryKey: ['customers', searchTerm],
    queryFn: () => ApiService.getCustomers({ search: searchTerm }),
    enabled: true,
  });
};
```

---

## 📁 WEEK 4: Remaining Masters & Integration

### Day 16-17: Items/Products Master

**Backend:**
```python
# Item master with all fields
class Item(CompanyScopedModel):
    __tablename__ = 'items'
    __table_args__ = {'schema': 'masters'}

    item_code = Column(String(50), nullable=False)
    item_name = Column(String(200), nullable=False)
    category_id = Column(UUID)
    unit_of_measure = Column(String(20))
    cost_method = Column(String(20), default='FIFO')
    standard_cost = Column(Numeric(15, 4))
    selling_price = Column(Numeric(15, 4))
    tax_group_id = Column(UUID)
    gl_sales_account = Column(UUID)
    gl_cogs_account = Column(UUID)
    gl_inventory_account = Column(UUID)
```

**Frontend Update:**
```typescript
// Remove any hardcoded items
// Update ItemSelect component to use API
// Update all item-related forms
```

### Day 18: Sales Types & Payment Terms

**Quick implementation for supporting masters**

### Day 19-20: Testing & Bug Fixes

**Testing Checklist:**
- [ ] All API endpoints working
- [ ] Frontend successfully fetches all masters
- [ ] No localStorage for master data
- [ ] No hardcoded values remaining
- [ ] Proper error handling
- [ ] Loading states implemented

---

## 🔍 WEEK 5: Data Migration & Cleanup

### Day 21-22: Data Migration Scripts

**Create migration scripts for existing data:**
```python
# scripts/migrate_localStorage_to_db.py
"""
Script to migrate localStorage data to PostgreSQL
Run once during transition
"""
import json
import asyncio
from app.models import *
from app.core.database import AsyncSessionLocal

async def migrate_customers(data: dict):
    async with AsyncSessionLocal() as db:
        for customer_data in data['customers']:
            customer = Customer(
                company_id=company_id,
                customer_code=customer_data['code'],
                customer_name=customer_data['name'],
                # ... map all fields
            )
            db.add(customer)
        await db.commit()
```

### Day 23-24: Frontend Final Cleanup

**Complete removal checklist:**

```typescript
// FILES TO DELETE:
// ❌ src/data/mockData.ts
// ❌ src/data/sampleTransactions.ts
// ❌ src/utils/localStorage.ts (if only used for mock data)
// ❌ Any .json files with test data

// FUNCTIONS TO REMOVE:
// ❌ initializeLocalStorage()
// ❌ generateMockData()
// ❌ seedDatabase()

// REPLACE ALL:
// ❌ localStorage.getItem('customers')
// ✅ apiService.getCustomers()

// ❌ localStorage.setItem('glAccounts', ...)
// ✅ apiService.createGLAccount(...)
```

### Day 25: Documentation

**Create documentation for:**
1. API endpoints reference
2. Frontend integration guide
3. Data model documentation
4. Migration guide

---

## 📋 WEEK 6: Final Integration & Deployment

### Day 26-27: Integration Testing

**Test Scenarios:**
1. Create new company → Create all masters
2. Login → Fetch masters → Display in dropdowns
3. Create customer → Verify in database
4. Update master → Verify changes
5. Delete master → Verify soft delete

### Day 28-29: Performance Optimization

**Backend Optimizations:**
```python
# 1. Add database indexes
CREATE INDEX idx_customers_code ON masters.customers(company_id, customer_code);
CREATE INDEX idx_items_code ON masters.items(company_id, item_code);

# 2. Implement Redis caching
@cache(expire=300)
async def get_gl_accounts(company_id: str):
    # Cache frequently accessed masters
    pass

# 3. Add pagination
GET /api/v1/masters/customers?page=1&size=50
```

**Frontend Optimizations:**
```typescript
// 1. Implement React Query for caching
// 2. Add virtual scrolling for large lists
// 3. Lazy load master data
// 4. Add debouncing to search
```

### Day 30: Deployment

**Deployment Steps:**
1. Deploy backend to production
2. Run database migrations
3. Deploy updated frontend
4. Verify all masters loading
5. Monitor for errors

---

## ✅ Success Criteria

### Backend Success Metrics:
- [ ] All 16 master endpoints implemented
- [ ] 100% test coverage for masters
- [ ] API documentation complete
- [ ] Authentication working
- [ ] Multi-tenant support working
- [ ] Response time < 200ms for masters

### Frontend Success Metrics:
- [ ] ZERO hardcoded master data
- [ ] ZERO localStorage for masters
- [ ] All dropdowns use API data
- [ ] Proper loading states
- [ ] Error handling for API failures
- [ ] Search/filter working

---

## 🚨 Critical Files to Clean

### Priority 1 - Remove Immediately:
```
src/data/*                    # All mock data files
src/utils/mockDataGenerator.ts
src/utils/sampleData.ts
```

### Priority 2 - Refactor:
```
src/hooks/useMasterData.ts    # Convert to API
src/components/*Select.tsx    # Update to use API
src/pages/Login.tsx           # Remove demo companies
```

### Priority 3 - Update:
```
src/pages/masters/*           # Add API integration
src/context/AppContext.tsx    # Remove localStorage deps
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