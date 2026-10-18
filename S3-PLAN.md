# TruthDB: S3 Workflow Checklist

Every S3 workflow, one numbered item each. When an item is done, overwrite its line with what was built.

## Buckets and objects

1. Manage buckets: create, list, inspect configuration, change settings, empty, delete.
2. ✅ Store objects: PutObject streams the body into the object engine in segments, up to 5 GiB. It verifies SigV4 chunk signatures, checksums and Content-MD5 as the body arrives. Replacing an object releases the old body after a grace period. Uploading many files is a client loop over PutObject (truthdb#203).
3. ✅ Multipart uploads: Create, UploadPart (a part can be retried or replaced), ListParts and ListMultipartUploads (to resume), Complete (a composite of the parts, no bytes copied; S3 order, ETag and size rules; at most 5 TiB) and Abort. Uploads left open for 7 days are aborted by garbage collection (truthdb#206).
4. ✅ Retrieve objects: GetObject streams a leased body; it supports single byte ranges (206/416), the If-Match, If-None-Match, If-Modified-Since and If-Unmodified-Since headers, and `versionId` (truthdb#203).
5. ✅ Inspect objects: HeadObject returns size, ETag (MD5), Last-Modified, content type, the stored system headers and `x-amz-meta-*`, without the body (truthdb#203).
6. ✅ Browse and organise: ListObjectsV2 and ListObjects support prefix, delimiter with CommonPrefixes, max-keys, start-after or marker, continuation tokens and `encoding-type=url`. A folder whose keys are all deleted is not listed, and each call scans a bounded number of rows (truthdb#204).
7. ✅ Copy and move: CopyObject shares the source body (no bytes copied) within or between buckets, with COPY/REPLACE metadata, copy-source conditions and `versionId`. UploadPartCopy shares a whole part or copies a byte range. Move is a copy followed by a delete on the client side. There is one region, so "across regions" does not apply (truthdb#207).
8. ✅ Delete objects: DeleteObject (a delete marker in a versioned bucket, a permanent delete with `versionId`) and DeleteObjects (up to 1000 keys, quiet mode, Content-MD5 or checksum required). Deleting everything under a prefix is a listing followed by DeleteObjects on the client side (truthdb#205).
9. Metadata and tags: read and set user metadata, content type, cache headers, download filename; add, change and remove tags.
10. Versioning: enable/suspend, list versions, retrieve an older version, restore overwritten data, recover deleted objects via delete markers.

## Search

11. Exact lookup by bucket and key.
12. Prefix search via native listing.
13. Substring or pattern match on names (client-side filtering or a name index).
14. Search by metadata (tags, size, storage class) through a queryable metadata index.
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
