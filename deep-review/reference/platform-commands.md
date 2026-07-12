# Platform Commands (GitHub / GitLab)

The review skills are host-agnostic. This file maps every git-host operation they need onto the
matching CLI — [`gh`](https://cli.github.com/) for GitHub, [`glab`](https://gitlab.com/gitlab-org/cli)
for GitLab — so `SKILL.md` can stay declarative ("run the platform-appropriate command") and never
hardcode one vendor.

## 1. Detect the platform from the git remote

Derive the platform from `origin` before doing anything else; do not assume GitHub.

```bash
origin=$(git remote get-url origin 2>/dev/null || git remote get-url "$(git remote | head -1)" 2>/dev/null)
case "$origin" in
  *github.com*) cli=gh   ;;                 # github.com
  *gitlab*)     cli=glab ;;                 # gitlab.com or *.gitlab.* self-hosted
  *)                                        # self-hosted GHE / self-hosted GitLab with a custom host
    if   gh   auth status >/dev/null 2>&1; then cli=gh
    elif glab auth status >/dev/null 2>&1; then cli=glab
    else cli=ask; fi ;;                     # ambiguous → ask the user which host this is
esac
```

Self-hosted refinement: a GitHub Enterprise or self-hosted GitLab host won't contain
`github.com`/`gitlab`. If both CLIs are authenticated, disambiguate by host —
`gh auth status --hostname <host>` succeeds only for GitHub-authenticated hosts; otherwise use `glab`.
When still unsure, ask rather than guess.

## 2. Vocabulary

Same concept, different name — use the host's term in the report ("PR" on GitHub, "MR" on GitLab):

| Concept                    | GitHub                     | GitLab                          |
| -------------------------- | -------------------------- | ------------------------------- |
| Change unit                | pull request (PR)          | merge request (MR)              |
| Reviewed head commit       | `headRefOid`               | diff head sha (`.sha`)          |
| Top-level comment          | issue comment              | note                            |
| Line/diff comment + thread | review comment / thread    | diff note / discussion          |
| Thread resolved flag       | `reviewThreads.isResolved` | discussion `.resolved`          |
| CI                         | checks                     | pipeline                        |

Both hosts render Markdown, HTML comments (`<!-- deep-review sha:… -->`), and Mermaid in comment
bodies, so the marker-stamp and Module Coupling Map mechanisms work unchanged on either.

## 3. Operation map

`<iid>` = MR internal id, `<id>` = comment/note id — capture each from the step that lists it.
For `glab api`, `:id` and `:fullpath` are auto-substituted with the current repo's project; the MR
iid is **not** a placeholder, so interpolate the value you fetched. `@file` reads the field from a file.

| Operation                        | GitHub (`gh`)                                                                          | GitLab (`glab`)                                                                                       |
| -------------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Identify change on branch        | `gh pr view`                                                                           | `glab mr view`                                                                                         |
| Metadata (title, body, issue)    | `gh pr view --json title,body,url`                                                     | `glab mr view -F json` → `.title`, `.description`, `.web_url`                                          |
| MR iid (needed for API calls)    | — (the PR number *is* the id)                                                          | `glab mr view -F json \| jq -r '.iid'`                                                                 |
| CI / pipeline status             | `gh pr checks`                                                                         | `glab ci status`                                                                                      |
| **Head SHA** (for the marker)    | `gh pr view --json headRefOid -q .headRefOid`                                          | `glab api "projects/:id/merge_requests/<iid>" \| jq -r '.sha'` (fallback `.diff_refs.head_sha`)        |
| List top-level comments          | `gh api repos/{owner}/{repo}/issues/{pr}/comments --paginate`                          | `glab api "projects/:id/merge_requests/<iid>/notes" --paginate`                                        |
| Post a comment (from file)       | `gh pr comment <pr> --body-file report.md`                                             | `glab mr note <iid> -m "$(cat report.md)"`                                                             |
| Update a comment (from file)     | `gh api repos/{owner}/{repo}/issues/comments/<id> -X PATCH -F body=@report.md`         | `glab api "projects/:id/merge_requests/<iid>/notes/<id>" -X PUT --field body=@report.md`               |
| List line/diff comments          | `gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate`                           | `glab api "projects/:id/merge_requests/<iid>/discussions" --paginate`                                  |
| Thread resolution state          | GraphQL `reviewThreads { isResolved }`                                                 | discussions payload → `.notes[].resolvable` / `.notes[].resolved`                                     |

### Finding the marker comment

On both hosts: list top-level comments, take the one whose body starts with `<!-- deep-review`
(or `<!-- review-followup` for the follow-up skill). Capture its id, then use the *update* row to
PATCH/PUT it — never post a second comment. GitHub issue-comment ids are updated via
`issues/comments/<id>`; GitLab note ids via `merge_requests/<iid>/notes/<id>`.
