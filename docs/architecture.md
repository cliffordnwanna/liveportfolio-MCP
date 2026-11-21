# Architecture

How the LivePortfolio assistant connection works. This describes what the server does today; where something is not verified, it says so. The application source is not public.

## System overview

```mermaid
flowchart LR
  A[Assistant client<br/>Claude, ChatGPT, Claude Code] -->|HTTPS + OAuth token| M[MCP endpoint<br/>/mcp]
  A -. sign-in .-> O[OAuth 2.1 server]
  O --> C[Consent page<br/>liveportfolio.site]
  M --> G{Checks on every call:<br/>token, app connection,<br/>access, rate limit}
  G --> T[Tools]
  T --> P[Operation layer<br/>validation + limits]
  P --> V[Versions and history<br/>compare-and-set]
  V --> D[(Database)]
  T -->|needs approval| Q[Approval request]
  Q -. link .-> U[You, on the website]
  U --> R[Approval page<br/>owner only]
  R --> P
  D --> S[Public portfolio site]
```

Key points:

- Assistant tokens are accepted only at `/mcp`. Browser sign-in tokens are rejected there, and assistant tokens are rejected on every other API route of the site.
- Every change, from the dashboard or an assistant, goes through the same operation layer. There is no second way to write portfolio content.
- Changes that could hurt (removals, whole-list replacements, batches, new links, sharp cuts) are not applied by the assistant. It receives a link; you approve on the website.

## OAuth 2.1 sign-in with PKCE, and token scoping

```mermaid
sequenceDiagram
  participant U as You
  participant A as Assistant client
  participant S as Auth server (OAuth 2.1)
  participant W as Consent page (liveportfolio.site)
  participant M as /mcp
  A->>M: first request, no token
  M-->>A: 401 + pointer to protected-resource metadata
  A->>S: discover, register as an OAuth client
  A->>S: authorization request (PKCE challenge)
  S->>W: send you to the consent page
  W->>U: who is asking, allow edits? share contact details? (off by default)
  U->>W: Connect
  W->>W: record the connection for you and this app
  W->>S: approve
  S-->>A: authorization code
  A->>S: token request (PKCE verifier)
  S-->>A: access token (carries the OAuth client id)
  A->>M: tool call + token
  M->>M: token valid? issued to an OAuth client? connection still active? account allowed? within rate limit?
  M-->>A: result
```

Token scoping:

- An access token issued to an assistant carries an OAuth client identifier; a browser sign-in token does not. `/mcp` accepts only the former. Every other API route refuses the former.
- Authorisation is not left to the token alone: each call also checks that you still have an active connection for that app. Revoking the connection stops the very next call, even while the token has not expired.
- The consent screen shows a "Verified" badge only when the return address is exactly `chatgpt.com` or `claude.ai` over https, and a warning for anything else.

## A direct edit (saved at once, undoable)

```mermaid
sequenceDiagram
  participant A as Assistant
  participant M as /mcp tool
  participant P as Operation layer
  participant V as Versions
  A->>M: tool call with the version it last read
  M->>M: rate limit, read-only connection check
  M->>M: policy: new link? sharp cut? then it needs approval instead
  M->>P: typed operations
  P->>P: validate (limits, links, plan rules) - all or nothing
  P->>V: snapshot the current state, then write only if the version is unchanged
  alt version changed since it was read
    V-->>M: conflict
    M-->>A: refused, read again and retry
  else
    V-->>M: saved as a new version
    M-->>A: what changed, the new version, warnings if not everything is shown
  end
  Note over A,V: "Undo" restores the snapshot as a new version, so an undo can be undone
```

## An approval-gated edit

```mermaid
sequenceDiagram
  participant A as Assistant
  participant M as /mcp tool
  participant R as Approval request
  participant U as You (browser)
  participant P as Operation layer
  A->>M: remove / replace / batch / new link
  M->>M: validate the change without saving it
  M->>R: store the proposal (expires in 30 minutes, single use)
  M-->>A: "NOTHING HAS CHANGED YET" + approval link (opaque id)
  A-->>U: open this link, approve, then reply "approved"
  U->>R: open the link (must be signed in as the owner)
  R-->>U: before and after
  alt you decline, or it expires
    R-->>U: nothing was changed
  else you approve
    R->>R: claim it (single use) - portfolio unchanged since the proposal?
    R->>P: apply, saved as a new version
    R-->>U: applied
  end
  U-->>A: "approved"
  A->>M: check the proposal
  M-->>A: applied
```

The assistant never holds the power to approve: the approval endpoint accepts browser sign-in tokens only, and assistant tokens are refused there.

## What is verified

Covered by automated tests that run against an in-memory database with throwaway users: token separation on every API route, per-call connection checks and revocation, access by plan, rate limits, isolation between users, version conflicts and concurrent edits, undo, the approval flow (before approval nothing changes, expiry, single use, owner-only), field limits, link validation, and hostile text stored as plain data.

Not verified by automation: the real OAuth sign-in with each vendor's client, how each assistant chooses tools, and how the approval page looks in each browser. These are checked by hand.
