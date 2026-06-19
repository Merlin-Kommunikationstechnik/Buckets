---
name: buckets
description: Use when a PowerShell project, script, automation, CLI tool, or agent workflow should store and retrieve structured PowerShell objects in simple directory-backed buckets. Covers installing/importing the Buckets module, bootstrapping it from an application's main entry point, choosing Buckets use cases, and using the core bucket/object commands safely.
---

# Buckets Skill

Use Buckets when a PowerShell codebase needs a lightweight local object store without introducing a database. Buckets stores PSObjects in named directory-backed buckets, hides file extensions from users, defaults to readable JSON, and can fall back to binary serialization when object fidelity requires it.

## Required Bootstrap

In a PowerShell application's main execution file, ensure Buckets is available before code uses it:

```powershell
if (!(Get-Module -ListAvailable -Name Buckets)) {
    Install-Module Buckets -Scope CurrentUser -Force
}

Import-Module Buckets
```

For one-off scripts where reinstall checks are less important, this minimal shape is acceptable:

```powershell
if (!(Get-Module Buckets)) {
    Install-Module Buckets -Scope CurrentUser -Force
}

Import-Module Buckets
```

Use `-Scope CurrentUser` for agent- and script-friendly installs that do not require admin rights. In locked-down environments, ask before installing modules or requiring network access.

## Good Uses

Use Buckets for:

- Persistent script state, checkpoints, and resumable jobs.
- Local caches for API responses, filesystem scans, inventory data, or expensive calculations.
- Small application data stores for PowerShell tools and CLIs.
- Test fixtures and demo data that should stay inspectable as JSON.
- Agent scratchpads where structured objects need to survive across commands.
- Logs, metrics, reports, and snapshots that should be queryable later.
- Simple import/export workflows where a full database would be unnecessary.

Avoid Buckets for high-concurrency writes, relational queries, large binary blobs, strict transactional guarantees, or storage that must be shared safely by many processes at once.

## Core Mental Model

- A bucket is a named collection of objects.
- An object has a key, which becomes the visible object name.
- JSON is the default format and is human-readable.
- `-AsBinary` preserves richer .NET types with PowerShell serialization.
- `-Compress` applies to binary storage.
- Keys are sanitized for filesystem safety; empty keys are rejected.
- Bucket paths should be treated through Buckets cmdlets, not by editing `.json` or `.dat` files directly.

## Basic Usage

Create a bucket:

```powershell
New-Bucket -Name tasks
```

Store one object with an explicit key:

```powershell
[pscustomobject]@{
    Id = 1
    Title = 'Write docs'
    Done = $false
} | New-BucketObject -Bucket tasks -Key task-1
```

Store many objects using a property as the key:

```powershell
$items | New-BucketObject -Bucket tasks -KeyProperty Id -AutoIndex
```

Read objects:

```powershell
Get-BucketObject -Bucket tasks
Get-BucketObject -Bucket tasks -Key task-1
Get-BucketObject -Bucket tasks -Match @{ Done = $false }
Get-BucketObject -Bucket tasks -Filter { $_.Title -like '*docs*' }
```

Update an object:

```powershell
$task = Get-BucketObject -Bucket tasks -Key task-1
$task.Done = $true
$task | Set-BucketObject -Bucket tasks -Key task-1
```

Remove objects safely:

```powershell
Remove-BucketObject -Bucket tasks -Key task-1 -WhatIf
Remove-BucketObject -Bucket tasks -Key task-1
```

Inspect buckets:

```powershell
Get-Bucket
Get-Bucket -Bucket tasks
Get-Bucket -Tree -Recurse
Get-BucketStats -Bucket tasks
Get-BucketKeys -Bucket tasks
```

## Patterns

Use timestamp keys for append-only logs:

```powershell
$event | New-BucketObject -Bucket logs/app -AsTimestamp
```

Use `-AutoIndex` when many inputs can produce the same key:

```powershell
$files | New-BucketObject -Bucket files -KeyProperty Name -AutoIndex
```

Use binary storage for objects that must preserve .NET type fidelity:

```powershell
Get-Item .\README.md | New-BucketObject -Bucket files -Key readme -AsBinary -Compress
```

Use a custom root in tests and automation so real user data is never touched:

```powershell
$root = Join-Path ([System.IO.Path]::GetTempPath()) "buckets-test-$([guid]::NewGuid())"
Set-BucketRoot $root
```

Use nested bucket names to organize data:

```powershell
$response | New-BucketObject -Bucket cache/github/issues -Key issue-123
Get-BucketObject -Bucket cache -Recurse
```

## Funnels

Use funnels when objects need the same transform on write or read. A funnel scriptblock must return the object to keep it and `$null` to drop it.

```powershell
New-Funnel -Name open-items -Transform {
    if (-not $_.Done) { $_ }
}

$items | New-BucketObject -Bucket tasks -Funnel open-items -KeyProperty Id
Get-BucketObject -Bucket tasks -Funnel open-items
```

Do not use a bare boolean filter as a funnel:

```powershell
# Wrong for funnels:
{ $_.Done -eq $false }

# Correct:
{ if ($_.Done -eq $false) { $_ } }
```

## Safety Rules

- Prefer `-WhatIf` before destructive operations.
- Use `Set-BucketRoot` in tests so commands never touch `$HOME/.buckets`.
- Do not manually traverse outside the bucket root or construct storage file paths from user input.
- Treat corrupted object reads as recoverable: Buckets warns and returns `$null`.
- Use `-Recurse` intentionally; by default, object reads do not recurse into nested buckets.
- For deletes by match/filter, expect a confirmation summary unless `-Force` is appropriate.
- Do not expose `.json` or `.dat` details in user-facing UI; show bucket names and keys.

## Common Command Map

- Create bucket: `New-Bucket -Name <bucket>`
- Store object: `<object> | New-BucketObject -Bucket <bucket> -Key <key>`
- Store by property: `<objects> | New-BucketObject -Bucket <bucket> -KeyProperty <property> -AutoIndex`
- Read object: `Get-BucketObject -Bucket <bucket> -Key <key>`
- Query objects: `Get-BucketObject -Bucket <bucket> -Match @{ Prop = 'value' }`
- Update object: `<object> | Set-BucketObject -Bucket <bucket> -Key <key>`
- Delete object: `Remove-BucketObject -Bucket <bucket> -Key <key>`
- List buckets: `Get-Bucket`
- Show tree: `Get-Bucket -Tree -Recurse`
- Export bucket: `Export-Bucket -Bucket <bucket> -OutputFile <file>`
- Import bucket: `Import-Bucket -Bucket <bucket> -InputFile <file>`

## Maintain This Skill

Update this file only when a meaningful Buckets behavior change would affect how another AI agent should install, bootstrap, choose, or use Buckets. Good reasons include new or changed cmdlet parameters, changed defaults, changed safety behavior, changed serialization/storage semantics, changed funnel behavior, or a new recommended integration pattern.

Do not update this file for cosmetic refactors, internal-only implementation changes, test-only changes, or wording churn. Keep it concise and operational; avoid adding examples that do not change how an agent should use Buckets.
