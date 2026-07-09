# StackIQ Action — translation table and routing tree

## Table of contents
- Notify-by-email (confirmed action)
- Billing-model action translation
- Routing decision tree
- Channel ladder
- Approval script

## Notify-by-email (confirmed action)
The one action StackIQ wires end to end: email the responsible party the required fix, then track it to closure. Use it for any finding whose remediation the client must perform themselves (the common case at `read_only`/`limited` access).

Steps:
1. **Find a send path — ask, don't assume.** Auto-detect a *send-capable* connector. Two rules:
   - A connector that only reads/searches mail (e.g. a search-only Outlook connector) is NOT a send path — do not treat its presence as "email works," and do not conclude "email won't work" from its absence.
   - Never silently send an internal note from an off-profile mailbox (e.g. Apollo's sales mailer). If the only send-capable connector isn't the user's own mailbox, treat it as "no clean send path" and go to the menu — offer it only with the user's explicit okay.

   When there's no clean same-mailbox send connector, ask the user which channel to use, offering: (a) the Outlook desktop app via computer use — recommend this for internal mail (steps below), (b) send from signed-in webmail via Chrome, (c) another connected email account, (d) a transactional provider like SendGrid/Twilio, (e) an off-profile connector like Apollo only if the user explicitly okays it, (f) StackIQ drafts it and the user sends it. Use their pick. Only if the user confirms none are usable → `blocked`, reason "no email channel available, per user." Never fall back to shell/HTTP mail.

### Outlook desktop send (computer use)
Use when the user picks the desktop Outlook path. This drives the native app on their machine, so it needs computer-use access and hard visual confirmation.
1. **Request access** to "Microsoft Outlook" (computer use), then `open_application` Outlook and take a `screenshot` to confirm it's frontmatter and signed in. If it isn't installed/open or asks for login, report that and offer another channel — do not force it.
2. **New message.** Open a compose window (New Email / Ctrl+N) and screenshot to confirm the compose window is focused.
3. **Fill fields** from the approved draft: To = the runtime recipient, Subject, and Body (the required action + sized impact + affected entities). Screenshot after filling so the exact content is verified before sending.
4. **Approval gate still applies** — the drafted content was already approved in step 4 of the main flow; if anything was edited on screen, re-confirm with the user before sending.
5. **Send** (Send button / Ctrl+Enter) and take a screenshot showing the compose window closed / message in Sent. Only treat the send as done on that visual confirmation.
6. **Record** channel `desktop_app` (Outlook) and set status `awaiting_confirmation` with `recipient` + `sent_date` — never `executed` on send.
Hard-stop and offer another channel on any account/permission prompt, MFA challenge, or if the send can't be visually confirmed.
2. **Ask recipient at runtime.** Prompt for the recipient email (the admin/owner who can act). Never infer or reuse an address without asking.
3. **Draft.** Subject names the tool + fix; body states the specific action from the finding's `recommendation_hook`, the sized impact, and the affected entities (e.g. the exact dormant seats). Keep it short and actionable.
4. **Approval gate (hard).** Show recipient + subject + body + impact; wait for explicit user YES. No YES → leave `pending_approval`.
5. **Send** via the detected provider; capture `sent_date` and `recipient`.
6. **Status `awaiting_confirmation`.** Write it to all three sinks and into the workbook Action tab immediately. Do NOT mark `executed` on send.
7. **Confirmation → executed.** Close the loop when any of these is true: a reply/confirmation is detected via the email provider, the user says YES it's done this session, or the next Diagnostic run shows the finding resolved. On close, set `confirmed_date` and flip the workbook status cell to `executed`. Until then it carries forward across runs.

## Billing-model action translation
Branch on the affected tool's `billing_model` (from the diagnostic snapshot). Never default to seat remediation.

| billing_model | Typical waste finding | Action to translate |
|---|---|---|
| `seat` | dormant / never-logged-in seats | revoke the dormant seat, or reassign it to a waitlisted user |
| `consumption` | idle API key, dead integration, over-committed spend | throttle/rotate the key, downgrade the usage tier, or kill the dormant integration |
| `flat` | a single subscription no one uses | cancel it, or consolidate into an overlapping tool |
| `tiered` | paying for a tier the usage doesn't justify | right-size the tier down to the usage-justified level |

Use the finding's `recommendation_hook` as the seed and make it concrete against the named tool/vendor from `intake.scope`. This applies identically whether the tool was named at intake or surfaced by Diagnostic's stack discovery — `billing_model` is what routes the action, not how the tool entered scope.

## Routing decision tree
For each translated action, classify into exactly one route:

1. **Prohibited-action boundary?** If the action is a permission/access change, a billing/subscription change, or a hard delete → route to **manual guidance** and record status `blocked` only if it also can't be guided; otherwise present it as manual steps (still gated by user YES) and, if the user performs it, mark `executed` via `channel: manual_guidance`. These are never auto-executed even with approval.
2. **Above the `intake.access` ceiling?** (e.g. action needs admin, access is `read_only` or `limited`) → `blocked`, with `blocked_reason` naming the missing permission.
3. **No programmatic path (no MCP, not doable in Chrome)?** → degrade to **manual guidance**; if not even guidable, `blocked` with reason.
4. **Within ceiling + programmatic path exists** → eligible for execution; send to the approval gate. Until the user says YES it sits as `pending_approval`; after YES and confirmed run it is `executed`.

## Channel ladder
Attempt in order, stop at the first that works:
1. `mcp` — a live connector action for the tool.
2. `chrome` — signed-in Claude in Chrome; permitted here to click action controls (Diagnostic could not). Require visual DOM confirmation before marking `executed`; hard-abort on CAPTCHA, rate-limit, account restriction, or any "unusual activity" prompt.
3. `manual_guidance` — numbered step-by-step instructions the user runs themselves.

For the notify-by-email action specifically: `mcp` = a send-capable email connector; `chrome` = compose and send from the user's signed-in webmail (require visual send confirmation); `desktop_app` = drive the Outlook desktop app via computer use (steps in the Outlook desktop send section, require visual send confirmation); `manual_guidance` = hand the fully drafted email (recipient, subject, body) to the user to send. All of these set `awaiting_confirmation` on send, not `executed`.

Record the channel actually used on every executed row.

## Approval script
Present, per action (or per sensible batch): the target tool + entity, the specific action, the sized impact (from the finding), the channel, and the reversibility. Then wait for an explicit YES. Silence, "maybe," or "looks good" without a clear YES is not consent — leave it `pending_approval`.
