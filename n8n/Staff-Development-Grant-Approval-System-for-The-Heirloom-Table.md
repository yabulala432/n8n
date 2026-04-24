# Staff Development Grant Approval System for The Heirloom Table

## 1. Create the Workflows

1. Log in to n8n.
2. Click `Create Workflow`.
3. Rename the first workflow to `The Heirloom Table - Discovery Grant Intake`.
4. Click `Save`.
5. Create a second workflow.
6. Rename it to `The Heirloom Table - Discovery Grant Decision Processor`.
7. Click `Save`.
8. Create a third workflow.
9. Rename it to `The Heirloom Table - Discovery Grant Error Alert`.
10. Click `Save`.

## 2. Prepare the Google Sheet

Create a Google Sheet named `Discovery Grant Requests`.

Use one sheet tab named `Requests`.

Add these columns in row 1:

- `request_id`
- `submitted_at`
- `employee_name`
- `employee_email`
- `department`
- `grant_type`
- `event_or_supplier`
- `request_reason`
- `request_amount`
- `approval_path`
- `status`
- `head_chef_decision`
- `gm_decision`
- `head_chef_notified_at`
- `gm_notified_at`
- `decision_notes`
- `final_decision_at`

Recommended status values:

- `Pending`
- `Approved`
- `Rejected`

Recommended approval decision values:

- `Pending`
- `Approved`
- `Rejected`
- `Not Required`

## 3. Main Workflow Node Order

`Typeform Trigger - Discovery Grant Form`  
-> `Set - Normalize Request`  
-> `Google Sheets - Log Pending Request`  
-> `If - Amount Over 150`  
-> `Gmail - Notify Head Chef Only`  
-> `Gmail - Notify Head Chef Large Grant`  
-> `Google Sheets - Mark Head Chef Notified`  
-> `Gmail - Notify General Manager Large Grant`  
-> `Google Sheets - Mark GM Notified`

## 4. Build the Main Workflow

### Node 1: Typeform Trigger - Discovery Grant Form

Purpose:  
Start the workflow when an employee submits the public grant request form.

Add Node:  
Click `+` and search for `Typeform Trigger`.

Settings:

- Connect your Typeform credential.
- Select the staff grant request form.
- Activate the trigger webhook in n8n.

Suggested Typeform fields:

- `employee_name`
- `employee_email`
- `department`
- `grant_type`
- `event_or_supplier`
- `request_reason`
- `request_amount`

Output:  
One item containing the form submission.

### Node 2: Set - Normalize Request

Purpose:  
Clean the incoming form data, generate consistent fields, and decide the approval path.

Add Node:  
Click `+` after the Typeform trigger and search for `Set` or `Edit Fields`.

Settings:

- Turn on `Keep Only Set`.

Fields:

- `request_id` = `DG-{{$now.toFormat('yyyyLLddHHmmss')}}`
- `submitted_at` = `{{$now}}`
- `employee_name` = `{{$json.employee_name}}`
- `employee_email` = `{{$json.employee_email}}`
- `department` = `{{$json.department}}`
- `grant_type` = `{{$json.grant_type}}`
- `event_or_supplier` = `{{$json.event_or_supplier}}`
- `request_reason` = `{{$json.request_reason}}`
- `request_amount` = `{{ Number($json.request_amount) }}`
- `approval_path` = `{{ Number($json.request_amount) > 150 ? 'Head Chef + General Manager' : 'Head Chef Only' }}`
- `status` = `Pending`
- `head_chef_decision` = `Pending`
- `gm_decision` = `{{ Number($json.request_amount) > 150 ? 'Pending' : 'Not Required' }}`
- `head_chef_notified_at` = ``
- `gm_notified_at` = ``
- `decision_notes` = ``
- `final_decision_at` = ``

Expressions:

- `{{$now}}`
- `{{$now.toFormat('yyyyLLddHHmmss')}}`
- `{{ Number($json.request_amount) }}`
- `{{ Number($json.request_amount) > 150 ? 'Head Chef + General Manager' : 'Head Chef Only' }}`

Output:  
One clean request object ready for logging and routing.

### Node 3: Google Sheets - Log Pending Request

Purpose:  
Write every request into Google Sheets immediately with status `Pending`.

Add Node:  
Click `+` after `Set - Normalize Request` and search for `Google Sheets`.

Settings:

- Credential: your Google Sheets credential
- Resource: `Sheet Within Document`
- Operation: `Append Row`
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Mapping mode: map each field manually

Map these columns:

- `request_id` = `{{$json.request_id}}`
- `submitted_at` = `{{$json.submitted_at}}`
- `employee_name` = `{{$json.employee_name}}`
- `employee_email` = `{{$json.employee_email}}`
- `department` = `{{$json.department}}`
- `grant_type` = `{{$json.grant_type}}`
- `event_or_supplier` = `{{$json.event_or_supplier}}`
- `request_reason` = `{{$json.request_reason}}`
- `request_amount` = `{{$json.request_amount}}`
- `approval_path` = `{{$json.approval_path}}`
- `status` = `Pending`
- `head_chef_decision` = `Pending`
- `gm_decision` = `{{$json.gm_decision}}`
- `head_chef_notified_at` = ``
- `gm_notified_at` = ``
- `decision_notes` = ``
- `final_decision_at` = ``

Important:  
Turn on `Include Row Number` if your Google Sheets node supports it. This makes later updates easier.

Output:  
The pending request is stored in the sheet.

### Node 4: If - Amount Over 150

Purpose:  
Split the workflow into the correct approval path.

Add Node:  
Click `+` after the Google Sheets node and search for `If`.

Settings:

- Condition type: `Number`

Condition:

- Left value = `{{$json.request_amount}}`
- Operation = `Larger`
- Right value = `150`

Output:

- `true` = grant requires both Head Chef and General Manager
- `false` = grant requires Head Chef only

### Node 5: Gmail - Notify Head Chef Only

Purpose:  
Email the Head Chef when the request amount is under or equal to `$150`.

Add Node:  
Click `+` from the `false` output of `If - Amount Over 150` and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `headchef@heirloomtable.com`
- Subject: `Discovery Grant Approval Needed - {{$json.employee_name}} - ${{$json.request_amount}}`

Email body:

```html
<p>A new Discovery Grant request needs your review.</p>
<p><strong>Request ID:</strong> {{$json.request_id}}</p>
<p><strong>Employee:</strong> {{$json.employee_name}}</p>
<p><strong>Email:</strong> {{$json.employee_email}}</p>
<p><strong>Department:</strong> {{$json.department}}</p>
<p><strong>Grant Type:</strong> {{$json.grant_type}}</p>
<p><strong>Event or Supplier:</strong> {{$json.event_or_supplier}}</p>
<p><strong>Reason:</strong> {{$json.request_reason}}</p>
<p><strong>Amount:</strong> ${{$json.request_amount}}</p>
<p>Please open the Google Sheet and update <strong>head_chef_decision</strong> to Approved or Rejected.</p>
```

Output:  
Head Chef receives the approval request.

### Node 6: Gmail - Notify Head Chef Large Grant

Purpose:  
Email the Head Chef when the grant is above `$150`.

Add Node:  
Click `+` from the `true` output of `If - Amount Over 150` and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `headchef@heirloomtable.com`
- Subject: `Discovery Grant Dual Approval Needed - {{$json.employee_name}} - ${{$json.request_amount}}`

Email body:

```html
<p>A Discovery Grant request needs dual approval.</p>
<p><strong>Request ID:</strong> {{$json.request_id}}</p>
<p><strong>Employee:</strong> {{$json.employee_name}}</p>
<p><strong>Department:</strong> {{$json.department}}</p>
<p><strong>Grant Type:</strong> {{$json.grant_type}}</p>
<p><strong>Event or Supplier:</strong> {{$json.event_or_supplier}}</p>
<p><strong>Reason:</strong> {{$json.request_reason}}</p>
<p><strong>Amount:</strong> ${{$json.request_amount}}</p>
<p>This request requires approval from both the Head Chef and the General Manager.</p>
<p>Please update <strong>head_chef_decision</strong> in the Google Sheet.</p>
```

Output:  
Head Chef receives the large-grant approval request.

### Node 7: Google Sheets - Mark Head Chef Notified

Purpose:  
Record when the Head Chef email was sent.

Add Node:  
Click `+` after the appropriate Head Chef Gmail node and search for `Google Sheets`.

Settings:

- Resource: `Sheet Within Document`
- Operation: `Update Row`
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Match column: `request_id`
- Match value: `{{$json.request_id}}`

Fields to update:

- `head_chef_notified_at` = `{{$now}}`

Output:  
The request row shows when the Head Chef was notified.

### Node 8: Gmail - Notify General Manager Large Grant

Purpose:  
Email the General Manager only for requests above `$150`.

Add Node:  
Click `+` after `Google Sheets - Mark Head Chef Notified` on the large-grant branch and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `gm@heirloomtable.com`
- Subject: `Discovery Grant GM Approval Needed - {{$json.employee_name}} - ${{$json.request_amount}}`

Email body:

```html
<p>A Discovery Grant request requires your approval.</p>
<p><strong>Request ID:</strong> {{$json.request_id}}</p>
<p><strong>Employee:</strong> {{$json.employee_name}}</p>
<p><strong>Department:</strong> {{$json.department}}</p>
<p><strong>Grant Type:</strong> {{$json.grant_type}}</p>
<p><strong>Event or Supplier:</strong> {{$json.event_or_supplier}}</p>
<p><strong>Reason:</strong> {{$json.request_reason}}</p>
<p><strong>Amount:</strong> ${{$json.request_amount}}</p>
<p>Please update <strong>gm_decision</strong> in the Google Sheet after review.</p>
```

Output:  
General Manager receives the approval request.

### Node 9: Google Sheets - Mark GM Notified

Purpose:  
Record when the General Manager email was sent.

Add Node:  
Click `+` after the GM Gmail node and search for `Google Sheets`.

Settings:

- Resource: `Sheet Within Document`
- Operation: `Update Row`
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Match column: `request_id`
- Match value: `{{$json.request_id}}`

Fields to update:

- `gm_notified_at` = `{{$now}}`

Output:  
The request row shows when the General Manager was notified.

## 5. Decision Processor Workflow Node Order

`Google Sheets Trigger - Request Decision Updates`  
-> `Set - Normalize Decision State`  
-> `If - Status Still Pending`  
-> `If - Head Chef Rejected`  
-> `Google Sheets - Finalize Rejected by Head Chef`  
-> `Gmail - Notify Employee Rejected by Head Chef`  
-> `If - GM Rejected`  
-> `Google Sheets - Finalize Rejected by GM`  
-> `Gmail - Notify Employee Rejected by GM`  
-> `If - Small Grant Approved`  
-> `Google Sheets - Finalize Small Grant Approved`  
-> `Gmail - Notify Employee Approved Small Grant`  
-> `If - Large Grant Fully Approved`  
-> `Google Sheets - Finalize Large Grant Approved`  
-> `Gmail - Notify Employee Approved Large Grant`

## 6. Build the Decision Processor Workflow

### Node 1: Google Sheets Trigger - Request Decision Updates

Purpose:  
Watch for changes to the approvals in the `Requests` sheet.

Add Node:  
Click `+` and search for `Google Sheets Trigger`.

Settings:

- Credential: your Google Sheets credential
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Trigger on:
  - `Updated Row`

Output:  
An item is produced whenever a request row changes.

### Node 2: Set - Normalize Decision State

Purpose:  
Prepare clean values for approval logic.

Add Node:  
Click `+` after the Google Sheets trigger and search for `Set`.

Settings:

- Turn on `Keep Only Set`.

Fields:

- `row_number` = `{{$json.rowNumber}}`
- `request_id` = `{{$json.request_id}}`
- `employee_name` = `{{$json.employee_name}}`
- `employee_email` = `{{$json.employee_email}}`
- `request_amount` = `{{ Number($json.request_amount) }}`
- `status` = `{{$json.status}}`
- `head_chef_decision` = `{{$json.head_chef_decision}}`
- `gm_decision` = `{{$json.gm_decision}}`
- `approval_path` = `{{$json.approval_path}}`

Output:  
Decision-ready row data.

### Node 3: If - Status Still Pending

Purpose:  
Stop the workflow from reprocessing rows that are already finalized.

Add Node:  
Click `+` after `Set - Normalize Decision State` and search for `If`.

Condition:

- Left value = `{{$json.status}}`
- Operation = `Equal`
- Right value = `Pending`

Output:

- `true` = continue evaluating decisions
- `false` = stop

### Node 4: If - Head Chef Rejected

Purpose:  
Reject immediately if the Head Chef has rejected the request.

Add Node:  
Click `+` from the `true` output of `If - Status Still Pending` and search for `If`.

Condition:

- Left value = `{{$json.head_chef_decision}}`
- Operation = `Equal`
- Right value = `Rejected`

### Node 5: Google Sheets - Finalize Rejected by Head Chef

Purpose:  
Update the Google Sheet with the final rejected status.

Add Node:  
Click `+` from the `true` output of `If - Head Chef Rejected` and search for `Google Sheets`.

Settings:

- Resource: `Sheet Within Document`
- Operation: `Update Row`
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Match column: `request_id`
- Match value: `{{$json.request_id}}`

Fields to update:

- `status` = `Rejected`
- `decision_notes` = `Rejected by Head Chef`
- `final_decision_at` = `{{$now}}`

### Node 6: Gmail - Notify Employee Rejected by Head Chef

Purpose:  
Tell the employee the request was declined.

Add Node:  
Click `+` after `Google Sheets - Finalize Rejected by Head Chef` and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `{{$json.employee_email}}`
- Subject: `Your Discovery Grant Request Was Not Approved`

Email body:

```html
<p>Hello {{$json.employee_name}},</p>
<p>Your Discovery Grant request <strong>{{$json.request_id}}</strong> was reviewed and was not approved.</p>
<p>Decision: Rejected by Head Chef</p>
<p>If you need more detail, please contact HR.</p>
```

### Node 7: If - GM Rejected

Purpose:  
If the Head Chef did not reject it, check whether the General Manager rejected it.

Add Node:  
Click `+` from the `false` output of `If - Head Chef Rejected` and search for `If`.

Condition:

- Left value = `{{$json.gm_decision}}`
- Operation = `Equal`
- Right value = `Rejected`

### Node 8: Google Sheets - Finalize Rejected by GM

Purpose:  
Set the request status to rejected when the GM rejects the request.

Add Node:  
Click `+` from the `true` output of `If - GM Rejected` and search for `Google Sheets`.

Settings:

- Resource: `Sheet Within Document`
- Operation: `Update Row`
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Match column: `request_id`
- Match value: `{{$json.request_id}}`

Fields to update:

- `status` = `Rejected`
- `decision_notes` = `Rejected by General Manager`
- `final_decision_at` = `{{$now}}`

### Node 9: Gmail - Notify Employee Rejected by GM

Purpose:  
Tell the employee the request was rejected by the General Manager.

Add Node:  
Click `+` after `Google Sheets - Finalize Rejected by GM` and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `{{$json.employee_email}}`
- Subject: `Your Discovery Grant Request Was Not Approved`

Email body:

```html
<p>Hello {{$json.employee_name}},</p>
<p>Your Discovery Grant request <strong>{{$json.request_id}}</strong> was reviewed and was not approved.</p>
<p>Decision: Rejected by General Manager</p>
<p>Please contact HR if you would like next-step guidance.</p>
```

### Node 10: If - Small Grant Approved

Purpose:  
Approve the request when the amount is `$150` or less and the Head Chef has approved it.

Add Node:  
Click `+` from the `false` output of `If - GM Rejected` and search for `If`.

Condition:

- Left value = `{{ $json.request_amount <= 150 && $json.head_chef_decision === 'Approved' }}`
- Operation = `Equal`
- Right value = `true`

### Node 11: Google Sheets - Finalize Small Grant Approved

Purpose:  
Set the request to approved for the single-approval path.

Add Node:  
Click `+` from the `true` output of `If - Small Grant Approved` and search for `Google Sheets`.

Settings:

- Resource: `Sheet Within Document`
- Operation: `Update Row`
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Match column: `request_id`
- Match value: `{{$json.request_id}}`

Fields to update:

- `status` = `Approved`
- `decision_notes` = `Approved by Head Chef`
- `final_decision_at` = `{{$now}}`

### Node 12: Gmail - Notify Employee Approved Small Grant

Purpose:  
Email the employee when a small request is approved.

Add Node:  
Click `+` after `Google Sheets - Finalize Small Grant Approved` and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `{{$json.employee_email}}`
- Subject: `Your Discovery Grant Request Was Approved`

Email body:

```html
<p>Hello {{$json.employee_name}},</p>
<p>Your Discovery Grant request <strong>{{$json.request_id}}</strong> has been approved.</p>
<p>Approval path: Head Chef only</p>
<p>HR can now continue with the next internal step.</p>
```

### Node 13: If - Large Grant Fully Approved

Purpose:  
Approve the request when it is above `$150` and both approvers have approved it.

Add Node:  
Click `+` from the `false` output of `If - Small Grant Approved` and search for `If`.

Condition:

- Left value = `{{ $json.request_amount > 150 && $json.head_chef_decision === 'Approved' && $json.gm_decision === 'Approved' }}`
- Operation = `Equal`
- Right value = `true`

### Node 14: Google Sheets - Finalize Large Grant Approved

Purpose:  
Set the request to approved when both approvers have signed off.

Add Node:  
Click `+` from the `true` output of `If - Large Grant Fully Approved` and search for `Google Sheets`.

Settings:

- Resource: `Sheet Within Document`
- Operation: `Update Row`
- Document: `Discovery Grant Requests`
- Sheet: `Requests`
- Match column: `request_id`
- Match value: `{{$json.request_id}}`

Fields to update:

- `status` = `Approved`
- `decision_notes` = `Approved by Head Chef and General Manager`
- `final_decision_at` = `{{$now}}`

### Node 15: Gmail - Notify Employee Approved Large Grant

Purpose:  
Email the employee when the large request reaches full approval.

Add Node:  
Click `+` after `Google Sheets - Finalize Large Grant Approved` and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `{{$json.employee_email}}`
- Subject: `Your Discovery Grant Request Was Approved`

Email body:

```html
<p>Hello {{$json.employee_name}},</p>
<p>Your Discovery Grant request <strong>{{$json.request_id}}</strong> has been approved.</p>
<p>Approval path: Head Chef and General Manager</p>
<p>HR will follow up with any final reimbursement or scheduling details.</p>
```

Important:  
If none of the final approval or rejection conditions are met, let the workflow end with no action. That means the request stays `Pending` until the needed decisions are entered in the sheet.

## 7. Error Workflow Node Order

`Error Trigger - Discovery Grant Failures`  
-> `Set - Error Details`  
-> `Gmail - Notify HR Manager`

## 8. Build the Error Workflow

### Node 1: Error Trigger - Discovery Grant Failures

Purpose:  
Catch workflow failures such as Google Sheets connection issues or email send errors.

Add Node:  
Click `+` and search for `Error Trigger`.

Settings:  
No special setup is needed after the node is added.

### Node 2: Set - Error Details

Purpose:  
Prepare a clean summary for HR.

Add Node:  
Click `+` after the Error Trigger and search for `Set`.

Settings:

- Turn on `Keep Only Set`.

Fields:

- `workflow_name` = `{{$json.workflow.name}}`
- `failed_node` = `{{$json.execution.lastNodeExecuted}}`
- `error_message` = `{{$json.execution.error.message}}`
- `failed_at` = `{{$now}}`

### Node 3: Gmail - Notify HR Manager

Purpose:  
Alert HR when the automation cannot send an email or cannot reach Google Sheets.

Add Node:  
Click `+` after `Set - Error Details` and search for `Gmail`.

Settings:

- Resource: `Message`
- Operation: `Send`
- To: `hr@heirloomtable.com`
- Subject: `Discovery Grant Workflow Error Alert`

Email body:

```html
<p>The Discovery Grant automation needs attention.</p>
<p><strong>Workflow:</strong> {{$json.workflow_name}}</p>
<p><strong>Failed Node:</strong> {{$json.failed_node}}</p>
<p><strong>Error:</strong> {{$json.error_message}}</p>
<p><strong>Time:</strong> {{$json.failed_at}}</p>
```

## 9. Link the Error Workflow

1. Open `The Heirloom Table - Discovery Grant Intake`.
2. Open workflow `Settings`.
3. Set the error workflow to `The Heirloom Table - Discovery Grant Error Alert`.
4. Save the workflow.
5. Repeat the same steps for `The Heirloom Table - Discovery Grant Decision Processor`.

## 10. Testing Steps

1. Submit a sample Typeform request for `$120`.
2. Confirm a new Google Sheets row appears with `Pending` status and `gm_decision` set to `Not Required`.
3. Confirm the Head Chef receives one approval email.
4. In Google Sheets, change `head_chef_decision` to `Approved`.
5. Confirm the status changes to `Approved`.
6. Confirm the employee receives the final approval email.
7. Submit a second Typeform request for `$240`.
8. Confirm a new Google Sheets row appears with both approval decisions set to `Pending`.
9. Confirm the Head Chef and General Manager both receive emails.
10. Change only `head_chef_decision` to `Approved`.
11. Confirm the row stays `Pending`.
12. Change `gm_decision` to `Approved`.
13. Confirm the row changes to `Approved`.
14. Submit one more test and set either approver field to `Rejected`.
15. Confirm the row changes to `Rejected` and the employee receives the rejection email.

## 11. Optional Google Forms Version

If you want Google Forms instead of Typeform:

1. Build the public form in Google Forms.
2. Link it to a Google Sheets response sheet.
3. Replace `Typeform Trigger - Discovery Grant Form` with `Google Sheets Trigger - Google Form Responses`.
4. Map the Google Forms column names into `Set - Normalize Request`.
5. Keep the rest of the workflow exactly the same.

## 12. Suggested Node Names

Use these exact names to keep the canvas clean:

- `Typeform Trigger - Discovery Grant Form`
- `Set - Normalize Request`
- `Google Sheets - Log Pending Request`
- `If - Amount Over 150`
- `Gmail - Notify Head Chef Only`
- `Gmail - Notify Head Chef Large Grant`
- `Google Sheets - Mark Head Chef Notified`
- `Gmail - Notify General Manager Large Grant`
- `Google Sheets - Mark GM Notified`
- `Google Sheets Trigger - Request Decision Updates`
- `Set - Normalize Decision State`
- `If - Status Still Pending`
- `If - Head Chef Rejected`
- `Google Sheets - Finalize Rejected by Head Chef`
- `Gmail - Notify Employee Rejected by Head Chef`
- `If - GM Rejected`
- `Google Sheets - Finalize Rejected by GM`
- `Gmail - Notify Employee Rejected by GM`
- `If - Small Grant Approved`
- `Google Sheets - Finalize Small Grant Approved`
- `Gmail - Notify Employee Approved Small Grant`
- `If - Large Grant Fully Approved`
- `Google Sheets - Finalize Large Grant Approved`
- `Gmail - Notify Employee Approved Large Grant`
- `Error Trigger - Discovery Grant Failures`
- `Set - Error Details`
- `Gmail - Notify HR Manager`

## 13. Final Result

This setup gives The Heirloom Table:

- public form intake
- automatic Google Sheets logging
- amount-based approval routing
- employee notification after approval or rejection
- simple error alerts to HR

It is also easy to demo because the approval step can be simulated directly by editing the `head_chef_decision` and `gm_decision` columns in Google Sheets.
