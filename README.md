# agentic-logistics-incident-response

# Overview <br>
A supply chain automation platform was built to assess the monetary consequences of transport disruptions, determine optimal rerouting strategies, and manage external processes via AI-driven agents and workflow automation.
Following repeated delivery slowdowns at PepsiCo, an initiative was launched to create an adaptive system that could compute route scenarios and timing estimates to select the option with the lowest financial impact.
# diagram <br>
![](https://github.com/CodeWithLuwam/agentic-logistics-incident-response/blob/main/Diagram.drawio.png?raw=true)

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


