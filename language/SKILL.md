---
name: language
description: Language handling conventions for AI agents - context language rules, default English, and response language matching
---

# Language Handling

Consistent language management for AI agents.

## Core Rules

1. **Context Language Priority**: When a specific language is explicitly set for a context, that language takes precedence

2. **Code, Projects and Documentation**: All projects, documentation, code, and comments use English when no other language is specified

3. **Internal Files**: AI-generated internal files (memories, logs, configuration) are stored in English

4. **Response Language Matching**: Respond in the same language the user communicates in

5. **User Communication**: With no further requirements, communicate to the user in his native language

6. **Swiss-German to German**: If the user's native language is Swiss-German, write in German, but use swiss grammar and rules, e.g. always «ss» no «ß» or «these quotes».

## Language Detection

Automatically detect user language from:
- USER.md
- Explicit instructions ("answer in German")
- Message content language
- Channel context settings

## Context-Specific Language

If you edit a document written in a specific language, continue in the same language.

If the user gives you a communication snippet in a specific language and asks you to write an aswer, then write the answer in the same language

When a specific language is set for a context:

```markdown
**Context Language: German**
All communication in this context uses German
```

This overrides default behavior for that specific conversation.

## Implementation

```markdown
### Response Language Determination

1. Check for explicit context language
2. If no context language, detect from user message
3. If unclear, default to user native language

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
```

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
| Mixed languages in conversation | Apply language detection per message |
