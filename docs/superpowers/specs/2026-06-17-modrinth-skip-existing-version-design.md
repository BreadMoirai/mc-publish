# Skip Modrinth publication when version already exists

## Problem

When `mc-publish` runs against Modrinth with a `version` number that has already
been published to the project, the action attempts to create the version again.
We want an early validation ("precheck") that detects this case, skips the
Modrinth publication gracefully, and lets the rest of the run continue.

## Goals

- Before publishing to Modrinth, check whether a version with the same
  `version_number` already exists on the target project.
- If it exists, skip the Modrinth publication and continue the run.
- Log a clear message explaining that Modrinth was skipped.

## Non-goals

- No new action output is added (skip is surfaced via logs only).
- The behavior for CurseForge and GitHub is unchanged. Only Modrinth is skipped;
  the run does not abort.
- No change to how versions are matched beyond exact `version_number` equality.

## Decisions

- **Skip scope:** Skip Modrinth only. CurseForge/GitHub continue to publish as
  normal. This is consistent with the existing per-platform loop in
  `program.ts:publish()`, which iterates platforms independently.
- **Surfacing:** Log only. No new action output.
- **Location:** The check lives at the start of
  `ModrinthUploader.uploadCore()`, after the project is resolved and before
  `createVersion()`. This keeps the logic Modrinth-specific and leaves the
  generic `program.ts` loop untouched.
- **Match rule:** A version "already exists" when any version returned by
  `getProjectVersions(id)` has `version_number` exactly equal to
  `request.version`.

## Components

### 1. `ModrinthApiClient` — new helper

```ts
async getProjectVersionByNumber(
    idOrSlug: string,
    versionNumber: string,
): Promise<ModrinthVersion | undefined>
```

- Calls the existing `getProjectVersions(idOrSlug)`.
- Returns the first version whose `version_number === versionNumber`, else
  `undefined`.
- Keeps the matching logic isolated and unit-testable.

File: `src/platforms/modrinth/modrinth-api-client.ts`

### 2. `ModrinthUploader.uploadCore()` — early skip

File: `src/platforms/modrinth/modrinth-uploader.ts`

After `const project = await this.getProject(...)` and before
`createVersion(...)`:

1. Call `api.getProjectVersionByNumber(project.id, request.version)`.
2. If a matching version is found:
   - Log: `🔍 Version "<version>" already exists on Modrinth; skipping publication`.
   - Return early with a report built from the **existing** version:
     ```ts
     {
       id: project.id,
       version: existing.id,
       url: `https://modrinth.com/${project.project_type}/${project.slug}/version/${existing.version_number}`,
       files: existing.files.map(x => ({ id: x.hashes.sha1, name: x.filename, url: x.url })),
     }
     ```
   - This makes the `modrinth-*` outputs point to the already-existing version
     rather than being empty/misleading, and adds no new output.
3. Otherwise, proceed with the existing flow (`createVersion` →
   `unfeaturePreviousVersions`).

## Flow & error handling

- Returning a report from `uploadCore` means the `program.ts` loop records
  Modrinth as processed and continues to the next platform — no special-casing in
  the loop.
- If the project is not readable by the token (placeholder project data / `404`
  from the API), `getProjectVersions` returns `[]`, no match is found, and
  publishing proceeds exactly as today. This is the safe default.

## Testing

- Unit test for `getProjectVersionByNumber`: match, no-match, and empty-list
  cases.
- Uploader test: when a matching version exists, `createVersion` is **not**
  called and the returned report reflects the existing version; when no match
  exists, behavior is unchanged (regression guard).
