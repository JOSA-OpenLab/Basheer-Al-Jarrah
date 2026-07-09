**Project:** [mdn/content](https://github.com/mdn/content) — the source repository for MDN Web Docs

---

## Task

Find a genuine, open issue in a major documentation project and ship a real PR: fork, branch, fix according to the project's style and tooling conventions, and write a proper PR description.

---

## What I Built

A documentation fix PR against `mdn/content` resolving [issue #44644](https://github.com/mdn/content/issues/44644) — the `<q>` element example on [`Web/HTML/Reference/Elements/q`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/q) rendered a visible space _inside_ the quotation marks because the quoted text sat on its own indented lines within the tag.

---

## Problems I Ran Into (and Fixed)

| Problem                                                    | Root Cause                                                                                                                                     | Fix                                                                                            |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `yarn install` failed: `packageManager: "yarn@npm@11.9.0"` | MDN migrated from Yarn to npm; the repo declares `"packageManager": "npm@11.9.0"`, and legacy Yarn 1.x garbles the field by prepending `yarn@` | Used `npm ci` instead — installs exactly what `package-lock.json` pins; no Corepack needed     |
| One-line fix exceeded Prettier's print width               | Prettier wraps at 80 chars, and `<q>` is whitespace-sensitive                                                                                  | Ran `npx prettier --write` and committed its hanging-bracket output instead of hand-formatting |
| Full clone would be multi-GB                               | `mdn/content` has enormous history                                                                                                             | `--depth 1` shallow clone; GitHub accepts PR pushes from shallow clones                        |
> [!status] 
> The PR was merged by the maintainer