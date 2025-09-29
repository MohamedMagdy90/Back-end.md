# 🔒 Separate Database Strategy for ERP Multi-Tenancy

## Executive Summary

**Decision: ALL tenants get separate databases, regardless of subscription plan.**

This document outlines the implementation strategy, benefits, and operational considerations for using separate databases for every tenant in your ERP SaaS platform.

---

## 🎯 Why Separate Databases for Everyone?

### Security Benefits
- **Complete Data Isolation**: Zero risk of data leaks between tenants
- **Compliance Ready**: Meets SOC2, GDPR, HIPAA requirements
- **Audit Trail**: Database-level audit logs per tenant
- **Access Control**: Database-level permissions per tenant

### Operational Benefits
- **Independent Backups**: Restore one tenant without affecting others
- **Performance Isolation**: One tenant's heavy queries don't impact others
- **Easy Scaling**: Scale individual tenant databases as needed
- **Simple Disaster Recovery**: Per-tenant recovery strategies

### Business Benefits
- **Premium Positioning**: "Enterprise-grade security for all"
- **Easier Sales**: Security-conscious customers prefer isolation
- **Reduced Support**: Fewer cross-tenant issues
- **Clearer Costs**: Easy to calculate per-tenant infrastructure costs

---

## 🏗️ Implementation Architecture

### Database Naming Convention
```python
def get_database_name(tenant_id: str) -> str:
    """
    Standard naming convention for tenant databases
    """
    return f"erp_{tenant_id.lower()}"

# Examples:
# erp_acme_corp
# erp_tech_startup_001
# erp_enterprise_500
```

### Connection Management
```python
# app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from typing import Dict
import asyncio

class TenantDatabaseManager:
    """
    Manages connections to multiple tenant databases
    """
    def __init__(self):
        self._engines: Dict[str, AsyncEngine] = {}
        self._lock = asyncio.Lock()

    async def get_engine(self, tenant_id: str) -> AsyncEngine:
        """
        Get or create engine for tenant database
        """
        db_name = f"erp_{tenant_id}"

        if db_name not in self._engines:
            async with self._lock:
                if db_name not in self._engines:  # Double-check
                    # Build database URL from environment variables
                    base_url = os.getenv("DATABASE_URL")
                    # Parse and modify URL to use tenant database
                    from urllib.parse import urlparse, urlunparse
                    parsed = urlparse(base_url)
                    tenant_url = urlunparse(parsed._replace(path=f"/{db_name}"))

                    self._engines[db_name] = create_async_engine(
                        tenant_url,
                        pool_size=5,  # Smaller pool per tenant
                        max_overflow=10,
                        pool_pre_ping=True,
                        pool_recycle=3600
                    )

        return self._engines[db_name]

    async def close_all(self):
        """
        Close all database connections
        """
        for engine in self._engines.values():
            await engine.dispose()

# Global instance
tenant_db_manager = TenantDatabaseManager()
```

---

## 📝 Tenant Provisioning Process

### Step-by-Step Database Creation

```python
# app/services/tenant_provisioning.py
import asyncpg
from sqlalchemy import text

class TenantProvisioning:
    """
    Handles creation of new tenant databases
    """

    async def provision_new_tenant(self, tenant_data: dict) -> dict:
        """
        Complete provisioning process for new tenant
        """
        tenant_id = tenant_data['tenant_id']
        db_name = f"erp_{tenant_id}"

        # Step 1: Create database
        await self._create_database(db_name)

        # Step 2: Create schemas
        await self._create_schemas(db_name)

        # Step 3: Run migrations
        await self._run_migrations(db_name)

        # Step 4: Insert default data
        await self._insert_defaults(db_name, tenant_data)

        # Step 5: Create first company and admin user
        await self._create_initial_company(db_name, tenant_data)

        # Step 6: Set up automated backups
        await self._configure_backups(db_name)

        return {
            "database_name": db_name,
            "status": "provisioned",
            "connection_string": f"postgresql://<configured_in_env>/{db_name}"
        }

    async def _create_database(self, db_name: str):
        """
        Create new PostgreSQL database
        """
        # Connect to postgres database to create new DB
        # Use environment variable for admin connection
        admin_db_url = os.getenv("ADMIN_DATABASE_URL")  # Should point to postgres database
        conn = await asyncpg.connect(admin_db_url)

        try:
            # Create database
            await conn.execute(f'''
                CREATE DATABASE {db_name}
                WITH
                    OWNER = erp_user
                    ENCODING = 'UTF8'
                    LC_COLLATE = 'en_US.UTF-8'
                    LC_CTYPE = 'en_US.UTF-8'
                    TABLESPACE = pg_default
                    CONNECTION LIMIT = 50;
            ''')

            # Grant permissions
            await conn.execute(f'''
                GRANT ALL PRIVILEGES ON DATABASE {db_name} TO erp_user;
            ''')

        finally:
            await conn.close()

    async def _create_schemas(self, db_name: str):
        """
        Create required schemas in tenant database
        """
        # Build connection string from environment variables
        base_url = os.getenv("DATABASE_URL")
        from urllib.parse import urlparse, urlunparse
        parsed = urlparse(base_url)
        tenant_url = urlunparse(parsed._replace(path=f"/{db_name}"))
        conn = await asyncpg.connect(tenant_url)

        try:
            schemas = ['core', 'masters', 'finance', 'inventory', 'sales', 'audit']

            for schema in schemas:
                await conn.execute(f'CREATE SCHEMA IF NOT EXISTS {schema};')

            # Enable extensions
            await conn.execute('CREATE EXTENSION IF NOT EXISTS "uuid-ossp";')
            await conn.execute('CREATE EXTENSION IF NOT EXISTS "pg_trgm";')

        finally:
            await conn.close()
```

---

## 🔄 Migration Management

### Applying Migrations to All Tenants

```python
# scripts/migrate_all_tenants.py
import asyncio
from alembic import command
from alembic.config import Config

class TenantMigrationManager:
    """
    Manages database migrations across all tenant databases
    """

    async def migrate_all_tenants(self, target_revision: str = "head"):
        """
        Apply migrations to all tenant databases
        """
        tenants = await self.get_all_active_tenants()
        results = []

        for tenant in tenants:
            db_name = f"erp_{tenant['id']}"

            try:
                # Run migration for this tenant
                await self.migrate_tenant(db_name, target_revision)
                results.append({
                    "tenant": tenant['id'],
                    "status": "success",
                    "database": db_name
                })

            except Exception as e:
                results.append({
                    "tenant": tenant['id'],
                    "status": "failed",
                    "error": str(e)
                })

                # Log but continue with other tenants
                logger.error(f"Migration failed for {db_name}: {e}")

        return results

    async def migrate_tenant(self, db_name: str, target: str):
        """
        Run Alembic migration for specific tenant database
        """
        alembic_cfg = Config("alembic.ini")

        # Override database URL for this tenant
        # Build database URL from environment variables
        base_url = os.getenv("DATABASE_URL")
        from urllib.parse import urlparse, urlunparse
        parsed = urlparse(base_url)
        tenant_url = urlunparse(parsed._replace(path=f"/{db_name}"))

        alembic_cfg.set_main_option(
            "sqlalchemy.url",
            tenant_url
        )

        # Run migration
        command.upgrade(alembic_cfg, target)

# Usage
async def main():
    manager = TenantMigrationManager()
    results = await manager.migrate_all_tenants()

    print(f"Migration complete: {len(results)} tenants processed")

    failures = [r for r in results if r['status'] == 'failed']
    if failures:
        print(f"Failed migrations: {failures}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 🔐 Security Configuration

### PostgreSQL Security per Tenant

```sql
-- For each tenant database, apply these security settings

-- 1. Revoke default public access
REVOKE ALL ON DATABASE erp_tenant_001 FROM PUBLIC;

-- 2. Create tenant-specific user (optional for extra security)
CREATE USER tenant_001_user WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE erp_tenant_001 TO tenant_001_user;
GRANT USAGE ON SCHEMA core, masters, finance TO tenant_001_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA core, masters, finance TO tenant_001_user;

-- 3. Enable SSL requirement
ALTER DATABASE erp_tenant_001 SET ssl = on;

-- 4. Set connection limits
ALTER DATABASE erp_tenant_001 CONNECTION LIMIT 50;

-- 5. Enable audit logging
ALTER DATABASE erp_tenant_001 SET log_statement = 'all';
ALTER DATABASE erp_tenant_001 SET log_connections = on;
ALTER DATABASE erp_tenant_001 SET log_disconnections = on;
```

---

## 💾 Backup Strategy

### Automated Per-Tenant Backups

```python
# app/services/backup_service.py
import asyncio
import subprocess
from datetime import datetime

class TenantBackupService:
    """
    Manages automated backups for all tenant databases
    """

    async def backup_tenant(self, tenant_id: str) -> str:
        """
        Create backup for specific tenant
        """
        db_name = f"erp_{tenant_id}"
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        backup_file = f"/backups/{tenant_id}/{db_name}_{timestamp}.sql"

        # Run pg_dump
        cmd = [
            "pg_dump",
            "-h", "localhost",
            "-U", "erp_user",
            "-d", db_name,
            "-f", backup_file,
            "--verbose",
            "--no-owner",
            "--no-acl",
            "--format=custom",
            "--compress=9"
        ]

        process = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )

        stdout, stderr = await process.communicate()

        if process.returncode != 0:
            raise Exception(f"Backup failed: {stderr.decode()}")

        # Upload to S3 or cloud storage
        await self._upload_to_cloud(backup_file, tenant_id)

        return backup_file

    async def restore_tenant(self, tenant_id: str, backup_file: str):
        """
        Restore tenant database from backup
        """
        db_name = f"erp_{tenant_id}"

        # Drop existing database
        await self._drop_database(db_name)

        # Create fresh database
        await self._create_database(db_name)

        # Restore from backup
        cmd = [
            "pg_restore",
            "-h", "localhost",
            "-U", "erp_user",
            "-d", db_name,
            backup_file,
            "--verbose",
            "--no-owner",
            "--no-acl"
        ]

        process = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )

        stdout, stderr = await process.communicate()

        if process.returncode != 0:
            raise Exception(f"Restore failed: {stderr.decode()}")

        return f"Database {db_name} restored successfully"

# Cron job for daily backups
# 0 2 * * * python -c "from backup_service import backup_all_tenants; backup_all_tenants()"
```

---

## 💰 Cost Considerations

### Database Cost Estimation

```python
def estimate_monthly_cost(num_tenants: int, avg_db_size_gb: float) -> dict:
    """
    Estimate monthly infrastructure costs for separate databases

    Based on AWS RDS PostgreSQL pricing (us-east-1)
    """
    # Storage cost: $0.115 per GB-month
    storage_cost = num_tenants * avg_db_size_gb * 0.115

    # Instance cost: Assuming db.t3.micro for small tenants
    # $0.018 per hour = ~$13 per month per instance
    # With connection pooling, estimate 1 instance per 50 tenants
    instances_needed = (num_tenants // 50) + 1
    instance_cost = instances_needed * 13 * 24 * 30 * 0.018

    # Backup storage: $0.095 per GB-month
    backup_cost = num_tenants * avg_db_size_gb * 0.095

    # Data transfer: Estimate 10GB per tenant per month at $0.09/GB
    transfer_cost = num_tenants * 10 * 0.09

    total_cost = storage_cost + instance_cost + backup_cost + transfer_cost

    return {
        "storage_cost": round(storage_cost, 2),
        "instance_cost": round(instance_cost, 2),
        "backup_cost": round(backup_cost, 2),
        "transfer_cost": round(transfer_cost, 2),
        "total_monthly_cost": round(total_cost, 2),
        "cost_per_tenant": round(total_cost / num_tenants, 2)
    }

# Example: 100 tenants with 1GB average database size
cost = estimate_monthly_cost(100, 1.0)
print(f"Monthly cost for 100 tenants: ${cost['total_monthly_cost']}")
print(f"Cost per tenant: ${cost['cost_per_tenant']}")
```

---

## ✅ Implementation Checklist

### Phase 1: Setup (Week 1)
- [ ] Define database naming convention
- [ ] Create provisioning scripts
- [ ] Set up connection pooling
- [ ] Create migration management tools
- [ ] Design backup strategy

### Phase 2: Automation (Week 2)
- [ ] Automate database creation
- [ ] Automate schema creation
- [ ] Automate initial data seeding
- [ ] Automate backup scheduling
- [ ] Create monitoring dashboard

### Phase 3: Operations (Week 3)
- [ ] Document runbooks
- [ ] Set up alerts
- [ ] Create tenant onboarding flow
- [ ] Test disaster recovery
- [ ] Performance testing

### Phase 4: Optimization (Week 4)
- [ ] Optimize connection pooling
- [ ] Implement query optimization
- [ ] Set up read replicas (if needed)
- [ ] Cost optimization review
- [ ] Security audit

---

## 🎯 Best Practices

1. **Naming Convention**: Use consistent, meaningful database names
2. **Connection Management**: Use connection pooling wisely
3. **Migration Strategy**: Test migrations on staging first
4. **Backup Policy**: Daily backups with 30-day retention
5. **Monitoring**: Alert on database size, connections, and slow queries
6. **Security**: Use SSL, separate users per tenant if needed
7. **Documentation**: Keep detailed records of each tenant's database
8. **Disaster Recovery**: Test restore procedures regularly
9. **Cost Management**: Monitor and optimize unused databases
10. **Compliance**: Maintain audit logs for all database operations

---

## 📚 Conclusion

Using separate databases for all tenants provides:
- **Maximum security** through complete isolation
- **Operational flexibility** for scaling and maintenance
- **Compliance readiness** for regulations
- **Premium positioning** in the market

While the operational overhead is higher than shared schemas, the benefits for an ERP system handling financial data far outweigh the costs.

---

**Document Version**: 1.0
**Last Updated**: September 2025
**Status**: Ready for Implementation