# ali-collibra - Workflow Implementation Reference

## Governance Workflows

### Workflow Structure

```yaml
# Governance Workflow Example: Data Asset Approval
workflow_name: Data Asset Approval
trigger: Asset status change to "Pending Approval"

steps:
  - step_1:
      name: Assign Reviewer
      action: Auto-assign to Data Steward based on domain
      script: |
        // Groovy script
        def domain = asset.getDomain()
        def steward = domain.getResponsibility("Data Steward")
        task.assignTo(steward)

  - step_2:
      name: Steward Review
      action: Human approval task
      form_fields:
        - name: Approval Decision
          type: radio
          options: [Approve, Reject, Request Changes]
        - name: Comments
          type: textarea

  - step_3:
      name: Update Status
      action: Set asset status based on decision
      script: |
        if (decision == "Approve") {
          asset.setStatus("Approved")
        } else if (decision == "Reject") {
          asset.setStatus("Rejected")
        } else {
          asset.setStatus("Changes Requested")
        }

  - step_4:
      name: Notify Requester
      action: Send email notification
      template: approval_notification.html
```

### Dev-to-Prod Promotion

```yaml
# Workflow promotion pattern
environments:
  - dev:
      url: https://dev.collibra.com
      workflow_version: 1.2.3

  - test:
      url: https://test.collibra.com
      workflow_version: 1.2.3

  - prod:
      url: https://prod.collibra.com
      workflow_version: 1.2.2

promotion_process:
  1. Export workflow from dev (JSON format)
  2. Review changes in test environment
  3. Execute workflow promotion script
  4. Import workflow to prod
  5. Version bump prod to 1.2.3
```

### Custom Workflow Logic (Groovy)

```groovy
// Custom validation step in workflow
import com.collibra.dgc.core.api.model.Asset

def asset = execution.getVariable("asset")
def assetType = asset.getType().getName()

if (assetType == "Database Table") {
    // Validate required attributes
    def owner = asset.getAttribute("Data Owner")
    def description = asset.getAttribute("Description")
    def classification = asset.getAttribute("Data Classification")

    def errors = []
    if (!owner) errors.add("Data Owner is required")
    if (!description) errors.add("Description is required")
    if (!classification) errors.add("Data Classification is required")

    if (errors.isEmpty()) {
        execution.setVariable("validationResult", "PASS")
    } else {
        execution.setVariable("validationResult", "FAIL")
        execution.setVariable("validationErrors", errors.join(", "))
    }
}
```
