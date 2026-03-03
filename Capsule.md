How do i safely give multiple team access to the same cluster without giving them rights of cluster administrator? <br>
Here, **Capsule** comes into the picture.

Capsule introduces new level of abstraction, called **Tenant**. <br>

**Tenant can:**
- Own multiple namespaces
- Limited cluster-wide permissions
- Quota restrictions
- Allowed storage classes
- allowed ingress classes

Instead of managing raw RBAC manually for each namespace you manage, you can use capsule.
```
tenant -> Namespace -> Policies
```
Much cleaner.

**Capsule controls:**
- Resource Quota
- Network policies
- PSS
- Allowed container registry
- Storage class

### Soft vs Hard Multi-tenancy:

1. **Hard Multi-tenancy**: Seperate cluster per team
2. **Soft Multi-tenancy**: `Capsule` healpps to achive this, single cluster, strong logical isolation.
