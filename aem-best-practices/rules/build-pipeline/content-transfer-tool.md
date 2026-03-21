---
title: Content Transfer Tool (CTT) for AEM Migration
impact: HIGH
impactDescription: CTT is the only supported path for migrating content to AEM as a Cloud Service — incorrect usage causes data loss, failed migrations, and broken references
tags: content-transfer-tool, migration, cloud-service, extraction, ingestion, cam, bpa, user-mapping
---

## Content Transfer Tool (CTT) for AEM Migration

The Content Transfer Tool (CTT) extracts content from a source AEM instance (6.3+, 6.5) and ingests it into AEM as a Cloud Service. It handles JCR content, users/groups mapping, and large-scale data transfer via Azure Blob Storage.

---

### 1. Migration Architecture

```
Source AEM (6.5)                          AEM as a Cloud Service
┌──────────────┐     Azure Blob Store     ┌──────────────────────┐
│              │  ──── Extraction ────►    │                      │
│  /content    │                           │  /content            │
│  /conf       │     Migration Set         │  /conf               │
│  /etc (parts)│  ◄──── Ingestion ────     │  /etc (mapped)       │
│              │                           │                      │
└──────────────┘                           └──────────────────────┘
                    ▲
                    │ Cloud Acceleration Manager (CAM)
                    │ orchestrates the entire process
```

**Phases:**
1. **Best Practice Analyzer (BPA)** — assess source instance readiness
2. **Content Transformation** — fix known incompatibilities
3. **Extraction** — pull content from source to Azure blob storage
4. **Ingestion** — push content from blob storage to Cloud Service

---

### 2. Prerequisites and Planning

#### Source Requirements

| Source Version | Supported | Notes |
|---------------|-----------|-------|
| AEM 6.3 | Yes (with SP3+) | Oldest supported |
| AEM 6.4 | Yes | Recommended: latest SP |
| AEM 6.5 | Yes | Recommended: latest SP |
| AEM as a Cloud Service | Yes | Environment-to-environment |

#### Pre-Migration Checklist

```
□ Run Best Practice Analyzer (BPA) on source
□ Review and resolve BPA findings
□ Identify content paths to migrate
□ Plan user/group mapping strategy
□ Estimate content size (GB/TB)
□ Identify custom node types and namespaces
□ Test extraction on a subset first
□ Plan maintenance window for final cutover
□ Coordinate with Adobe Support for large migrations (>500GB)
```

---

### 3. Best Practice Analyzer (BPA)

Run BPA **before** starting CTT to identify migration blockers:

```
Access: AEM > Tools > Operations > Best Practices Analyzer

Key findings to address:
├── Custom node types → May need registration in Cloud Service
├── Workflow models → ECMA scripts not supported
├── Deprecated features → Identify replacements
├── Large binary content → Plan for blob storage
├── Custom indexes → Migrate Oak index definitions
└── /etc content → Map to new locations (/conf, /content)
```

#### BPA Report Categories

| Category | Action |
|----------|--------|
| **Blocking** | Must fix before migration |
| **Major** | Should fix; may cause issues |
| **Info** | Awareness; review after migration |

---

### 4. Migration Set Configuration

#### Creating a Migration Set

```
Cloud Acceleration Manager (CAM):
1. Navigate to cam.adobe.com
2. Create/select project
3. Content Transfer → Create Migration Set

Migration Set Configuration:
  Name: "mysite-full-migration"
  Source URL: https://author-source.example.com
  Paths to include:
    - /content/mysite
    - /content/dam/mysite
    - /content/experience-fragments/mysite
    - /conf/mysite
  Paths to exclude:
    - /content/dam/mysite/temp
    - /content/mysite/test-pages
```

#### Recommended Path Strategy

```
Phase 1 — Structure and config:
  /conf/mysite          (templates, policies, CF models)
  /content/dam/mysite   (assets — largest, extract first)

Phase 2 — Content:
  /content/mysite       (pages)
  /content/experience-fragments/mysite

Phase 3 — Remaining:
  /content/launches     (if needed)
  /etc/tags/mysite      (tags)
```

---

### 5. User Mapping

AEM as a Cloud Service uses IMS (Adobe Identity Management). Local AEM users must be mapped to IMS users.

#### User Mapping CSV

```csv
# user-mapping.csv
source-user,ims-user
admin,admin@mycompany.com
jdoe,john.doe@mycompany.com
content-author,content-author@mycompany.com
dam-user,dam-admin@mycompany.com
```

#### Group Mapping

```
Source Groups → Cloud Service Groups:
  content-authors     → content-authors (auto-mapped)
  dam-users           → dam-users (auto-mapped)
  custom-approvers    → Create in Admin Console first, then map
  template-authors    → template-authors (auto-mapped)
```

**Important:**
- Create all target groups in Adobe Admin Console **before** ingestion
- System users (replication-service, workflow-service) are auto-mapped
- Service users require Repo Init configuration, not user mapping

---

### 6. Extraction

#### Full Extraction

```
Extraction Types:
├── Full Extraction — all content in migration set paths
│   └── Use for: initial migration, complete refresh
└── Top-Up Extraction — only changes since last extraction
    └── Use for: delta sync, minimizing cutover window
```

#### Extraction Best Practices

```
□ Run extraction during low-traffic period
□ Disable workflow launchers on source during extraction
□ Monitor extraction progress in CAM dashboard
□ Verify extraction log for errors/warnings
□ Check extracted content size matches expectations
```

#### Extraction Performance

| Content Size | Estimated Time | Notes |
|-------------|---------------|-------|
| < 50 GB | 2-6 hours | Standard extraction |
| 50-200 GB | 6-24 hours | Monitor closely |
| 200-500 GB | 1-3 days | Contact Adobe Support |
| > 500 GB | 3+ days | Requires Adobe assistance |

---

### 7. Ingestion

#### Ingestion Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Wipe** | Deletes existing content, then ingests | Clean migration, full refresh |
| **Non-Wipe (Top-Up)** | Merges with existing content | Incremental updates |

#### Ingestion Process

```
1. Select migration set in CAM
2. Choose target environment (dev/stage/prod)
3. Select ingestion mode (wipe/non-wipe)
4. Start ingestion
5. Monitor progress in CAM dashboard
6. Verify content post-ingestion
```

**Post-Ingestion Validation:**

```
□ Verify page rendering on author and publish
□ Check asset renditions are generated
□ Validate internal links and references
□ Test content fragment delivery
□ Verify user permissions
□ Check workflow status
□ Run automated regression tests
```

---

### 8. Content Transformation

CTT includes transformation rules for known incompatibilities:

```
Automatic transformations:
├── /etc/blueprints → /libs/msm (Live Copy blueprints)
├── /etc/designs → Content policies in /conf
├── /etc/clientlibs → /apps/clientlibs (if configured)
├── /etc/tags → /content/cq:tags
└── Custom transformations via configuration

Manual transformations needed:
├── /etc/workflow/models → Update ECMA → Java
├── /etc/replication → Cloud Service config
├── /etc/cloudservices → Developer Console config
└── Custom /etc structures → Project-specific mapping
```

---

### 9. Incremental Migration Strategy

For minimizing downtime during cutover:

```
Week 1: Full extraction + ingestion to dev
  └── Validate, fix issues

Week 2: Full extraction + ingestion to stage
  └── Full testing, UAT

Week 3: Top-up extraction (delta only)
  └── Captures changes since Week 2

Cutover Day:
  1. Content freeze on source (read-only)
  2. Final top-up extraction
  3. Ingestion to production
  4. DNS switch
  5. Validate production
  6. Decommission source
```

---

### 10. Troubleshooting

#### Common Issues

| Issue | Cause | Solution |
|-------|-------|---------|
| Extraction timeout | Large binary content | Split into smaller migration sets |
| Missing renditions | Asset microservices reprocessing | Trigger reprocessing post-ingestion |
| Broken references | Cross-path references not in migration set | Include all referenced paths |
| Permission errors | Service user not configured | Check Repo Init and service user mapping |
| Node type errors | Custom node types missing | Register node types before ingestion |
| Slow ingestion | Large volume of small nodes | Contact Adobe Support for optimization |

#### Extraction Logs

```
Location: CAM Dashboard → Migration Set → Extraction Log

Key log entries:
  INFO  — Normal progress
  WARN  — Non-blocking issues (review post-migration)
  ERROR — Blocking issues (fix before re-extracting)
```

---

### 11. Anti-Patterns

#### Migrating Without BPA

```
// WRONG — skip BPA and go straight to CTT
// Results: failed ingestion due to unsupported node types, broken workflows

// CORRECT — always run BPA first, fix blocking findings, then extract
```

#### Migrating Everything at Once

```
// WRONG — single migration set with /content and /content/dam (2TB total)
// Results: 5-day extraction, timeout failures, impossible to debug

// CORRECT — split into logical migration sets:
// Set 1: /content/dam/mysite (assets)
// Set 2: /content/mysite (pages)
// Set 3: /conf/mysite (config)
```

#### Skipping User Mapping

```
// WRONG — ingest without user mapping
// Results: all content owned by "admin", broken permissions, lost audit trail

// CORRECT — prepare user-mapping.csv, create IMS groups, map before ingestion
```

#### Not Using Incremental Extraction

```
// WRONG — full extraction for every iteration
// Each run takes hours/days, content freeze required for entire duration

// CORRECT — full extraction once, then top-up extractions for deltas
// Minimizes cutover window to hours instead of days
```

#### Forgetting Post-Migration Validation

```
// WRONG — "ingestion completed successfully" = migration done
// CTT reports success even if references are broken or renditions missing

// CORRECT — run comprehensive validation:
// - Automated link checker
// - Visual regression tests
// - Asset rendition spot-check
// - GraphQL endpoint validation
// - Permissions audit
```
