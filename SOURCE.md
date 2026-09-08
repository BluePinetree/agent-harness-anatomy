# Source repository

The system analysed in this repository is archived separately.

| | |
|---|---|
| **Repository** | `BluePinetree/MARS` |
| **Contents** | ~14,500 lines of Python, a React streaming UI, three framework prototypes |
| **History** | 26 commits, 2026-06-04 to 2026-08-25 |
| **State** | Archived, read-only. Not maintained |

## Why the code is not duplicated here

This repository is the *analysis*. The original is the *primary source* — its value is that
it is dated and unedited. Copying the code here would mean maintaining two versions of the
same artifact and re-auditing 14,500 lines for a second publication.

If you want to read the implementation, read it there. Everything in [findings/](findings/)
cites specific files by path.

## A note on the original repository history

The original local and remote histories diverged. A `filter-branch` rewrite — removing a
355 MB dataset blob that exceeded the file-size limit — produced a local branch sharing no
common ancestor with the published one. The published history is the complete line through
2026-08-25; the divergent local branch is 9 commits ending 2026-08-06, and a small amount of
its work — the shared stability platform under `rsp/`, for one — was never pushed.

This is left as it is rather than reconciled. A study record should show what happened, and
breaking your own history with `filter-branch` and not noticing for two months is part of
what happened.

## Third-party material

The original working tree contained a vendored copy of a proprietary codebase whose own
licence file declared it non-redistributable. It was never committed — verified as zero
objects in any ref — and has been deleted.

The Korean architecture analysis I wrote while reading it is preserved in
[notes/claude-code-analysis/](notes/claude-code-analysis/), with all verbatim source
excerpts removed. What remains is my own prose and my own diagrams.
