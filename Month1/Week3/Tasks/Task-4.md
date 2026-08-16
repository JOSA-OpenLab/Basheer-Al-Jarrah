
## Repository Information

- **Repository name:** Textualize/rich
- **Repository URL:** https://github.com/Textualize/rich
## Issue Information

- **Title:** [BUG] rich.progress.Progress transient option not working when exception is thrown within context
- **Issue URL:** https://github.com/Textualize/rich/issues/2762
- **Date opened:** 2023‑01‑18 (reporter: TerranWorks)
- **Reporter's environment:** rich 12.6.0, Python on Linux
- **Current status:** Open, labeled `Needs triage`, no comments, no linked/closing PR (verified via the issue page and the open‑issues listing at review time)

## Journal Entry (first person)

For this triage task I chose [Textualize/rich](https://github.com/Textualize/rich), a large and actively maintained terminal‑formatting library (latest release 15.0.0 on 2026‑04‑12), because it has a real backlog of older, untriaged issues where triage is genuinely useful. I picked [issue #2762](https://github.com/Textualize/rich/issues/2762), *"Progress transient option not working when exception is thrown within context,"* which had been open and untriaged since January 2023 with no comments.

I noticed the reporter had added an **Edit 2** retracting the title's premise: the exception is not the trigger — the stray blank line actually appears whenever a `Progress(transient=True)` block renders no visible task (an invisible task, or no task at all). I found that an older issue, [#2381](https://github.com/Textualize/rich/issues/2381), describes exactly this behavior and was even independently confirmed by another user, so #2762 is really a later report of the same root cause.

To verify rather than guess, I reproduced the behavior on two versions. On the reporter's exact version (rich 12.6.0) the blank line is left behind for the invisible‑task and no‑task cases, matching the reports; on the current release (rich 15.0.0) it no longer happens. The changelog also records a related fix, "Fixed extraneous blank line on non‑interactive disabled Progress" (#3905), in 14.3.0.

Based on that evidence I concluded the issue appears already fixed. I drafted a respectful [comment]((https://github.com/Textualize/rich/issues/2762#issuecomment-4642295076)) that corrects the misleading title using the reporter's own Edit 2, links the related #2381, presents a minimal runnable example, states the 12.6.0‑vs‑15.0.0 result, and asks a maintainer or the reporter to confirm before closing. I did not close the issue or claim any maintainer authority. The outcome is a clear, evidence‑backed recommendation that should let a maintainer close #2762 (and likely #2381) quickly, or point me to a case that still reproduces.
