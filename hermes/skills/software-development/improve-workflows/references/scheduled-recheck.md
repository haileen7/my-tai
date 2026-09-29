# Scheduling a Follow-up Verdict on a Thread

Depth for the "Scheduled Re-check on a Thread" section of SKILL.md: "give your
opinion now, then again in N minutes" as a two-phase job.

## Shape

Phase 1 runs in the live session: do the analysis, write the body to a file,
post it, read it back. Phase 2 is a one-shot cron job that runs in a FRESH
session with none of this context, so everything it needs goes in the prompt:

- who the assistant is and which repo/branch/owner
- the thread under review and the topic, restated
- the verdict already published, and every measurement already taken (so the
  follow-up re-uses numbers instead of re-running a multi-minute suite)
- the rule that its reply answers the NEWEST comment and must not restate the
  previous one
- the posting command, verbatim
- whether it may post without asking the user to confirm

```
cronjob_manage action=create
  name='<thread> follow-up comment'
  schedule='in 10m'      # one-shot by duration; do NOT hand-compute a timestamp
  repeat=1
  script='<name>.sh'     # filename only — see below
  workdir='<repo path>'
  prompt='<self-contained as above>'
```

## Pitfalls

- **`script` takes a FILENAME, not a command line.** `"gh_discussion.sh fetch_last 730"` fails validation with a path-not-found error. Create a tiny wrapper whose whole body is the invocation with its arguments baked in, and point `script` at the wrapper.
- **The cron session does not inherit your interactive environment.** A token exported from a shell rc file is present in a `terminal` call but absent in the scheduled session. Make the script self-sufficient: if the credential env var is empty, re-export it from the rc file before calling the CLI, and fail loudly if it is still empty. Capture that fix inside the script — do not leave the job depending on an env var that only a login shell has.
- **A script that fails should fail loudly.** A wrapper whose last command is an unguarded `exec` returns non-zero, and the agent prompt receives nothing rather than a plausible-looking empty thread.
- **Do not restrict toolsets unless you have a reason.** The `enabled_toolsets` array is easy to get wrong through a generic tool-call wrapper (an array of one item can arrive wrapped and fail validation). Omitting it costs a little context and removes a failure mode; add it only after the job is confirmed working.
- **Deliver a real report, not a confirmation request.** If the user said confirmation is not needed, the follow-up's final response goes straight to the chat with the comment id, the URL and a short bullet summary. A job that ends by asking the user "shall I post this?" has undone the whole point.

## What the follow-up job is for

The value of the second comment is that the thread may have moved. It should
contain at least one of: agreement with a point that has now been measured,
a correction where code or a new measurement disagrees with an earlier claim,
or a genuinely new finding from the repo. A re-run of the first answer, or an
"any updates?" stub, is a wasted post — cut the first answer's content down to
one link and spend the space on what is new.
