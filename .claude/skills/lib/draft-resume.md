# eng-flow draft and resume (shared reference)

**Status:** implemented for Stage 1 (`eng-flow-spec`, live-tested T6, 2026-10-26) and rolled out to Stages 2 (`eng-flow-domain-model`), 3 (`eng-flow-architecture`), 3.5 (`eng-flow-eng-review`), 3.6 (`eng-flow-ui-design`) and 4 (`eng-flow-epics-stories-tasks`, Steps 0–6 only) on 2026-10-27. The rollout is not yet live-tested. Drafts are gitignored (T8, 2026-10-27).
**Origin:** D4 in `docs/improvements/session-mgmgt.md`.

The goal is to save as the interview goes, so a crash or closed terminal loses at most the step (or loop item) in progress, never the user's earlier answers.

Each stage's `SKILL.md` keeps its own Step 0 trigger and per-step `save` calls inline. This file holds the rules they share: **Resume**, **Review**, **Discard**, stale-step `truncate`, and the in-progress and input-drift conventions. When a `SKILL.md` says "see `lib/draft-resume.md`", Read this file before acting.

---

## Rules

- **One draft file per run, not per session.** A resumed session keeps writing to the same file.
- **Location:** project-level `eng-flow/.drafts/`, named `<prefix>--<dated-slug>.json`, where `<prefix>` is the skill name without `eng-flow-` (`spec--`, `domain-model--`, `architecture--`, `eng-review--`, `ui-design--`, `epics-stories-tasks--`).
- **Slug:** Stage 1 computes `<dated-slug>` at its Step 0, from the topic and the start date. Downstream stages reuse the spec folder's existing `<dated-slug>`, so a stage has at most one draft per spec and there is no "separate run" option. Starting over means discarding.
- **Format:** JSON. The model is less likely to reshape JSON than Markdown. Large content (full drafts, ADRs, HTML mockups) goes inline as JSON strings too; JSON-escape quotes and newlines.
- **When it saves:** as the last action of every numbered step. Per-item loops also save after each item, as `in_progress` (below).
- **What it saves:** the agreed answers in short form, not the conversation, plus any expensive output a resume shouldn't redo (subagent findings, research fetches).
- **Skipped steps are recorded as skipped**, so a resume doesn't ask them again.
- **Gitignored, never committed.** On first write the helper creates `eng-flow/.drafts/.gitignore` containing `*`, so drafts stay out of git in any project with no setup. Decided by the user, 2026-10-27 (T8).

## Draft shape

```json
{
  "skill": "eng-flow-eng-review",
  "topic": "csv export",
  "dated_slug": "2026-10-26-csv-export",
  "path": null,
  "last_completed_step": 3,
  "created": "2026-10-27T09:12:40Z",
  "updated": "2026-10-27T09:51:03Z",
  "inputs": {
    "eng-flow/specs/2026-10-26-csv-export/spec.md": "<sha256>",
    "eng-flow/specs/2026-10-26-csv-export/architecture.md": "<sha256>"
  },
  "steps": {
    "0": { "status": "done", "name": "Inputs", "answers": { "topic": "csv export", "spec_folder": "eng-flow/specs/2026-10-26-csv-export" } },
    "3": { "status": "done", "name": "Independent review", "answers": { "findings": ["…"] } },
    "4": { "status": "in_progress", "name": "Issues", "answers": { "resolved": [{ "title": "…", "resolution": "accepted" }] } }
  }
}
```

`topic` comes from step 0's answers (Stage 1: the user's topic; downstream: the spec's title). `path` is Stage 1's routing path and `null` elsewhere. `inputs` is downstream only.

---

## Statuses

| Status | Meaning | Counts in `last_completed_step`? |
|---|---|---|
| `done` | Step finished; `answers` holds what was agreed | Yes |
| `skipped` | Step didn't apply; `reason` says why | Yes |
| `in_progress` | A per-item loop partway through; `answers` holds the items finished so far | **No.** A resume re-enters this step |

**Per-item loops** save after each item with the same step number, status `in_progress`, and `answers` listing every item finished so far (the whole list each time, not just the new item). When the loop's last item is done, the step saves once more as `done`. Loops that do this: domain-model Steps 1 (per domain) and 8 (per finding), eng-review Step 4 (per issue), ui-design Step 3 (per journey dimension).

On resume into an `in_progress` step, skip the items already in its `answers` and carry on with the next one. Say so in the resume line ("3 of 7 issues resolved").

## Input drift (downstream stages)

Step 0 of a new downstream run records a hash of every input file it read:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" inputs <skill> "<dated-slug>" <file>...
```

On resume, before the resume line:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" check <skill> "<dated-slug>"
```

Each output line is `<file>\tchanged` or `<file>\tmissing`. No output means the inputs match. If any changed, still resume, but name the files in the resume line and suggest *review*: "spec.md has changed since this draft started — say *review* to check the saved answers against it." Don't judge whether the change matters ("it just confirms…") and skip the hint: whether the saved answers still hold is the user's call, so the file name and the *review* hint always appear. If an input is missing, say so and ask before going on, because the step that reads it can't run. Log the outcome: `risk open_question <user_confirmed|user_revised> "resumed with changed inputs: <files>"`.

One expected case: a stage that edits its own input in its final step (eng-review Step 5 updates `architecture.md`). If that step crashed partway, `check` reports the file as changed. The skill's resume note covers this.

---

## Shared flows

Each `SKILL.md` decides *when* to reach these flows. The flows themselves are the same everywhere. In the commands below, `<skill>` is the full skill name (e.g. `eng-flow-architecture`).

### Resume

Picking a draft, or picking a spec that has one, means resume. Don't ask again.

1. Read `eng-flow/.drafts/<prefix>--<dated-slug>.json`. Reuse `dated_slug` exactly as saved.
2. Downstream only: run `check` (above).
3. Don't ask whether the saved answers are still right. Run step `last_completed_step + 1`'s analytics checkpoint first. Then say in one line, directly above that step's content (so tool output doesn't separate them), where it's picking up and how to review. For example: "Resuming **csv export** at Step 4 (Issues), 3 of 7 resolved. Steps 0–3 are saved; say *review* to go over or change them, or *discard* to drop this draft." With input drift: "Resuming **csv export** at Step 4 (Issues), 3 of 7 resolved. **spec.md has changed since this draft started.** Steps 0–3 are saved; say *review* to check them against it, or *discard* to drop this draft." The *review* and *discard* hints are required in every resume line, not optional.
4. Run step `last_completed_step + 1` as normal, including its skip rules and draft save, but restored from the draft:
   - Read the inputs again from disk. The draft stores answers, not the input files.
   - Rebuild any working text from the saved answers. If the resumed step is the stage's save step, write the files from the draft's saved content, not regenerated content.
   - Don't redo work whose output is saved (subagent findings, research fetches).
   - Saved `done` and `skipped` steps are the record: don't re-derive, re-merge or re-save them on resume, even if they look inconsistent with each other. If something looks wrong (e.g. a merged list missing an earlier finding), say so in the resume line and let the user fix it through *review*.
5. Log it: `eng-flow-decision-log <skill> "Step 0" risk open_question user_confirmed "resumed draft at Step <n+1>" "<dated-slug>"`.

A resume skips the stage's own step-0 questions and step-0 save, because the draft already has both.

### Review

Only when the user asks (they say "review", or want to change an earlier answer). Play back a short summary of the saved answers: a line or two per `done` step, each `skipped` step with its reason, and the items finished so far for an `in_progress` step. Then take any corrections.

- **Corrected:** re-save each changed step with its corrected answers (same `save` call, same step number), then continue at `last_completed_step + 1`.
- **A correction makes later saved steps stale:** drop those steps *first*, so a crash before they're redone can't leave them counted as finished. Run `truncate` to the last step that's still valid, re-save the changed steps, then continue from there:

  ```bash
  python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" truncate <skill> "<dated-slug>" <keep-up-to-step>
  ```

  Each `SKILL.md` names its own stale cases (e.g. a changed Stage 1 routing path, or an earlier answer changed after the "Draft and confirm" step was saved).

Log it: `... user_revised "resumed draft, corrected: <what changed>" ...`.

### Discard

Reached from a stage's Step 0 question, or when the user says "discard" at any point after a resume starts. Confirm first: "Delete the draft for `<topic>`? Its saved answers can't be recovered." Only on yes:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" clear <skill> "<dated-slug>"
```

Log it: `eng-flow-decision-log <skill> "Step 0" risk open_question user_confirmed "discarded draft <dated-slug>" "<dated-slug>"`. Then go back to the top of Step 0. Downstream, that's a new run on the same spec. On no, carry on where the user was.

### Finish

The stage's save step writes its final files. Run `clear` as the last action of that step, only once every file has been written. If any write failed, keep the draft so a re-run can resume at the save step, and tell the user which file didn't save.

---

## Scenarios

| Scenario | What happens |
|---|---|
| **Normal step** | At the end of the step, add its answers to the draft and move `last_completed_step` forward. |
| **Normal loop item** | Re-save the loop's step as `in_progress` with every finished item. `last_completed_step` doesn't move. |
| **Crash partway through a step** | The draft still shows the previous step as the last one finished. Only the step in progress is lost, or for a loop, only the item in progress. |
| **Resume** | See **Resume**. Picking a draft resumes it with no extra question. (Changed 2026-10-26 after the Stage 1 live test: the confirm-first flow asked too many questions.) |
| **Inputs changed since the draft started** | Downstream: resume anyway, flag the files, suggest *review*. |
| **Correction that makes later steps stale** | `truncate` before re-saving. |
| **Normal finish** | See **Finish**. |
| **Abandoned run** | Keep the draft. Step 0 keeps offering it until the user resumes or discards it. Never delete it automatically. |
| **Discard** | See **Discard**. |

---

## Stage 1 specifics (`eng-flow-spec`)

Stage 1 has no spec folder until Step 7, so it builds its own slug and matches drafts on the topic.

| Situation at Step 0 | What happens |
|---|---|
| Topic passed in (`/eng-flow-spec csv export`) | Use it as the topic. If a draft has the same topic, ask "Resume it, or start a separate spec?" If not, go straight to the routing question. |
| No topic, no drafts | Ask "What are we speccing?", then the routing question. |
| No topic, one or more drafts | One question that lists the drafts plus "Something new". "Something new" leads to "What are we speccing?" |

- A separate spec on the same topic gets its own file (a `-2`, `-3` slug suffix). If several drafts share a topic, list them by slug.
- Skipped steps: Step 1 off the idea path, Step 4 off the feature path, Step 5 when no research is done.
- Stale cases: a changed routing path → `truncate 0`; an earlier answer changed after Step 6 was saved → `truncate 5`.
- `dated_slug` is worked out at Step 0 from the start date. Step 7 reuses it for the spec folder.

## Downstream specifics (Stages 2, 3, 3.5, 3.6, 4)

Step 0 first finds the spec folder as before. Then:

1. Run `list <skill>` to get the drafts. Each line starts with the spec folder's `<dated-slug>`.
2. When the user is asked which spec folder to use (more than one exists), each option for a folder with a draft gets "draft: stopped after Step n, age" in its description. Picking that folder → **Resume**.
3. When there's only one candidate folder and it has a draft → **Resume**. The resume line's *discard* hint is how the user starts over.
4. No draft for the chosen folder → new run. Once the inputs are found and read, `save` step 0 with `topic` (the spec's title) and `spec_folder`, then `inputs` with the files read.

If Step 0 stops early (inputs missing, or ui-design finds no UI journeys), don't save. Nothing worth resuming exists yet.

The full helper reference is the docstring at the top of `lib/bin/eng-flow-draft`.
