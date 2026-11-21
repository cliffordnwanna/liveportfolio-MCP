# Design decisions

Short, honest trade-offs. Each says what was chosen, why, and what it costs.

## One operation layer for every edit

**Chosen:** the dashboard, the undo feature and every assistant tool all change a portfolio through one validated set of typed operations (about twenty operation kinds, applied all or nothing).
**Why:** one place for limits, link rules and plan rules, so an assistant can never save something the dashboard would reject, and the dashboard can never be bypassed.
**Cost:** each new kind of edit needs an operation first, then a tool. We added a few narrow tools (edit one education entry, rename a skill group) because replacing a whole list for a small change forced needless approvals.

## Compare-and-set on a version number

**Chosen:** an edit carries the version it was based on and is refused if the portfolio changed since.
**Why:** two assistants, or an assistant and a browser tab, must never silently overwrite each other. It also makes index-based edits ("fix certification 2") safe: a stale position fails instead of editing the wrong item.
**Cost:** the assistant must read before it writes and sometimes retry. Tools that target an entry by position require the version.

## Approval happens out of band, on the website

**Chosen:** risky changes (removals, whole-list replacements, batches, new links, sharp cuts, restoring an old version) create a request. You approve on liveportfolio.site; the assistant only receives a link.
**Why:** a confirmation dialog in an assistant app is a hint the client may or may not show, and the same model that proposed a change could "approve" it in a two-step preview-then-apply tool. A separate signed-in page the assistant cannot click is a real boundary.
**Cost:** friction. After approving you switch back to the chat and say "approved". We reduced the number of approvals by adding narrow tools and by letting additive, undoable edits save at once.

## Everything saved is a version, and the first one is kept

**Chosen:** every change stores the state before it. The state before an assistant first edited a portfolio is never deleted, even when old versions are pruned. Undo only reverses an assistant's own latest change.
**Why:** an assistant that edits often must not be able to push your original work out of history, and "undo" must never wipe an edit you made yourself.

## Links need approval

**Chosen:** adding or changing a link on a public page waits for approval.
**Why:** a swapped link on a public page is a phishing risk. Removing a link does not need approval.

## No custom token audience, a per-app connection check instead

**Chosen:** we rely on the OAuth client marker in the token, a rule that assistant tokens work only at `/mcp`, and a connection record checked on every call. We did not add a custom claim that binds tokens to the MCP address.
**Why:** that claim would sit in the sign-in path of every user, and we had no separate environment to test it in. The per-call connection check gives immediate revocation.
**Cost:** tokens are not bound by audience. If a client later requires it, we will add it.

## Known gap: returning you to the chat after approval

A website cannot push a message into ChatGPT or Claude. After you approve, the page tells you to go back to the chat and say "approved". The MCP specification has a mechanism for this, URL-mode elicitation, where the server asks the client to send you to a URL and then notifies the client when it finishes.

We did **not** build it. We do not know which assistant clients support it, and we have not confirmed that ChatGPT or Claude do. The server logs, per connection and without any message content, which capabilities each client declares, so the decision can use real data. If a client advertises it, we would use it and keep the current screen as the fallback.

## What the assistant is deliberately not given

Publishing, the address, billing, sign-in, account deletion, and creating the first portfolio. They are not tools, rather than tools that refuse, so there is no instruction that reaches them.

## Not built yet

- Per-change undo from an activity feed in the dashboard (the history already records which changes came from an assistant).
- A "review before it goes live" setting for published portfolios.
- A read tool for visitor analytics.
