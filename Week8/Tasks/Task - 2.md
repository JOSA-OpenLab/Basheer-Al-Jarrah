
## Goal

Pick an open-source repo, browse for a N+1 problem
## Target 

[django-helpdesk](https://github.com/django-helpdesk/django-helpdesk)
## Setup
>[!warning] I didn't find an exactly N+1 queries problem, but I've found a SQL performance problem 

Minimal Django 6.0.7 project (no demo app), SQLite, `django-helpdesk` installed
editable from a git clone. One non-obvious requirement:
`HELPDESK_TEAMS_MODE_ENABLED = False`, or migrations abort with a lazy
reference to `pinax_teams.Team` that isn't installed.

Instead of eyeballing a SQL log by hand, I measured query *counts* with
`CaptureQueriesContext` while scaling the number of objects on the page.
## Finding 1 — follow-ups have no eager loading at all

`get_followups_for_ticket()` in `views/staff.py` returns
`ticket.followup_set.order_by(order)` — no `select_related`, no
`prefetch_related`. Then `templates/helpdesk/ticket.html` touches, per follow up:

- `followup.user` — FK, one query each
- `followup.ticketchange_set.all` — called **twice**, once in `{% if %}`
  (line 138) and again in `{% for %}` (line 140). Each call on a related
  manager builds a fresh queryset, so it's two queries, not one.
- `followup.followupattachment_set.all` — same double-call pattern (lines
  150, 153)

Measured cost: exactly **6.0 queries per follow up**, identical at every step
of the scale (1 → 10 → 25 → 50 → 100). Textbook linear.

## Finding 2 — a second, independent N+1 in the same view

After fixing #1 the scaling was still 1.0 queries per follow up. Rather than
call it done, I traced it: `get_attachments_for_ticket()` does
`.select_related("followup")`, but `include/attachment_list_item.html` checks
`attachment.followup.user`. The follow-up is joined, the *user* isn't — one
query per attachment.

Finding this was the most valuable part of the exercise. The obvious fix made
the graph look almost flat, and "almost flat" is exactly where you stop if
you're chasing a feeling instead of a number.

## Numbers

Ticket detail view, one follow-up per author, each with a ticket change and an
attachment. Django test client, response warmed once, SQLite.

| follow-ups | queries before | queries after | ms before | ms after |
|---|---|---|---|---|
| 1 | 37 | 33 | 37 | 41 |
| 10 | 91 | 33 | 79 | 55 |
| 25 | 181 | 33 | 166 | 144 |
| 50 | 331 | 33 | 246 | 131 |
| 100 | 631 | 33 | 462 | 203 |

Before: O(N). After: **constant 33 queries** regardless of follow-up count.
19x fewer queries at 100 follow-ups.

Honest caveat on the wall-clock column: only ~2.3x faster, because SQLite runs
in-process and a query costs microseconds. The real-world impact is bigger than
this table suggests — on Postgres over a network, 598 eliminated round trips at
even 1 ms each is most of a second of pure latency. Query count is the portable
metric here; milliseconds are environment-specific.
