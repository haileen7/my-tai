# Reviewing a Plan and Publishing the Verdict

Depth for the "Plan Review & Verdict Publishing" section of SKILL.md.

## Claim-verification checklist

Walk these before writing a single objection. Each row is a claim type that
appears in tooling/DX plans, the command that settles it, and what "not
reproducible" means for the plan.

| Claim in the plan | Settle it with | If it does not reproduce |
|---|---|---|
| Tool X is too slow for a local gate | `/usr/bin/time -f "%e s" <exact command>` | The gate should run the full tool; a scoped variant buys seconds at the cost of a second code path that can drift from CI |
| Run Y has been flaky | Open the cited PR/issue; read the body | The citation does not support the claim — say so and ask for the failure list |
| Must handle gotcha Z | `grep` the existing scripts and CI workflow for Z | Z is already handled; the wrapper should not duplicate it — name the one step that is actually missing |
| The new command reproduces CI | Diff the CI workflow's steps and flags against the proposed script | Narrow the acceptance claim to the job that is reproducible locally; coverage gates and env setup are frequently not |
| Fast feedback depends on hook Z | `ls -la .git/hooks/<hook>`; `git config core.hooksPath` | The hook may be silently absent; an existence check in the wrapper is worth more than extending the hook |
| Select subset S of tests from changed files | Try to build the mapping; inspect test naming and whether the source files are PHP at all | The mapping has no basis — recommend manual selection over a heuristic, and flag the failure mode: a skipped broken test still reports green |

A framework where component classes live inline in template files has no
`app/… → tests/…` file mapping to exploit. Check that before accepting any
"run the tests related to changed files" proposal.

## Reading a discussion

`gh issue view N` and issue-scoped MCP tools return Not Found for a discussion
number. Read the rendered page instead — it carries the body and all comments:

```
web_extract(urls=["https://github.com/<owner>/<repo>/discussions/<N>"])
```

Read every existing comment before replying. The thread's consensus is what a
plan claims to summarize, and contradicting an already-settled point is
wasted post.

## Posting a comment to a discussion

Two calls: resolve the node ID, then mutate. The ID is not the issue number.

```
# 1. node ID + existing comment count/authors
gh api graphql -f query='
query($owner:String!,$name:String!,$number:Int!){repository(owner:$owner,name:$name){discussion(number:$number){
  id
  comments(first:100){nodes{author{login} body createdAt}}
}}}' -F owner=<owner> -F name=<repo> -F number=<N> \
 --jq '.data.repository.discussion | {id, commentCount:(.comments.nodes|length)}'

# 2. post — body from a file so long Persian/Arabic text is not mangled by shell quoting
gh api graphql -f query='
mutation($discussionId:ID!,$body:String!){addDiscussionComment(input:{discussionId:$discussionId,body:$body}){comment{url createdAt author{login}}}}' \
 -F discussionId='<node id from step 1>' \
 -F body=@/path/to/comment.md --jq '.data.addDiscussionComment.comment'
```

Write the body with `write_file` first, then pass it via `-F body=@file`.
Passing a multi-line non-Latin string inline risks shell mangling and costs a
rewrite.

## Confirming the post landed

Write the response, then read the state back — never report a comment as
posted on the strength of a mutation return alone:

```
gh api graphql -f query='
query($owner:String!,$name:String!,$number:Int!){repository(owner:$owner,name:$name){discussion(number:$number){comments(first:100){nodes{author{login} createdAt body}}}}}' \
 -F owner=<owner> -F name=<repo> -F number=<N> \
 --jq '.data.repository.discussion.comments.nodes[-1] | {author:.author.login, at:.createdAt, head:(.body[0:180])}'
```

Check that `author.login` is the account you expect and the body head matches
what you wrote. Return the comment URL to the user.
