# ali-collibra - Licensing and Sizing Reference

## Licensing and Sizing

### License Tiers

| Tier | Capabilities | Use For |
|------|--------------|---------|
| **Admin** | Full config, workflow design, API access | Platform admins, architects |
| **Steward/Editor** | Create/edit assets, approve workflows | Data stewards, data engineers |
| **Consumer/Viewer** | Read-only, search, view lineage | Analysts, data scientists, business users |

### Sizing Guidance

| Factor | Small (< 1000 assets) | Medium (1000-10,000) | Large (> 10,000) |
|--------|----------------------|---------------------|------------------|
| **Users** | 10-50 | 50-500 | 500+ |
| **Edge Agents** | 1-2 | 3-10 | 10+ |
| **Databases** | 1-5 | 5-20 | 20+ |
| **Lineage Complexity** | Basic SQL | ETL tools + SQL | Multi-platform, custom |
| **Recommended Model** | Cloud SaaS | Cloud SaaS | Cloud SaaS or On-Prem |
