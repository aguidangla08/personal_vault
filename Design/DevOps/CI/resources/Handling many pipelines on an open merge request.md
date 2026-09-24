# Handling many pipelines on an open merge request

In GitLab a pull request is a merge request (MR). Each push to its branch starts a new pipeline, and depending on your rules it may start two: a branch pipeline and a merge request pipeline. With multi-hour, license-bound jobs, those pipelines pile up in the queue for the license, and the older ones are usually obsolete by the time they run.

## 1. Avoid duplicate branch and MR pipelines

Run only the MR pipeline when the branch has an open merge request:

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
```

This alone halves the load if you currently get both.

## 2. Cancel superseded pipelines

Mark the heavy jobs `interruptible` and cancel older pipelines when a new commit arrives (limited to MRs so main and release pipelines are not affected):

```yaml
workflow:
  auto_cancel:
    on_new_commit: interruptible

sim_full:
  interruptible: true
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

This only cancels if all jobs that have already started in the older pipeline are interruptible. The older project setting "Auto-cancel redundant pipelines" does something similar on older versions.

## 3. Run the cheap jobs on every push, the expensive ones only when needed

Smoke tests on each push, full regression only on demand:

```yaml
sim_full:
  when: manual
  allow_failure: true
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
```

Full regression only when a label is set:

```yaml
sim_full:
  rules:
    - if: $CI_MERGE_REQUEST_LABELS =~ /run-full/
```

Skip drafts, so people can push freely while working:

```yaml
sim_full:
  rules:
    - if: $CI_MERGE_REQUEST_TITLE =~ /^Draft:/
      when: never
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

Only when relevant files change:

```yaml
sim_full:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes: [rtl/**/*, ver/**/*]
```

## 4. Skip CI for pushes that don't need it

Developers can push work-in-progress commits without triggering a pipeline:

```bash
git push -o ci.skip
```

Or include `[ci skip]` in the commit message.

## 5. Prioritize the newest pipeline in the license queue

With `resource_group`, set the group's process mode to `newest_first` through the Resource Groups API, so the latest pipeline gets the license first instead of the oldest waiting one. It is not certain whether older waiting jobs are then dropped or only reordered, so test it on your version. Combine it with auto-cancel so stale pipelines don't linger.

## 6. Run the full regression only at merge time

Use merged results pipelines or merge trains (available in higher GitLab tiers) so the expensive validation happens once, when the MR is about to merge, not on every push.

## Recommended combination

1. The `workflow:rules` above to remove duplicates.
2. Auto-cancel with `interruptible` on MR pipelines.
3. A quick smoke tier on each push, with the full regression manual, label-triggered, or only after Draft is removed.

Together these keep the license queue short without giving up the full checks before merge.