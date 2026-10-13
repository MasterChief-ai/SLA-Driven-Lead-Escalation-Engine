# SLA Driven Lead Escalation Engine

## Overview
This repository contains the configuration for the `SLA Driven Lead Escalation Engine.json` file, which is an automated workflow built on the n8n platform. 

The primary goal of this system is to enforce a Service Level Agreement (SLA) on lead response times. When a prospective client submits a lead, the system does not simply email a group of agents and hope someone replies. Instead, it systematically identifies one available agent at a time, checks their calendar, and sends them a direct message on Telegram. The agent is given exactly 3 minutes to claim the lead. If they fail to respond in time, the system registers the missed attempt and escalates the lead to the next available agent, eventually reaching the broker in charge if necessary.

## Core Tools and Integrations
This workflow connects several external services to function as a unified engine:
* **n8n:** The central automation platform where the workflow logic is executed.
* **Webhooks:** A digital listener that waits to receive the incoming lead data (such as the lead's name, phone number, and property of interest) from a website or form.
* **Airtable:** Serves as the database. It uses two separate tables: one to track the status of the leads and one to maintain a directory of active agents.
* **Google Calendar:** Used to check the assigned agent's schedule to ensure they do not have a scheduling conflict in the immediate future before sending them a lead.
* **Telegram:** The messaging platform used to alert agents. It utilizes inline keyboard buttons so agents can accept leads directly from their chat interface.

## Detailed Workflow Breakdown

The system follows a strict, step-by-step logical path from the moment a lead is generated to the final resolution.

### Phase 1: Lead Ingestion and Registration
1. **Data Reception:** A webhook receives a payload containing the `lead_name`, `property`, and `lead_phone`.
2. **Database Logging:** The system immediately creates a new record in the "Engine Leads" Airtable database. It logs the lead's details, records the exact timestamp, and sets the lead's status to "New".
3. **Attempt Tracking Initialization:** The workflow sets up internal tracking variables, setting the current attempt count to 1 and preparing a blank variable to exclude agents who have already been contacted.

### Phase 2: Tier 1 Agent Routing
1. **Agent Search:** The system searches the "Lead Escalation Engine" Airtable database for an agent whose status is marked as "Active". 
2. **Availability Check:** Once an active agent is found, the workflow takes their email address and queries Google Calendar. It specifically checks if the agent is free for the next 30 minutes.
3. **Telegram Notification:** If the agent is available, the system sends a formatted text message to their Telegram ID. The message displays the lead's name, property, and phone number, and includes a clickable "Accept Lead" inline button. 
4. **The SLA Timer:** The workflow suspends itself and waits for up to 3 minutes for the agent to click the button. 
5. **Resolution or Timeout:** 
   * If the agent clicks the link within 3 minutes, the workflow updates the lead's Airtable record status to "Assigned" and logs the agent's name.
   * If the 3 minutes pass without a click, the workflow proceeds to Phase 3.

### Phase 3: Tier 2 Agent Routing
1. **Agent Exclusion:** The system updates its tracking variables, changing the attempt count to 2 and adding the first agent's ID to an exclusion list so they are not contacted again.
2. **Secondary Search:** It searches the Airtable database for another "Active" agent, strictly filtering out the first agent.
3. **Repeat Process:** The system checks this second agent's Google Calendar for 30 minutes of availability and sends them the same Telegram alert with a new 3-minute SLA timer.
4. **Resolution or Timeout:** If the second agent accepts, the Airtable record is updated to show they are the assigned agent. If they also fail to respond, the system escalates to Phase 4.

### Phase 4: Broker Escalation
1. **Broker Alert:** If two consecutive agents miss the lead, the workflow bypasses standard routing and sends a message to a specific, hardcoded Telegram ID belonging to the Broker in Charge.
2. **Escalation Message:** The message explicitly states: "ESCALATED TO BROKER — LEAD UNCLAIMED. 2 agents did not respond in time. This lead now needs your direct attention," alongside the lead's details.
3. **Final Resolution:** The system waits for the broker to accept. If the broker accepts, the lead is updated in Airtable as assigned to the "Broker in Charge". If the broker also fails to act, the system permanently updates the lead's status to "Expired" in the database, ensuring there is a clear record that the lead was missed entirely.

## Installation and Deployment

To implement this workflow in your own environment, you will need to recreate the database structures and import the logic into an n8n instance.

### 1. Database Configuration
You must create two separate bases in Airtable:
* **Base 1: Lead Escalation Engine**
  * This base manages your agents.
  * Required columns: `Status` (must include an "Active" option), `Agent Name`, `Phone Number`, `Email / GCal ID` (used for calendar checking), `Tier`, and `Telegram ID`.
* **Base 2: Engine Leads**
  * This base stores the incoming prospects.
  * Required columns: `Lead Name`, `Phone`, `Property Address`, `Status` (must contain options for "New", "Assigned", and "Expired"), `Assigned Agent`, and `Timestamp`.

### 2. Importing the Workflow
1. Log into your n8n workspace.
2. Create a new, blank workflow.
3. Open the provided `SLA Driven Lead Escalation Engine.json` file in a text editor and copy all the contents.
4. Paste the copied JSON text directly into the n8n visual editor canvas. The nodes and connections will automatically generate.

### 3. Configuring Credentials
The workflow will display warning symbols on several nodes indicating missing credentials. You must authorize n8n to communicate with your external accounts:
* **Airtable Nodes:** Open each Airtable node (e.g., "Create a record", "Search records") and create a new credential using an Airtable Personal Access Token. Update the Base and Table dropdowns to point to the databases you created in Step 1.
* **Google Calendar Nodes:** Open the Google Calendar nodes and connect your Google account using OAuth2 credentials. Ensure the account you connect has access to view the calendars associated with the email addresses stored in your Airtable database.
* **Telegram Nodes:** Open the Telegram nodes and supply a Telegram Bot API Token. You will need to create a bot via the BotFather on Telegram to generate this token. Ensure your agents have started a chat with this bot so it can message them.

### 4. Updating Hardcoded Values
* Open the "Escalated to Broker in Charge" Telegram node. You must change the `chatId` value from `5991474775` to the actual Telegram ID of your managing broker.

### 5. Finalizing Deployment
1. Double-click the initial "Webhook" node to locate your unique Webhook URL.
2. Copy this URL and paste it into the system generating your leads (such as your website's form submission settings or a lead generation ad platform). Ensure the external system sends a POST request formatted with `lead_name`, `property`, and `lead_phone`.
3. Click "Test Workflow" in n8n and send a test payload to the webhook. Check Telegram to ensure the messages arrive and verify that Airtable updates correctly.
4. Once verified, toggle the workflow from "Inactive" to "Active" in the top right corner of the n8n interface.
