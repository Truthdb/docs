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
15. Full-text search inside object contents.
16. Query structured file contents (Athena/Spark-style SQL over stored files).
17. Semantic similarity search over embeddings (S3 Vectors equivalent).

## Operations

18. Access control: grant and revoke read/write/list/delete, scope to buckets or prefixes, cross-account access, block public access.
19. Sharing: presigned download URLs, presigned uploads, public read for selected content.
20. Data protection: encryption at rest, key management, retention periods (Object Lock), legal holds.
21. Lifecycle: transition to cheaper storage classes, expire objects after a period, expire old versions, restore archived objects before retrieval.
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
