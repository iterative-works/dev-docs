---
status: draft
created: 2025-04-06
last_updated: 2025-04-06
version: 0.1
tags: guide
---

> [!info] Draft Document
> This document is an initial draft and may change significantly.

# Simplified Maturity Indicators

If you're having trouble with the custom CSS styling, you can use this simplified approach that works with Obsidian's built-in callout types:

## Maturity Level Callouts

### AI-Generated Documents
```markdown
> [!danger] AI-Generated Content
> This is AI-generated content pending human review.
```

### Draft Documents
```markdown
> [!info] Draft Document
> This document is an initial draft and may change significantly.
```

### In-Review Documents
```markdown
> [!warning] In-Review Document
> This document is under review and feedback is welcome.
```

### Tested Documents
```markdown
> [!success] Tested Document
> This document has been validated in real projects.
```

### Stable Documents
```markdown
> [!quote] Stable Document
> This document is considered stable and authoritative.
```

## Important Notes

1. This approach uses Obsidian's built-in callout types with their default styling:
   - AI-Generated: Red danger callout
   - Draft: Blue info callout
   - In-Review: Yellow warning callout
   - Tested: Green success callout
   - Stable: Grey quote callout (since purple isn't built-in)

2. You can still use the frontmatter status tags and other parts of the maturity system.

3. The Dataview queries in the Document Maturity Dashboard will still work correctly.

4. When you resolve the CSS issues, you can return to using the custom styling approach.
