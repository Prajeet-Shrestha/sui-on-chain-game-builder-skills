---
name: json_output_format
description: Output format instructions for file-mode code generation — forces the LLM to return pure JSON
---

# OUTPUT FORMAT REQUIREMENT — YOU MUST FOLLOW THIS EXACTLY

Your response MUST be ONLY a valid JSON object. Nothing else.

## Structure

```json
{
  "files": [
    { "path": "Move.toml", "content": "..." },
    { "path": "sources/game.move", "content": "..." }
  ]
}
```

## Rules

- Your ENTIRE response must be parseable by `JSON.parse()`
- Do NOT include ANY text before or after the JSON
- Do NOT wrap in markdown code blocks
- Do NOT include explanations, plans, or commentary
- Start your response with the character `{` and end with `}`
- The `"path"` should be relative (e.g. `"sources/game.move"`)
- The `"content"` should be the complete file content as a string
- Always include `Move.toml` and at least one `.move` source file

---

## REMINDER

Your response MUST be ONLY a JSON object: `{"files": [...]}`.
No markdown, no text, no explanations. Start with `{` and end with `}`.
