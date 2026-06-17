# Skip Modrinth Publication When Version Exists — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Before publishing to Modrinth, skip the publication (and continue with other platforms) if a version with the same version number already exists on the project.

**Architecture:** Add a small lookup helper to `ModrinthApiClient` that finds a project version by its `version_number`. Call it as an early validation at the start of `ModrinthUploader.uploadCore()`; if a match is found, log and return a report built from the existing version instead of creating a new one. The generic per-platform loop in `program.ts` is untouched — returning a report skips Modrinth while other platforms still run.

**Tech Stack:** TypeScript, Jest (`ts-jest`), `mock-fs`, the repo's `createFakeFetch` test helper.

## Global Constraints

- This is a Modrinth-only change. Do not touch CurseForge or GitHub uploaders.
- No new action output is added — surface the skip via `this._logger` only.
- Match rule: a version "already exists" when its `version_number` is **exactly equal** to `request.version`.
- Follow existing code style: JSDoc on new public methods, 4-space indentation, no semicolon/style deviations from surrounding code.
- Run unit tests with: `npm run test:unit`.

---

### Task 1: Add `getProjectVersionByNumber` to `ModrinthApiClient`

**Files:**
- Modify: `src/platforms/modrinth/modrinth-api-client.ts` (insert a new method after `getProjectVersions`, which ends at line 226, before `unfeaturePreviousProjectVersions`)
- Test: `tests/unit/platforms/modrinth/modrinth-api-client.spec.ts`

**Interfaces:**
- Consumes: existing `getProjectVersions(idOrSlug: string, template?): Promise<ModrinthVersion[]>` on the same class; `ModrinthVersion` (already imported in the file).
- Produces: `getProjectVersionByNumber(idOrSlug: string, versionNumber: string): Promise<ModrinthVersion | undefined>` — returns the first version whose `version_number === versionNumber`, otherwise `undefined`.

Test fixtures available in `tests/content/modrinth/versions.json`: project slug `lithium` (id `gvQqBUqZ`) has a version with `version_number` `"mc1.20.4-0.12.1"` and `id` `"nMhjKWVE"`. The spec's `MODRINTH_FETCH` already mocks `GET /project/:idOrSlug/version`.

- [ ] **Step 1: Write the failing tests**

Add this `describe` block inside the top-level `describe("ModrinthApiClient", ...)` in `tests/unit/platforms/modrinth/modrinth-api-client.spec.ts`, immediately after the closing `});` of the existing `describe("getProjectVersions", ...)` block (around line 619):

```ts
    describe("getProjectVersionByNumber", () => {
        test("returns the version matching the given version number", async () => {
            const api = new ModrinthApiClient({ fetch: MODRINTH_FETCH, token: "token" });

            const version = await api.getProjectVersionByNumber("lithium", "mc1.20.4-0.12.1");

            expect(version).toBeDefined();
            expect(version.id).toBe("nMhjKWVE");
        });

        test("returns `undefined` when no version matches the given version number", async () => {
            const api = new ModrinthApiClient({ fetch: MODRINTH_FETCH, token: "token" });

            const version = await api.getProjectVersionByNumber("lithium", "0.0.0-does-not-exist");

            expect(version).toBeUndefined();
        });

        test("returns `undefined` when the project doesn't exist", async () => {
            const api = new ModrinthApiClient({ fetch: MODRINTH_FETCH, token: "token" });

            const version = await api.getProjectVersionByNumber("not-a-real-slug", "1.0.0");

            expect(version).toBeUndefined();
        });
    });
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm run test:unit -- tests/unit/platforms/modrinth/modrinth-api-client.spec.ts -t "getProjectVersionByNumber"`
Expected: FAIL — `api.getProjectVersionByNumber is not a function`.

- [ ] **Step 3: Implement the method**

In `src/platforms/modrinth/modrinth-api-client.ts`, insert the following method immediately after the `getProjectVersions` method (after its closing `}` on line 226) and before `unfeaturePreviousProjectVersions`:

```ts
    /**
     * Fetches a project version by its version number.
     *
     * @param idOrSlug - The project id or slug.
     * @param versionNumber - The version number to search for.
     *
     * @returns The version with the specified version number, or `undefined` if no such version exists.
     */
    async getProjectVersionByNumber(idOrSlug: string, versionNumber: string): Promise<ModrinthVersion | undefined> {
        const versions = await this.getProjectVersions(idOrSlug);
        return versions.find(x => x.version_number === versionNumber);
    }
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm run test:unit -- tests/unit/platforms/modrinth/modrinth-api-client.spec.ts -t "getProjectVersionByNumber"`
Expected: PASS (3 passing).

- [ ] **Step 5: Commit**

```bash
git add src/platforms/modrinth/modrinth-api-client.ts tests/unit/platforms/modrinth/modrinth-api-client.spec.ts
git commit -m "feat(modrinth): add getProjectVersionByNumber helper

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: Skip publication in `ModrinthUploader.uploadCore` when the version exists

**Files:**
- Modify: `src/platforms/modrinth/modrinth-uploader.ts` (inside `uploadCore`, between `getProject` at line 62 and `createVersion` at line 63)
- Test: `tests/unit/platforms/modrinth/modrinth-uploader.spec.ts`

**Interfaces:**
- Consumes: `ModrinthApiClient.getProjectVersionByNumber` from Task 1; existing `this._logger` (a `Logger`, defaults to `NULL_LOGGER`); `ModrinthProject` (`id`, `slug`, `project_type`); `ModrinthVersion` (`id`, `version_number`, `files[].filename`, `files[].url`, `files[].hashes.sha1`).
- Produces: no new exported symbol. Behavior change only: `uploadCore` returns a `ModrinthUploadReport` built from the existing version when one is found, without calling `createVersion`.

**Why the existing test needs a fetch tweak:** Adding the precheck makes `uploadCore` issue `GET /project/AAAAAAAA/version` (no query string). `createFakeFetch` rejects any unmocked route, so the existing "fulfills the specified upload request" test would break unless that route is mocked to return an empty list (no existing version → publication proceeds).

- [ ] **Step 1: Update the existing fetch mock so the current test still passes**

In `tests/unit/platforms/modrinth/modrinth-uploader.spec.ts`, find the `GET` block of `MODRINTH_FETCH` (around lines 32-48). Add a new entry **immediately after** the existing `"^\\/project\\/AAAAAAAA\\/version\\?featured=true": () => HttpResponse.json([]),` line:

```ts
        "^\\/project\\/AAAAAAAA\\/version$": () => HttpResponse.json([]),
```

The `$` anchor ensures this matches only the no-query precheck request, not the `?featured=true` request used for unfeaturing (which keeps its own, earlier-listed handler). `HttpResponse` is already imported in this file.

- [ ] **Step 2: Run the existing test to confirm it still passes against current code**

Run: `npm run test:unit -- tests/unit/platforms/modrinth/modrinth-uploader.spec.ts`
Expected: PASS — the existing "fulfills the specified upload request" test still passes (the new mock entry is currently unused because the precheck doesn't exist yet).

- [ ] **Step 3: Write the failing skip test**

In the same file, add a dedicated fetch mock at module scope, immediately after the existing `MODRINTH_FETCH` definition (after its closing `});` around line 80):

```ts
const MODRINTH_FETCH_EXISTING_VERSION = createFakeFetch({
    baseUrl: MODRINTH_API_URL,
    requiredHeaders: ["Authorization"],

    GET: {
        "^\\/project\\/foo$": () => ({
            id: "AAAAAAAA",
            slug: "foo",
            project_type: "mod",
        }),

        "^\\/project\\/AAAAAAAA\\/version$": () => HttpResponse.json([{
            id: "BBBBBBBB",
            version_number: "1.0.0",
            files: [{
                filename: "file.txt",
                url: "https://cdn.modrinth.com/data/AAAAAAAA/versions/BBBBBBBB/file.txt",
                hashes: { sha1: "sha1" },
            }],
        }]),
    },

    POST: {
        "^\\/version$": () => {
            throw new Error("createVersion must not be called when the version already exists");
        },
    },
});
```

Then add this test inside the existing `describe("upload", ...)` block, after the existing `test("fulfills the specified upload request", ...)`:

```ts
        test("skips publication and returns the existing version when a version with the same number already exists", async () => {
            const uploader = new ModrinthUploader({ fetch: MODRINTH_FETCH_EXISTING_VERSION });

            const report = await uploader.upload({
                token: SecureString.from("token"),
                id: "foo",
                name: "Version v1.0.0",
                version: "1.0.0",
                versionType: VersionType.ALPHA,
                changelog: "Changelog",
                files: [FileInfo.of("file.txt")],
                gameVersions: ["1.18.2"],
                loaders: ["fabric"],
            });

            expect(report).toEqual({
                id: "AAAAAAAA",
                version: "BBBBBBBB",
                url: "https://modrinth.com/mod/foo/version/1.0.0",
                files: [{
                    id: "sha1",
                    name: "file.txt",
                    url: "https://cdn.modrinth.com/data/AAAAAAAA/versions/BBBBBBBB/file.txt",
                }],
            });
        });
```

The `POST /version` handler throws, so this test also proves `createVersion` is never reached.

- [ ] **Step 4: Run the skip test to verify it fails**

Run: `npm run test:unit -- tests/unit/platforms/modrinth/modrinth-uploader.spec.ts -t "skips publication"`
Expected: FAIL — the upload reaches `POST /version` and throws `createVersion must not be called when the version already exists` (because the precheck isn't implemented yet).

- [ ] **Step 5: Implement the early skip in `uploadCore`**

In `src/platforms/modrinth/modrinth-uploader.ts`, inside `uploadCore`, insert the following block between the `getProject` call (line 62) and the `createVersion` call (line 63):

```ts
        const existingVersion = await api.getProjectVersionByNumber(project.id, request.version);
        if (existingVersion) {
            this._logger.info(`🔍 Version "${request.version}" already exists on Modrinth; skipping publication`);
            return {
                id: project.id,
                version: existingVersion.id,
                url: `https://modrinth.com/${project.project_type}/${project.slug}/version/${existingVersion.version_number}`,
                files: existingVersion.files.map(x => ({ id: x.hashes.sha1, name: x.filename, url: x.url })),
            };
        }
```

For reference, the surrounding code should read:

```ts
        const project = await this.getProject(request.id, api);

        const existingVersion = await api.getProjectVersionByNumber(project.id, request.version);
        if (existingVersion) {
            this._logger.info(`🔍 Version "${request.version}" already exists on Modrinth; skipping publication`);
            return {
                id: project.id,
                version: existingVersion.id,
                url: `https://modrinth.com/${project.project_type}/${project.slug}/version/${existingVersion.version_number}`,
                files: existingVersion.files.map(x => ({ id: x.hashes.sha1, name: x.filename, url: x.url })),
            };
        }

        const version = await this.createVersion(request, project, api);
        await this.unfeaturePreviousVersions(version, unfeatureMode, api);
```

- [ ] **Step 6: Run the skip test to verify it passes**

Run: `npm run test:unit -- tests/unit/platforms/modrinth/modrinth-uploader.spec.ts -t "skips publication"`
Expected: PASS.

- [ ] **Step 7: Run the full Modrinth uploader suite to verify no regressions**

Run: `npm run test:unit -- tests/unit/platforms/modrinth/modrinth-uploader.spec.ts`
Expected: PASS (both "fulfills the specified upload request" and the new "skips publication" test pass).

- [ ] **Step 8: Commit**

```bash
git add src/platforms/modrinth/modrinth-uploader.ts tests/unit/platforms/modrinth/modrinth-uploader.spec.ts
git commit -m "feat(modrinth): skip publication when version already exists

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: Full verification

**Files:** none (verification only).

- [ ] **Step 1: Run lint and the full unit suite**

Run: `npm run test:lint && npm run test:unit`
Expected: lint passes with no errors; all unit tests pass.

- [ ] **Step 2: Confirm no unintended changes**

Run: `git status` and `git diff --stat`
Expected: only the four files from Tasks 1-2 are modified (plus the spec/plan docs already committed). No changes under `dist/` (the build step is a separate, maintainer-run process and is out of scope for this plan).
