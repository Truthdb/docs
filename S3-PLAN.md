# TruthDB: S3 Workflow Checklist

Every S3 workflow, one numbered item each. When an item is done, overwrite its line with what was built.

## Buckets and objects

1. ✅ Manage buckets: CreateBucket (AWS naming rules), ListBuckets, HeadBucket, GetBucketLocation, versioning settings (Put/GetBucketVersioning), and DeleteBucket only when no object or open upload is left. Emptying a bucket is a listing followed by DeleteObjects on the client side (truthdb#202, #209).
2. ✅ Store objects: PutObject streams the body into the object engine in segments, up to 5 GiB. It verifies SigV4 chunk signatures, checksums and Content-MD5 as the body arrives. Replacing an object releases the old body after a grace period. Uploading many files is a client loop over PutObject (truthdb#203).
3. ✅ Multipart uploads: Create, UploadPart (a part can be retried or replaced), ListParts and ListMultipartUploads (to resume), Complete (a composite of the parts, no bytes copied; S3 order, ETag and size rules; at most 5 TiB) and Abort. Uploads left open for 7 days are aborted by garbage collection (truthdb#206).
4. ✅ Retrieve objects: GetObject streams a leased body; it supports single byte ranges (206/416), the If-Match, If-None-Match, If-Modified-Since and If-Unmodified-Since headers, and `versionId` (truthdb#203).
5. ✅ Inspect objects: HeadObject returns size, ETag (MD5), Last-Modified, content type, the stored system headers and `x-amz-meta-*`, without the body (truthdb#203).
6. ✅ Browse and organise: ListObjectsV2 and ListObjects support prefix, delimiter with CommonPrefixes, max-keys, start-after or marker, continuation tokens and `encoding-type=url`. A folder whose keys are all deleted is not listed, and each call scans a bounded number of rows (truthdb#204).
7. ✅ Copy and move: CopyObject shares the source body (no bytes copied) within or between buckets, with COPY/REPLACE metadata, copy-source conditions and `versionId`. UploadPartCopy shares a whole part or copies a byte range. Move is a copy followed by a delete on the client side. There is one region, so "across regions" does not apply (truthdb#207).
8. ✅ Delete objects: DeleteObject (a delete marker in a versioned bucket, a permanent delete with `versionId`) and DeleteObjects (up to 1000 keys, quiet mode, Content-MD5 or checksum required). Deleting everything under a prefix is a listing followed by DeleteObjects on the client side (truthdb#205).
9. ✅ Metadata and tags: system headers (content type, cache control, content disposition, encoding, language, expires) and `x-amz-meta-*` at write time and on GET/HEAD; CopyObject REPLACE to change them; tags through `x-amz-tagging` and Put/Get/DeleteObjectTagging, within S3's limits (truthdb#203, #208).
10. ✅ Versioning: enable or suspend; version ids on every response; ListObjectVersions with delete markers; GET by `versionId`; restore by copying an older version; recover a deleted object by deleting its marker (truthdb#209).

## Search

11. ✅ Exact lookup by bucket and key: HeadObject and GetObject, with 404 for a missing key and `x-amz-delete-marker` when its newest version is a delete marker (truthdb#203, #209; tests #210).
12. ✅ Prefix search via native listing: ListObjectsV2/ListObjects `prefix`, in UTF-8 byte order, with delimiters and pagination (truthdb#204; tests #210).
13. ✅ Substring or pattern match on names: `SELECT ... FROM sys.s3_objects(@bucket [, @prefix]) WHERE [key] LIKE ...` over TDS, a live inventory read from the object engine's own trees (truthdb#211).
14. ✅ Search by metadata: `sys.s3_objects`, `sys.s3_object_versions`, `sys.s3_object_tags` and `sys.s3_object_metadata` (size, storage class, content type, tags, user metadata), plus `sys.s3_journal(@bucket [, @after])`, the change journal in S3 Metadata's journal-table shape (truthdb#211, #213).
15. ✅ Full-text search inside object contents: opt-in per bucket (`s3 enable fulltext <bucket> [max_bytes]`), indexed in the background from the change journal, and queried from SQL with `sys.s3_search(@bucket, @condition [, @top])`, a CONTAINS-style condition ranked by BM25 (truthdb#215, #218, #220).
16. ✅ Query structured file contents: `sys.s3_read(@bucket, @key, 'csv'|'json'|'parquet' [, @options])` streams a stored object (or, with `{"prefix": true}`, every object under a prefix) as typed rows to any SQL client; Parquet files are read defensively against hostile input (truthdb#216, #217).
17. ✅ Semantic similarity search over embeddings: the S3 Vectors JSON API on the S3 port (SigV4 service `s3vectors`; vector buckets, indexes, Put/Get/List/DeleteVectors and QueryVectors) over a new vector engine on the shared log, with exact cosine or euclidean top-k search and S3 Vectors metadata filters; embeddings are supplied by the client, and a query returns all its results in one response (truthdb#221, #222).

## Operations

18. ✅ Access control: accounts, their users and access keys; AWS-style bucket policies and user policies (inline and managed, through an IAM API subset and STS GetCallerIdentity) evaluated as AWS does, scoping read/write/list/delete to buckets or prefixes, across accounts and for anonymous callers; Block Public Access per bucket and per account, on for new buckets; ACLs disabled (BucketOwnerEnforced) (truthdb#223–#227).
19. ✅ Sharing: presigned URLs (query-string SigV4, up to 7 days, for GET, HEAD, PUT, DELETE and part uploads, with `response-*` header overrides); browser form uploads (POST Object, the SDKs' presigned POST) under a signed policy document (expiration, eq, starts-with and content-length-range conditions, every field covered) or anonymously under a bucket policy; public read for selected content by bucket policy on a prefix or an object tag (truthdb#228, #230).
20. ✅ Data protection: the whole database encrypted at rest (pages AES-256-XTS with a MAC, log records and backups AES-GCM, standbys and restores included) under a database key wrapped by a master key from a key file or a wrap/unwrap command; online encryption of an existing database and rotation of both keys, resumable after a crash; SSE-S3 as that encryption, a bucket's encryption configuration, and SSE-C (customer-provided keys, over TLS, copies decrypting and encrypting again); Object Lock: retention in GOVERNANCE or COMPLIANCE mode, legal holds and event holds (variable retention), per object and as a bucket default, enforced wherever versions are removed (truthdb#231, #233–#235, #240–#242, #244, #246, #248, #250).
21. ✅ Lifecycle: the eight storage classes, kept with each version and shown everywhere; cold classes stored zstd-compressed (packed in place in the background, or as they arrive); GLACIER and DEEP_ARCHIVE readable only while restored (RestoreObject opens a time window, no copy); bucket lifecycle configurations as S3 validates them (prefix, tag and size filters; transitions, expirations, noncurrent-version rules, expired delete markers, abort of incomplete uploads), applied by a daily background pass, at most one action per version a day, through the engine's own paths so Object Lock applies; `x-amz-expiration` and `x-amz-restore` headers; every change in the journal, which consumers follow from their own place (truthdb#251–#257).
22. Synchronise and migrate: sync local directories, migrate datasets, transfer between storage systems.
23. Event notifications: emit events on upload and delete; trigger functions or queue jobs for processing pipelines.
24. Bulk jobs: apply operations to large object collections (Batch Operations).
25. Serve content: images, downloads, static website hosting, CDN integration, CORS.
26. Monitoring and audit: storage usage, request/error/cost metrics, access logs, API activity logs, inventory reports.

## Replication

27. Configure replication: choose source and destination, grant required permissions.
28. Same-region replication.
29. Cross-region replication.
30. Cross-account replication.
31. Selective replication by prefix or tag.
32. Replication to multiple destinations.
33. Replicate existing objects and retry failed replication (Batch Replication).
34. Delete-marker replication setting; permanent version deletes never replicate.
35. Replication monitoring: status, failures, lag.
36. Change or stop replication rules; existing destination copies remain.

## Not covered by the source conversation

37. Conditional writes: `If-None-Match` (create only if absent) and `If-Match` (compare-and-swap on ETag).
38. Strong read-after-write consistency for all operations, stated and tested as a guarantee.
39. Checksums and integrity: client-supplied checksums (CRC32, CRC32C, SHA-1, SHA-256) verified on upload and returned on download.
40. Directory buckets (Express One Zone equivalent): low-latency bucket type with native rename.
41. Table buckets (S3 Tables equivalent): managed Apache Iceberg tables.
42. Access Points: named endpoints with their own access policies per application.
43. Object Lambda: transform objects on read.
44. Requester Pays buckets.
45. Transfer acceleration.
46. Storage analytics dashboard (Storage Lens equivalent).
47. Mount a bucket as a filesystem (Mountpoint equivalent).
