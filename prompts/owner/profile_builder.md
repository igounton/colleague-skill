# Owner Profile Builder

Turn the intake answers and any provided material into one Markdown file with
the structure below. Write in the owner's preferred language. Section headings
may be translated except the summary heading, which must be exactly
`## Always-on summary` (or `## 常驻摘要` in Chinese) so the installer can find it.

Write about the owner in the third person or as directives to the agent
("Prefers...", "Ask before..."). Do not write it as a role-play persona.

```markdown
# Owner profile: {preferred name}

> Created by Distilly on {YYYY-MM-DD}. Reviewed by the owner.

## Always-on summary

- Name / how to address: ...
- Role and context: ...
- Timezone and languages: ...
- Current focus: ...
- Communication: ...
- Act vs. ask: ...
- Never: ...
- Rhythm: ...

## Work and projects
## Goals
## Communication preferences
## Decision style and autonomy
## Routines and availability
## Tools and accounts
## Boundaries and dislikes
## Open questions
```

Rules:

1. The always-on summary is loaded every session, so keep it under 3,000
   characters (OpenClaw gives `USER.md` a 4,000-character budget). Short
   bullets, highest-value facts first.
2. The other sections hold detail the agent reads only when needed. Skip any
   section with no information instead of padding it.
3. Every line must come from the owner's answers or material. Mark inferences
   with `(inferred)` and list unknowns under `## Open questions`.
4. Follow the privacy rules in `prompts/owner/intake.md`.

Show the full profile to the owner and apply their corrections before saving.
