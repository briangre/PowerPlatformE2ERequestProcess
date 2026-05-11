# PowerPlatformE2ERequestProcess

This repo contains a complete end to end demo scenario for an app request, review, environment provisioning, and ALM process for Power Platform.

## Implementation plan

### Goal

Create a request process that allows a user to submit a Power Platform app request, routes that request through CoE triage, provisions the required ALM environments, and creates deployment pipelines for approved requests.

### Solution components

#### 1. Power Platform apps

1. **Requestor app (canvas app)**
   - Used by business users to submit new app requests.
   - Captures business purpose, app name, data sensitivity, team/department, approvers, and requested makers.
   - Shows request history and current status.
2. **CoE triage app (model-driven app)**
   - Used by the approval team to review, approve, reject, and manage requests.
   - Exposes request details, environment assignments, provisioning status, and pipeline status.
3. **Operations/admin app (optional model-driven experience)**
   - Used by platform admins to monitor failed provisioning jobs, re-run automations, and audit approvals.

#### 2. Dataverse tables

| Table | Purpose | Key columns |
| --- | --- | --- |
| App Request | Primary request record | Request ID, Title, Business justification, Requested by, Department, Data classification, Status, Decision, Decision notes |
| Request Approval | Stores review activity | Request, Approver, Outcome, Comments, Decision date |
| Environment Assignment | Tracks provisioned environments | Request, Environment type (Dev/Test/Prod), Environment name, Region, Provisioning status, Environment ID |
| Pipeline Configuration | Tracks deployment pipeline metadata | Request, Pipeline name, Pipeline ID, Source environment, Target environment, Status |
| Request Activity Log | Auditing and operational tracking | Request, Activity type, Activity timestamp, Details, Performed by |

#### 3. Power Automate cloud flows

1. **Request submission flow**
   - Triggered when an App Request row is created.
   - Sets initial status to `Submitted`.
   - Sends acknowledgement to the requestor.
   - Creates approval work items for CoE.
2. **Approval orchestration flow**
   - Starts approval for CoE triage.
   - On rejection:
     - Updates request status to `Rejected`.
     - Stores rejection reason.
     - Notifies requestor and closes the request.
   - On approval:
     - Updates request status to `Approved`.
     - Initiates environment provisioning.
3. **Environment provisioning flow**
   - Creates or assigns Dev/Test/Prod environments based on approved request criteria such as data classification, department/team ownership, region, and naming standards.
   - Applies required settings and security.
   - Adds the requestor as **Environment Maker** in the Dev environment.
   - Writes Environment Assignment records back to Dataverse.
4. **Pipeline creation flow**
   - Runs after all required environments are ready.
   - Creates a deployment pipeline with stages for Test and Prod.
   - Associates the correct source and target environments.
   - Saves pipeline metadata to Dataverse.
5. **Monitoring/retry flow**
   - Handles failed provisioning or pipeline actions.
   - Alerts admins and supports retry/recovery processing.

#### 4. Security roles

| Role | Access |
| --- | --- |
| Requestor | Create and read own requests; view status and notifications |
| CoE Reviewer | Read all requests; approve/reject; update triage fields |
| Platform Admin | Full access to requests, assignments, pipelines, and logs |
| Automation Service Account | Minimum required permissions for Dataverse updates, environment provisioning, and pipeline setup |

#### 5. ALM environments

1. **Development**
   - Created or assigned per approved request.
   - Requestor is added as Environment Maker.
   - Used to build the requested solution.
2. **Test**
   - Provisioned for validation and managed deployments.
   - Connected as the first deployment target in the pipeline.
3. **Production**
   - Provisioned for final deployment.
   - Connected as the final deployment target in the pipeline.

### End-to-end process flow

1. User submits a request in the Requestor app.
2. Request submission flow creates tracking records and notifies the approval team.
3. CoE reviews the request in the triage app.
4. If rejected:
   - Request is marked rejected.
   - Requestor receives the rejection reason.
   - Process ends.
5. If approved:
   - Dev/Test/Prod environments are provisioned or assigned.
   - Requestor is granted Environment Maker in Dev.
   - Pipeline is created with Test and Prod stages.
   - Request is marked `Ready for Development`.

### Recommended implementation phases

1. **Phase 1 - Data model and security**
   - Create Dataverse tables, relationships, choices, and security roles.
2. **Phase 2 - Request intake**
   - Build the Requestor app and request submission flow.
3. **Phase 3 - Triage and approval**
   - Build the CoE triage app and approval orchestration flow.
4. **Phase 4 - Provisioning automation**
   - Implement environment assignment/provisioning automation and security assignment.
5. **Phase 5 - Pipeline automation**
   - Create deployment pipeline automation for Test and Prod.
6. **Phase 6 - Operations**
   - Add monitoring, retry handling, and audit reporting.

### Key design decisions to confirm

| Decision | Timing |
| --- | --- |
| Whether environments are always newly created or can be selected from a pre-provisioned pool | Required before Phase 4; affects both provisioning automation and environment flow logic |
| Required naming conventions for environments, apps, solutions, and pipelines | Required before Phase 1 |
| Required approval levels for high-risk or high-sensitivity requests | Required before Phase 3 |
| Licensing, capacity, and admin connector prerequisites for provisioning automation | Required before Phase 4 |
