# Pull requests and review history

> **Read this when** you have a commit and need the reasoning behind it, or a PR number
> and need to know what it changed and whether it landed.

**In this file**

| Section | Answers |
|---|---|
| Why PRs matter | Where Bitcoin's reasoning actually lives |
| Commit → PR | The merge-commit trick |
| Reading a PR | `gh` without cloning anything |
| Searching PRs | By topic, file, author, state |
| Review as evidence | ACKs, NACKs, and what they mean here |
| Release notes | The curated version |
| **Contributing** | **Core's AI policy — hard limits** |

---

## Why PRs matter

Bitcoin Core commit messages are terse. The *reasoning* — why a rule is shaped this way,
what alternatives were rejected, what the security argument was — lives in pull-request
descriptions and review threads. For a researcher, the PR is usually the primary source and
the commit is just the artefact.

This works without a clone. `gh` reads the GitHub API directly.

## Commit → PR

Core merges via merge commits whose subject names the PR, so the mapping is right there:

```bash
git log -1 --format='%s' b811aeabad
# Merge bitcoin/bitcoin#36048: util: keep wallet names literal in notification commands
```

Given an arbitrary commit, find the merge that brought it in:

```bash
# the PR number, extracted
git log --merges --oneline --ancestry-path <commit>..master | tail -1 | grep -oE '#[0-9]+'

# or search the log for a known PR number
git log --oneline --grep="#36048" -1
```

`--ancestry-path` matters: without it you get every merge after the commit, not the one
that introduced it.

## Reading a PR

```bash
gh pr view 36048 --repo bitcoin/bitcoin
gh pr view 36048 --repo bitcoin/bitcoin --json number,title,state,mergedAt,author,body
```

Verified output shape:

```
#36048 [MERGED] util: keep wallet names literal in notification commands
by l0rinc, merged 2026-09-02T13:55:11Z
```

More of the surface:

```bash
gh pr view 36048 --repo bitcoin/bitcoin --comments        # the review discussion
gh pr diff 36048 --repo bitcoin/bitcoin                   # the changes
gh pr diff 36048 --repo bitcoin/bitcoin --name-only       # just which files
gh api repos/bitcoin/bitcoin/pulls/36048/reviews --jq '.[] | "\(.user.login)\t\(.state)"'
```

**Check `state` before quoting a PR.** `OPEN` means proposed, not adopted — an enormous
share of confident claims about Bitcoin come from reading an open or closed-unmerged PR as
though it were the implementation. `MERGED` plus a `mergedAt` is the fact that matters, and
even then see "which release" in `research.md`, because merged is not released.

## Searching PRs

```bash
gh search prs --repo bitcoin/bitcoin --merged "taproot activation" --limit 5 \
  --json number,title --jq '.[] | "#\(.number) \(.title)"'
# #31978 kernel: pre-29.x chainparams and headerssync update
# #32386 mining: rename gbt_force and gbt_force_name
# #26201 Remove Taproot BIP 9 deployment
```

Other axes:

```bash
gh search prs --repo bitcoin/bitcoin --state open  "mempool policy" --limit 10
gh search prs --repo bitcoin/bitcoin --author achow101 --merged --limit 10
gh search issues --repo bitcoin/bitcoin "descriptor wallet" --limit 5
```

To find PRs touching a specific file, go through the commits rather than the search API:

```bash
git log --oneline --merges -- src/policy/rbf.cpp | head -10
```

That gives the merge commits — each names its PR — which is more reliable than text search
for "what changed this file".

## Review as evidence

Bitcoin Core review uses a convention worth understanding before quoting it:

| Marker | Means |
|---|---|
| **ACK `<sha>`** | Reviewer approves, pinned to that exact commit |
| **utACK** | "Untested ACK" — read the code, didn't run it |
| **tACK** | Tested it |
| **Concept ACK** | Likes the idea; not an approval of this code |
| **NACK** | Objects, usually with reasoning worth reading |
| **Approach ACK/NACK** | About the design, not the diff |

An ACK is pinned to a sha: if the branch was force-pushed afterwards, that ACK does not
cover the merged code. Check what the ACK references before citing it as approval.

NACKs and their arguments are frequently the most informative part of a thread — they are
where the tradeoffs get stated explicitly. When a question is "why doesn't Bitcoin do X",
a NACK thread is often the real answer.

## Release notes

The curated view, in the tree:

```bash
ls doc/release-notes/ | tail -5
grep -rn "taproot" doc/release-notes/release-notes-22.0.md | head -3
```

Release notes state user-visible changes and deprecations per version — better than a PR
list when the question is "what changed in this release".

## Contributing

Core ships [`doc/AI_POLICY.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/AI_POLICY.md).
Read it in full before helping anyone contribute. The operative limits:

> **"Pull requests should not be opened or driven by autonomous agents."**
> **"Do not include agents as authors or co-authors of your commits."**

Also binding: comments to maintainers and reviewers are expected to be **written by
humans** and may be moderated if they look AI-generated; a contributor should only open a
PR if they know the language, could have written the code, and understand the surrounding
code; and AI-derived context in a comment must be disclosed with human commentary.

What this means in practice:

- **Fine:** researching, reading, explaining, summarising a PR for a user, helping someone
  understand review feedback, drafting code a human will read, understand, and own.
- **Not fine:** opening or driving a PR, writing review or issue comments to be posted as
  the user's own words, or adding an agent as a commit co-author.

If asked to do one of the latter, say so plainly and point at the policy. Bitcoin Core's
review capacity is a scarce shared resource, and the policy exists because wasting it has a
real cost.
