---
status: stable
created: 2025-04-06
last_updated: 2025-04-06
version: 1.0
tags: guide
---

> [!stable] Stable Document
> This document is considered stable and authoritative.

# How to Update Document Status

This guide explains how to change the status of a document as it progresses through our maturity framework.

## Step 1: Update the Frontmatter

Change the following values in the document's frontmatter:

```yaml
---
status: [new-status]  # Change to: ai-generated, draft, in-review, tested, or stable
last_updated: [current-date]
version: [increment-version-number]  # e.g., 0.1 → 0.2
---
```

## Step 2: Change the Callout Block

Replace the existing callout block at the top of the document with the appropriate one:

### For AI-Generated Documents:
```markdown
> [!danger] AI-Generated Content
> This is AI-generated content pending human review. Information may be inaccurate or misaligned with actual processes.
```

### For Draft Documents:
```markdown
> [!info] Draft Document
> This document is an initial draft and may change significantly.
```

### For In-Review Documents:
```markdown
> [!warning] In-Review Document
> This document is under review and feedback is welcome.
```

### For Tested Documents:
```markdown
> [!success] Tested Document
> This document has been validated in real projects.
```

### For Stable Documents:
```markdown
> [!purple] Stable Document
> This document is considered stable and authoritative.
```

## Step 3: Update the Document History

Add a new entry to the document history table at the bottom of the document:

```markdown
| [new-version] | [current-date] | [brief description of changes made] | [your-name] |
```

## Step 4: Check the Dashboard

After saving your changes, check the [Document Maturity Dashboard](Document%20Maturity%20Dashboard.md) to ensure your document appears in the correct section.

---

Remember to follow the criteria outlined in the [Document Maturity Framework](Document%20Maturity%20Framework.md) when determining whether a document is ready to progress to the next status level.
