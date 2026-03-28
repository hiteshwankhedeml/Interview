# 🟢 Workflow

```
Research Agent
    ↓
Analysis Agent
    ↓
JD Generation Agent
    ↓
Self-Reflection (Generation)
    ↓
Validation Agent
    ↓
Self-Reflection (Validation)
    ↓
Communication Layer (Agent Discussion)
    ↓
Reasoning Agent ──→ APPROVE → END
    │                REJECT  → END
    ↓
  REGENERATE
    ↓
JD Generation Agent (loop back — validation feedback in prompt)
```
