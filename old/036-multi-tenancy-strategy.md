# ADR-036: Multi-Tenancy Strategy

## Status
Accepted

## Context

The Ameide platform needs to support multiple tenants (organizations) while ensuring:
- Complete data isolation between tenants
- Scalability to thousands of tenants
- Cost-effective resource utilization
- Simple operational management
- Flexibility to adapt to different deployment scenarios

We considered three main approaches:
1. **Separate Databases**: Each tenant gets their own database
2. **Separate Schemas**: Each tenant gets their own schema within a shared database
3. **Shared Schema with Row-Level Security**: All tenants share tables with tenant_id column

## Decision

We will implement **Shared Schema with Row-Level Security** as the primary multi-tenancy strategy, with the flexibility to migrate specific tenants to separate schemas or databases when needed.

### Implementation Details

1. **Database Schema**
   - Add `tenant_id` column to all tables
   - Create composite indexes on (tenant_id, primary_key) for performance
   - Enforce tenant consistency with foreign key constraints

2. **Application Layer**
   - Extract tenant_id from JWT tokens (custom claim)
   - Use TenantRepository to automatically filter all queries
   - Prevent cross-tenant data access at API layer

3. **Authentication Integration**
   - Support both Auth0 and Keycloak identity providers
   - Store tenant_id as custom claim in tokens
   - Automatic user provisioning on first login

4. **Security Measures**
   - Tenant validation in authentication layer
   - Automatic tenant filtering in repositories
   - Optional database-level Row Level Security (RLS)
   - Audit logging for cross-tenant operations

## Consequences

### Positive
- **Simple Operations**: Single database to backup, monitor, and maintain
- **Cost Effective**: Shared resources reduce infrastructure costs
- **Easy Deployment**: No per-tenant provisioning required
- **Cross-Tenant Features**: Easier to implement platform-wide features
- **Performance**: Efficient resource utilization with proper indexing

### Negative
- **Noisy Neighbor**: One tenant could impact others' performance
- **Compliance Challenges**: Some regulations may require physical separation
- **Migration Complexity**: Moving to separate databases requires data migration
- **Query Complexity**: Must always include tenant_id in queries

### Mitigation Strategies

1. **Performance Isolation**
   - Implement query timeouts and resource limits
   - Monitor per-tenant resource usage
   - Use read replicas for heavy tenants

2. **Compliance Requirements**
   - Design for tenant migration from day one
   - Support schema-per-tenant for specific cases
   - Implement data residency controls

3. **Security Hardening**
   - Regular security audits of tenant isolation
   - Automated testing of cross-tenant access
   - Database-level RLS as additional protection

## Migration Path

### Phase 1: Current Implementation
- Shared tables with tenant_id
- Application-level filtering
- Basic tenant management

### Phase 2: Enhanced Security (Future)
- Enable database-level RLS
- Implement tenant resource quotas
- Add tenant-specific encryption keys

### Phase 3: Hybrid Model (Future)
- Support schema-per-tenant for enterprise
- Automated tenant migration tools
- Multi-region tenant placement

## Code Examples

### Repository Usage
```python
# Automatically filtered by tenant
async def get_user_workflows(tenant_id: str, user_id: str):
    repo = TenantRepository(session, Workflow, tenant_id)
    return await repo.find({"owner_id": user_id})
```

### Security Context
```python
@router.get("/workflows")
async def list_workflows(
    security_context: SecurityContext = Depends(get_security_context)
):
    # tenant_id automatically extracted from token
    tenant_id = security_context.tenant_id
    # All queries filtered by tenant
```

### Database Schema
```sql
CREATE TABLE workflows (
    id VARCHAR(255) PRIMARY KEY,
    tenant_id VARCHAR(255) NOT NULL,
    owner_id VARCHAR(255) NOT NULL,
    -- other fields
    INDEX ix_workflows_tenant_id (tenant_id),
    INDEX ix_workflows_tenant_id_owner_id (tenant_id, owner_id)
);
```

## References
- [Designing Your SaaS Database for Scale](https://www.citusdata.com/blog/2016/10/03/designing-your-saas-database-for-scale/)
- [Multi-tenant SaaS patterns](https://docs.microsoft.com/en-us/azure/sql-database/saas-tenancy-app-design-patterns)
- [Row Level Security in PostgreSQL](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)