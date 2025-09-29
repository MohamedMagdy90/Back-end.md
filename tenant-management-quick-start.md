# 🚀 Tenant Management Platform - Quick Start Guide

## 🎯 Executive Summary

As an ERP operator, you need a platform to:
- **Create & manage multiple tenants** (your clients)
- **Control limits** (companies, users, transactions)
- **Handle billing** (subscriptions, trials, upgrades)
- **Ensure isolation** (data security between tenants)
- **Monitor usage** (for billing and performance)

---

## 📊 Recommended Architecture for ERP SaaS

### Database Strategy - Separate Database for ALL

| Customer Type | Annual Revenue | Database Strategy | Why |
|--------------|---------------|-------------------|-----|
| **Trial/Demo** | $0 | Separate Database | Full isolation even for trials |
| **Small Business** | < $10K | Separate Database | Complete data security |
| **Mid-Market** | $10K - $100K | Separate Database | Compliance ready |
| **Enterprise** | > $100K | Separate Database + Dedicated Resources | Maximum performance |

**Why Separate Database for Everyone?**
- ✅ **No data leak risk** - Complete isolation between tenants
- ✅ **Easier compliance** - Each tenant's data is physically separated
- ✅ **Independent backups** - Restore one tenant without affecting others
- ✅ **Better performance** - No noisy neighbor problems
- ✅ **Simpler security model** - Database-level access control

### Subscription Tiers Example

```javascript
const subscriptionPlans = {
  trial: {
    price: 0,
    duration: "14 days",
    limits: {
      companies: 1,
      usersPerCompany: 3,
      transactionsPerMonth: 100
    },
    database: "separate_database"  // Full isolation for all tenants
  },

  starter: {
    price: 49,
    duration: "monthly",
    limits: {
      companies: 1,
      usersPerCompany: 5,
      transactionsPerMonth: 1000
    },
    database: "separate_database"  // Full isolation for all tenants
  },

  professional: {
    price: 199,
    duration: "monthly",
    limits: {
      companies: 3,
      usersPerCompany: 20,
      transactionsPerMonth: 10000
    },
    database: "separate_database"  // Full isolation for all tenants
  },

  enterprise: {
    price: "custom",
    duration: "annual",
    limits: {
      companies: "unlimited",
      usersPerCompany: "unlimited",
      transactionsPerMonth: "unlimited"
    },
    database: "separate_database"
  }
};
```

---

## 🔨 Day 1: Essential Database Tables

```sql
-- Minimum viable tenant management tables

-- 1. Tenants table (your clients)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_code VARCHAR(20) UNIQUE NOT NULL,  -- e.g., "ACME001"
    company_name VARCHAR(200) NOT NULL,
    subdomain VARCHAR(50) UNIQUE NOT NULL,    -- acme.yourerp.com

    -- Subscription
    plan VARCHAR(50) DEFAULT 'trial',
    status VARCHAR(20) DEFAULT 'active',      -- active, suspended, cancelled
    trial_ends_at DATE,

    -- Limits (from plan)
    max_companies INT DEFAULT 1,
    max_users_per_company INT DEFAULT 5,

    -- Database config
    database_name VARCHAR(100),               -- null for shared

    created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Quick tracking table
CREATE TABLE tenant_usage (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id),
    date DATE NOT NULL,
    companies_count INT DEFAULT 0,
    users_count INT DEFAULT 0,
    transactions_count INT DEFAULT 0,
    UNIQUE(tenant_id, date)
);
```

---

## 💻 Day 2: Core API Endpoints

```python
# main.py - Minimal tenant management API

from fastapi import FastAPI, HTTPException, Depends
from datetime import datetime, timedelta
import asyncpg

app = FastAPI(title="ERP Tenant Management")

# 1. Create tenant
@app.post("/api/tenants")
async def create_tenant(
    company_name: str,
    subdomain: str,
    plan: str = "trial"
):
    # Generate tenant code
    tenant_code = f"{company_name[:3].upper()}{datetime.now().strftime('%Y%m%d%H%M%S')}"

    # Set limits based on plan
    limits = {
        "trial": {"companies": 1, "users": 3},
        "starter": {"companies": 1, "users": 5},
        "professional": {"companies": 3, "users": 20}
    }

    # Create tenant record
    tenant = {
        "tenant_code": tenant_code,
        "company_name": company_name,
        "subdomain": subdomain,
        "plan": plan,
        "max_companies": limits[plan]["companies"],
        "max_users_per_company": limits[plan]["users"],
        "trial_ends_at": datetime.now() + timedelta(days=14) if plan == "trial" else None
    }

    # TODO: Insert into database
    # TODO: Create tenant database/schema
    # TODO: Send welcome email

    return {"message": "Tenant created", "tenant": tenant}

# 2. Check limits before creating resources
@app.post("/api/tenants/{tenant_id}/check-limit")
async def check_limit(
    tenant_id: str,
    resource_type: str  # "company" or "user"
):
    # Get tenant limits
    # TODO: Fetch from database
    tenant = {"max_companies": 3, "current_companies": 2}

    if resource_type == "company":
        can_create = tenant["current_companies"] < tenant["max_companies"]
        return {
            "can_create": can_create,
            "current": tenant["current_companies"],
            "limit": tenant["max_companies"],
            "message": "Company limit reached" if not can_create else "OK"
        }

# 3. List all tenants (admin view)
@app.get("/api/tenants")
async def list_tenants(
    status: str = None,
    plan: str = None
):
    # TODO: Fetch from database with filters
    return {
        "tenants": [
            {
                "id": "123",
                "company_name": "Acme Corp",
                "plan": "professional",
                "status": "active",
                "users_count": 15,
                "companies_count": 2
            }
        ]
    }

# 4. Suspend/activate tenant
@app.put("/api/tenants/{tenant_id}/status")
async def update_tenant_status(
    tenant_id: str,
    status: str,
    reason: str = None
):
    if status not in ["active", "suspended", "cancelled"]:
        raise HTTPException(400, "Invalid status")

    # TODO: Update database
    # TODO: Revoke sessions if suspended
    # TODO: Send notification

    return {"message": f"Tenant {status}", "reason": reason}
```

---

## 🔐 Day 3: Enforcement Middleware

```python
# middleware.py - Enforce limits automatically

from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse

class TenantLimitMiddleware:
    """
    Automatically enforce tenant limits
    """

    async def __call__(self, request: Request, call_next):
        # Extract tenant from subdomain or header
        tenant_id = request.headers.get("X-Tenant-ID")

        if tenant_id:
            # Check if tenant is active
            tenant = await self.get_tenant(tenant_id)

            if tenant["status"] == "suspended":
                return JSONResponse(
                    status_code=403,
                    content={"error": "Account suspended. Please contact support."}
                )

            # For POST requests, check limits
            if request.method == "POST":
                if "/companies" in str(request.url):
                    if not await self.can_create_company(tenant_id):
                        return JSONResponse(
                            status_code=403,
                            content={"error": "Company limit reached. Please upgrade."}
                        )

                elif "/users" in str(request.url):
                    company_id = request.path_params.get("company_id")
                    if not await self.can_create_user(tenant_id, company_id):
                        return JSONResponse(
                            status_code=403,
                            content={"error": "User limit reached. Please upgrade."}
                        )

        response = await call_next(request)
        return response

# Add to main.py
app.add_middleware(TenantLimitMiddleware)
```

---

## 🖥️ Day 4: Admin Portal Components

```typescript
// TenantManagement.tsx - Simple admin interface

import React, { useState, useEffect } from 'react';
import { Table, Button, Tag, Modal, Form, Input, Select } from 'antd';

const TenantManagement = () => {
  const [tenants, setTenants] = useState([]);
  const [createModalOpen, setCreateModalOpen] = useState(false);

  // Table columns
  const columns = [
    {
      title: 'Company',
      dataIndex: 'company_name',
      key: 'company_name',
    },
    {
      title: 'Subdomain',
      dataIndex: 'subdomain',
      render: (subdomain) => `${subdomain}.yourerp.com`,
    },
    {
      title: 'Plan',
      dataIndex: 'plan',
      render: (plan) => {
        const colors = {
          trial: 'orange',
          starter: 'blue',
          professional: 'green',
          enterprise: 'purple',
        };
        return <Tag color={colors[plan]}>{plan.toUpperCase()}</Tag>;
      },
    },
    {
      title: 'Status',
      dataIndex: 'status',
      render: (status) => (
        <Tag color={status === 'active' ? 'green' : 'red'}>
          {status.toUpperCase()}
        </Tag>
      ),
    },
    {
      title: 'Usage',
      render: (_, record) => (
        <div>
          Companies: {record.companies_count}/{record.max_companies}<br />
          Users: {record.users_count}/{record.max_users}
        </div>
      ),
    },
    {
      title: 'Actions',
      render: (_, record) => (
        <div>
          <Button size="small" onClick={() => viewTenant(record.id)}>
            View
          </Button>
          <Button
            size="small"
            danger={record.status === 'active'}
            onClick={() => toggleStatus(record)}
          >
            {record.status === 'active' ? 'Suspend' : 'Activate'}
          </Button>
        </div>
      ),
    },
  ];

  // Create tenant form
  const CreateTenantForm = () => (
    <Form onFinish={handleCreateTenant}>
      <Form.Item name="company_name" label="Company Name" required>
        <Input />
      </Form.Item>
      <Form.Item name="subdomain" label="Subdomain" required>
        <Input addonAfter=".yourerp.com" />
      </Form.Item>
      <Form.Item name="plan" label="Plan" required>
        <Select>
          <Select.Option value="trial">Trial (14 days free)</Select.Option>
          <Select.Option value="starter">Starter ($49/mo)</Select.Option>
          <Select.Option value="professional">Professional ($199/mo)</Select.Option>
        </Select>
      </Form.Item>
      <Form.Item name="admin_email" label="Admin Email" required>
        <Input type="email" />
      </Form.Item>
      <Button type="primary" htmlType="submit">
        Create Tenant
      </Button>
    </Form>
  );

  return (
    <div>
      <h1>Tenant Management</h1>

      <Button type="primary" onClick={() => setCreateModalOpen(true)}>
        Create New Tenant
      </Button>

      <Table columns={columns} dataSource={tenants} />

      <Modal
        title="Create New Tenant"
        open={createModalOpen}
        onCancel={() => setCreateModalOpen(false)}
        footer={null}
      >
        <CreateTenantForm />
      </Modal>
    </div>
  );
};
```

---

## 🎯 Critical Success Factors

### 1. **Start Simple**
Begin with:
- Basic tenant creation
- Simple limit checking
- Manual billing

Then add:
- Automated provisioning
- Usage tracking
- Stripe integration

### 2. **Database Isolation Strategy**

```python
# Always use separate database for maximum ERP security:

def get_database_strategy(tenant):
    """
    All tenants get separate databases
    Critical for ERP systems handling financial data
    """
    return "separate_database"  # Always separate for ERP

    # Benefits:
    # - Complete data isolation
    # - Independent backup/restore
    # - No risk of data leaks
    # - Easier compliance (SOC2, GDPR, etc.)
    # - Better performance isolation
    # - Simpler security model
```

### 3. **Limit Enforcement Points**

```javascript
// Where to check limits:
const limitCheckPoints = {
  beforeCreating: {
    company: checkCompanyLimit,
    user: checkUserLimit,
    transaction: checkTransactionLimit
  },

  dailyChecks: {
    storage: checkStorageUsage,
    api_calls: checkAPIUsage
  },

  monthlyBilling: {
    calculate_overages: calculateOverageCharges,
    send_invoices: generateInvoices
  }
};
```

### 4. **Trial to Paid Conversion**

```python
# Automate trial conversion
async def handle_trial_expiry(tenant_id: str):
    tenant = await get_tenant(tenant_id)

    if tenant["has_payment_method"]:
        # Auto-convert to paid
        await convert_to_paid(tenant_id)
        await send_email("Welcome to Starter Plan!")
    else:
        # Give grace period
        await extend_trial(tenant_id, days=7)
        await send_email("Trial ending - Add payment method")

        # After grace period, suspend
        if still_no_payment:
            await suspend_tenant(tenant_id)
```

---

## 🚦 Go-Live Checklist

### Week 1: Foundation
- [ ] Create platform database
- [ ] Build tenant CRUD API
- [ ] Set up admin authentication
- [ ] Create basic admin UI

### Week 2: Limits & Usage
- [ ] Implement limit checking
- [ ] Add usage tracking
- [ ] Create middleware for enforcement
- [ ] Build usage dashboard

### Week 3: Billing
- [ ] Integrate Stripe/payment gateway
- [ ] Create subscription management
- [ ] Add invoice generation
- [ ] Handle upgrades/downgrades

### Week 4: Production
- [ ] Set up monitoring
- [ ] Add backup strategy
- [ ] Create onboarding flow
- [ ] Deploy to production

---

## 🎓 Key Decisions You Need to Make

1. **Database Strategy** ✅ DECIDED
   - **Separate database for ALL tenants** (Best security for ERP)
   - Each tenant gets their own isolated database
   - No data leak risk between tenants
   - Compliance-ready from day one

2. **Pricing Model**
   - Per user pricing?
   - Per company pricing?
   - Transaction-based?
   - Flat rate with limits?

3. **Trial Strategy**
   - How long? (7, 14, or 30 days)
   - Full features or limited?
   - Credit card required upfront?

4. **Support Levels**
   - Email only for Starter?
   - Priority support for Professional?
   - Dedicated support for Enterprise?

---

## 💡 Pro Tips from ERP Veterans

1. **Start with higher prices** - It's easier to lower than raise
2. **Grandfather existing customers** when changing plans
3. **Soft limits for good customers** - Warn before suspending
4. **Export functionality** - Prevent vendor lock-in concerns
5. **Transparent usage dashboard** - Reduces support tickets
6. **Automatic backups** - Before any suspension/deletion
7. **Grace periods** - For payment failures
8. **Tiered support** - Higher plans get faster response

---

## 📞 Next Steps

1. **Review the full documentation** in `tenant-management-platform-documentation.md`
2. **Choose your database strategy** based on your market
3. **Set up the basic tables** from this guide
4. **Implement the core API** endpoints
5. **Add limit enforcement** middleware
6. **Build simple admin UI** for tenant management
7. **Test with 2-3 pilot customers** before scaling

Remember: Start simple, iterate based on customer feedback!

---

**Ready to build?** This foundation will handle your first 100 customers. Scale up as you grow! 🚀