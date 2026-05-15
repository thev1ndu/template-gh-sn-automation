# Installation Guide

Complete setup for the GitHub ↔ ServiceNow integration. Follow both parts in order.

---

## Part 1 — GitHub Setup

### 1.1 Add repository secrets

Go to **Repository Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret | Value |
|---|---|
| `SERVICENOW_URL` | Full API endpoint: `https://<instance>.service-now.com/api/<scope>/gh_integration/case` |
| `SERVICENOW_UI_URL` | Base URL only: `https://<instance>.service-now.com` |
| `SERVICENOW_USERNAME` | ServiceNow user with REST API access |
| `SERVICENOW_PASSWORD` | Password for the above user |

### 1.2 Configure `.github/servicenow-config.yml`

Edit this file and commit. It holds ServiceNow constants applied to every Case created from a GitHub issue.

```yaml
case:
  category: "Issue"                    # Category display value from SN
  account: "Your Account Name"         # Exact Account display name from SN
  announcement_type: "General"         # Announcement Type choice value from SN

project:
  name: "Your Project Name"            # Project reference — exact display name from SN
  product: "Your Product Name"         # Product reference — exact display name from SN
  wso2_product: "Your WSO2 Product"    # WSO2 Product reference — exact display name from SN
```

> For reference fields (`account`, `project`, `product`, `wso2_product`), the value must exactly match the display name in ServiceNow. A mismatch silently leaves the field blank — no error is returned.

To find exact values: open any existing Case in ServiceNow, right-click the field label → **Show value**.

---

## Part 2 — ServiceNow Setup

### Step 1 — Add custom columns to the Case table

These two columns link a Case back to the GitHub issue that created it.

1. Go to **System Definition → Tables**
2. Search for `sn_customerservice_case` and open it
3. Click the **Columns** tab → **New** for each column:

| Setting | Column 1 | Column 2 |
|---|---|---|
| Column label | GitHub Issue Number | GitHub Issue URL |
| Column name | `u_github_issue_number` | `u_github_issue_url` |
| Type | String (length 20) | URL |

4. Save both columns.

> Use String (not Integer) for `u_github_issue_number` — the integration queries and compares it as a string throughout.

---

### Step 2 — Create the `github.dispatch.config` system property

This property holds GitHub credentials keyed by Account display name. Every outbound flow reads it at runtime — no credentials are hardcoded anywhere.

#### Create the property

1. Go to **System Definition → System Properties**
2. Click **New** and fill in:

| Field | Value |
|---|---|
| Name | `github.dispatch.config` |
| Type | `String` (or `Password` if your org encrypts integration credentials) |
| Description | GitHub PAT + owner + repo keyed by Account display name |

3. Set the **Value** to this JSON (one entry per team/account):

```json
{
  "Your Account Display Name": {
    "token": "github_pat_xxxxx",
    "owner": "your-github-org-or-user",
    "repo":  "your-repo-name"
  }
}
```

| Key | What to put |
|---|---|
| Top-level key | Exact value of the **Account** field on the Case — copy it character-for-character |
| `token` | GitHub Personal Access Token — classic with `repo` scope, or fine-grained with **Contents: Read and write** |
| `owner` | GitHub org name or username (the part before `/` in the repo URL) |
| `repo` | Repository name (the part after `/` in the repo URL) |

4. Save. Tick **Ignore cache** so the value takes effect immediately.

#### Multiple accounts / teams

Add one key per team. Cases route automatically by their Account field — no flow changes needed.

```json
{
  "Team Alpha Account": {
    "token": "github_pat_aaa",
    "owner": "my-org",
    "repo":  "alpha-repo"
  },
  "Team Beta Account": {
    "token": "github_pat_bbb",
    "owner": "my-org",
    "repo":  "beta-repo"
  }
}
```

#### Finding the exact account display name

1. Open a Case that belongs to the team
2. Right-click the **Account** field label → **Show value**
3. Copy that string exactly as the JSON key (case-sensitive, spaces included)

#### Updating the property

1. Go to **All > System Properties > System Properties**
2. Search for `github.dispatch.config`
3. Edit the **Value** field — paste the full JSON object
4. Tick **Ignore cache** → click **Update**

> Use a JSON formatter before pasting to catch syntax errors. A malformed value causes all flows to log `config_missing` and stop sending.

---

### Step 3 — Create the Scripted REST API

This exposes the `POST /case` and `PATCH /case` endpoints that GitHub Actions calls.

#### 3.1 Create the REST API

1. Go to **System Web Services → Scripted REST APIs**
2. Click **New**:
   - **Name**: `GitHub Case Integration`
   - **API ID**: `gh_integration`
   - **Application**: `Global`
3. Click **Submit**

#### 3.2 Add the POST resource — Create Case

On the saved API record, scroll to **Resources** → **New**:

| Field | Value |
|---|---|
| Name | `Create Case` |
| HTTP method | `POST` |
| Relative path | `/case` |

Paste this script:

```javascript
(function process(request, response) {
  var body = request.body.data;
  var issueNumber = body.issue_number + '';
  if (!issueNumber) {
    response.setStatus(400);
    response.setBody({ error: 'issue_number is required' });
    return;
  }

  // Idempotency: return existing case if already created
  var existing = new GlideRecord('sn_customerservice_case');
  existing.addQuery('u_github_issue_number', issueNumber);
  existing.query();
  if (existing.next()) {
    response.setStatus(200);
    response.setBody({
      case_number: existing.number + '',
      sys_id: existing.sys_id + '',
      case_sys_id: existing.sys_id + '',
      message: 'Case already exists for this issue'
    });
    return;
  }

  var gr = new GlideRecord('sn_customerservice_case');
  gr.initialize();
  gr.u_github_issue_number = issueNumber;
  gr.u_github_issue_url    = body.issue_url + '';
  gr.short_description.setDisplayValue(body.title + '');
  gr.description           = body.description + '';
  gr.category.setDisplayValue(body.category + '');
  gr.account.setDisplayValue(body.account + '');
  gr.u_announcement_type   = body.announcement_type + '';
  gr.project.setDisplayValue(body.project + '');
  gr.product.setDisplayValue(body.product + '');
  gr.u_wso2_product.setDisplayValue(body.wso2_product + '');
  gr.priority.setDisplayValue(body.priority + '');
  gr.u_project_environment = body.u_project_environment + '';
  gr.u_request_details     = body.u_request_details + '';
  if (body.u_short_description) gr.u_short_description = body.u_short_description + '';
  if (body.u_impact)            gr.u_impact = body.u_impact + '';
  if (body.u_impact_description_overall)  gr.u_impact_description_overall  = body.u_impact_description_overall + '';
  if (body.u_impact_description_customer) gr.u_impact_description_customer = body.u_impact_description_customer + '';
  if (body.u_affected_component)  gr.u_affected_component  = body.u_affected_component + '';
  if (body.u_affected_services)   gr.u_affected_services   = body.u_affected_services + '';
  if (body.u_service_outage)      gr.u_service_outage      = body.u_service_outage + '';
  if (body.u_implementation_plan) gr.u_implementation_plan = body.u_implementation_plan + '';
  if (body.u_test_plan)           gr.u_test_plan           = body.u_test_plan + '';

  var sysId = gr.insert();
  if (!sysId) {
    response.setStatus(500);
    response.setBody({ error: 'Case creation failed' });
    return;
  }

  gr.get(sysId);
  response.setStatus(201);
  response.setBody({
    case_number: gr.number + '',
    sys_id: sysId + '',
    case_sys_id: sysId + '',
    message: 'Case created successfully'
  });
})(request, response);
```

#### 3.3 Add the PATCH resource — Update Case

On the same API record, **Resources** → **New**:

| Field | Value |
|---|---|
| Name | `Update Case` |
| HTTP method | `PATCH` |
| Relative path | `/case` |

Paste this script:

```javascript
(function process(request, response) {
  var body = request.body.data;
  var issueNumber = body.issue_number + '';
  if (!issueNumber) {
    response.setStatus(400);
    response.setBody({ error: 'issue_number is required' });
    return;
  }

  var gr = new GlideRecord('sn_customerservice_case');
  gr.addQuery('u_github_issue_number', issueNumber);
  gr.query();
  if (!gr.next()) {
    response.setStatus(404);
    response.setBody({ error: 'No case found for issue ' + issueNumber });
    return;
  }

  var tracked = ['priority','u_project_environment','u_request_details','u_impact',
    'u_short_description','u_impact_description_overall','u_impact_description_customer',
    'u_affected_component','u_affected_services','u_service_outage','u_implementation_plan','u_test_plan'];
  var changes = [];
  tracked.forEach(function(f) {
    if (body[f] !== undefined && body[f] + '' !== gr[f] + '') {
      changes.push({ field: f, old: gr[f] + '', new: body[f] + '' });
      gr[f] = body[f] + '';
    }
  });
  if (body.title) gr.short_description.setDisplayValue(body.title + '');
  gr.description = body.description + '';

  gr.update();
  response.setStatus(200);
  response.setBody({
    case_number: gr.number + '',
    sys_id: gr.sys_id + '',
    case_sys_id: gr.sys_id + '',
    changes: changes.length,
    change_log: changes,
    message: 'Case updated with ' + changes.length + ' change(s)'
  });
})(request, response);
```

---

### Step 4 — Create the GitHub Integration REST Message

This is a container used by all outbound flows. Credentials are **not** stored here — they come from `github.dispatch.config` at runtime.

#### 4.1 Create the REST Message record

1. Go to **All > System Web Services > Outbound > REST Messages**
2. Click **New**:

| Field | Value |
|---|---|
| Name | `GitHub Integration` |
| Application | `Global` |
| Accessible from | `All application scopes` |
| Description | `Sends repository_dispatch events to GitHub Actions` |
| Endpoint | `https://api.github.com` |

3. Click the **Authentication** tab → **Authentication type**: `No authentication`
4. Click **Submit**

#### 4.2 Add HTTP Method: `dispatch` (used by comment and case-update flows)

On the saved REST Message, scroll to **HTTP Methods** → **New**:

| Field | Value |
|---|---|
| Name | `dispatch` |
| HTTP method | `POST` |
| Endpoint | leave blank |
| Content | leave blank |

- **Authentication tab**: `Inherit from parent`
- **HTTP Request tab** → HTTP Headers:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |

Click **Submit**.

#### 4.3 Add HTTP Method: `dispatch_cr` (used by CR notification flows)

Same as above but with **Name**: `dispatch_cr`. Same headers, same settings.

Click **Submit**.

#### 4.4 Verify with Script Background

Go to **All > System Definition > Scripts - Background** and run:

```javascript
var configJson  = gs.getProperty('github.dispatch.config');
var accountName = 'Your Account Display Name'; // replace with your actual account name
var config      = JSON.parse(configJson)[accountName];

gs.info('Owner: ' + config.owner);
gs.info('Repo:  ' + config.repo);
gs.info('Token starts with: ' + config.token.substring(0, 10));

var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');
rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches');
rm.setRequestHeader('Authorization', 'token ' + config.token);
rm.setRequestBody(JSON.stringify({
  event_type: 'servicenow-note',
  client_payload: {
    issue_number: '1',
    note_text: 'Test from Script Background',
    note_type: 'additional_comments',
    case_number: 'CS0000001',
    sn_user: gs.getUserDisplayName()
  }
}));

var response = rm.execute();
gs.info('HTTP Status: ' + response.getStatusCode());
```

| HTTP status | Meaning |
|---|---|
| `204` | Success |
| `401` | Token wrong or expired |
| `404` | Owner or repo is wrong |
| `422` | Workflow file not on default branch |

---

### Step 5 — Flow: SN Comment → GitHub

Fires when an agent posts an additional comment on a Case. Sends it to the linked GitHub issue.

**GitHub Actions file:** `sn-comment-to-github.yml`  
**Event type sent:** `servicenow-note`

#### 5.1 Create the flow

**All > Process Automation > Flow Designer → New > Flow**

- **Name**: `SN Comment to GitHub`
- **Run as**: `System User`

#### 5.2 Trigger

- **Record > Updated** on `Customer Service Case [sn_customerservice_case]`
- Condition: `Additional comments` **changes**

#### 5.3 Script Step 1 — Read comment and prepare data

Input variables (all String, mapped from Trigger > Customer Service Case Record):

| Variable | Data pill |
|---|---|
| `github_issue_number` | `u_github_issue_number` |
| `case_number` | `Number` |
| `case_sys_id` | `Sys ID` |
| `account_name` | `Account > Name` |

> Do NOT map `Additional comments` as a data pill — it returns a Java object. Read it via `GlideRecord` instead.

Script:

```javascript
(function execute(inputs, outputs) {
  var issueNumber = inputs.github_issue_number + '';
  var caseSysId   = inputs.case_sys_id + '';

  var gr = new GlideRecord('sn_customerservice_case');
  gr.get(caseSysId);
  var commentText = gr.comments.getJournalEntry(1) + '';

  outputs.issue_number  = issueNumber;
  outputs.note_text     = commentText;
  outputs.note_type     = 'additional_comments';
  outputs.case_number   = inputs.case_number + '';
  outputs.case_sys_id   = caseSysId;
  outputs.sn_user       = gs.getUserDisplayName();
  outputs.account_name  = inputs.account_name + '';
  outputs.should_send   = (issueNumber.length > 0 && commentText.length > 0) ? 'true' : 'false';
})(inputs, outputs);
```

Output variables: `issue_number`, `note_text`, `note_type`, `case_number`, `case_sys_id`, `sn_user`, `account_name`, `should_send`

#### 5.4 If Condition

- `Script step 1 > should_send` **is** `true`

All remaining steps go inside the **then** branch.

#### 5.5 Script Step 2 — Call GitHub REST Message (inside **then**)

Input variables mapped from Script step 1 outputs: `issue_number`, `note_text`, `note_type`, `case_number`, `case_sys_id`, `sn_user`, `account_name`

Script:

```javascript
(function execute(inputs, outputs) {
  try {
    var configJson = gs.getProperty('github.dispatch.config');
    var config = JSON.parse(configJson)[inputs.account_name];
    if (!config) {
      gs.error('No config entry for account "' + inputs.account_name + '"');
      outputs.http_status = 'config_missing'; outputs.success = 'false'; return;
    }
    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');
    rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches');
    rm.setRequestHeader('Authorization', 'token ' + config.token);
    rm.setRequestBody(JSON.stringify({
      event_type: 'servicenow-note',
      client_payload: {
        issue_number: inputs.issue_number,
        note_text:    inputs.note_text,
        note_type:    inputs.note_type,
        case_number:  inputs.case_number,
        case_sys_id:  inputs.case_sys_id,
        sn_user:      inputs.sn_user
      }
    }));
    var response = rm.execute();
    var status = response.getStatusCode();
    outputs.http_status = status + '';
    outputs.success = (status == 204 || status == 200) ? 'true' : 'false';
  } catch (ex) {
    outputs.http_status = 'error'; outputs.success = 'false';
    gs.error('GitHub dispatch exception: ' + ex.message);
  }
})(inputs, outputs);
```

Output variables: `http_status`, `success`

#### 5.6 Save and Activate

---

### Step 6 — Flow: SN Case Updates → GitHub

Fires when a Case is **closed** or **assigned**. Posts a comment on the GitHub issue; closure also closes the issue.

**GitHub Actions file:** `sn-case-updates.yml`  
**Event type sent:** `servicenow-case-update`

#### 6.1 Create the flow

- **Name**: `SN Case Updates → GitHub`
- **Run as**: `System User`

#### 6.2 Trigger

- **Record > Updated** on `sn_customerservice_case`
- Conditions:
  - `u_github_issue_number` **is not empty** AND
  - `State` **changes to** `Closed` OR `Assigned to` **changes**

> To add the OR: after adding the Closed row, click the AND connector between rows 2 and 3 and switch it to OR.

#### 6.3 Script Step 1 — Determine what changed

Input variables:

| Variable | Data pill |
|---|---|
| `github_issue_number` | `u_github_issue_number` |
| `case_number` | `Number` |
| `case_sys_id` | `Sys ID` |
| `case_resolution` | `Resolution notes` |
| `assigned_to_name` | `Assigned to > Name` |
| `account_name` | `Account > Name` |

Script:

```javascript
(function execute(inputs, outputs) {
  var issueNumber = inputs.github_issue_number + '';
  var caseSysId   = inputs.case_sys_id + '';
  var assignedTo  = inputs.assigned_to_name + '';

  var gr = new GlideRecord('sn_customerservice_case');
  gr.get(caseSysId);
  var isClosed = (gr.state.getDisplayValue() + '').toLowerCase() === 'closed';

  var isAssigned = false;
  if (!isClosed && assignedTo.length > 0) {
    var audit = new GlideRecord('sys_audit');
    audit.addQuery('tablename', 'sn_customerservice_case');
    audit.addQuery('documentkey', caseSysId);
    audit.addQuery('fieldname', 'assigned_to');
    audit.orderByDesc('sys_created_on');
    audit.setLimit(1);
    audit.query();
    if (audit.next()) isAssigned = true;
  }

  var action = isClosed ? 'closed' : (isAssigned ? 'assigned' : '');

  outputs.issue_number     = issueNumber;
  outputs.case_number      = inputs.case_number + '';
  outputs.case_sys_id      = caseSysId;
  outputs.resolution_notes = inputs.case_resolution + '';
  outputs.assigned_to      = assignedTo;
  outputs.sn_user          = gs.getUserDisplayName();
  outputs.action           = action;
  outputs.account_name     = inputs.account_name + '';
  outputs.should_send      = (issueNumber.length > 0 && action.length > 0) ? 'true' : 'false';
})(inputs, outputs);
```

Output variables: `issue_number`, `case_number`, `case_sys_id`, `resolution_notes`, `assigned_to`, `sn_user`, `action`, `account_name`, `should_send`

#### 6.4 If Condition

- `Script step 1 > should_send` **is** `true`

#### 6.5 Script Step 2 — Call GitHub (inside **then**)

Input variables mapped from Script step 1: `issue_number`, `case_number`, `case_sys_id`, `resolution_notes`, `assigned_to`, `sn_user`, `action`, `account_name`

Script:

```javascript
(function execute(inputs, outputs) {
  try {
    var configJson = gs.getProperty('github.dispatch.config');
    var config = JSON.parse(configJson)[inputs.account_name];
    if (!config) {
      outputs.http_status = 'config_missing'; outputs.success = 'false'; return;
    }
    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');
    rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches');
    rm.setRequestHeader('Authorization', 'token ' + config.token);
    rm.setRequestBody(JSON.stringify({
      event_type: 'servicenow-case-update',
      client_payload: {
        action:              inputs.action,
        github_issue_number: inputs.issue_number,
        case_number:         inputs.case_number,
        case_sys_id:         inputs.case_sys_id,
        resolution_notes:    inputs.resolution_notes,
        assigned_to:         inputs.assigned_to,
        sn_user:             inputs.sn_user
      }
    }));
    var status = rm.execute().getStatusCode();
    outputs.http_status = status + '';
    outputs.success = (status == 204 || status == 200) ? 'true' : 'false';
  } catch (ex) {
    outputs.http_status = 'error'; outputs.success = 'false';
    gs.error('GitHub case-update dispatch exception: ' + ex.message);
  }
})(inputs, outputs);
```

Output variables: `http_status`, `success`

#### 6.6 Save and Activate

---

### Step 7 — Flow A: SN CR Created → GitHub

Fires when a Change Request is created from a Case. Posts a CR notification on the GitHub issue.

**GitHub Actions file:** `sn-cr-notifier.yml`  
**Event type sent:** `servicenow-cr-update` with `action: created`

#### 7.1 Create the flow

- **Name**: `SN CR Created → GitHub`
- **Run as**: `System User`

#### 7.2 Trigger

- **Record > Created** on `Change Request [change_request]`
- Condition: `Parent` **is not empty**

#### 7.3 Look Up Record step

- Table: `Customer Service Case [sn_customerservice_case]`
- Filter: `Sys ID` is → `Trigger > Change Request Record > Parent > Sys ID`

#### 7.4 Script Step 1

Input variables:

| Variable | Data pill |
|---|---|
| `github_issue_number` | Look Up Record > Customer Service Case > `u_github_issue_number` |
| `case_sys_id` | Look Up Record > Customer Service Case > `Sys ID` |
| `cr_number` | Trigger > Change Request > `Number` |
| `cr_sys_id` | Trigger > Change Request > `Sys ID` |
| `cr_state` | Trigger > Change Request > `State` |
| `account_name` | Look Up Record > Customer Service Case > `Account > Name` |

Script:

```javascript
(function execute(inputs, outputs) {
  outputs.issue_number = inputs.github_issue_number + '';
  outputs.case_sys_id  = inputs.case_sys_id + '';
  outputs.cr_number    = inputs.cr_number + '';
  outputs.cr_sys_id    = inputs.cr_sys_id + '';
  outputs.cr_state     = inputs.cr_state + '';
  outputs.account_name = inputs.account_name + '';
  outputs.should_send  = (outputs.issue_number.length > 0 && outputs.cr_number.length > 0) ? 'true' : 'false';
})(inputs, outputs);
```

Output variables: `issue_number`, `case_sys_id`, `cr_number`, `cr_sys_id`, `cr_state`, `account_name`, `should_send`

#### 7.5 If Condition → `should_send` is `true`

#### 7.6 Script Step 2 — Call GitHub (inside **then**)

Input variables from Script step 1 outputs.

Script:

```javascript
(function execute(inputs, outputs) {
  try {
    var config = JSON.parse(gs.getProperty('github.dispatch.config'))[inputs.account_name];
    if (!config) { outputs.http_status = 'config_missing'; outputs.success = 'false'; return; }
    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch_cr');
    rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches');
    rm.setRequestHeader('Authorization', 'token ' + config.token);
    rm.setRequestBody(JSON.stringify({
      event_type: 'servicenow-cr-update',
      client_payload: {
        github_issue_number: inputs.issue_number,
        cr_number:           inputs.cr_number,
        cr_sys_id:           inputs.cr_sys_id,
        cr_state:            inputs.cr_state,
        previous_state:      '',
        case_sys_id:         inputs.case_sys_id,
        action:              'created'
      }
    }));
    var status = rm.execute().getStatusCode();
    outputs.http_status = status + '';
    outputs.success = (status == 204 || status == 200) ? 'true' : 'false';
  } catch (ex) {
    outputs.http_status = 'error'; outputs.success = 'false';
    gs.error('GitHub CR dispatch exception: ' + ex.message);
  }
})(inputs, outputs);
```

Output variables: `http_status`, `success`

#### 7.7 Save and Activate

---

### Step 8 — Flow B: SN CR Updated → GitHub

Fires when a CR's state, assignee, or planned dates change. Sends `state_changed` or `dates_updated` action.

**GitHub Actions file:** `sn-cr-notifier.yml`  
**Event type sent:** `servicenow-cr-update` with `action: state_changed` or `dates_updated`

#### 8.1 Create the flow

- **Name**: `SN CR Updated → GitHub`
- **Run as**: `System User`

#### 8.2 Trigger

- **Record > Updated** on `Change Request [change_request]`
- Conditions (use OR between field conditions, AND for Parent):
  - `State` **changes** OR `Assigned to` **changes** OR `Planned start date` **changes** OR `Planned end date` **changes**
  - AND `Parent` **is not empty**

#### 8.3 Look Up Record step

Same as Flow A (Step 7.3).

#### 8.4 Script Step 1

Input variables (all String):

| Variable | Data pill |
|---|---|
| `github_issue_number` | Look Up Record > Customer Service Case > `u_github_issue_number` |
| `case_sys_id` | Look Up Record > Customer Service Case > `Sys ID` |
| `cr_number` | Trigger > Change Request > `Number` |
| `cr_sys_id` | Trigger > Change Request > `Sys ID` |
| `cr_state` | Trigger > Change Request > `State` |
| `cr_assigned_to` | Trigger > Change Request > `Assigned to > Name` |
| `cr_planned_start` | Trigger > Change Request > `Planned start date` |
| `cr_planned_end` | Trigger > Change Request > `Planned end date` |
| `account_name` | Look Up Record > Customer Service Case > `Account > Name` |

Script:

```javascript
(function execute(inputs, outputs) {
  var dataPillState = inputs.cr_state + '';
  var crSysId = inputs.cr_sys_id + '';
  var crState = dataPillState;

  var crGr = new GlideRecord('change_request');
  if (crGr.get(crSysId)) crState = crGr.getValue('state') + '';

  outputs.issue_number  = inputs.github_issue_number + '';
  outputs.case_sys_id   = inputs.case_sys_id + '';
  outputs.cr_number     = inputs.cr_number + '';
  outputs.cr_sys_id     = crSysId;
  outputs.cr_state      = crState;
  outputs.assigned_to   = inputs.cr_assigned_to + '';
  outputs.planned_start = inputs.cr_planned_start + '';
  outputs.planned_end   = inputs.cr_planned_end + '';
  outputs.action        = (dataPillState !== crState) ? 'state_changed' : 'dates_updated';
  outputs.account_name  = inputs.account_name + '';
  outputs.should_send   = (outputs.issue_number.length > 0 && outputs.cr_number.length > 0) ? 'true' : 'false';
})(inputs, outputs);
```

Output variables: `issue_number`, `case_sys_id`, `cr_number`, `cr_sys_id`, `cr_state`, `assigned_to`, `planned_start`, `planned_end`, `action`, `account_name`, `should_send`

#### 8.5 If Condition → `should_send` is `true`

#### 8.6 Script Step 2 — Call GitHub (inside **then**)

Input variables from Script step 1 outputs.

Script:

```javascript
(function execute(inputs, outputs) {
  try {
    var config = JSON.parse(gs.getProperty('github.dispatch.config'))[inputs.account_name];
    if (!config) { outputs.http_status = 'config_missing'; outputs.success = 'false'; return; }
    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch_cr');
    rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches');
    rm.setRequestHeader('Authorization', 'token ' + config.token);
    rm.setRequestBody(JSON.stringify({
      event_type: 'servicenow-cr-update',
      client_payload: {
        github_issue_number: inputs.issue_number,
        cr_number:           inputs.cr_number,
        cr_sys_id:           inputs.cr_sys_id,
        cr_state:            inputs.cr_state,
        assigned_to:         inputs.assigned_to,
        case_sys_id:         inputs.case_sys_id,
        planned_start:       inputs.planned_start,
        planned_end:         inputs.planned_end,
        action:              inputs.action
      }
    }));
    var status = rm.execute().getStatusCode();
    outputs.http_status = status + '';
    outputs.success = (status == 204 || status == 200) ? 'true' : 'false';
  } catch (ex) {
    outputs.http_status = 'error'; outputs.success = 'false';
    gs.error('GitHub CR dispatch exception: ' + ex.message);
  }
})(inputs, outputs);
```

Output variables: `http_status`, `success`

#### 8.7 Save and Activate

---

## Troubleshooting

### Flow logs

**All > Process Automation > Flow Designer → Executions tab**

Find the flow name, click the execution row, expand each Script step to see input/output values and `http_status`.

### Common errors

| Symptom | Cause | Fix |
|---|---|---|
| `config_missing` in flow logs | Property missing or JSON is invalid | Verify `github.dispatch.config` exists and value parses as valid JSON |
| `No config entry for account "..."` | Account name mismatch | Copy account name exactly from ServiceNow — check spaces, capitalisation, special characters |
| `http_status: 401` | Token expired or wrong scope | Regenerate PAT with `repo` scope (classic) or `Contents: Read and write` (fine-grained) |
| `http_status: 404` | Wrong owner or repo | Verify against `github.com/{owner}/{repo}` |
| `http_status: 422` | Workflow file not on default branch | Push the workflow file to the default branch |
| Changes not taking effect | Cached property value | Enable **Ignore cache** on the `github.dispatch.config` property record |
| ServiceNow case fields blank | Display name mismatch in `servicenow-config.yml` | Right-click the field in SN → **Show value** and copy exactly |
| Case creation returns 500 | Script error in Scripted REST API | Check **System Log > All** for GlideRecord errors |
