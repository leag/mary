---
name: pipedrive-hot-deal-detector
description: Evaluates sales leads to detect urgency signals. Checks Pipedrive via MCP to determine the actual deal status (Open/Won/Lost). Updates the CRM for Open deals and maintains a centralized sorted list at ~/HOT_DEALS.md, automatically removing deals that Pipedrive reports as closed.
---

# Pipedrive & Local Hot Deal Detector

Act as a sales qualification expert. Your goal is to fetch the current deal status from Pipedrive, analyze customer interactions to assign a probability score based on timeframe signals, use the Pipedrive MCP server tools to register or update the opportunity, and maintain a centralized, sorted list of active qualified leads at `~/HOT_DEALS.md`.

## 0. Inputs the skill expects

Before doing anything else, extract the following from the user's message. If you cannot confidently extract an identifier, ask the user before proceeding.

* **Identifier** (use the first available, in this priority order):
  1. An explicit Pipedrive Deal ID (a number, often written as `#1234` or "deal 1234").
  2. A company / organization name.
  3. The domain portion of a contact email (e.g. `acme.com`).
  4. A contact's full email or name.
* **Timeframe context** — the customer's own phrasing about *when* they want to act. Use the customer's words, not the user's paraphrase.

If the user supplies a Deal ID directly, skip search and fetch that deal by ID.

## 1. Scoring Rules (For Open Deals Only)

Read the customer's words for **future intent** to act within a given timeframe. Phrases describing past events (e.g. "I sent that yesterday") do not count — only forward-looking phrasing does.

Signals **stack additively**, with a **global cap of 99%** and a **floor of 0%**.

* **Base Score:** `40%`.
* **High Urgency (+45%):** customer expresses intent to act within roughly the next week. Example phrasings: "urgent", "ASAP", "as soon as possible", "immediately", "by Friday", "this week", "we need it yesterday".
* **Short Term (+25%):** customer expresses intent to act within roughly the next quarter. Example phrasings: "this quarter", "Q1/Q2/Q3/Q4", "this month", "next few weeks", "before our renewal".
* **Stall (−25%):** customer is de-prioritizing or pushing the deal out. Example phrasings: "on hold", "next year", "after Q4", "budget paused", "circle back later", "not a priority right now".
* **No Signal (+0%):** no clear forward-looking timeframe.

Worked examples:
- "ASAP this quarter" → 40 + 45 + 25 = **99%** (cap).
- "ASAP" alone → 40 + 45 = **85%**.
- "This quarter" alone → 40 + 25 = **65%**.
- "On hold until next year" → 40 − 25 = **15%**.

## 2. Workflow (Pipedrive Status First)

When the skill is triggered, follow these steps strictly:

1. **Search & fetch status (Pipedrive MCP).** Use the Pipedrive search tool to find the deal by the identifier extracted in §0.
   * If **multiple organizations** match the identifier, list the candidates and ask the user to pick before continuing.
   * If a single organization has **multiple deals**, pick:
     1. the most recently updated **Open** deal, otherwise
     2. the most recently updated Won/Lost deal.
   * If nothing matches, treat this as a new "Open" deal.
   * Capture the **Deal ID**, **current status** (Open / Won / Lost), and the **current Pipedrive probability** (if set).

2. **Handle Closed Deals.** If Pipedrive reports the deal as **Won** or **Lost**:
   * Do NOT calculate or update probability. You may add a note if the user provided new context worth recording.
   * Skip directly to step 4 to clean up the local list.

3. **Handle Open Deals (analyze + update).** If the deal is Open or new:
   * Calculate the score per §1.
   * **Filter:** if the score is `< 60%` **and** the Deal ID is **not** already on the local list, respond briefly explaining why this lead is low priority and **stop** — no Pipedrive write, no local-list edit.
   * **Pipedrive write rule:** when writing a probability to Pipedrive, set it to `max(computed_score, current_pipedrive_probability)`. Never lower a value a human sales rep already set.
   * **Update existing deal:** apply the write rule above and add a note describing the detected timeframe signal.
   * **Create new deal:** if the deal does not exist and the score is `>= 60%`, create it via the MCP creation tool. Title: `[Company Name] - Hot Deal`. Set the probability and add a summary note. Capture the new **Deal ID**.
   * **Downgrade path:** if the Deal ID is already on the local list and the new score has dropped (stall signal hit, or signals weakened), still apply the Pipedrive write rule, then let step 4 update or remove the local entry based on the new score.

4. **Local list maintenance (`~/HOT_DEALS.md`).**
   * Open `~/HOT_DEALS.md`. Create it if missing.
   * Read existing rows.
   * **Cleanup (Won/Lost):** remove the row matching the Deal ID. If the Deal ID isn't present, do nothing — no error.
   * **Cleanup (cooled):** if the deal is Open but the new score is `< 60%`, remove the row matching the Deal ID (no-op if not present).
   * **Insert / update:** if the deal is Open and the score is `>= 60%`, add a new row, or update the existing row matched by Deal ID.
   * **Sort:** by probability **descending**, then by Deal ID **ascending** as a tiebreaker.
   * Overwrite `~/HOT_DEALS.md` using the format in §3.

5. **Report.** Display a concise success message indicating which action was taken (created, updated, downgraded, cleaned up, or filtered out) and confirm `~/HOT_DEALS.md` was synced.

## 3. Required Format for `~/HOT_DEALS.md`

The file must contain only this title and table. Probability is always the first column.

# High Priority Pipeline

| Probability | Deal ID | Company / Lead | Timeframe Signal | Context Summary |
| :--- | :--- | :--- | :--- | :--- |
| 85% | 1045 | Acme Corp | High Urgency (ASAP) | Needs implementation before the end-of-month accounting close. |
| 65% | 1089 | Globex Inc | Short Term (This quarter) | Budget is allocated for Q3 deployment. |

## 4. Agent Behavior

* If MCP tool calls fail (missing parameter, auth issue, etc.), ask the user for the missing information before retrying. Do not edit `~/HOT_DEALS.md` until the Pipedrive action succeeds.
* If multiple organizations match the extracted identifier, list the candidates and let the user pick before continuing.
* If you cannot confidently extract an identifier from the user's message, ask before searching.
