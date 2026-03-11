# ali-wherescape - Version Control Reference

This reference provides detailed guidance on WhereScape's version control integration patterns and deployment workflows.

---

## Metadata Export

WhereScape supports exporting metadata to XML:

```bash
# Export metadata for specific objects
ws_export -objects "STG_CUSTOMER,DIM_CUSTOMER" -output metadata.xml

# Export entire project
ws_export -project "DataWarehouse" -output project_metadata.xml
```

---

## Git Integration Pattern

```bash
# Export metadata after changes
ws_export -project "DataWarehouse" -output metadata/project.xml

# Commit to git
git add metadata/project.xml
git commit -m "feat: add customer dimension SCD Type 2"

# Deploy to target environment
ws_import -input metadata/project.xml -environment PROD
```

---

## Deployment Workflow

```
Development → Export Metadata → Git Commit → Pull Request → Merge → Import to TEST → Import to PROD
```

**Challenges:**
- XML diffs are hard to read
- Merge conflicts require WhereScape expertise
- No native git integration (manual export/import)

**Best practices:**
- Export metadata frequently (daily or per feature)
- Use feature branches for major changes
- Test imports in lower environments first
- Document deployment procedures

---

## WhereScape CLI Commands

```bash
# Export metadata
ws_export -objects "TABLE1,TABLE2" -output metadata.xml
ws_export -project "MyProject" -output project.xml

# Import metadata
ws_import -input metadata.xml -environment PROD

# Execute job
ws_execute -job Load_Customer -environment PROD

# Validate metadata
ws_validate -project "MyProject"

# Generate documentation
ws_document -project "MyProject" -output docs/
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
