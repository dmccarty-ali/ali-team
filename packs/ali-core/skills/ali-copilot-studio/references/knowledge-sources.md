# ali-copilot-studio - Knowledge Sources Reference

This reference provides detailed configuration patterns for knowledge sources in Microsoft CoPilot Studio.

---

## Knowledge Source Types

### Comparison Table

| Source | Use Case | Update Frequency | Cost |
|--------|----------|------------------|------|
| **SharePoint** | Internal documentation, policies | Real-time sync | Included |
| **File Upload** | Static docs, PDFs, Word files | Manual update | Included |
| **Website Crawl** | Public docs, external content | Scheduled crawl | Included |
| **Dataverse** | Structured business data | Real-time query | Extra license |

---

## SharePoint Integration

### Setup Steps

1. **Connect to SharePoint site**
   - Navigate to Knowledge Sources in CoPilot Studio
   - Select "Add knowledge source" > SharePoint
   - Authenticate with Microsoft 365 account

2. **Select document libraries**
   - Choose specific libraries or folders
   - Set permissions (who can query)
   - Configure metadata fields for filtering

3. **Configure sync schedule**
   - Real-time sync: Changes reflected immediately
   - Hourly sync: Batch updates every hour
   - Daily sync: Scheduled nightly updates

4. **Set permissions**
   - Define who can query the knowledge source
   - Align with SharePoint permissions or override
   - Test with different user roles

### Best Practices

- **Use folders/metadata for topic organization**: Makes retrieval more accurate
- **Keep docs under 5MB per file**: Larger files slow indexing
- **Use approved file formats**: PDF, DOCX, TXT for best results
- **Implement document approval workflow**: Ensure quality before indexing
- **Tag documents with keywords**: Improves search relevance

### SharePoint Knowledge Hub Pattern

**Use case:** Centralized documentation for agent grounding

**Structure:**
```
SharePoint Site: Company Knowledge Hub
├── IT Documentation (library)
│   ├── VPN Setup.docx
│   ├── Password Policy.pdf
│   └── Software Catalog.xlsx
├── HR Policies (library)
│   ├── PTO Policy.docx
│   ├── Benefits Guide.pdf
│   └── Onboarding Checklist.docx
└── Product Docs (library)
    ├── User Manual.pdf
    ├── API Reference.docx
    └── Release Notes.txt
```

**Agent configuration:**
- Connect to site
- Select libraries by topic
- Real-time sync enabled
- Permissions: All employees (read)

---

## File Upload Pattern

### When to Use
Static reference documents, PDFs, limited update frequency

### Limitations

- **Max file size**: 10MB per file
- **Max total**: 100 files per agent
- **Manual refresh required**: Updates don't sync automatically
- **No versioning**: Replaced files lose history

### Supported Formats

- PDF, DOCX, PPTX, TXT, CSV, XLSX

### Upload Process

1. Navigate to Knowledge Sources
2. Select "Upload files"
3. Choose files from local system
4. Wait for indexing to complete (may take several minutes)
5. Test retrieval with sample queries

### Best Practices

- **Consolidate related content**: Combine related docs to reduce file count
- **Use clear file names**: Include keywords for discoverability
- **Update periodically**: Replace outdated files manually
- **Test after upload**: Verify content is indexed correctly

---

## Website Crawl

### Setup Steps

1. **Provide root URL**
   - Enter starting URL for crawl
   - Must be publicly accessible (no authentication)

2. **Configure crawl depth**
   - 1 level: Only pages linked from root
   - 2 levels: Pages linked from level 1
   - 3 levels: Pages linked from level 2 (max)

3. **Set crawl schedule**
   - Daily: Crawl every 24 hours
   - Weekly: Crawl once per week
   - Manual: Trigger crawl on demand

4. **Exclude patterns**
   - Admin pages: /admin/*, /dashboard/*
   - Login pages: /login, /auth/*
   - Dynamic content: /search?*, /filter?*

### Best Practices

- **Use sitemap.xml**: Helps crawler discover all pages
- **Exclude auth-required pages**: Crawler cannot authenticate
- **Set crawl frequency based on content update rate**: Daily for news, weekly for docs
- **Monitor crawl errors**: Review crawl logs for 404s and failures
- **Respect robots.txt**: Ensure crawler is allowed

### Limitations

- **Max 100 URLs per agent**: Choose most important pages
- **JavaScript-rendered content may not index**: Use server-side rendering if possible
- **Requires public accessibility**: No authentication wall allowed

### Example Sitemap Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://docs.example.com/</loc>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://docs.example.com/getting-started</loc>
    <priority>0.8</priority>
  </url>
  <url>
    <loc>https://docs.example.com/api-reference</loc>
    <priority>0.8</priority>
  </url>
</urlset>
```

---

## Dataverse Integration

### When to Use
Structured business data (CRM records, sales data, customer information)

### Setup

1. Connect to Dataverse environment
2. Select tables to include
3. Configure query permissions
4. Define search fields and filters

### Benefits

- **Real-time data**: Always up-to-date
- **Structured queries**: Filter by specific fields
- **Rich metadata**: Leverage entity relationships
- **Security**: Row-level security applied

### Licensing

Requires Power Apps or Dynamics 365 license

---

## Troubleshooting

### SharePoint Not Syncing

**Symptoms**: New documents don't appear in knowledge source

**Solutions**:
- Check sync schedule (may be hourly, not real-time)
- Verify document library permissions
- Ensure file format is supported
- Check file size (must be under 5MB)
- Trigger manual sync in CoPilot Studio

### File Upload Indexing Slow

**Symptoms**: Files uploaded but not searchable

**Solutions**:
- Wait 10-15 minutes for indexing to complete
- Check file format compatibility
- Reduce file size if over 10MB
- Check error logs in portal

### Website Crawl Incomplete

**Symptoms**: Expected pages not indexed

**Solutions**:
- Check crawl depth setting
- Verify pages are publicly accessible
- Review exclude patterns
- Check robots.txt for disallow rules
- Increase crawl depth if needed

---

## Migration from Claude Code

### CLAUDE.md Files → SharePoint

**Process**:
1. Export CLAUDE.md files to Word/PDF
2. Organize into SharePoint library structure
3. Add metadata tags for topics
4. Configure SharePoint knowledge source
5. Test retrieval accuracy

### Skill Files → Knowledge Sources

**Process**:
1. Convert skill markdown to Word/PDF
2. Extract key principles and patterns
3. Upload to SharePoint or file upload
4. Map skill domains to agent topics
5. Validate grounding quality
