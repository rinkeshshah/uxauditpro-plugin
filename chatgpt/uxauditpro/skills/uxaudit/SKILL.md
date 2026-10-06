---
name: uxaudit
description: Audit a live web page with UXAuditPro, map each finding to the code in this repository, apply only the fixes the user approves, then re-audit the live page and show the before and after comparison.
---

# UXAuditPro audit loop

Run a UXAuditPro audit loop on the code in this repository:

1. Get an audit of the live page.
2. Map each finding to the code that causes it.
3. Propose fixes and **wait for the user to approve them**.
4. Apply only the approved fixes.
5. Get the fixes deployed (commit and push **only with the user's yes**), then wait until the live page serves them.
6. Re-audit the live page and show the comparison.

The UXAuditPro tools are `get_profile`, `list_audits`, `get_audit`, `start_audit` and `compare_audits`. If they aren't available, ask the user to connect UXAuditPro, then stop.

## Rules

- **Never edit a file before the user approves the change.** Show the plan first. Asking "shall I go ahead?" is not approval. The user has to answer.
- **Only use free audits.** `start_audit` uses an open free re-audit for this page, or the free monthly audit. If `get_profile` shows no free audit left this month, say when the next one is (`next_free_audit_at`), offer to work from the newest delivered audit instead, and stop there.
- **Only claim what the audit says.** Every proposed fix names the finding it answers. Don't add fixes the audit didn't ask for, and don't promise a result before the re-audit shows it.
- **Never commit or push without the user's yes.** The re-audit reads the live page, so the fixes have to be deployed first. Show the exact commit and where it will be pushed, and wait for the user to say yes. If they would rather deploy themselves, tell them exactly what to do.
- **Never re-audit stale content.** Before starting the re-audit, confirm the live page is serving the fixes (step 5). A re-audit of the old page wastes the audit and reports nothing as fixed.

## 1. Find the page and its audit

- Use the page URL the user gave. Otherwise look for it in the repository: a `CNAME` file, `homepage` in `package.json`, or the README. If you still can't tell, ask.
- Call `list_audits` and take the newest `DELIVERED` audit of that exact URL.
  - If there is one, tell the user its date and score, and ask whether to use it or start a fresh audit.
  - If there is none, or they want a fresh one, call `get_profile`, then `start_audit` if a free audit is left. If none is left, follow the rule above.
- Poll `get_audit` about every 30 seconds until the status is `DELIVERED`, which takes about 4 minutes. Read every page of findings (`findings_page`) until `has_more_findings` is false.
- If `full_report` is false, only one finding is open to you. Say so plainly and work on that one.

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

## 5. Deploy the fixes

The re-audit reads the live URL, not these files. Tell the user plainly: **the fixes have to be live before the re-audit, or it will audit the old page.**

Work out how the site deploys: a `CNAME` file or a `*.github.io` URL means GitHub Pages; a workflow in `.github/workflows/` that deploys on push; or ask.

- **The site deploys on push (GitHub Pages, or a deploy workflow):** offer to commit and push. Show the commit message, the branch, and the remote. Run `git commit` and `git push` **only after the user says yes**. If they say no, or you can't run commands here, give them the two commands to run and wait for them to say they've pushed.
- **Any other deploy:** tell the user what has to go live (the changed files) and wait for them to say it's deployed.

Then **wait for the deploy to finish**. Don't trust a fixed delay.

- **GitHub Pages:** a push takes about a minute to build, and can take longer. If `gh` is available, check every 20 seconds:
  `gh api repos/OWNER/REPO/pages/builds/latest --jq '.status + " " + .commit'`
  Wait until it reads `built` with the commit you just pushed. If it reads `errored`, stop and show the user.
- **Every deploy, Pages included:** confirm the live site serves the change. Fetch each changed file from the live URL, for example `curl -s https://USER.github.io/REPO/styles.css`, and check that a value you changed is there (the new colour, the new padding, the new `alt` text). Check again every 20 seconds.

**Give it 10 minutes.** If the live site still serves the old content, stop. Tell the user what you checked and what you saw, and don't start the re-audit.

## 6. Re-audit and compare

Once the live page serves the fixes:

- **Use the free re-audit when it is open.** Some audits come with one free re-audit of the same page. Call `get_audit` on the earlier audit:
  - If `free_reaudit_available_until` is set, say so and call `start_audit` with the same URL.
  - If it is not set, say plainly why it isn't available (already used, or the 30 days have passed), then start the re-audit the same way as step 1.
- When it's `DELIVERED`, call `compare_audits` with the earlier audit first.
- Show the result in the tool's own words:
  - **Resolved**: not reported in the later run.
  - **New**: not reported in the earlier run.
  - **In both**: still reported, with the trend where there is one (for example "9 newly failing, 2 no longer reported").
  - **Possibly the same**: pairs the tool couldn't match with confidence. These count as neither resolved nor new.
- For each fix you applied, say which finding it targeted and whether that finding is now Resolved. If one isn't, say so, and suggest a next step.
