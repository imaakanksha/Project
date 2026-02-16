---
inclusion: always
---

# Project Structure

## Current Organization
```
GeneRX-AI/
├── .kiro/              # Kiro configuration and steering rules
│   └── steering/       # AI assistant guidance documents
```

## Recommended Structure
As the project develops, consider organizing code into:

- `/src` - Source code
- `/tests` - Test files
- `/docs` - Documentation
- `/config` - Configuration files
- `/scripts` - Build and utility scripts

## Conventions
- Use clear, descriptive names for files and directories
- Keep related functionality grouped together
- Separate concerns (business logic, data access, UI, etc.)
- Document architectural decisions as the project grows

## File Naming
- Use kebab-case for directories and configuration files
- Follow language-specific conventions for source files
- Keep names concise but descriptive
