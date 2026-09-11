# Data Gateway & Storage — Features, Enhancements, and Security Findings

**For:** picking what to build next in `blocks-data`.
**Scope:** Data Gateway and Storage only — the two services the e-commerce/POS/inventory
platform (`ECOMMERCE_PLATFORM_ON_BLOCKS.md`) leans on for nearly everything, since that
project builds no backend of its own (`AGENTS.md`). Everything below is grounded in
`blocks-data`'s actual source in this workspace, with `file:line` references — nothing here
is guessed from documentation. Nothing in `blocks-data/` was changed to produce this document.
**How to use it:** pick items below and implement them directly in `blocks-data`. Security
findings are ordered by severity; features and enhancements by how much they unblock the
e-commerce project specifically. Each item stands alone — you don't need the others to do one.

This is a companion to, and for Data Gateway/Storage goes deeper than, the earlier
`BLOCKS_FEATURE_SUGGESTIONS.md` (which also covers IAM and Release). Where an item appears in
both, this document has the fuller technical detail; treat this one as authoritative for
Data Gateway/Storage.

---

## Part A — Security findings

### A1. [CRITICAL] [Storage] Azure Blob container is created world-readable — the DMS access-policy layer doesn't apply to Azure at all

**Where:** `server/Storage.DomainService/Storage/Services/AzureBlobStorageService.cs:22-23`

```csharp
var blobServiceClient = new BlobServiceClient(StorageProvider.ConnectionString);
_containerClient = blobServiceClient.GetBlobContainerClient(BlocksContext.GetContext()?.TenantId.ToLower());
_containerClient.CreateIfNotExists(PublicAccessType.Blob);
```

**What this means:** `PublicAccessType.Blob` grants **anonymous, unauthenticated read access to
every blob in the container**, to anyone who has (or guesses, or derives) the blob's URL —
no SAS token, no bearer token, no `x-blocks-key` required. This is a container-level ACL,
independent of anything the DMS layer decides.

Every other piece of Storage's design — `ObjectAccessResolver`'s RLS-like policy resolution,
`AccessModifier.Private` vs `Public`, per-resource share/grant/revoke, the whole point of
`GenerateDownloadUriByAccessType` handing out a scoped SAS URL for private files
(`AzureBlobStorageService.cs:60-65`) — assumes the *only* way to read a blob is through that
resolved, authorized URL. The container setting means that assumption is false: the blob
itself is public the instant it's created, regardless of what `AccessModifier` the
application-level `File` record says.

**Concretely, for this project:** a "Private" invoice or receipt stored via Storage
(`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §4.2) is named `Private/{fileId}/{fileVersionId}/{fileName}`
(`Shared/Services/FileManagementService.cs:453-455`) — a real, guessable-if-leaked path, and
anonymously fetchable from `https://<account>.blob.core.windows.net/<tenant-container>/Private/...`
with **zero** authentication, forever. Revoking a share (`objects.revokeAccess`) or disabling a
user's role does nothing to this URL if it was ever observed — in a browser's network tab, in
a server log, in a Referer header, anywhere. There is no way to un-leak it short of deleting
and re-uploading the blob under a new key.

**Fix:** create the container with `PublicAccessType.None` and make every read go through a
SAS the app issues after `ObjectAccessResolver` approves it (which is already the code path
for `AccessModifier.Private` — `GetDownloadUrlAsync` already builds a scoped, time-limited SAS
regardless of container ACL). For files that are genuinely meant to be public
(`AccessModifier.Public`), keep issuing a SAS-free URL from the app, but there's no need for
the *container* itself to be anonymously readable to do that — the app already decides which
URL to hand out.

**Note on scope:** the AWS S3 provider (`AwsS3StorageService.cs:96-108`) does **not** set a
public ACL or bucket policy when creating the bucket, so this specific issue is Azure-only as
far as the code in this repo goes — but it's worth an explicit check of whatever bucket
policies exist outside this codebase, since the code doesn't rule out a public bucket policy
set some other way.

---

### A2. [CRITICAL] [Data Gateway] The legacy `filter`/`input.filter` string argument is deserialized as a raw, unsanitized MongoDB filter — reachable without authentication on any Public schema

**Where:**
- Query path: `Services/Implementations/QueryService.cs:149-165` (`GetUserFilterBson`), fed
  by `input: { filter: "<string>" }` (`GraphTypes/QueryInputType.cs:10`).
- Mutation path: `Helpers/MutationFilterHelper.cs:50-51` and `:105-106`
  (`BuildFilterWithRls`/`BuildBaseFilter`), fed by a standalone `filter: String` argument
  (`Resolvers/SchemaResolver.cs`, e.g. line 71 on `updateX`).
- Both do the same thing: `BsonSerializer.Deserialize<BsonDocument>(filterJson)`, straight from
  caller-supplied text, with **no operator allowlist, no key blocklist, no depth limit.**

**Contrast with the typed `where` path**, which is properly hardened
(`Conversion/WhereToMongoFilterConverter.cs:69-78`): it rejects any key starting with `$`,
rejects any field not on the schema's allowlist, and `Regex.Escape`s every string comparison
value before building a regex (`:188-191`). The legacy `filter` string bypasses every one of
those protections, because it's just handed to the BSON deserializer as-is.

**Exploit scenario:** a filter payload like
`{"$where": "function(){ ... }"}` or `{"$expr": {"$function": {...}}}` runs arbitrary
server-side JavaScript inside MongoDB during query evaluation — classic NoSQL injection, with
the same class of impact as SQL injection (data exfiltration across the whole collection
regardless of RLS — since this filter is combined with, not restricted by, whatever RLS
produces; denial of service via a deliberately expensive `$where`/`$regex`; potentially probing
for or affecting other tenants' data if the collection isn't otherwise isolated per request).

**Reachable without authentication today.** `ReadSchemaAccessMiddleware` skips the auth check
entirely when a schema's read access level is `Public`
(`Middlewares/SchemaAccessMiddlewareHelper.cs:37-40`: `if (accessLevel == SchemaAccessLevel.Public) { await next(context); return; }`).
The e-commerce project's own product catalog is configured exactly this way on purpose
(`ecommerce-back-office/AGENTS.md`: "Product reads are configured Public on the Data Gateway...
the separate ecommerce-consumer app relies on that to browse the catalog with no session") —
which the tenant's `x-blocks-key` is, by design, embedded in a public frontend bundle. So this
injection surface is reachable by any anonymous internet visitor to the storefront, on the
Products query, with no credentials beyond a value already shipped to their browser.

**Fix:** either remove the legacy `filter`/`input.filter` string argument entirely (it's
already documented as deprecated — every resolver's own GraphQL description says "Use where
instead"), or, if backward compatibility is required, route it through the exact same
allowlist/escape logic `WhereToMongoFilterConverter` already applies to `where`, rather than
deserializing it directly.

---

### A3. [HIGH] [Storage] Path traversal via an unsanitized file name — Local (SFTP-backed) storage writes outside its intended per-file directory

**Where:**
- `Storage/Validators/LocalStorageUploadRequestValidator.cs:12-22` validates `request.Name` for
  non-empty, has-an-extension, and extension-not-denylisted — **never** for path separators or
  `..`.
- `Shared/Services/FileManagementService.cs:1029-1030` (`BuildLocalStorageKey`) builds
  `"{tenantId}/{fileId}/{version}/{fileName.TrimStart('/')}"` from that same unsanitized name.
- `Storage/Services/SftpStorageService.cs` (`UploadFileToSftpAsync`) builds
  `remoteFilePath = $"{remoteDirectory}/{fileName.TrimStart('/')}"` and calls
  `client.UploadFile(fileStream, remoteFilePath, true)` — `TrimStart('/')` strips a single
  leading slash; it does nothing about `..` segments.

**Exploit scenario:** a `Name` of `../../../../etc/cron.d/evil` (with any allowed extension
appended, e.g. `../../../../home/svc/.ssh/authorized_keys.evil`, or simply an extension the
denylist doesn't cover) resolves, once the SFTP server itself normalizes the path, to a
location outside the intended `<remoteBase>/<tenant>/<fileId>/<version>/` sandbox — an
authenticated user with ordinary upload permission can write a file anywhere the SFTP service
account can write, limited only by that account's own filesystem permissions.

**Contrast:** the directory/rename validators elsewhere in this same file do check for exactly
this (`DmsValidationRules.BeAUsableName`, `Storage/Validators/DmsObjectValidators.cs:26-32`,
rejecting `/`, `\`, and `.`/`..`) — the upload-name validator is the one path that doesn't reuse
it.

**Fix:** apply the same `BeAUsableName`-style check (or stricter: reject any `..` segment and
any path separator) to `GetPreSignedUrlForUploadRequest.Name` and
`LocalStorageUploadRequest.Name` before they ever reach `BuildLocalStorageKey` or the SFTP
path. Cloud (S3/Azure) storage keys aren't vulnerable in the traditional filesystem sense —
object stores treat a key as an opaque string, not a path to resolve — but sanitizing at the
one shared entry point (the upload validators) is simpler and safer than reasoning about which
backend is safe today and staying right as backends change.

---

### A4. [MEDIUM-HIGH] [Data Gateway] No upper bound on `pageSize` — a single query can request unbounded rows

**Where:** `Services/Implementations/QueryService.cs:248-255` (`ComputePagination`):

```csharp
private static (int skip, int limit) ComputePagination(int? pageNo, int? pageSize)
{
    const int defaultLimit = 10;
    if (pageNo is not null && pageSize is not null)
        return ((pageNo.Value - 1) * pageSize.Value, pageSize.Value);
    return (0, defaultLimit);
}
```

Supplying both `pageNo` and `pageSize` skips the default entirely — `pageSize` is passed
straight to Mongo's `.Limit()` with no ceiling. The same absence of a cap applies to
`insertMany`/`updateMany`/`deleteMany` batch sizes (`Services/Implementations/MutationService.cs`,
around the `InsertManyAsync`/bulk-update code paths) — no maximum item count is enforced on
the input array either.

**Why it matters:** this directly contradicts the platform's own stated non-functional
requirement ("Paginate all large datasets. Prevent unbounded queries" —
`ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md` §14). A single `getProducts(input: {pageSize: 5000000})`
— reachable unauthenticated per A2 above, on any Public schema — can force the gateway to
materialize an enormous result set, and a single `insertManyProduct` with an enormous array
can do the equivalent on write.

**Fix:** enforce a maximum `pageSize` (configurable per project/schema, with a hard global
ceiling) and a maximum item count on every `Many` mutation, rejecting the request with a clear
error rather than silently clamping — a silent clamp would hide the mistake from a legitimate
caller who actually needed more than one page.

---

### A5. [MEDIUM] [Storage] No server-side file size limit, quota, or content verification on any upload path

**Where:** `Shared/Services/FileManagementService.cs` (presigned-upload flow,
`HandleNewFileAsync`/`HandleExistingFileAsync`) and the S3/Azure presigned-URL generators
(`Storage/Services/AwsS3StorageService.cs:63-71`, `AzureBlobStorageService.cs:87-100`).

**What's missing:**
- The S3 presigned PUT URL (`GetPreSignedUrlRequest { Verb = HttpVerb.PUT }`) and the Azure SAS
  (`BlobSasBuilder` + `SetPermissions(BlobSasPermissions.Write)`) impose no size constraint —
  S3 supports a `Content-Length-Range` condition (via a presigned **POST** policy, not the PUT
  URL used here) and Azure SAS has no native size cap at all, but neither is used here to bound
  anything.
- Because the upload goes straight from the client to the storage backend on that URL, **the
  API server never sees the bytes** — there's no hook at which a size check, a content-type
  sniff, or a malware scan could happen even if one were added later without changing this
  architecture.
- The `File`/`FileVersion` metadata (`SizeInBytes`, `ContentType`) is whatever the client
  *declared* when requesting the URL — nothing reconciles it against what was actually
  uploaded. A client can request a URL claiming a 10KB PNG and upload a 10GB file, or an
  executable relabeled with an image content-type, and the database will keep showing the
  original, false metadata indefinitely.

**Why it matters for this project:** product media, invoices, and receipts (`ECOMMERCE_PLATFORM_ON_BLOCKS.md`
§4.1/§4.2) all go through this path. Without a cap, a single malicious or buggy client upload
is an unbounded storage-cost and bandwidth liability, and any later feature that trusts stored
`ContentType`/`SizeInBytes` (a thumbnail renderer, a "safe to preview inline" check, a quota
dashboard) is trusting attacker-controlled metadata.

**Fix:** for S3, switch to a presigned POST policy with a `content-length-range` condition
(and optionally a `starts-with $Content-Type` condition); for Azure, there's no native SAS size
limit, so enforce it by reconciling the actual blob size against a declared maximum after
upload (a `HEAD`/`GetPropertiesAsync` check before the file is marked available), and consider
a client-visible per-project storage quota enforced the same way. Malware scanning needs a
step after upload, before a file is served (a scan-then-publish state), which is a bigger
architectural addition — worth flagging as its own feature ask (see B7) rather than folding
into this fix.

---

### A6. [MEDIUM] [Storage] Extension filtering is a denylist, not content verification — spoofed content-type is undetectable

**Where:** `Shared/Utilities/UnsupportedFile.cs` (the denylist), consulted at
`FileManagementService.cs:137` and `LocalStorageUploadRequestValidator.cs:19-20`, plus an
optional per-directory allowlist (`FileManagementService.cs:150`).

**What's missing:** the check is purely on the *claimed* file name's extension. Nothing
verifies the uploaded bytes match either the extension or the `ContentType` recorded in
metadata (which, per A5, isn't verified either). An HTML file with an embedded script, renamed
to `.jpg` and uploaded with `ContentType: image/jpeg`, passes every check here. If that file is
ever served with a browser-sniffable or trust-the-declared-type response (a future "view
receipt inline" or "preview product image" feature is exactly this shape), it's a stored-XSS
vector — compounded by A1, since on Azure the file is anonymously fetchable regardless of
what access policy the app thinks applies.

**Fix:** this is inherently a defense-in-depth item, not a single fix — options in rough order
of effort: (a) always serve user-uploaded content with `Content-Disposition: attachment` and
`X-Content-Type-Options: nosniff` rather than inline rendering, unless the file has been
verified; (b) sniff the actual leading bytes of small/known types (images, PDFs) server-side
before marking a version as the current one; (c) move toward an allowlist-by-default model
(only accept file types a project has explicitly enabled) rather than a denylist of known-bad
ones, which is always chasing the last known threat rather than defining the trusted set.

---

### A7. [MEDIUM] [Storage] Fail-open default: a resource with no explicit access policy is fully public

**Where:** `Shared/Services/ObjectAccessResolver.cs:187-192` (`Decide`):

```csharp
// Resources without an access policy are public. This is equivalent to an
// implicit Everyone Allow at every permission level, but does not persist a
// synthetic entry or interfere with an explicit policy when one exists.
if (candidates.Count == 0) return true;
```

This is a deliberate, documented design choice, not an oversight — but it's worth naming as a
security posture decision rather than leaving it implicit: **every new file and directory is
public-by-default** to any authenticated caller (and, combined with A1 on Azure, to anonymous
callers too) until someone explicitly narrows it. A default-deny model (nothing visible until
a policy grants it, at minimum "creator only" as the implicit floor) is the safer default for
a DMS holding invoices, receipts, and customer PII, and would still let a project *opt into*
public-by-default for a specific directory tree if that's genuinely wanted.

**Fix, if changed:** flip the implicit default to owner-only rather than everyone, and let a
project (or the object's creator) explicitly grant broader access — this is a breaking
behavior change for any existing deployment relying on the current default, so it should ship
as an opt-in project-level setting first, not a silent flip.

---

### A8. [LOW] [Data Gateway] Verbose `Console.WriteLine` diagnostics on every schema-access check, in production code paths

**Where:** `Middlewares/SchemaAccessMiddlewareHelper.cs:29-49` — six separate
`Console.WriteLine` calls per request that touches any schema access middleware (read, write,
edit, delete), logging the middleware name, the resolved access level, tenant validity, and
authentication status.

**Why it matters:** this runs on *every* GraphQL field resolution across every schema in every
project — real overhead on the hottest path in the service, written to stdout rather than
through the structured `ILogger` the rest of the codebase uses (e.g.
`Services/DataChangeEventPublisher.cs`), which means it bypasses log levels, sampling,
correlation IDs, and any redaction policy the platform applies to its real logs. It's also
information a container's raw stdout stream shouldn't need to carry per-request (access level
and auth outcome for every call), depending on how that stream is collected and who can read it
downstream.

**Fix:** replace with `ILogger<T>` at `Debug`/`Trace` level, or remove — this looks like
diagnostic output left over from development rather than an intentional operational log.

---

## Security findings summary

| # | Severity | Service | Finding |
|---|---|---|---|
| A1 | **Critical** | Storage | Azure container created with anonymous blob-read; DMS access policy doesn't actually gate access |
| A2 | **Critical** | Data Gateway | Legacy `filter` string is unsanitized — NoSQL injection, reachable unauthenticated on Public schemas |
| A3 | High | Storage | Path traversal via unsanitized upload `Name` in Local/SFTP storage |
| A4 | Medium-High | Data Gateway | No max `pageSize` / no max batch size on `Many` mutations |
| A5 | Medium | Storage | No file size limit, quota, or upload content verification |
| A6 | Medium | Storage | Extension denylist only — no content verification, spoofable `ContentType` |
| A7 | Medium | Storage | Resources with no explicit policy default to fully public |
| A8 | Low | Data Gateway | Per-request `Console.WriteLine` diagnostics on the schema-access hot path |

---

## Part B — New features

### B1. [Data Gateway] Atomic numeric operators (`$inc`, `$push`, `$pull`, `$addToSet`)

**What's missing:** every mutation issues `$set` only
(`Repositories/MongoCollectionOperations.cs:54,64`). No increment, no array append/remove.

**Why the e-commerce project specifically needs it:** every inventory quantity change
(reserve, commit, release, receive, adjust) is currently read → compute → guarded
compare-and-swap write → retry (`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.1), purely because `$inc`
isn't available. An `increment:` argument alongside `input:` on `updateX`, honoring the same
`where` guard, turns that into one atomic, unconditional-on-read operation. `$push`/`$addToSet`
would do the same for array fields (adding a line to a cart, tagging a record) without a
read-modify-write cycle. **This is the single highest-leverage change for this project.**

---

### B2. [Data Gateway] A real unique index, declarable in the schema file

**What's missing:** `IsUniqueData: true` on a field is enforced by querying for a conflict
before inserting (`Services/Implementations/MutationService.cs:309`,
`ValidateUniquenessOrThrowAsync`) — not a database constraint, so two concurrent requests with
the same value can both pass the check. Meanwhile, a real MongoDB unique index does exist as a
capability (`SchemaIndexService.cs`, `POST /schemas/indexes`), but it's REST-only: no `blocks
data index *` CLI command, no SDK method, and no way to declare one inside
`blocks/data/schemas/<Schema>.json`, so it can't ride along with `blocks data schema push` /
`blocks data sync`.

**Why it matters:** idempotency keys (on orders, POS transactions, payments, inventory
movements — anything that might be retried, including a POS terminal replaying a queued sale
after days offline) need a real constraint, not a check-then-act query, to actually prevent
double-processing. Making `IsUniqueData` create a genuine unique index at push time (or adding
first-class index declarations to the schema file) closes this without a separate,
hand-maintained provisioning step outside the normal schema workflow.

---

### B3. [Data Gateway] A general, queryable change-history / audit trail — extending what already exists for deletes

**What's there today:** deletes are archived into a `DataMutationRecords` collection
(`Services/Implementations/MutationService.cs:596-613`, `BuildDeletedRecordDocument`) — but
only for delete operations, and it's not exposed through any query field a client can read.

**What's missing:** the same treatment for insert and update — a full before/after record of
every mutation, queryable per schema/record, would give every Data Gateway project a built-in
immutable audit trail. For this project, that's directly the plan's requirement for an audit
log of sensitive operations and (with an insert-only access policy on the collection, following
the same default-deny pattern already used elsewhere) the same shape as the hand-rolled
immutable-ledger pattern this project has to build itself today
(`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.2). Promoting this from a delete-only side effect to a
first-class, documented feature — with a `getDataMutationRecords`-style query field, filterable
by schema/record/operation/date — would remove the need for every project to reinvent it per
collection.

---

### B4. [Data Gateway] Transactional multi-collection batch mutation

**What's missing:** no way to commit writes across more than one collection (or more than one
document beyond what a single `updateMany` touches) as a single MongoDB transaction.

**Why it matters:** an inventory movement is always a balance update plus a ledger append
(`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.5); today these are two separate calls, ordered
correctness-first, with a reconciliation pass as the safety net for the window between them. A
`batch(operations: [...])` mutation that opens one MongoDB session/transaction across the listed
writes would remove that reconciliation need entirely for any project willing to accept
transaction overhead on the write path.

---

### B5. [Data Gateway] Return the updated document from `updateX`

**What's missing:** mutation responses carry only `acknowledged` and `totalImpactedData`
(`Repositories/MongoCollectionOperations.cs:56`) — never the document's new state.

**Why it matters:** every stock-changing operation currently needs a follow-up read to learn
the resulting balance, costing a full round trip exactly where the plan sets its tightest
latency target (POS checkout, p95 < 1s). `FindOneAndUpdate` with `ReturnDocument.After` is a
drop-in replacement for the current `UpdateOneAsync` internally.

---

### B6. [Data Gateway] Aggregation primitives on the query surface

**What's missing:** the generated GraphQL surface is exactly `getXs`/`insertX`/`updateX`/`deleteX`
(plus `Many` variants) — no `$group`, `$sum`, `$avg`, no joins.

**Why it matters:** inventory valuation, sales metrics, stock-ageing, and supplier-performance
dashboards need sums and grouping over potentially large collections
(`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.6). Even a scoped version — count/sum/avg grouped by one
field, no arbitrary pipeline — would remove the need for client-side aggregation over paged
results or hand-maintained summary schemas for most dashboard tiles.

---

### B7. [Storage] A post-upload scan/verify step before a file version is servable

**What's missing:** there is no state between "upload URL issued" and "file is live and
servable" at which the actual bytes are inspected. See A5/A6 for the underlying gap this
would close.

**Proposal:** add a `Scanning` (or `PendingVerification`) status to `FileVersion`, set when the
presigned URL is issued; a webhook or polling check (Azure/S3 both support event notifications
on blob-created) flips it to `Available` after a size/content-type/malware check passes, or
`Rejected` if it fails. Every existing read path already filters by version status implicitly
by reading "the latest version" — this only requires that status to gate what counts as
latest/servable.

---

### B8. [Storage] Per-project storage quota

**What's missing:** no notion of "this project/tenant may store at most N GB" anywhere in
`Configuration`/`ConfigurationRepository`.

**Why it matters:** ties directly to A5 — without both a per-upload size cap and an aggregate
quota, storage cost for a project is fully caller-controlled. Useful independently of A5's
per-file fix.

---

## Part C — Enhancements

### C1. [Data Gateway] Enforce a maximum `pageSize` and a maximum batch size (implementation of A4)

Listed here too because it's as much a correctness/cost feature as a security fix — pick it up
alongside A4 either way.

### C2. [Data Gateway] TTL index support

A TTL index (declarable the same way as B2's unique index) on a reservation's `ExpiresAt`
would let the database expire abandoned reservations with no scheduler at all — directly
relevant given this project runs no background process
(`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §6.3).

### C3. [Data Gateway] Reliable delivery for `DataChangeEvent`

`DataChangeEventPublisher.PublishAsync` catches and logs its own exceptions
(`Services/DataChangeEventPublisher.cs:74`) — a committed write can silently fail to publish
its event. Moving this to an outbox (persist the event in the same write path, deliver from
there with retry) makes "committed" and "will eventually publish" the same guarantee.

### C4. [Data Gateway] Raise or document the 15-index-per-schema ceiling

`MaxIndexesPerSchema = 15` (`Services/Implementations/SchemaIndexService.cs:18`). A heavily
queried `Orders` or `InventoryMovements` collection with tenant, variant, location, lot,
reference-type, and idempotency-key access paths can reach that; worth either raising it or
clearly documenting the ceiling so schema design accounts for it up front.

### C5. [Storage] Relevance-ranked full-text search across file/object metadata

Object/file search today is name/keyword matching over listing metadata
(`Shared/Services/ObjectDiscoveryService.cs`, `ObjectSearchRequestValidator`) — fine for a DMS
browser, but not enough to search invoices/receipts by content. A lower priority than
everything above, but worth noting for a mature DMS.

### C6. [Data Gateway] Bulk upsert (insert-or-update) mutation

Useful for catalog import/export and idempotent event replay — today, achieving "insert if
absent, else update" requires a client-side read-then-branch; a single upsert mutation keyed
on a caller-specified field (not just `ItemId`) would remove that round trip.

---

## Summary — everything, by service

| # | Part | Service | Item |
|---|---|---|---|
| A1 | Security | Storage | Azure container anonymous blob-read bypasses access policy |
| A2 | Security | Data Gateway | Unsanitized legacy `filter` — NoSQL injection, unauth-reachable |
| A3 | Security | Storage | Path traversal via unsanitized upload name (Local/SFTP) |
| A4 | Security | Data Gateway | No max `pageSize` / batch size |
| A5 | Security | Storage | No upload size limit, quota, or content verification |
| A6 | Security | Storage | Extension denylist only, no content verification |
| A7 | Security | Storage | Fail-open default: no policy = fully public |
| A8 | Security | Data Gateway | `Console.WriteLine` diagnostics on the access-check hot path |
| B1 | Feature | Data Gateway | Atomic `$inc`/`$push`/`$pull`/`$addToSet` |
| B2 | Feature | Data Gateway | Real unique index, declarable in schema file |
| B3 | Feature | Data Gateway | General change-history/audit trail (extend delete-archival) |
| B4 | Feature | Data Gateway | Transactional multi-collection batch mutation |
| B5 | Feature | Data Gateway | Return updated document from `updateX` |
| B6 | Feature | Data Gateway | Aggregation primitives on the query surface |
| B7 | Feature | Storage | Post-upload scan/verify step before a version is servable |
| B8 | Feature | Storage | Per-project storage quota |
| C1 | Enhancement | Data Gateway | Enforce max `pageSize`/batch size (A4's fix) |
| C2 | Enhancement | Data Gateway | TTL index support |
| C3 | Enhancement | Data Gateway | Outbox delivery for `DataChangeEvent` |
| C4 | Enhancement | Data Gateway | Raise/document the 15-index-per-schema ceiling |
| C5 | Enhancement | Storage | Full-text search across file/object metadata |
| C6 | Enhancement | Data Gateway | Bulk upsert mutation |
