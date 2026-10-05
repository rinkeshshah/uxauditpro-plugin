# UXAuditPro for Claude Code

Audit a live page with [UXAuditPro](https://www.uxauditpro.com), let Claude map
each finding to your code, apply only the fixes you approve, then re-audit and
see what was resolved.

## Install

In Claude Code (v2.1.275 or later):

```
/plugin install uxauditpro --marketplace rinkeshshah/uxauditpro-plugin
```

Then run `/mcp`, choose **uxauditpro** and authenticate with your UXAuditPro
account.

## Use

In the repository for your site:

```
/uxaudit https://www.example.com/
```

`/uxauditpro:uxaudit` works too, if another command already uses `/uxaudit`.

The skill:

1. Uses your latest audit of the page, or starts your free monthly audit.
2. Maps each finding to the file and line that causes it, using the measured
   element's selector where there is one.
3. Shows the proposed fixes and waits. Nothing is edited until you approve.
4. Applies the fixes you chose.
5. Once they're live, re-audits and compares the two audits: Resolved (not
   reported in the later run), New (not reported in the earlier run), and
   what's still there.

A paid audit is only started after you say yes to it. More at
https://www.uxauditpro.com/claude/.
