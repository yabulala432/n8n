# Evergreen Turf & Aquatics: Multi-Stage Field Service Incident & Completion Intelligence Workflow

Prepared for: Evergreen Turf & Aquatics  
Document type: n8n implementation guide  
Purpose: Exact node-by-node build instructions for a field report workflow that validates weather claims, routes critical incidents to Slack, emails homeowners on routine completions, logs every outcome to Google Sheets, and alerts the Project Manager when data or weather checks fail.

## 1. Recommended Build Stack

To keep the implementation clean and easy to prove, this guide uses:

- `Webhook` for public intake
- `OpenWeatherMap` via `HTTP Request` for weather validation
- `Slack` for critical incident alerts
- `Gmail` for homeowner notifications and PM alerts
- `Google Sheets` for the master log
- `Error Trigger` in a second workflow for unexpected failures

You can swap `Google Sheets` for `Airtable` later, but Google Sheets is the fastest option for a clean demo and weekly PM review.

---

## 2. What You Will Build

You will create two workflows:

1. `Evergreen - Field Report Intelligence`
2. `Evergreen - Field Report Error Handler`

### Main workflow logic

`Webhook - Field Report Intake`  
`-> Set - Normalize Report`  
`-> IF - Required Fields Present`

Invalid path:  
`-> Set - Invalid Report Context`  
`-> Gmail - PM Invalid Input Alert`  
`-> Set - Finalize Invalid Log`  
`-> Google Sheets - Append Report Log`

Valid path:  
`-> HTTP Request - OpenWeather Current Conditions`  
`-> IF - Weather Data Available`

Weather success path:  
`-> Set - Weather Success Context`  
`-> Switch - Route Report`

Weather failure path:  
`-> Set - Weather Failure Context`  
`-> Gmail - PM Weather API Failure`  
`-> Switch - Route Report`

Critical branch:  
`-> Slack - Critical Incident Alert`  
`-> Set - Finalize Critical Log`  
`-> Google Sheets - Append Report Log`

Routine branch:  
`-> Gmail - Homeowner Completion Summary`  
`-> Set - Finalize Routine Log`  
`-> Google Sheets - Append Report Log`

### Error workflow logic

`Error Trigger - Unexpected Workflow Failure`  
`-> Set - Format Failure Alert`  
`-> Gmail - PM Unexpected Workflow Failure`

---

## 3. Before You Open n8n

Create these credentials first:

- `OpenWeatherMap API key`
- `Slack credential` for the workspace and target incident channel
- `Gmail credential` for outbound email
- `Google Sheets credential`

Create one Google Sheet named `Evergreen Field Report Log` and one tab named `Reports`.

Use these exact columns in row 1:

```text
report_id
received_at
routing_type
technician_name
homeowner_name
homeowner_email
property_address
city
state
zip
service_category
job_summary
incident_category
incident_details
weather_claimed
weather_claim_notes
weather_status
openweather_summary
temp_c
wind_mps
rain_1h
weather_validation_result
notification_channel
notification_status
pm_alert_status
execution_status
```

This exact column naming is important because the final `Set` nodes will produce matching keys, which keeps your Google Sheets mapping very clean.

---

## 4. Test Payloads to Prepare

Use these two payloads when testing from the Webhook node.

### Routine completion test

```json
{
  "report_type": "routine_completion",
  "technician_name": "Jordan Ellis",
  "homeowner_name": "Ava Monroe",
  "homeowner_email": "ava.monroe@example.com",
  "property_address": "118 Cedar Glen Dr",
  "city": "Austin",
  "state": "TX",
  "zip": "78738",
  "service_category": "Pool Maintenance",
  "job_summary": "Completed weekly pool balancing and filter rinse.",
  "incident_category": "",
  "incident_details": "",
  "weather_claimed": "yes",
  "weather_claim_notes": "Delayed 45 minutes due to heavy rain."
}
```

### Critical incident test

```json
{
  "report_type": "critical_incident",
  "technician_name": "Maya Brooks",
  "homeowner_name": "Samuel Kent",
  "homeowner_email": "samuel.kent@example.com",
  "property_address": "41 Laurel Point Rd",
  "city": "Tampa",
  "state": "FL",
  "zip": "33602",
  "service_category": "Landscape Irrigation",
  "job_summary": "Site stabilized and water isolated pending repair.",
  "incident_category": "Equipment Failure",
  "incident_details": "Main irrigation controller failed and left zone 3 flooding near driveway.",
  "weather_claimed": "no",
  "weather_claim_notes": ""
}
```

---

## 5. Main Workflow Build: Exact Node-by-Node Steps

Create a new workflow and name it:

`Evergreen - Field Report Intelligence`

### Node 1: Webhook - Field Report Intake

### Purpose

This node receives technician submissions from a public form, portal, or direct POST request.

### How to add it

1. Create a new workflow.
2. Click `Add first step`.
3. Search for `Webhook`.
4. Add the node.
5. Rename it to `Webhook - Field Report Intake`.

### Settings

| Setting | Value |
|---|---|
| `HTTP Method` | `POST` |
| `Path` | `evergreen-field-report` |
| `Authentication` | `None` for testing, then `Header Auth` in production |
| `Respond` | `Immediately` |
| `Response Code` | `200` |

### Notes

- During testing, use the test URL from n8n.
- In production, switch your form or app to the production webhook URL.

### Expected input

The workflow expects either:

- flat JSON fields at the root, or
- form JSON under `$json.body`

This guide supports both.

---

### Node 2: Set - Normalize Report

### Purpose

This node creates one consistent data structure, regardless of how the payload was submitted.

### How to add it

1. Click the `+` after `Webhook - Field Report Intake`.
2. Search for `Set`.
3. Add the node.
4. Rename it to `Set - Normalize Report`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `true` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `report_id` | `String` | `={{ 'ETA-' + $now.toFormat('yyyyLLddHHmmss') + '-' + Math.floor(Math.random() * 1000) }}` |
| `received_at` | `String` | `={{ $now.toISO() }}` |
| `routing_type` | `String` | `={{ ['critical_incident','critical','incident'].includes((($json.body?.report_type || $json.report_type || '') + '').trim().toLowerCase().replace(/\s+/g, '_')) || ['equipment failure','property damage','chemical spill'].includes((($json.body?.incident_category || $json.incident_category || '') + '').trim().toLowerCase()) ? 'critical_incident' : 'routine_completion' }}` |
| `technician_name` | `String` | `={{ ($json.body?.technician_name || $json.technician_name || '').toString().trim() }}` |
| `homeowner_name` | `String` | `={{ ($json.body?.homeowner_name || $json.homeowner_name || '').toString().trim() }}` |
| `homeowner_email` | `String` | `={{ ($json.body?.homeowner_email || $json.homeowner_email || '').toString().trim().toLowerCase() }}` |
| `property_address` | `String` | `={{ ($json.body?.property_address || $json.property_address || '').toString().trim() }}` |
| `city` | `String` | `={{ ($json.body?.city || $json.city || '').toString().trim() }}` |
| `state` | `String` | `={{ ($json.body?.state || $json.state || '').toString().trim() }}` |
| `zip` | `String` | `={{ ($json.body?.zip || $json.zip || '').toString().trim() }}` |
| `service_category` | `String` | `={{ ($json.body?.service_category || $json.service_category || '').toString().trim() }}` |
| `job_summary` | `String` | `={{ ($json.body?.job_summary || $json.job_summary || '').toString().trim() }}` |
| `incident_category` | `String` | `={{ ($json.body?.incident_category || $json.incident_category || '').toString().trim() }}` |
| `incident_details` | `String` | `={{ ($json.body?.incident_details || $json.incident_details || '').toString().trim() }}` |
| `weather_claimed` | `String` | `={{ ((($json.body?.weather_claimed || $json.weather_claimed || 'no') + '').trim().toLowerCase() === 'yes') ? 'yes' : 'no' }}` |
| `weather_claim_notes` | `String` | `={{ ($json.body?.weather_claim_notes || $json.weather_claim_notes || '').toString().trim() }}` |

### What this node does

- Standardizes the payload
- Detects whether the report should be treated as `critical_incident` or `routine_completion`
- Normalizes email, address, service, and weather claim values

---

### Node 3: IF - Required Fields Present

### Purpose

This node prevents malformed submissions from quietly entering the workflow.

### How to add it

1. Click the `+` after `Set - Normalize Report`.
2. Search for `If`.
3. Add the node.
4. Rename it to `IF - Required Fields Present`.

### Condition mode

Use a single expression condition.

### Expression

```javascript
={{ 
  !!$json.technician_name &&
  !!$json.property_address &&
  !!$json.city &&
  !!$json.state &&
  !!$json.zip &&
  !!$json.service_category &&
  !!$json.job_summary &&
  (
    ($json.routing_type === 'critical_incident' && !!$json.incident_details) ||
    ($json.routing_type === 'routine_completion' && !!$json.homeowner_email)
  )
}}
```

### True output

Use the `true` branch for valid records.

### False output

Use the `false` branch for malformed records.

---

## 6. Invalid Input Branch

This branch handles malformed submissions without crashing the workflow.

### Node 4A: Set - Invalid Report Context

### Purpose

Adds a final status package for bad input before notifying the PM.

### How to add it

1. From the `false` output of `IF - Required Fields Present`, click `+`.
2. Add a `Set` node.
3. Rename it to `Set - Invalid Report Context`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `false` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `weather_status` | `String` | `not_checked` |
| `openweather_summary` | `String` | `Skipped because input was invalid` |
| `temp_c` | `String` | `` |
| `wind_mps` | `String` | `` |
| `rain_1h` | `String` | `` |
| `weather_validation_result` | `String` | `skipped_invalid_input` |
| `notification_channel` | `String` | `pm_email` |
| `notification_status` | `String` | `pm_invalid_input_email_sent` |
| `pm_alert_status` | `String` | `invalid_input_alert_sent` |
| `execution_status` | `String` | `rejected_invalid_input` |

---

### Node 5A: Gmail - PM Invalid Input Alert

### Purpose

Emails the Project Manager so malformed submissions get reviewed instead of disappearing.

### How to add it

1. Click `+` after `Set - Invalid Report Context`.
2. Search for `Gmail`.
3. Add the node.
4. Rename it to `Gmail - PM Invalid Input Alert`.

### Settings

| Setting | Value |
|---|---|
| `Resource` | `Message` |
| `Operation` | `Send` |
| `To` | `pm@evergreenturfandaquatics.com` |
| `Subject` | `Invalid field report received: {{$json.report_id}}` |

### Email body

Use HTML mode and paste:

```html
<p>A field report was rejected because required data was missing.</p>
<p><strong>Report ID:</strong> {{$json.report_id}}</p>
<p><strong>Routing Type:</strong> {{$json.routing_type}}</p>
<p><strong>Technician:</strong> {{$json.technician_name}}</p>
<p><strong>Address:</strong> {{$json.property_address}}, {{$json.city}}, {{$json.state}} {{$json.zip}}</p>
<p><strong>Homeowner Email:</strong> {{$json.homeowner_email}}</p>
<p><strong>Incident Details:</strong> {{$json.incident_details}}</p>
<p><strong>Job Summary:</strong> {{$json.job_summary}}</p>
<p>Please review the source form or webhook payload.</p>
```

---

### Node 6A: Set - Finalize Invalid Log

### Purpose

Shapes the output to exactly match the Google Sheet column headers.

### How to add it

1. Click `+` after `Gmail - PM Invalid Input Alert`.
2. Add a `Set` node.
3. Rename it to `Set - Finalize Invalid Log`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `true` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `report_id` | `String` | `={{ $('Set - Invalid Report Context').item.json.report_id }}` |
| `received_at` | `String` | `={{ $('Set - Invalid Report Context').item.json.received_at }}` |
| `routing_type` | `String` | `={{ $('Set - Invalid Report Context').item.json.routing_type }}` |
| `technician_name` | `String` | `={{ $('Set - Invalid Report Context').item.json.technician_name }}` |
| `homeowner_name` | `String` | `={{ $('Set - Invalid Report Context').item.json.homeowner_name }}` |
| `homeowner_email` | `String` | `={{ $('Set - Invalid Report Context').item.json.homeowner_email }}` |
| `property_address` | `String` | `={{ $('Set - Invalid Report Context').item.json.property_address }}` |
| `city` | `String` | `={{ $('Set - Invalid Report Context').item.json.city }}` |
| `state` | `String` | `={{ $('Set - Invalid Report Context').item.json.state }}` |
| `zip` | `String` | `={{ $('Set - Invalid Report Context').item.json.zip }}` |
| `service_category` | `String` | `={{ $('Set - Invalid Report Context').item.json.service_category }}` |
| `job_summary` | `String` | `={{ $('Set - Invalid Report Context').item.json.job_summary }}` |
| `incident_category` | `String` | `={{ $('Set - Invalid Report Context').item.json.incident_category }}` |
| `incident_details` | `String` | `={{ $('Set - Invalid Report Context').item.json.incident_details }}` |
| `weather_claimed` | `String` | `={{ $('Set - Invalid Report Context').item.json.weather_claimed }}` |
| `weather_claim_notes` | `String` | `={{ $('Set - Invalid Report Context').item.json.weather_claim_notes }}` |
| `weather_status` | `String` | `={{ $('Set - Invalid Report Context').item.json.weather_status }}` |
| `openweather_summary` | `String` | `={{ $('Set - Invalid Report Context').item.json.openweather_summary }}` |
| `temp_c` | `String` | `={{ $('Set - Invalid Report Context').item.json.temp_c }}` |
| `wind_mps` | `String` | `={{ $('Set - Invalid Report Context').item.json.wind_mps }}` |
| `rain_1h` | `String` | `={{ $('Set - Invalid Report Context').item.json.rain_1h }}` |
| `weather_validation_result` | `String` | `={{ $('Set - Invalid Report Context').item.json.weather_validation_result }}` |
| `notification_channel` | `String` | `={{ $('Set - Invalid Report Context').item.json.notification_channel }}` |
| `notification_status` | `String` | `={{ $('Set - Invalid Report Context').item.json.notification_status }}` |
| `pm_alert_status` | `String` | `={{ $('Set - Invalid Report Context').item.json.pm_alert_status }}` |
| `execution_status` | `String` | `={{ $('Set - Invalid Report Context').item.json.execution_status }}` |

---

## 7. Valid Branch: Weather Validation

### Node 4B: HTTP Request - OpenWeather Current Conditions

### Purpose

Fetches current weather conditions to validate a technician weather-delay claim.

### How to add it

1. From the `true` output of `IF - Required Fields Present`, click `+`.
2. Search for `HTTP Request`.
3. Add the node.
4. Rename it to `HTTP Request - OpenWeather Current Conditions`.

### Settings

| Setting | Value |
|---|---|
| `Method` | `GET` |
| `URL` | `https://api.openweathermap.org/data/2.5/weather` |
| `Send Query Parameters` | `true` |
| `Response Format` | `JSON` |
| `Put Output in Field` | `weather_api` |

### Query parameters

| Name | Value |
|---|---|
| `zip` | `={{ $json.zip + ',US' }}` |
| `appid` | `YOUR_OPENWEATHERMAP_API_KEY` |
| `units` | `metric` |

### Important options

Open the node `Settings` tab and set:

| Option | Value |
|---|---|
| `Continue On Fail` | `true` |
| `Timeout` | `10000` |

### Important note

If your service area is outside the United States, replace:

`{{$json.zip + ',US'}}`

with the correct country code, for example:

`{{$json.zip + ',KE'}}`

---

### Node 5B: IF - Weather Data Available

### Purpose

Separates a successful weather lookup from an API failure or malformed weather response.

### How to add it

1. Click `+` after `HTTP Request - OpenWeather Current Conditions`.
2. Add an `If` node.
3. Rename it to `IF - Weather Data Available`.

### Expression

```javascript
={{ !!$json.weather_api && !!$json.weather_api.weather && Array.isArray($json.weather_api.weather) }}
```

### True output

Use the `true` branch when weather data exists.

### False output

Use the `false` branch when the API failed or returned an unexpected payload.

---

### Node 6B: Set - Weather Success Context

### Purpose

Converts the raw weather payload into business-friendly fields.

### How to add it

1. From the `true` output of `IF - Weather Data Available`, click `+`.
2. Add a `Set` node.
3. Rename it to `Set - Weather Success Context`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `false` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `weather_status` | `String` | `available` |
| `openweather_summary` | `String` | `={{ $json.weather_api.weather[0].main + ' - ' + $json.weather_api.weather[0].description }}` |
| `temp_c` | `Number` | `={{ $json.weather_api.main.temp }}` |
| `wind_mps` | `Number` | `={{ $json.weather_api.wind?.speed || 0 }}` |
| `rain_1h` | `Number` | `={{ $json.weather_api.rain?.['1h'] || 0 }}` |
| `weather_validation_result` | `String` | `={{ $json.weather_claimed === 'yes' ? (((Number($json.weather_api.rain?.['1h'] || 0) > 0) || ['Rain','Thunderstorm','Drizzle','Snow'].includes($json.weather_api.weather?.[0]?.main) || Number($json.weather_api.wind?.speed || 0) >= 10) ? 'claim_supported' : 'claim_not_supported') : 'no_weather_claim_submitted' }}` |
| `pm_alert_status` | `String` | `not_needed` |

### Business rule used here

This guide treats a weather claim as supported when at least one of these is true:

- rain volume is greater than `0`
- condition is `Rain`, `Thunderstorm`, `Drizzle`, or `Snow`
- wind speed is `10 m/s` or higher

You can tighten or relax this later.

---

### Node 7B: Set - Weather Failure Context

### Purpose

Builds a fallback payload so the workflow still continues when weather data is unavailable.

### How to add it

1. From the `false` output of `IF - Weather Data Available`, click `+`.
2. Add a `Set` node.
3. Rename it to `Set - Weather Failure Context`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `false` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `weather_status` | `String` | `unavailable` |
| `openweather_summary` | `String` | `Weather API unavailable or invalid response` |
| `temp_c` | `String` | `` |
| `wind_mps` | `String` | `` |
| `rain_1h` | `String` | `` |
| `weather_validation_result` | `String` | `={{ $json.weather_claimed === 'yes' ? 'weather_claim_unverified_api_unavailable' : 'weather_api_unavailable_no_claim' }}` |
| `pm_alert_status` | `String` | `weather_api_alert_triggered` |

### Connection rule

This node should connect to:

- `Gmail - PM Weather API Failure`
- `Switch - Route Report`

That way the PM gets notified, but the workflow still completes the business action instead of stopping.

---

### Node 8B: Gmail - PM Weather API Failure

### Purpose

Notifies the Project Manager that weather validation could not be completed.

### How to add it

1. Click `+` after `Set - Weather Failure Context`.
2. Add a `Gmail` node.
3. Rename it to `Gmail - PM Weather API Failure`.

### Settings

| Setting | Value |
|---|---|
| `Resource` | `Message` |
| `Operation` | `Send` |
| `To` | `pm@evergreenturfandaquatics.com` |
| `Subject` | `Weather API unavailable for report {{$json.report_id}}` |

### Email body

```html
<p>The weather validation step could not confirm current conditions.</p>
<p><strong>Report ID:</strong> {{$json.report_id}}</p>
<p><strong>Technician:</strong> {{$json.technician_name}}</p>
<p><strong>Service Category:</strong> {{$json.service_category}}</p>
<p><strong>Address:</strong> {{$json.property_address}}, {{$json.city}}, {{$json.state}} {{$json.zip}}</p>
<p><strong>Weather Claim:</strong> {{$json.weather_claimed}}</p>
<p><strong>Weather Notes:</strong> {{$json.weather_claim_notes}}</p>
<p>The workflow will continue using fallback weather status so operations are not blocked.</p>
```

---

## 8. Routing Logic

### Node 9: Switch - Route Report

### Purpose

Separates routine completions from critical incidents.

### How to add it

1. Add a `Switch` node after the valid weather branch.
2. Rename it to `Switch - Route Report`.

### Connections

Connect both of these nodes into this switch:

- `Set - Weather Success Context`
- `Set - Weather Failure Context`

### Switch value

Use:

```javascript
={{ $json.routing_type }}
```

### Rules

| Output | Rule |
|---|---|
| `0` | `critical_incident` |
| `1` | `routine_completion` |

---

## 9. Critical Incident Branch

### Node 10: Slack - Critical Incident Alert

### Purpose

Sends an immediate alert into the incident channel for equipment failure, property damage, spills, or similar urgent issues.

### How to add it

1. From output `0` on `Switch - Route Report`, click `+`.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Slack - Critical Incident Alert`.

### Settings

| Setting | Value |
|---|---|
| `Resource` | `Message` |
| `Operation` | `Post` |
| `Channel` | `#field-incidents` |

### Message text

```text
:rotating_light: *Critical Field Incident*

*Report ID:* {{$json.report_id}}
*Technician:* {{$json.technician_name}}
*Category:* {{$json.incident_category}}
*Service Type:* {{$json.service_category}}
*Address:* {{$json.property_address}}, {{$json.city}}, {{$json.state}} {{$json.zip}}
*Details:* {{$json.incident_details}}
*Weather Status:* {{$json.weather_status}}
*Weather Validation:* {{$json.weather_validation_result}}
*Submitted At:* {{$json.received_at}}
```

### Recommended Slack formatting note

If your Slack node supports blocks and you prefer a richer layout, you can convert this message into block format later. For the first working version, plain formatted text is faster and easier to test.

---

### Node 11: Set - Finalize Critical Log

### Purpose

Builds the exact final row that will be written to Google Sheets after the Slack alert is sent.

### How to add it

1. Click `+` after `Slack - Critical Incident Alert`.
2. Add a `Set` node.
3. Rename it to `Set - Finalize Critical Log`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `true` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `report_id` | `String` | `={{ $('Switch - Route Report').item.json.report_id }}` |
| `received_at` | `String` | `={{ $('Switch - Route Report').item.json.received_at }}` |
| `routing_type` | `String` | `={{ $('Switch - Route Report').item.json.routing_type }}` |
| `technician_name` | `String` | `={{ $('Switch - Route Report').item.json.technician_name }}` |
| `homeowner_name` | `String` | `={{ $('Switch - Route Report').item.json.homeowner_name }}` |
| `homeowner_email` | `String` | `={{ $('Switch - Route Report').item.json.homeowner_email }}` |
| `property_address` | `String` | `={{ $('Switch - Route Report').item.json.property_address }}` |
| `city` | `String` | `={{ $('Switch - Route Report').item.json.city }}` |
| `state` | `String` | `={{ $('Switch - Route Report').item.json.state }}` |
| `zip` | `String` | `={{ $('Switch - Route Report').item.json.zip }}` |
| `service_category` | `String` | `={{ $('Switch - Route Report').item.json.service_category }}` |
| `job_summary` | `String` | `={{ $('Switch - Route Report').item.json.job_summary }}` |
| `incident_category` | `String` | `={{ $('Switch - Route Report').item.json.incident_category }}` |
| `incident_details` | `String` | `={{ $('Switch - Route Report').item.json.incident_details }}` |
| `weather_claimed` | `String` | `={{ $('Switch - Route Report').item.json.weather_claimed }}` |
| `weather_claim_notes` | `String` | `={{ $('Switch - Route Report').item.json.weather_claim_notes }}` |
| `weather_status` | `String` | `={{ $('Switch - Route Report').item.json.weather_status }}` |
| `openweather_summary` | `String` | `={{ $('Switch - Route Report').item.json.openweather_summary }}` |
| `temp_c` | `String` | `={{ $('Switch - Route Report').item.json.temp_c }}` |
| `wind_mps` | `String` | `={{ $('Switch - Route Report').item.json.wind_mps }}` |
| `rain_1h` | `String` | `={{ $('Switch - Route Report').item.json.rain_1h }}` |
| `weather_validation_result` | `String` | `={{ $('Switch - Route Report').item.json.weather_validation_result }}` |
| `notification_channel` | `String` | `slack` |
| `notification_status` | `String` | `slack_incident_alert_sent` |
| `pm_alert_status` | `String` | `={{ $('Switch - Route Report').item.json.pm_alert_status }}` |
| `execution_status` | `String` | `completed` |

---

## 10. Routine Completion Branch

### Node 12: Gmail - Homeowner Completion Summary

### Purpose

Sends a personalized completion update to the homeowner for standard service visits.

### How to add it

1. From output `1` on `Switch - Route Report`, click `+`.
2. Add a `Gmail` node.
3. Rename it to `Gmail - Homeowner Completion Summary`.

### Settings

| Setting | Value |
|---|---|
| `Resource` | `Message` |
| `Operation` | `Send` |
| `To` | `={{ $json.homeowner_email }}` |
| `Subject` | `Evergreen service completed at {{$json.property_address}}` |

### Email body

```html
<p>Hello {{$json.homeowner_name || 'Homeowner'}},</p>
<p>Your {{$json.service_category}} visit has been completed by {{$json.technician_name}}.</p>
<p><strong>Service summary:</strong> {{$json.job_summary}}</p>
<p><strong>Property:</strong> {{$json.property_address}}, {{$json.city}}, {{$json.state}} {{$json.zip}}</p>
<p><strong>Weather check:</strong> {{$json.weather_validation_result}}</p>
<p>Thank you for choosing Evergreen Turf &amp; Aquatics.</p>
```

### Why weather validation is included

Including the weather validation result creates transparent customer communication if the technician reported a delay caused by weather.

---

### Node 13: Set - Finalize Routine Log

### Purpose

Builds the final Google Sheets row after the homeowner email is sent.

### How to add it

1. Click `+` after `Gmail - Homeowner Completion Summary`.
2. Add a `Set` node.
3. Rename it to `Set - Finalize Routine Log`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `true` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `report_id` | `String` | `={{ $('Switch - Route Report').item.json.report_id }}` |
| `received_at` | `String` | `={{ $('Switch - Route Report').item.json.received_at }}` |
| `routing_type` | `String` | `={{ $('Switch - Route Report').item.json.routing_type }}` |
| `technician_name` | `String` | `={{ $('Switch - Route Report').item.json.technician_name }}` |
| `homeowner_name` | `String` | `={{ $('Switch - Route Report').item.json.homeowner_name }}` |
| `homeowner_email` | `String` | `={{ $('Switch - Route Report').item.json.homeowner_email }}` |
| `property_address` | `String` | `={{ $('Switch - Route Report').item.json.property_address }}` |
| `city` | `String` | `={{ $('Switch - Route Report').item.json.city }}` |
| `state` | `String` | `={{ $('Switch - Route Report').item.json.state }}` |
| `zip` | `String` | `={{ $('Switch - Route Report').item.json.zip }}` |
| `service_category` | `String` | `={{ $('Switch - Route Report').item.json.service_category }}` |
| `job_summary` | `String` | `={{ $('Switch - Route Report').item.json.job_summary }}` |
| `incident_category` | `String` | `={{ $('Switch - Route Report').item.json.incident_category }}` |
| `incident_details` | `String` | `={{ $('Switch - Route Report').item.json.incident_details }}` |
| `weather_claimed` | `String` | `={{ $('Switch - Route Report').item.json.weather_claimed }}` |
| `weather_claim_notes` | `String` | `={{ $('Switch - Route Report').item.json.weather_claim_notes }}` |
| `weather_status` | `String` | `={{ $('Switch - Route Report').item.json.weather_status }}` |
| `openweather_summary` | `String` | `={{ $('Switch - Route Report').item.json.openweather_summary }}` |
| `temp_c` | `String` | `={{ $('Switch - Route Report').item.json.temp_c }}` |
| `wind_mps` | `String` | `={{ $('Switch - Route Report').item.json.wind_mps }}` |
| `rain_1h` | `String` | `={{ $('Switch - Route Report').item.json.rain_1h }}` |
| `weather_validation_result` | `String` | `={{ $('Switch - Route Report').item.json.weather_validation_result }}` |
| `notification_channel` | `String` | `gmail` |
| `notification_status` | `String` | `homeowner_completion_email_sent` |
| `pm_alert_status` | `String` | `={{ $('Switch - Route Report').item.json.pm_alert_status }}` |
| `execution_status` | `String` | `completed` |

---

## 11. Shared Logging Node

### Node 14: Google Sheets - Append Report Log

### Purpose

Writes every final outcome into the master review sheet.

### How to add it

1. Add a `Google Sheets` node.
2. Rename it to `Google Sheets - Append Report Log`.
3. Connect these nodes into it:
   `Set - Finalize Invalid Log`
   `Set - Finalize Critical Log`
   `Set - Finalize Routine Log`

### Settings

| Setting | Value |
|---|---|
| `Resource` | `Sheet Within Document` |
| `Operation` | `Append row` |
| `Document` | `Evergreen Field Report Log` |
| `Sheet` | `Reports` |

### Mapping strategy

Use `Auto-map input data to columns` if your sheet columns exactly match the field names in this guide.  
If not, switch to manual mapping and map each field one by one.

### Why this is the final shared node

Using one shared log node keeps your audit trail consistent and makes weekly PM review much easier.

---

## 12. Error Workflow

Create a second workflow named:

`Evergreen - Field Report Error Handler`

This catches anything unexpected that was not already handled by the invalid-input branch or the weather-failure branch.

### Node 1: Error Trigger - Unexpected Workflow Failure

### Purpose

Starts when the main workflow fails unexpectedly.

### How to add it

1. Create a new workflow.
2. Click `Add first step`.
3. Search for `Error Trigger`.
4. Add the node.
5. Rename it to `Error Trigger - Unexpected Workflow Failure`.

### Important note

This workflow only runs for unhandled errors, such as a broken credential, a Slack delivery problem, or a Google Sheets failure.

---

### Node 2: Set - Format Failure Alert

### Purpose

Formats the error details into a readable PM alert.

### How to add it

1. Click `+` after `Error Trigger - Unexpected Workflow Failure`.
2. Add a `Set` node.
3. Rename it to `Set - Format Failure Alert`.

### Settings

| Setting | Value |
|---|---|
| `Keep Only Set` | `true` |

### Add these fields

| Field name | Type | Value |
|---|---|---|
| `workflow_name` | `String` | `={{ $json.workflow?.name || 'Unknown workflow' }}` |
| `execution_id` | `String` | `={{ $json.execution?.id || 'Unknown execution' }}` |
| `last_node` | `String` | `={{ $json.lastNodeExecuted || 'Unknown node' }}` |
| `error_message` | `String` | `={{ $json.error?.message || 'No error message available' }}` |
| `error_time` | `String` | `={{ $now.toISO() }}` |

---

### Node 3: Gmail - PM Unexpected Workflow Failure

### Purpose

Sends a clean escalation email whenever the main workflow crashes unexpectedly.

### How to add it

1. Click `+` after `Set - Format Failure Alert`.
2. Add a `Gmail` node.
3. Rename it to `Gmail - PM Unexpected Workflow Failure`.

### Settings

| Setting | Value |
|---|---|
| `Resource` | `Message` |
| `Operation` | `Send` |
| `To` | `pm@evergreenturfandaquatics.com` |
| `Subject` | `Unexpected n8n failure in {{$json.workflow_name}}` |

### Email body

```html
<p>An unexpected error occurred in the Evergreen field report automation.</p>
<p><strong>Workflow:</strong> {{$json.workflow_name}}</p>
<p><strong>Execution ID:</strong> {{$json.execution_id}}</p>
<p><strong>Last Node:</strong> {{$json.last_node}}</p>
<p><strong>Error:</strong> {{$json.error_message}}</p>
<p><strong>Detected At:</strong> {{$json.error_time}}</p>
```

---

## 13. Activation Checklist

Before you activate the workflows, confirm these items:

- `Webhook` path is correct
- `OpenWeatherMap` API key works
- `Slack` channel is correct
- `Gmail` credential is authorized
- `Google Sheets` points to the right spreadsheet and tab
- Both workflows are saved
- Both workflows are activated

---

## 14. Proof of Execution Checklist

To satisfy the proof requirement, capture both a routine run and a critical run.

### Routine success run screenshots

Capture:

1. The incoming test payload in `Webhook - Field Report Intake`
2. The normalized record in `Set - Normalize Report`
3. The weather response in `HTTP Request - OpenWeather Current Conditions`
4. The `Gmail - Homeowner Completion Summary` execution result
5. The appended row in `Google Sheets - Append Report Log`

### Critical incident run screenshots

Capture:

1. The incoming incident payload in `Webhook - Field Report Intake`
2. The route decision in `Switch - Route Report`
3. The `Slack - Critical Incident Alert` execution result
4. The final Google Sheets row showing `notification_channel = slack`

### Error-handling proof

For error-path proof, run one malformed payload such as a routine completion without `homeowner_email`.

Capture:

1. The `false` branch on `IF - Required Fields Present`
2. The `Gmail - PM Invalid Input Alert` result
3. The Google Sheets row showing `execution_status = rejected_invalid_input`

---

## 15. Workflow Export and Visual Documentation

### Export the workflow JSON

For each workflow:

1. Open the workflow in n8n.
2. Click the workflow menu in the top-right.
3. Choose `Download`.
4. Save the exported JSON.

Suggested filenames:

- `evergreen-field-report-intelligence.json`
- `evergreen-field-report-error-handler.json`

### Capture the workflow canvas

Use these screenshot rules:

- Zoom out until the full branch logic is visible
- Capture the main workflow on one clean canvas screenshot
- Capture the error workflow on one separate screenshot
- If the main workflow is too wide, take two overlapping high-resolution screenshots

### Screen recording option

If you prefer a short screen recording:

1. Start recording.
2. Trigger the routine payload.
3. Show the Gmail send result and the Google Sheets row.
4. Trigger the incident payload.
5. Show the Slack alert and the Google Sheets row.
6. Show the error workflow setup briefly at the end.

---

## 16. Suggested Enhancements After the First Working Version

Once the base workflow is working, you can improve it with:

- `Airtable` instead of Google Sheets for richer filtering
- `Respond to Webhook` for custom intake responses
- `Geocoding` for lat/long precision instead of ZIP lookup
- `Severity scoring` based on incident category and keywords
- `Photo attachment support` for field evidence
- `Deduplication logic` using a submission ID from the source form

---

## 17. Final Outcome

When this build is complete:

- routine completion reports send a homeowner summary email
- critical incidents alert Slack immediately
- weather claims are cross-checked against public weather data
- malformed input is rejected and escalated to the PM
- weather API outages do not stop the business workflow
- every result is logged for weekly review

This gives Evergreen Turf & Aquatics a field intake process that is fast for technicians, visible for management, and resilient when external services fail.
