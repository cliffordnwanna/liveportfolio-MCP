# Changelog

## 1.1.0

- New Awards section (prizes and placings): `portfolio_add_award`, `portfolio_update_award`, `portfolio_remove_award` (needs approval), `awards` in `portfolio_reorder`, and a `set_awards` operation for batches. 59 tools.
- `portfolio_get` returns `display_report` and `text_usage` (characters used, limit, remaining, words). `portfolio_preview_changes` returns `would_display`, so an assistant knows before saving whether everything will show.
- The public page now shows every stack item, highlight line and testimonial on a computer (before, it showed 5 stack items, one testimonial and, in one layout, one highlight line). On a phone, long lists fold behind a labelled "Show N more" button.
- Project text and testimonials are no longer cut at 800 and 500 characters; the page shows up to the 1500 and 600 character limits.

## 1.0.0

- First public version of the LivePortfolio MCP server at `https://liveportfolio.site/mcp` (Streamable HTTP, OAuth 2.1 with PKCE), Pro accounts.
- 56 tools covering reading, profile and contact, skills, experience, projects, education, certifications, testimonials, pictures, design, undo and batches.
- Approval on the website for removals, whole-list replacements, batches, new links and sharp cuts.
- Listed in the official MCP registry as `io.github.cliffordnwanna/liveportfolio`.
- Documentation added: architecture, security, tools and design decisions.
