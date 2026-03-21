---
title: AEM Workflow Development Patterns
impact: HIGH
impactDescription: Workflows orchestrate content approval, asset processing, and publishing — incorrect patterns cause hung workflows, performance degradation, and unreliable content operations
tags: workflows, process-steps, launchers, transient, payload, granite-workflow, cloud-service
---

## AEM Workflow Development Patterns

Workflows in AEM orchestrate multi-step processes: content approval, asset processing, translation, and publishing. AEM as a Cloud Service uses the Granite Workflow engine with significant differences from on-premise (no ECMA scripts, microservice-based asset processing).

---

### 1. Workflow Architecture (Cloud Service)

#### Key Differences from On-Premise

| Feature | AEM 6.5 On-Premise | AEM as a Cloud Service |
|---------|-------------------|----------------------|
| ECMA workflow scripts | Supported | **Not supported** |
| Custom Java process steps | Full access | Supported (with restrictions) |
| DAM Update Asset workflow | Customizable | **Asset microservices** (not customizable) |
| Transient workflows | Optional | **Default for performance workflows** |
| Workflow purge | Manual/scheduled | **Automatic** |
| Launcher execution | Synchronous option | **Always asynchronous** |

#### Workflow Types

```
Granite Workflows (modern, preferred):
  └── /var/workflow/models/   — workflow model definitions
  └── /var/workflow/instances/ — running/completed instances

Legacy CQ Workflows (avoid):
  └── /etc/workflow/models/   — deprecated location
```

---

### 2. Custom Workflow Process Steps

#### WorkflowProcess Interface

**Correct — properly implemented process step:**

```java
@Component(
    service = WorkflowProcess.class,
    property = {
        "process.label=My Custom Process Step"
    }
)
public class ContentApprovalStep implements WorkflowProcess {

    private static final Logger log = LoggerFactory.getLogger(ContentApprovalStep.class);

    @Reference
    private ResourceResolverFactory resolverFactory;

    private static final Map<String, Object> AUTH_INFO = Collections.singletonMap(
        ResourceResolverFactory.SUBSERVICE, "workflow-service"
    );

    @Override
    public void execute(WorkItem workItem, WorkflowSession workflowSession,
                        MetaDataMap processArgs) throws WorkflowException {
        String payloadPath = workItem.getWorkflowData().getPayload().toString();
        log.info("Processing payload: {}", payloadPath);

        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH_INFO)) {
            Resource resource = resolver.getResource(payloadPath);
            if (resource == null) {
                throw new WorkflowException("Payload resource not found: " + payloadPath);
            }

            ModifiableValueMap props = resource.adaptTo(ModifiableValueMap.class);
            if (props != null) {
                props.put("approvalStatus", "approved");
                props.put("approvedAt", Calendar.getInstance());
                resolver.commit();
            }

            // Advance the workflow
            log.info("Content approved: {}", payloadPath);

        } catch (LoginException e) {
            throw new WorkflowException("Service user login failed", e);
        } catch (PersistenceException e) {
            throw new WorkflowException("Failed to save approval status", e);
        }
    }
}
```

**Incorrect — common mistakes:**

```java
// WRONG: Using deprecated admin session
@Override
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    // Never use getAdministrativeResourceResolver
    ResourceResolver resolver = resolverFactory.getAdministrativeResourceResolver(null);

    // Never forget to close the resolver (use try-with-resources)
    // Never swallow exceptions silently
    try {
        // do work
    } catch (Exception e) {
        log.error("Error", e); // Workflow appears to succeed but didn't
    }
}
```

#### Process Step Arguments

Access arguments passed from the workflow model dialog:

```java
@Override
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    // Single argument from PROCESS_ARGS
    String args = processArgs.get("PROCESS_ARGS", String.class);

    // Parse structured arguments
    String targetStatus = processArgs.get("PROCESS_ARGS", "published");

    // Access workflow metadata (shared across steps)
    WorkflowData workflowData = workItem.getWorkflowData();
    MetaDataMap wfMeta = workflowData.getMetaDataMap();
    String initiator = wfMeta.get("initiator", String.class);
}
```

---

### 3. Workflow Launchers

Launchers trigger workflows automatically based on JCR events.

#### Launcher Configuration

```xml
<!-- /apps/myproject/config/com.adobe.granite.workflow.core.launcher.WorkflowLauncherConfig-myLauncher.cfg.json -->
{
    "eventType": "NODE_MODIFIED",
    "nodetype": "cq:PageContent",
    "glob": "/content/mysite/**",
    "workflow": "/var/workflow/models/content-approval",
    "runModes": ["author"],
    "enabled": true,
    "excludeList": ["jcr:lastModified", "jcr:lastModifiedBy", "cq:lastModified"],
    "condition": ""
}
```

| Event Type | Trigger |
|-----------|---------|
| `NODE_CREATED` | New node created |
| `NODE_MODIFIED` | Node properties changed |
| `NODE_REMOVED` | Node deleted |
| `WORKFLOW_COMPLETED` | Another workflow completed |

#### Launcher Best Practices

```java
// Use excludeList to prevent infinite loops
// When a workflow modifies a property, it triggers the launcher again
// Exclude properties your workflow writes:
"excludeList": ["approvalStatus", "approvedAt", "jcr:lastModified"]
```

**Anti-pattern — launcher loop:**

```
Launcher triggers on NODE_MODIFIED for /content/mysite/**
  → Workflow Step sets property "status" on the payload
    → NODE_MODIFIED event fires
      → Launcher triggers again → infinite loop
```

---

### 4. Transient Workflows

Transient workflows do not persist execution history to JCR, dramatically improving performance. **Required for high-throughput scenarios.**

```xml
<!-- Mark a workflow model as transient -->
<!-- In the workflow model definition node -->
<model
    jcr:primaryType="cq:WorkflowModel"
    transient="true"
    title="My Transient Workflow">
</model>
```

#### When to Use Transient vs Persistent

| Scenario | Type | Reason |
|----------|------|--------|
| Asset processing | Transient | High volume, no audit needed |
| Content approval | Persistent | Audit trail required |
| Notification sending | Transient | Fire-and-forget |
| Legal review | Persistent | Compliance tracking |
| Bulk tag application | Transient | Performance critical |

---

### 5. Workflow Payload Handling

#### Payload Types

```java
@Override
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    WorkflowData data = workItem.getWorkflowData();
    String payloadType = data.getPayloadType();

    if ("JCR_PATH".equals(payloadType)) {
        String path = data.getPayload().toString();
        // Handle page or asset path
    } else if ("JCR_UUID".equals(payloadType)) {
        String uuid = data.getPayload().toString();
        // Resolve UUID to path via JCR Session
    }
}
```

#### DAM Asset Payload Patterns

```java
// Processing a DAM asset payload
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    String payloadPath = workItem.getWorkflowData().getPayload().toString();

    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH_INFO)) {
        Resource assetResource = resolver.getResource(payloadPath);
        if (assetResource == null) return;

        Asset asset = assetResource.adaptTo(Asset.class);
        if (asset == null) return;

        // Access original rendition
        Rendition original = asset.getOriginalRendition();
        String mimeType = original.getMimeType();

        // Access metadata
        String title = asset.getMetadataValue(DamConstants.DC_TITLE);

        // Add/update metadata
        Resource metadataResource = resolver.getResource(payloadPath + "/jcr:content/metadata");
        if (metadataResource != null) {
            ModifiableValueMap meta = metadataResource.adaptTo(ModifiableValueMap.class);
            meta.put("dam:customField", "processed");
            resolver.commit();
        }
    } catch (Exception e) {
        throw new WorkflowException("Asset processing failed", e);
    }
}
```

---

### 6. Custom Participant Chooser

Dynamic participant assignment based on content or context:

```java
@Component(
    service = ParticipantStepChooser.class,
    property = {
        "chooser.label=Regional Approver Chooser"
    }
)
public class RegionalApproverChooser implements ParticipantStepChooser {

    @Reference
    private ResourceResolverFactory resolverFactory;

    @Override
    public String getParticipant(WorkItem workItem, WorkflowSession workflowSession,
                                  MetaDataMap metaDataMap) throws WorkflowException {
        String payloadPath = workItem.getWorkflowData().getPayload().toString();

        try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(AUTH_INFO)) {
            Resource page = resolver.getResource(payloadPath + "/jcr:content");
            if (page != null) {
                ValueMap props = page.getValueMap();
                String region = props.get("region", "global");

                return switch (region) {
                    case "emea" -> "emea-approvers";
                    case "apac" -> "apac-approvers";
                    case "americas" -> "americas-approvers";
                    default -> "content-approvers";
                };
            }
        } catch (LoginException e) {
            throw new WorkflowException("Cannot resolve service user", e);
        }

        return "content-approvers"; // fallback group
    }
}
```

---

### 7. Workflow Metadata and Step Communication

#### Passing Data Between Steps

```java
// Step 1: Set metadata for downstream steps
@Override
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    MetaDataMap wfMeta = workItem.getWorkflow().getMetaDataMap();
    wfMeta.put("validationResult", "passed");
    wfMeta.put("validatedBy", "system");
    wfMeta.put("validatedAt", System.currentTimeMillis());
}

// Step 2: Read metadata from previous step
@Override
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    MetaDataMap wfMeta = workItem.getWorkflow().getMetaDataMap();
    String result = wfMeta.get("validationResult", "unknown");

    if (!"passed".equals(result)) {
        // Terminate or route differently
        workflowSession.terminateWorkflow(workItem.getWorkflow());
        return;
    }
}
```

---

### 8. Programmatic Workflow Management

#### Starting a Workflow

```java
@Reference
private WorkflowService workflowService;

public void startApprovalWorkflow(ResourceResolver resolver, String contentPath) {
    try {
        WorkflowSession wfSession = workflowService.getWorkflowSession(
            resolver.adaptTo(Session.class));
        WorkflowModel model = wfSession.getModel("/var/workflow/models/content-approval");

        if (model != null) {
            WorkflowData data = wfSession.newWorkflowData("JCR_PATH", contentPath);
            // Set initial metadata
            data.getMetaDataMap().put("initiator", resolver.getUserID());
            wfSession.startWorkflow(model, data);
        }
    } catch (WorkflowException e) {
        log.error("Failed to start workflow for: {}", contentPath, e);
    }
}
```

#### Querying Workflow Status

```java
public String getWorkflowStatus(ResourceResolver resolver, String contentPath) {
    try {
        WorkflowSession wfSession = workflowService.getWorkflowSession(
            resolver.adaptTo(Session.class));

        Workflow[] workflows = wfSession.getWorkflows(new String[]{contentPath});
        for (Workflow wf : workflows) {
            if (wf.isActive()) {
                return "IN_PROGRESS";
            }
        }
        return "COMPLETE";
    } catch (WorkflowException e) {
        log.error("Failed to query workflow status", e);
        return "UNKNOWN";
    }
}
```

---

### 9. Error Handling and Timeouts

#### Retry Logic

```java
@Component(
    service = WorkflowProcess.class,
    property = { "process.label=Reliable External Call Step" }
)
public class ReliableExternalCallStep implements WorkflowProcess {

    private static final int MAX_RETRIES = 3;

    @Override
    public void execute(WorkItem workItem, WorkflowSession workflowSession,
                        MetaDataMap processArgs) throws WorkflowException {
        MetaDataMap wfMeta = workItem.getWorkflow().getMetaDataMap();
        int retryCount = wfMeta.get("retryCount", 0);

        try {
            callExternalService(workItem);
        } catch (Exception e) {
            if (retryCount < MAX_RETRIES) {
                wfMeta.put("retryCount", retryCount + 1);
                log.warn("Retry {}/{} for payload: {}", retryCount + 1, MAX_RETRIES,
                    workItem.getWorkflowData().getPayload());
                // Route back to this step or a retry handler
                throw new WorkflowException("Retriable failure", e);
            }
            log.error("Max retries exceeded, failing workflow", e);
            workflowSession.terminateWorkflow(workItem.getWorkflow());
        }
    }
}
```

#### Workflow Timeout Configuration

```xml
<!-- OSGi config for workflow timeout -->
<!-- com.adobe.granite.workflow.core.WorkflowSessionFactory.cfg.json -->
{
    "granite.workflow.maxTimeout": 86400,
    "granite.workflow.maxRetry": 3
}
```

---

### 10. Anti-Patterns

#### Never Use ECMA Scripts (Cloud Service)

```
// WRONG — not supported on AEM as a Cloud Service
/etc/workflow/scripts/my-script.ecma
```

#### Never Block the Workflow Engine

```java
// WRONG — long-running synchronous call blocks workflow thread pool
@Override
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    // This blocks a workflow thread for minutes
    Thread.sleep(300000); // wait for external system
    callSlowExternalAPI(); // synchronous 30-second call
}

// CORRECT — use Sling Jobs for long-running work
@Override
public void execute(WorkItem workItem, WorkflowSession workflowSession,
                    MetaDataMap processArgs) throws WorkflowException {
    // Queue a Sling Job and let the workflow continue
    Map<String, Object> jobProps = new HashMap<>();
    jobProps.put("payload", workItem.getWorkflowData().getPayload().toString());
    jobProps.put("workflowId", workItem.getWorkflow().getId());
    jobManager.addJob("myproject/external-call", jobProps);
}
```

#### Never Modify Asset Processing Workflows on Cloud Service

```
// WRONG — DAM Update Asset is handled by asset microservices on Cloud Service
// Customizing /var/workflow/models/dam/update_asset has no effect

// CORRECT — use Processing Profiles for custom asset processing
// CORRECT — use Asset Compute workers for custom renditions
```

#### Never Ignore Idempotency

```java
// WRONG — running twice creates duplicate entries
props.put("comments", existingComments + "\nApproved");

// CORRECT — idempotent operation
props.put("approvalStatus", "approved");
props.put("approvedAt", Calendar.getInstance());
```
