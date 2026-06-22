## Review Norms: rust-lang/rust

Read three merged PRs spanning different eras and topics — old, contentious, and architecturally deep, to get a real spread rather than one snapshot:

- https://github.com/rust-lang/rust/pull/12081 — _Robin Hood hashmap_ (2014, 52 comments) — DS-heavy, safe/unsafe tradeoff debate
- https://github.com/rust-lang/rust/pull/42388 — _Downgrade ProjectionTy's TraitRef to its substs_ (2017, 60 comments) — internal refactor, mostly mechanical
- https://github.com/rust-lang/rust/pull/95548 — _Add fine-grained LLVM CFI support_ (2022, 65 comments, 100+ inline) — large feature, long-running, spec-heavy

### What gets flagged

- **Unjustified `unsafe`.** On the hashmap PR, the reviewer (alexcrichton) didn't block on a single bug — he was explicit that nothing was _currently_ wrong — but kept pushing because unsafe code is a standing risk for _future_ refactors, not just present correctness.
- **Spec/grammar mismatches stated precisely.** On the CFI PR, the reviewer cited exact Itanium C++ ABI mangling productions line-by-line to show a specific encoding was non-conforming — not "this looks off," but "the grammar says X, you produced Y."
- **Incomplete parallel cleanup.** On the refactor PR, the reviewer insisted a sibling type (`ExistentialProjection`) get the same treatment as the one actually being fixed — leaving it half-done wasn't acceptable even though nothing was technically broken.
- **Long-term maintainability of non-code artifacts.** A reviewer objected to citing a personal website as a permanent reference for compiler internals — "would be a shame if... we found a 404" — durability of _documentation_, not just code.
- **Style nits, called out as nits.** "Use `///` doc-comments" sat right next to serious ABI-correctness comments, clearly lower-stakes, not blocking.

### What doesn't get flagged

- Mechanical fixups matching existing convention pass with a one-word "Done." — no debate once the author conforms to what the reviewer already asked for.
- Genuinely good engineering gets named explicitly, not just implied by silence: "You've done an awesome job containing the unsafety of this implementation, thank you!" — praise and continued pushback for more changes coexist in the same comment.
- Open-ended architectural questions about _future_ work don't block the _current_ PR — they get raised, the author commits to a separate RFC, and review moves on without resolving the bigger question now.

### How disagreements get resolved

- **Defer, don't block.** The C-integer-type/mangling-scheme disagreement on the CFI PR never got fully resolved in-thread — the author proposed writing a separate RFC, the reviewer accepted that as sufficient to keep moving, and the PR proceeded with the open question explicitly logged as future work.
- **Senior reviewer proposes a path, not just an objection.** On the hashmap PR, alexcrichton didn't just say "too unsafe" — he sketched a concrete staged design (start with a safe `Vec<Option<...>>`-backed version, swap to the unsafe one later behind the same interface) so the disagreement became "which path" rather than "yes/no."
- **Reviewer handoff when too close to the work.** On the CFI PR, nagisa explicitly stepped back from being primary reviewer because he'd helped shape the design and didn't feel impartial, handing off to someone less involved — review independence is treated as something to actively protect, not just assume.
- **Git mechanics resolved collaboratively, not adversarially.** When the refactor PR accidentally picked up stray submodule commits, the reviewer didn't just flag it — he gave the exact commands to fix it, and the two went back and forth comparing workflows.

### Infrastructure norms (worth noting separately)

A `r?` reviewer-assignment bot, a CI-failure bot that auto-posts a guessed root cause on every red build, and `bors` managing the actual merge queue and rebase-conflict pings — review is heavily bot-assisted so humans spend their time on substance, not babysitting CI. The welcome bot also codifies a norm directly: push fixup commits during review instead of squashing early, so the reviewer can see exactly what changed since their last pass.