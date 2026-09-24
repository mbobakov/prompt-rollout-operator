# Writing a PR Description

## Step: Write the PR description

Default template (use if no `.github/PULL_REQUEST_TEMPLATE.md`):

```markdown
## RATIONALE

<1–3 short sentences. State the problem. Do not describe the fix.>

## CHANGES

<Bullet list. One change per bullet.>
```

### Language rules — strict, no exceptions

Write the full description in **ASD-STE100 Simplified Technical English**.

- One idea per sentence. Target 20 words or fewer per sentence.
- Active voice only. Subject, then verb, then object.
  - Correct: "This change fixes the retry loop."
  - Wrong: "The retry loop was fixed by this change."
- Present tense for what the code does now. Past tense only for the prior problem.
- One approved word per meaning. Do not swap synonyms for the same thing (pick one of "delete"/"remove", one of "fix"/"resolve", and reuse it).
- No noun strings. Break them apart.
  - Correct: "the value that sets the session timeout"
  - Wrong: "the session timeout configuration value"
- No jargon, idioms, or filler words ("simply", "basically", "in order to", "leverage", "utilize").
- No compound or chained sentences. Split any "and"/"but"/"which" clause into its own sentence.
- Use the ubiquitous-language terms for every domain concept. Never paraphrase a defined term.

### Content rules

- **RATIONALE**: the problem and the motivation only. Max 3 sentences.
- **CHANGES**: one bullet per change. Max 12 words per bullet.
- Do not repeat the title in the body.
- Keep the whole description short. It is a fast summary for a reviewer, not documentation.

## Step: Write description to a temp file

```bash
cat > /tmp/pr_body_$$.md << 'EOF'
<generated PR body here>
EOF
```

## Step: Create the PR

Try the GitHub MCP server first. Use the connected GitHub MCP tool to create the PR with:
- **title**: as constructed above
- **body**: contents of `/tmp/pr_body_$$.md`
- **base**: the base branch determined earlier
- **head**: current branch
- **draft**: `false` (ready for review)
- **assignees**: `["mbobakov"]`
- **labels**: `["Ready for Review"]`

Do **not** set reviewers.

If the MCP server fails, fall back to the `gh` CLI:

```bash
gh pr create \
  --title "<title>" \
  --body-file /tmp/pr_body_$$.md \
  --base <base-branch>
```

## Step: Clean up the temp file

```bash
rm /tmp/pr_body_$$.md
```
