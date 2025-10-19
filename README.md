# agentic-logistics-incident-response

# Overview <br>
An automated supply chain incident processing system for PepsiCo's logistics operations. The system uses AI agents in ServiceNow to analyze financial impacts of delivery truck breakdowns, make optimal routing decisions, and coordinate external execution through workflow orchestration in n8n.
# diagram <br>
![](https://github.com/CodeWithLuwam/agentic-logistics-incident-response/blob/main/Diagram.drawio.png?raw=true)


# Implementation Steps
## Step 1.1: ServiceNow Scoped Application
Application name: PepsiCo Deliveries

*This precise naming will auto-generate the scope: x_snc_pepsico_de_0*



### 1.2: Table Setup <br>
**Delivery Delay Table** holds the information about the various truck breakdowns reported from Schneider (Trucking Logistics Provider).

Delivery Delay Table Fields:
- `route_id` (Integer, Primary Key)
- `truck_id` (Integer)
- `customer_id` (Integer, Default: 1)
- `problem_description` (String, 4000)
- `proposed_routes` (String, 4000) - JSON format with route options
- `calculated_impact` (String, 4000) - JSON format with financial analysis
- `chosen_option` (String, 4000) - Selected route details
- `status` (String, 16) - Workflow progression: pending/calculated/approved/dispatched
- `assigned_to` (Reference to User) - Critical: Used for trigger execution context and permissions
- `incident_sys_id` (String, 32) - Links to associated incident records


**Supply Agreement Table** holds the contractual penalties for late shipments from Whole Foods (Retail Client).

Supply Agreement Table Fields:
- `customer_id` (Integer, Primary Key)
- `customer_name` (String, 100)
- `deliver_window_hours` (Integer) - Contractual delivery timeframe <br>
  *(The contractual timeframe (in hours) within which deliveries must be completed to avoid penalties. For Whole Foods, deliveries must be completed within 3 hours of departure to avoid charges.)*
- `stockout_penalty_rate` (Integer) - Cost per hour of delay in dollars <br>
*(The financial penalty (in dollars) assessed for every hour a delivery exceeds the contractual delivery window. Whole Foods charges PepsiCo $250 for each hour beyond the 3-hour delivery window. For example, a 5-hour delivery would incur penalties for 2 hours (5 - 3 = 2 x 250), resulting in a $500 penalty charge.)*


## Step 2. AI Agent Setup
### Agent 1: Route Financial Analysis Agent <br>

*Purpose*: Analyzes financial impact of delivery disruptions, calculates the cost of alternate routes, and creates incident tracking
Tools Configured:

**Role**
```
You are a financial analysis specialist for PepsiCo's supply chain operations. Always format financial calculations
clearly and ensure all monetary values are in USD. Always present the calculated financial impact to the user.
```
**Description**
```
Analyzes delivery disruptions by calculating penalty costs for each route option based on customer contract terms,
creates incident records with appropriate priority levels, and prepares comprehensive financial impact data for
route optimization decisions.
```

**Instructions**
``` 
When a delivery delay occurs, you must:
1. **Retrieve Delivery Information**
   1.1 Look up the delivery delay record using the provided route_id
   1.2 pull the oldest route_id in status = pending
   1.3 Store the proposed route in memory

2. Use the Customer ID of the Delayed Delivery to Locate the Supply Agreement for the Customer. Store the numerical value
Customer's Delivery Window Hours and numerical value for Stockout Penalty Rate in memory.

FINANCIAL ANALYSIS INSTRUCTIONS
3.1 There are multiple options in the proposed routes. Store the numerical value of each ETA Minutes in memory.  
3.2 Run the Financial Impact Calculation tool separately for EACH of the delivery's proposed route option values.
There will be a unique calculation for each of the ETA minutes.

4. Return the Calculation for each proposed route combined in **USD with $** in the output format
"Alternative Route Option 1 Calculated Impact: , Alternative Route Option 2 Calculated Impact: " with a line break after
 each option. NOTE: IF THE CALCULATED IMPACT IS A NEGATIVE NUMBER, USE $0.00. This is the Calculated Impact. 
4.1 Store the Calculated Impact in memory.

5. **Create Incident Record**
  5.1 save the sys_id value in your memory log

6 **Update the Delivery Delay record that matches the user provided Route ID with the Calculated Impact and
the Incident Sys ID**
```

**Tools** <br>
**Look Up Delivery Delay** (record lookup) <br>
**Look Up Supply Agreement** (record lookup) <br>
**Create Incident** (record creation) <br>
**Update Delivery Delay** ( updates with calculated financial impact & updates status to Calculated) <br>
**Financial Impact Calculation** (script) ( Calculates the monetary impact of a delivery delay by converting ETA minutes to hours, comparing against the delivery window, and applying the stockout penalty rate.) <br>

![]()

### Agent 2: Route Financial Analysis Agent
*Purpose: Selects optimal routes and coordinates external execution*

**Roles**
```
You will be provided with the Route ID.

- First, locate the Delivery Delay record using the Route ID.
- Second, choose the route option with the most cost-savings.
- Third, Route ID's Delivery Delay record with the Decision and Truck ID.
- Fourth, update the incident record.
- Fifth, call the web hook to send Chosen Option to vendor.

1. Use the provided Route ID to Pull Delivery Delay Details.

2. Analyze the route options and select an option_id that optimizes the corresponding distance_miles and calculated_impact. Store the **entire route object** (including option_id, route_number, distance_miles, and eta_minutes) in your memory as Decision, not just the option_idd string. Decision must be a structured JSON object, not plain text.

3. Update Delivery Delay Record of Route ID with the entire Decision JSON object, prepending the Route ID and Truck ID to the objet in the format "route_id": "truck_id":. Do not put the values for route_id and truck_id in quotations.

4. Update the associated Incident record's Urgency based on financial severity. Use the following guidelines to set Urgency:
- If the option cost LESS than $500, then set Urgency to 3- Low.
- If the option cost BETWEEN $501- $1000, then set Urgency to 2- Medium.
-  If the option cost MORE then $1000, then set Urgency to 1- High.

5. Retrieve Webhook Value. You will use this data to trigger the webhook.

6. Trigger n8n webhook with the Route ID and Retrieved Webhook Value.
```
**Description**
```
You will use the Route ID to find the optimal route based on cost and time constraint, then update the Delivery Record,
then update incident's priority, then communicate the decision to via web hook.
```

**Tools** <br>
**Look Up Delivery Delay** (Locate the Delivery Delay record by searching the table for the provided Route ID) <br>
**Update Delivery Delay** (Finds records that match a specific Route ID and has Status = "Calculated", then updates the Status to "Approved") <br>
**Update Incident** (Updates a specific Incident record by setting its Impact and Urgency fields based on agent-determined values) <br>
**Trigger n8n Workflow** (webhook script tool - Automatically triggers external workflows in n8n when delivery delays occur, enabling automated responses like notifications, rerouting, or escalations.) <br>

Key Responsibilities: <br>

- Analyzes route options using financial impact calculations
- Selects optimal routing based on lowest penalty cost
- Updates incident priority based on financial severity:
  - Penalty > $500: Priority 1 (Critical)
  - Penalty $200-$500: Priority 2 (High)
  - Penalty < $200: Priority 3 (Moderate)
- Triggers external execution workflow via webhook to n8n
- Updates workflow status to "approved" for tracking
![]()

### Step 3: Use Case and Trigger Setup
use case is set up to utilize one data point across both agents for a consistent experience. I chose the route_id because it is a Primary Key in our Delivery Delay records.
**Description** <br>
```
This use case is triggered when external agents relay a Delayed Delivery Route ID, runs a cost analysis on
alternative routes, and make a recommendation best suited for business needs.
```
**Instructions**
```
When Pending Delivery Delay records are created or updated, then first run the Delivery Delay Financial Analyzer
and THEN run the Route Decision Agent to select the most optimal alternative route.
You will receive a Route ID.
Store this Route ID in memory to run both Agents.
Use the tools as exactly as directed.
1. Use Financial Analyzer to find the calculated impact of delivery delay.
2. Use the Route Decision Agent to choose the most optimal route.
```
![](https://github.com/CodeWithLuwam/agentic-logistics-incident-response/blob/main/Images/Describe%20and%20Connect%20Use%20Case.png?raw=true) <br>
This use case is an automated trigger that monitors the Delivery Delay table records. It activates on create/update events when the status is 'Pending' and starts the AI analysis workflow to help resolve or manage the delay. <br>
![](https://github.com/CodeWithLuwam/agentic-logistics-incident-response/blob/main/Images/Use%20Case%20Trigger.png?raw=true) <br>

## Step 4: n8n Workflow Setup
Workflow Logic
1. The n8n AI agent receives webhook payloads from ServiceNow **Route Decision Agent**. <br>
2. Parse and validate payload structure. <br>
3. Send data to **Logistics**, **Retail**, and **ServiceNow MCP systems**. <br>
4. Confirm route dispatch and return a success message. <br>

![](https://github.com/CodeWithLuwam/agentic-logistics-incident-response/blob/main/Images/n8n%20Workflow%20.png?raw=true)

---








breakdown is reported (or detected) and logged as an incident in ServiceNow

Agent 1 immediately reviews the issue details, creates an incident, checks the relevant SLA commitments and calculates the impact using ServiceNow's Now language model.

Agent 2 calculates the best chosen option sourced from the Agent 1 calculation and passes the selected route to Agent 3.

Then Agent 3 orchestrates the execution: it communicates with external systems (like Schneider’s fleet management and Whole Foods’ delivery scheduling system) to implement the reroute and provide updates. It then updates the status of the issue to Dispatched (full circle).

This ensures that both PepsiCo and its partners have real-time updates on the delivery status

ServiceNow The two agents are fired when the built Use Case detects a change in the status of the issue (status is either created as or updated with "pending").

Agent 1 (Route Financial Analysis Agent) and Agent 2 (Route Decision Agent) are configured on the ServiceNow platform. Amazon Bedrock powers the Agent 3 execution. Amazon Bedrock powers the Agent 3 execution.

Agent 1 pulls relevant SLA data (e.g. proposed routes, inside delivery delayed {passed externally}) and uses the LLM to assess impact, while Agent 2 uses the LLM to choose the best rerouting solution based on business rule set cost and estimated time of delivery. —-- Agent 3 [handles cross-system orchestration.] - workflow hosted on n8n(automation platform) ServiceNow triggers Agent 3 (e.g. via a webhook or IntegrationHub action) once Agents 1 and 2 have formulated a plan. Agent 3 runs in n8n to perform external communications that ServiceNow alone cannot directly handle. Specifically, Agent 3 uses the Model Context Protocol (MCP) to interface with external systems. MCP is an open standard that allows AI-driven applications to connect to external data sources and tools in a uniform way. Think of MCP as a “USB-C port” for AI applications, standardizing how Agent 3 communicates with outside systems.

In this project, Schneider (the carrier managing the truck fleet) and Whole Foods (the delivery recipient) each host an MCP server. Agent 3 (running in n8n) acts as an MCP client: it sends a reroute request to Schneider’s MCP endpoint and sends a delivery delay notification to Whole Foods’ MCP endpoint.

(ServiceNow to n8n, n8n to external MCP servers)

Tables in ServiceNow - Supply Agreement (containing the penalty rate per hour) Incident (representing the breakdown event) Delivery Delays table that tracks shipment delays and statuses. Records the status of the issue (pending → calculated → approved → dispatched) with notes on actions taken, as well as the new expected delivery time, delay duration, and reason (truck breakdown reroute) for downstream visibility.

Amazon Bedrock Access Obtain AWS credentials (Access Key ID and Secret Key) that have permissions to use Amazon Bedrock’s API. You may need to request access to Bedrock if it’s not generally available in your AWS region. Once you have access, note the Bedrock endpoint URL for your region (from AWS console or docs) to configure in ServiceNow. Using Bedrock may incur AWS costs, so ensure you have billing awareness and have set up any necessary cost controls.

n8n Orchestration Environment n8n has a custom package that provides an MCP Server Trigger and MCP Client node. In our case, we use an MCP Client node in n8n to connect to the external MCP servers.


