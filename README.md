# SharePoint Tenant Audit and Migration

A read-only method for auditing a SharePoint Online tenant before a restructure or migration: every site, library, folder and file is extracted once, then analysed offline so the folder tree, file mix, path lengths and storage can be re-cut without touching the tenant again. I have used it for current-state reports ahead of information architecture redesigns, a SharePoint to Azure Files migration and a legacy Office file conversion.

> Case study. Scripts, reports and tenant data are not published. This repository documents the method and the lessons. All names and values shown are generic.

## Method

```mermaid
flowchart TD
    A["Active sites export<br/>from SharePoint admin centre"] --> B["Sign in once<br/>PnP PowerShell"]
    B --> C["For each site:<br/>list document libraries (REST)"]
    C --> D["For each library:<br/>page items by ID, 5,000 at a time"]
    D --> E["Write CSV in batches"]
    C -->|"access refused"| X["Log site + file count<br/>as a coverage gap"]
    E --> F["Offline analysis"]
    F --> G["Folder tree with<br/>files-beneath totals"]
    F --> H["File type mix"]
    F --> I["Path length bands<br/>vs 400-character limit"]
    F --> J["Sampled version storage"]
    G & H & I & J --> K["Current-state report<br/>+ searchable HTML tree"]
    X --> K
```

About 1,500 rows a second; roughly 880,000 items in about ten minutes.

## Why ID paging and not the obvious options

| Tried | Result |
|---|---|
| OData next-link paging | Silently stopped after one page on every library |
| Filtering on folder vs file | Not indexed, refused on any library over 5,000 items |
| Search API | About 10,000 rows per query, security trimmed, totals unreliable |
| Graph site search | Rejected or failed to resolve many sites |
| One call per folder | Does not scale past a few levels |
| **Page by ID, skip folders in code** | **Deterministic, fast, no threshold errors** |

## Restructure approach

```mermaid
flowchart LR
    A["Inventory sites and folders<br/>owner, created, last used, size"] --> B["Pilot design of the new<br/>structure, 2 to 3 folder levels"]
    B --> C["Move content<br/>(move keeps version history)"]
    C --> D["Owners review and clean up<br/>their own content by a deadline"]
    D --> E["Archive what is left<br/>after review"]
```

## Azure Files migration

```mermaid
flowchart LR
    A["SharePoint library"] -->|"rclone copy"| B["Validation copy<br/>of one folder"]
    B --> C{"Validation<br/>checks pass?"}
    C -->|no| A
    C -->|yes| D["Full copy to Azure Files"]
    D --> E["Approval gates"]
    E --> F["Retire source"]
```

## Legacy Office files

Old .doc and .xls files were being blocked by current Office policy. The audit found them by extension and by reading the Word file header, converted them to current formats, and checked each Excel conversion by comparing populated cell and formula counts rather than the used range, which old files overstate.

## Reporting rules I follow

- Coverage statement first: how much of the tenant the figures cover and what was refused.
- Label items and files separately (library item counts include folders).
- Source and date for every figure; reconcile totals from different sources.
- Measured facts only. If the reason something exists is not established, say so and name who would know.

## Stack

PnP PowerShell, SharePoint REST API, offline analysis scripts, rclone, Azure Files, Excel and Word reports.
