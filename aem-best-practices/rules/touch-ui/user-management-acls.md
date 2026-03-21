---
title: User Management, Permissions, and ACLs
impact: HIGH
impactDescription: Incorrect permission patterns cause security vulnerabilities, broken authoring workflows, and data exposure — IMS integration on Cloud Service changes everything from on-premise
tags: permissions, acls, ims, admin-console, repo-init, service-users, cug, principal-access-control, product-profiles
---

## User Management, Permissions, and ACLs

AEM as a Cloud Service uses Adobe IMS (Identity Management System) for authentication. Users are managed in Adobe Admin Console, not in AEM. Permissions are configured via Repo Init scripts, content policies, and product profiles.

---

### 1. IMS Integration (Cloud Service)

```
Adobe Admin Console                    AEM as a Cloud Service
├── Organization                       ├── Author instance
│   ├── Product Profile: AEM Authors   │   ├── IMS users synced
│   │   ├── user1@company.com          │   ├── IMS groups synced
│   │   └── user2@company.com          │   └── rep:policy nodes (ACLs)
│   ├── Product Profile: AEM Admins    │
│   │   └── admin@company.com          └── Publish instance
│   └── Product Profile: AEM Developers    ├── Anonymous access (public)
│       └── dev@company.com                └── CUG for gated content
```

**Key difference from on-premise:**
- No local AEM user creation (all via Admin Console)
- No Classic UI user admin
- Groups synced from IMS product profiles
- Service users still managed via Repo Init

#### Default Product Profiles

| Profile | AEM Group | Access |
|---------|-----------|--------|
| AEM Administrators | administrators | Full admin access |
| AEM Users | content-authors | Author content |
| AEM Developers | developers | CRXDE, packages, OSGi console |

---

### 2. Repo Init for Permission Management

Repo Init is the primary way to configure permissions as code on Cloud Service:

#### Service User Creation

```
# org.apache.sling.jcr.repoinit.RepositoryInitializer-myproject.cfg.json
{
    "scripts": [
        "create service user myproject-content-reader with path system/myproject\ncreate service user myproject-content-writer with path system/myproject\ncreate service user myproject-workflow-user with path system/myproject"
    ]
}
```

#### ACL Configuration

```
# Set ACLs for service users
set ACL for myproject-content-reader
    allow jcr:read on /content/mysite
    allow jcr:read on /content/dam/mysite
    allow jcr:read on /conf/mysite
end

set ACL for myproject-content-writer
    allow jcr:read,rep:write on /content/mysite
    allow jcr:read on /content/dam/mysite
    allow jcr:read on /conf/mysite
end

# Deny access to sensitive content
set ACL for myproject-content-reader
    deny jcr:read on /content/mysite/internal
end
```

#### Group Creation and Permissions

```
# Create custom groups
create group myproject-authors with path /home/groups/myproject
create group myproject-approvers with path /home/groups/myproject
create group myproject-dam-users with path /home/groups/myproject

# Set ACLs for groups
set ACL for myproject-authors
    allow jcr:read,jcr:modifyProperties,jcr:addChildNodes,jcr:removeChildNodes on /content/mysite
    allow jcr:read on /content/dam/mysite
    allow jcr:read on /conf/mysite
    deny jcr:read on /content/mysite/admin-only
end

set ACL for myproject-approvers
    allow jcr:read,rep:write on /content/mysite
    allow jcr:read,rep:write on /content/dam/mysite
    allow jcr:read on /conf/mysite
end

set ACL for myproject-dam-users
    allow jcr:read,rep:write on /content/dam/mysite
    allow jcr:read on /content/mysite
end

# Add group membership
add myproject-authors to group content-authors
add myproject-approvers to group content-authors
```

---

### 3. ACL Structure (rep:policy)

#### JCR Permission Model

```
/content/mysite
  └── rep:policy
        ├── allow (principal: myproject-authors)
        │     privileges: jcr:read, jcr:modifyProperties
        ├── allow (principal: myproject-approvers)
        │     privileges: jcr:all
        └── deny (principal: myproject-authors)
              privileges: jcr:removeNode
              restrictions: rep:glob=/*/jcr:content/locked-section
```

#### JCR Privileges Reference

| Privilege | Description |
|-----------|-------------|
| `jcr:read` | Read nodes and properties |
| `jcr:modifyProperties` | Set, modify, remove properties |
| `jcr:addChildNodes` | Add child nodes |
| `jcr:removeChildNodes` | Remove child nodes |
| `jcr:removeNode` | Remove the node itself |
| `rep:write` | Aggregate: modifyProperties + addChildNodes + removeChildNodes + removeNode |
| `jcr:all` | All privileges |
| `crx:replicate` | Replicate (publish) content |
| `jcr:lockManagement` | Lock/unlock nodes |
| `jcr:versionManagement` | Create/restore versions |

---

### 4. Principal-Based Access Control

Oak supports restrictions to narrow ACL scope:

```
# rep:glob — path pattern restriction
set ACL for myproject-authors
    allow jcr:read on /content/mysite restriction(rep:glob,/*/jcr:content)
    # Only read jcr:content nodes, not all descendants
end

# rep:ntNames — restrict to specific node types
set ACL for myproject-authors
    allow jcr:read on /content/mysite restriction(rep:ntNames,cq:Page,cq:PageContent)
end

# rep:itemNames — restrict to specific properties
set ACL for myproject-authors
    allow jcr:modifyProperties on /content/mysite restriction(rep:itemNames,jcr:title,jcr:description,cq:tags)
    # Can only modify these specific properties
end

# rep:prefixes — restrict to namespace prefixes
set ACL for myproject-authors
    deny jcr:modifyProperties on /content/mysite restriction(rep:prefixes,rep,cq)
    # Cannot modify rep: or cq: prefixed properties
end
```

---

### 5. Closed User Groups (CUG)

Restrict read access on publish for gated content:

#### CUG Configuration

```xml
<!-- /content/mysite/members-only/jcr:content -->
<jcr:content
    jcr:primaryType="cq:PageContent"
    cq:cugEnabled="{Boolean}true"
    cq:cugPrincipals="[members]"
    cq:cugLoginPage="/content/mysite/login"/>
```

#### Repo Init for CUG

```
# Create CUG-related group
create group mysite-members with path /home/groups/mysite

# Set CUG policy
set CUG for /content/mysite/members-only
    principals: mysite-members
end
```

#### Login Page Configuration

```java
// Authentication handler for CUG login
// com.adobe.granite.auth.ims.impl.IMSProviderImpl handles IMS auth
// For custom login, configure Sling Authenticator:

// org.apache.sling.engine.impl.auth.SlingAuthenticator.cfg.json
{
    "auth.requirements": [
        "+/content/mysite/members-only"
    ]
}
```

---

### 6. Service Users (System Operations)

#### Service User Mapping

```json
// org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-myproject.cfg.json
{
    "user.mapping": [
        "com.myproject.core:content-reader=myproject-content-reader",
        "com.myproject.core:content-writer=myproject-content-writer",
        "com.myproject.core:workflow-service=myproject-workflow-user",
        "com.myproject.core:dam-service=myproject-dam-user"
    ]
}
```

#### Usage in Code

```java
@Reference
private ResourceResolverFactory resolverFactory;

// Each subservice gets its own resolver with specific permissions
private static final Map<String, Object> READER = Map.of(
    ResourceResolverFactory.SUBSERVICE, "content-reader");

private static final Map<String, Object> WRITER = Map.of(
    ResourceResolverFactory.SUBSERVICE, "content-writer");

public List<Page> readContent() {
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(READER)) {
        return findPages(resolver);
    } catch (LoginException e) {
        log.error("content-reader service user login failed", e);
        return Collections.emptyList();
    }
}

public void updateContent(String path, Map<String, Object> properties) {
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(WRITER)) {
        Resource resource = resolver.getResource(path);
        if (resource != null) {
            ModifiableValueMap props = resource.adaptTo(ModifiableValueMap.class);
            props.putAll(properties);
            resolver.commit();
        }
    } catch (LoginException | PersistenceException e) {
        log.error("Failed to update content at: {}", path, e);
    }
}
```

---

### 7. Token-Based Authentication

#### IMS Service Credentials

```java
// For server-to-server API calls (Cloud Service)
// Use IMS service credentials from Developer Console

// com.adobe.granite.auth.ims.impl.IMSAccessTokenRequestCustomizerImpl.cfg.json
{
    "ims.token.endpoint": "https://ims-na1.adobelogin.com/ims/token/v3",
    "client.id": "$[secret:IMS_CLIENT_ID]",
    "client.secret": "$[secret:IMS_CLIENT_SECRET]",
    "technical.account.id": "$[env:IMS_TECHNICAL_ACCOUNT_ID]",
    "org.id": "$[env:IMS_ORG_ID]",
    "meta.scopes": "ent_aem_cloud_api"
}
```

#### Local Development Token

```bash
# Generate local dev token via Developer Console
# Token valid for 24 hours (dev only, never production)
curl -X GET "https://author-p12345-e67890.adobeaemcloud.com/content/mysite.json" \
  -H "Authorization: Bearer eyJhbGci..."
```

---

### 8. Debugging Permissions

#### Access Control Editor

```
CRXDE Lite → Navigate to node → Access Control tab
  Shows effective ACLs (inherited + direct)
  Evaluate permissions for specific principal

URL: /crx/de/index.jsp#/content/mysite
```

#### Query-Based Permission Check

```java
// Programmatic permission check
Session session = resolver.adaptTo(Session.class);
AccessControlManager acm = session.getAccessControlManager();

Privilege[] privileges = new Privilege[]{
    acm.privilegeFromName(Privilege.JCR_READ),
    acm.privilegeFromName(Privilege.JCR_MODIFY_PROPERTIES)
};

boolean hasAccess = acm.hasPrivileges("/content/mysite/en", privileges);
log.info("User {} has read+write on /content/mysite/en: {}",
    session.getUserID(), hasAccess);
```

#### Permission Audit Logging

```json
// Enable audit logging for permission troubleshooting
// org.apache.sling.commons.log.LogManager.factory.config~acl-debug.cfg.json
{
    "org.apache.sling.commons.log.names": [
        "org.apache.jackrabbit.oak.security.authorization"
    ],
    "org.apache.sling.commons.log.level": "DEBUG",
    "org.apache.sling.commons.log.file": "logs/acl-debug.log"
}
```

---

### 9. Anti-Patterns

#### Using Admin Session

```java
// WRONG — removed in Cloud Service
ResourceResolver resolver = resolverFactory.getAdministrativeResourceResolver(null);
Session session = repository.loginAdministrative(null);

// CORRECT — use service users with least privilege
try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(READER)) { }
```

#### Overly Broad Permissions

```
// WRONG — giving content authors jcr:all on /content
set ACL for content-authors
    allow jcr:all on /content
end

// CORRECT — least privilege, specific paths
set ACL for myproject-authors
    allow jcr:read,jcr:modifyProperties,jcr:addChildNodes on /content/mysite
    allow jcr:read on /content/dam/mysite
    deny jcr:removeNode on /content/mysite/critical-pages
end
```

#### Hardcoding User Checks

```java
// WRONG — checking specific user names
if ("admin".equals(resolver.getUserID())) {
    // allow operation
}

// CORRECT — check permissions, not users
Session session = resolver.adaptTo(Session.class);
if (session.hasPermission("/content/mysite", Session.ACTION_SET_PROPERTY)) {
    // allow operation
}
```

#### Managing Users in AEM (Cloud Service)

```
// WRONG — creating users via CRXDE or UserManager on Cloud Service
// These users won't persist across deployments

// CORRECT — manage users in Adobe Admin Console
// CORRECT — manage groups via Repo Init scripts
// CORRECT — manage service users via Repo Init scripts
```
