---
name: pipedrive-hot-deal-detector
description: Evaluates sales leads to detect urgency signals. Checks Pipedrive via MCP to determine the actual deal status (Open/Won/Lost). Updates the CRM for Open deals and maintains a local sorted list in HOT_DEALS.md, automatically removing deals that Pipedrive reports as closed.
---

# Pipedrive & Local Hot Deal Detector

Act as a sales qualification expert. Your goal is to fetch the current deal status from Pipedrive, analyze customer interactions to assign a probability score based on timeframe signals, use the Pipedrive MCP server tools to register or update the opportunity, and maintain a centralized, sorted list of active qualified leads in a local file.

## 1. Scoring Rules (For Open Deals Only)
Evaluate the provided text looking for explicit or implicit timeframe signals to calculate probability:
* **Base Score:** Starts at `40%`.
* **High Urgency (+45%):** The customer uses phrases like "urgent", "ASAP", "as soon as possible", "immediately", "yesterday", or "this week". (Maximum cap of 99%).
* **Short Term (+25%):** The customer uses phrases like "this quarter", "Q1", "Q2", "Q3", "Q4", "this month", or "next few weeks".
* **No Urgency (+0%):** There is no clear mention of timing or it is a very long-term project.

## 2. Dual Workflow (Pipedrive Status First)
When this skill is triggered, follow these steps strictly:

1. **Search & Fetch Status (Pipedrive MCP):** Use the Pipedrive search tool (e.g., `search_deals` or `search_organizations`) to find the deal.
   * If it exists, retrieve its **current status** from Pipedrive (e.g., Open, Won, Lost) and its **Deal ID**.
   * If it does not exist, treat it as a new "Open" deal.
2. **Handle Closed Deals:** If Pipedrive reports the deal status as **Won or Lost**:
   * Do NOT calculate probability or update the deal's score in Pipedrive (you may add a note if the user provided new context).
   * Skip directly to Step 4 to remove it from the local list.
3. **Handle Open Deals (Analyze & Update):** If the deal is Open (or new):
   * Calculate the probability score based on the rules in Section 1.
   * *Filter:* If the score is **less than 60%**, respond to the user explaining briefly why the lead has low priority and **stop here** (do not edit Pipedrive or the local file).
   * *Update:* If the deal exists and the score is **>= 60%**, use the MCP update tool to raise its probability and add a note detailing the detected "Timeframe Signal".
   * *Create:* If the deal DOES NOT exist and the score is **>= 60%**, use the MCP creation tool. Set the title as "[Company Name] - Hot Deal", set the probability, and add a summary note. **Capture the newly created Deal ID.**
4. **Local List Maintenance (File System):**
   * Look for the file `HOT_DEALS.md` in the current directory. If it does not exist, create it.
   * Read the contents of `HOT_DEALS.md` and extract the existing leads.
   * **Cleanup Step:** If the deal is Won/Lost in Pipedrive, find it in the extracted list using its **Deal ID** and **remove it entirely**.
   * **Update Step:** If the deal is Open and scored >= 60%, add the new lead (or update its probability and context if the Deal ID is already on the list).
   * Sort all active leads from **highest to lowest** probability.
   * Overwrite `HOT_DEALS.md` with the updated list, strictly using the table format below.
5. **Report:** Display a concise success message to the user in the terminal. Indicate whether the deal was updated, created, or cleaned up (if closed), and confirm the local `HOT_DEALS.md` was synced.

## 3. Required Format for HOT_DEALS.md
The generated or modified file must contain only a title and this table in Markdown format. The Probability must always be the first column. Respect this structure when writing the file:

# High Priority Pipeline

| Probability | Deal ID | Company / Lead | Timeframe Signal | Context Summary |
| :--- | :--- | :--- | :--- | :--- |
| 85% | 1045 | Acme Corp | High Urgency (ASAP) | Needs implementation before the end-of-month accounting close. |
| 65% | 1089 | Globex Inc | Short Term (This quarter) | Budget is allocated for Q3 deployment. |

## 4. Agent Behavior
If you experience any errors while using the MCP tools (for example, if a parameter required by the Pipedrive API is missing), politely ask the user for the missing information before attempting the action again. Do not update or clean the local list until the Pipedrive action is successful.
