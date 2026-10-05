---
name: uxaudit
description: Audit a live page with UXAuditPro, map each finding to the code in this repository, apply only the fixes the user approves, then re-audit and show the before and after comparison.
argument-hint: "[page-url]"
disable-model-invocation: true
allowed-tools: Read Grep Glob
---

# /uxaudit

Run a UXAuditPro audit loop on the code in this repository:

1. Get an audit of the live page.
2. Map each finding to the code that causes it.
3. Propose fixes and **wait for the user to approve them**.
4. Apply only the approved fixes.
5. Re-audit the page once the fixes are live, and show the comparison.

The UXAuditPro tools come from this plugin's MCP server: `get_profile`, `list_audits`, `get_audit`, `start_audit`, `start_paid_audit`, `compare_audits` and `get_checkout_link`. If they aren't available, tell the user to run `/mcp`, choose **uxauditpro** and authenticate, then stop.

## Rules

- **Never edit a file before the user approves the change.** Show the plan first. Asking "shall I go ahead?" is not approval. The user has to answer.
- **Never spend a credit without asking.** `start_audit` uses the free monthly audit. `start_paid_audit` spends a paid credit: call it only after the user says yes to that specific audit, and pass `confirm_spend: true` only then. If they have no credit, offer `get_checkout_link` and stop.
- **Only claim what the audit says.** Every proposed fix names the finding it answers. Don't add fixes the audit didn't ask for, and don't promise a result before the re-audit shows it.
- **Don't commit or push unless asked.** A re-audit sees the live page, so the fixes have to be deployed first. Say so, and let the user decide how to deploy.

## 1. Find the page and its audit

- The page URL is `$ARGUMENTS` if the user gave one. Otherwise look for it in the repository: a `CNAME` file, `homepage` in `package.json`, or the README. If you still can't tell, ask.
- Call `list_audits` and take the newest `DELIVERED` audit of that exact URL.
  - If there is one, tell the user its date and score, and ask whether to use it or start a fresh audit.
  - If there is none, or they want a fresh one, call `get_profile`. If a free audit is left, call `start_audit`. If not, ask before `start_paid_audit`, as in the rules.
- Poll `get_audit` about every 30 seconds until the status is `DELIVERED`, which takes about 4 minutes. Read every page of findings (`findings_page`) until `has_more_findings` is false.
- If `full_report` is false, only one finding is open. Say so plainly, work on that one, and mention that the full report unlocks the rest.

## 2. Map each finding to the code

Work through findings in order of severity. For each one, find the file and line that causes it. Use the most exact evidence the finding has:

- **Measured elements first.** A finding can list `elements`, each with a CSS `selector`, a text `snippet`, a `rule` and the measured values:
  - `#id`, `tag[data-testid="..."]`, `tag[name="..."]`: search the markup and components for that id or attribute.
  - A path such as `#main > section:nth-of-type(2) > div:nth-of-type(1) > p:nth-of-type(1)`: start at the anchor (`#main` here), then walk the children in that template or component. Confirm the match with the `snippet` text.
  - `contrast`: find where that element gets its colour. Search the stylesheets for the measured foreground hex, then confirm the rule applies to the element.
  - `tap_target_size`, `tap_target_spacing`, `skip_link`: find the size, padding, margin or line-height that sets the measured box.
  - `missing_alt`: find the `<img>` whose `src` matches the snippet.
- **Then the evidence quote and the title.** A finding with no elements, such as "hero has no call to action", is mapped by its text. Mark these as lower confidence.
- If you can't find the code for a finding, say so. Don't guess a location.

## 3. Propose, then wait

Show one numbered list. For each item give:

- the finding: severity, title, and the measured value where there is one;
- the file and line;
- the exact change, as a short diff;
- confidence: **measured** (matched by selector and value) or **inferred** (matched by text).

For a contrast fix, pick the nearest colour to the original that reaches 4.5:1 (or 3:1 for large text) against the measured background, and state the new ratio. For a tap target, bring the box to 44 x 44px, or space neighbours at least 8px apart.

Then ask: **"Which of these should I apply? Say 'all', list the numbers, or 'none'."** Stop and wait for the answer.

## 4. Apply what was approved

Make only the approved edits, then show a short summary of the changed files. Run the project's own checks if it has them, such as the build, lint or tests. Report the results honestly: if a check fails, say so.

Then tell the user the re-audit needs the fixes live, and ask how they want to deploy. Offer to commit and push if the project deploys on push, for example GitHub Pages.

## 5. Re-audit and compare

Once the user says the fixes are live:

- Start the re-audit the same way as step 1, asking first if it would spend a credit.
- When it's `DELIVERED`, call `compare_audits` with the earlier audit first.
- Show the result in the tool's own words:
  - **Resolved**: not reported in the later run.
  - **New**: not reported in the earlier run.
  - **In both**: still reported, with the trend where there is one (for example "9 newly failing, 2 no longer reported").
  - **Possibly the same**: pairs the tool couldn't match with confidence. These count as neither resolved nor new.
- For each fix you applied, say which finding it targeted and whether that finding is now Resolved. If one isn't, say so, and suggest a next step.
