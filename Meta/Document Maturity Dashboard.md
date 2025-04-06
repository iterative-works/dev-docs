---
status: stable
created: 2025-04-06
last_updated: 2025-04-06
version: 1.0
tags: dashboard
---

> [!stable] Stable Document
> This document is considered stable and authoritative.

# Document Maturity Dashboard

This dashboard provides an overview of all documentation in the vault organized by maturity level.

## 🤖 AI-Generated Documents

```dataview
TABLE 
  file.cday as "Created",
  file.mday as "Last Modified"
FROM "/"
WHERE status = "ai-generated"
SORT file.mtime DESC
```

## 📝 Draft Documents

```dataview
TABLE 
  file.cday as "Created",
  file.mday as "Last Modified"
FROM "/"
WHERE status = "draft"
SORT file.mtime DESC
```

## 🔍 In-Review Documents

```dataview
TABLE 
  file.cday as "Created",
  file.mday as "Last Modified"
FROM "/"
WHERE status = "in-review"
SORT file.mtime DESC
```

## ✅ Tested Documents

```dataview
TABLE 
  file.cday as "Created",
  file.mday as "Last Modified"
FROM "/"
WHERE status = "tested"
SORT file.mtime DESC
```

## 🌟 Stable Documents

```dataview
TABLE 
  file.cday as "Created",
  file.mday as "Last Modified"
FROM "/"
WHERE status = "stable"
SORT file.mtime DESC
```

## Documents Missing Status

```dataview
TABLE 
  file.cday as "Created",
  file.mday as "Last Modified"
FROM "/"
WHERE !status AND file.name != "Document Maturity Dashboard"
SORT file.mtime DESC
```