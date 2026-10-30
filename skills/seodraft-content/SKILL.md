---
name: seodraft-content
description: Turns OpenSEO research (keyword research, clusters, Search Console queries, AI answer research, audits) into briefed, evidence-backed article drafts through the seodraft MCP. Use when the user has OpenSEO findings and asks to write, plan, or queue content from them.
license: MIT
compatibility: Requires the OpenSEO MCP and the seodraft MCP (https://seodraft.app/mcp) connected to the agent.
---

# seodraft content from OpenSEO research

OpenSEO finds what to write about. seodraft turns a chosen topic into a draft that passes the rules (a saved brief, at least three pieces of first-hand evidence, the answer first under every H2) and delivers it to the user's CMS as a draft. This skill is the handoff between the two. Nothing here publishes: publishing stays a human click.

## Setup (once)

1. Connect the OpenSEO MCP to your agent (MCP server or plugin, see https://openseo.so).
2. Connect the seodraft MCP: add the remote server `https://seodraft.app/mcp` and sign in when the agent opens the OAuth window.
3. Verify both: OpenSEO `whoami` and `list_projects` should return your account and project; seodraft `list_workspaces` should return your site. If either fails, stop and fix the connection first.

## Inputs

- An OpenSEO `projectId` (call `list_projects`) and its matching seodraft workspace (call `list_workspaces` in seodraft; pass its id as `workspaceId` on every seodraft call when there is more than one).
- A source of topics, any of: saved keywords or a tag, a keyword-clustering or keyword-gap result, Search Console queries, AI prompt research, or a site audit.
- How many articles the user wants and how often. If not given, ask.

## Step 1. Read both sides (free)

1. OpenSEO `get_project_context`: business, goal, competitors, key pages, research log. Reuse research from the last 30 days instead of buying it again.
2. seodraft `get_profile`: business, voice, language, accepted evidence. If `profileApprovedAt` is null, stop and send the user to finish setup in seodraft; `plan_topics` refuses until then.
3. seodraft `import_sitemap` (free) if `siteInventory` is null, so new topics are checked against pages the site already has.

## Step 2. Pick topics from OpenSEO

Pick from whichever source the user has. Prefer questions and informational or commercial intent; a transactional cluster usually needs a product page, not an article.

| OpenSEO source | Tool | What to take |
|---|---|---|
| Saved keywords | `list_saved_keywords` (filter by tag) | Terms with intent and volume already cached |
| Clusters | output of keyword-clustering | One primary keyword per "new page" cluster, never one per keyword |
| Keyword gap | `get_ranked_keywords` on competitors | Gap groups that need a missing article |
| Search Console | `get_search_console_performance` with `dimensions: ["query","page"]` | Queries with impressions and no page that answers them; queries at positions 8 to 20 split across URLs (fix, do not add) |
| Opportunities | `get_search_opportunities` | Pages at positions 4 to 20 worth a supporting article that links to them |
| AI answers | `research_ai_visibility_prompts`, `get_ai_visibility_sources` | Questions where the brand is missing and the domains the answers cite |
| Audit | `get_audit_issues`, `get_audit_pages` | Thin pages to rewrite instead of new topics |

Show the user the shortlist (term, source, why) and ask which to send to seodraft. Do not send more than they approve.

## Step 3. Hand topics to seodraft

1. `add_topics` with the approved terms in one call (one batch, about USD 0.11 on the user's own DataForSEO, whatever the count). Read `duplicates`, `similar` and `published`: those terms were withheld because they compete with an existing topic or page. Tell the user; resend with `allowSimilar: true` only if they confirm.
2. `research_topic` per term for the citability brief (People Also Ask, cited domains, AI Overview). About USD 0.004 per term when already measured.
3. `submit_topic_analysis` with gaps and angles. Every gap must cite an exact string seodraft already holds (a cited domain, a result title, a PAA question, a topic term). You may add domains OpenSEO found in AI answer sources only if they also appear in the stored brief.
4. Ask before `plan_topics`. It schedules onto the calendar; `everyDays` sets the gap between articles.

## Step 4. Evidence

The rules block any draft with fewer than three accepted evidence pieces. Read `list_evidence`. If the bank is thin, propose pieces from OpenSEO data the user owns, for example a Search Console query with its clicks, or a GA4 landing page outcome. Add them with `add_evidence` and `saidByHuman: false`: they wait for the user's approval in the seodraft app. Only mark `saidByHuman: true` when transcribing what the user said in this conversation.

## Step 5. Write, check, approve

1. `get_skill` once for the write pipeline, then `next_write` for the next scheduled term.
2. `upsert_post` with the `brief` first (intent, template, angle, three accepted evidence pieces, outline, image slots), then the body.
3. `get_linkgraph` for internal links. Include the key pages from OpenSEO project context that match the topic.
4. `run_gate`. Fix every finding and run it again. Do not approve a draft that has not passed.
5. Read `pipeline_status`. Call `approve_post` yourself only when it shows `approvalMode: agent`. With `approvalMode: human`, show the draft and ask the person; approve only after they say yes in the conversation. Approval delivers a draft to their CMS; it never publishes.

## Step 6. Close the loop in OpenSEO

- Tag the sent terms with `save_keywords`, tag `seodraft` (ask first).
- Append to the research log with `update_project_context`: `{ appendResearchLog: { summary: "seodraft: <n> topics sent, <n> drafts approved" } }`.
- After publishing, suggest adding the article's main term to a rank tracker (`add_rank_tracking_keywords`; show `estimate_rank_tracker_cost` first).

## Output

A short summary: topics sent, topics withheld and why, drafts written and whether each passed the rules, drafts delivered, and evidence waiting for the user's review.

## Notes on cost and stopping

- OpenSEO tools spend OpenSEO credits; seodraft measurement spends the user's own DataForSEO balance. Say the estimate before any paid call and stop if `degraded: spend-limit` comes back.
- Stop and ask when: the seodraft profile is not approved, a topic collides with a published page, evidence is pending, or a cluster maps to an existing page (update it instead).
- No tool here guarantees rankings or traffic. The skill produces briefed drafts; results depend on the site.
