---
name: language
description: Language handling conventions for AI agents - context language rules, default English, and response language matching
---

# Language Handling

Follow these rules for consistent language management.

## Core Rules

1. **Context Language Priority**: When a specific language is explicitly set for a context, that language takes precedence over the English default

2. **Default English**: All projects, documentation, code, and comments use English when no other language is specified

3. **Internal Files**: AI-generated internal files (memories, logs, configuration) must be stored in English, regardless of the user's language

4. **Response Language Matching**: Respond in the same language the user communicates in

## Language Detection

Automatically detect user language from:
- Explicit instructions ("answer in German")
- Message content language
- Channel context settings

## Context-Specific Language

When a specific language is set for a context:

```markdown
**Context Language: German**
All communication in this context uses German
```

This overrides default behavior for that specific conversation.

## Implementation

### Response Language Determination

1. Check for explicit context language
2. If no context language, detect from user message
3. If unclear, default to English

### Code and Documentation

- All code comments in English
- All README files in English
- All API documentation in English
- Variable names and functions in English

### Internal Files

- Memory files: English
- Log files: English
- Configuration: English
- Error messages: English

## Examples

### Response Language Matching

```
// User writes in German
User: "Kannst du mir helfen?"

// Agent responds in German
Agent: "Ja, gerne! Wie kann ich dir helfen?"
```

### Default English

```
// No explicit language, default to English
User: "Help"

Agent: "I'm here to help! What do you need?"
```

### Context Language

```
// In a German context
**Context Language: German**

User: "Help"

// Even English requests get German responses
Agent: "Gerne! Wie kann ich dir helfen?"
```

## Troubleshooting

| Issue | Solution |
|---|---|
| Response in wrong language | Check for context language setting |
| Inconsistent language in files | Enforce English default for all internal files |
| Mixed languages in conversation | Re-apply language detection rules for each new message |