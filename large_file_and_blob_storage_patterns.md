# Large File and Blob Storage: Patterns, Pitfalls, and Production Design

> **What this doc covers:** Everything you need to design a production-grade large file and blob storage system — from why you cannot use a relational database for binary data, to how object stores work internally, how to upload and serve files at scale, how CDNs integrate with origin storage, how to handle access control with signed URLs, how to build media processing pipelines, and every tricky failure mode that appears at scale (partial uploads, hot objects, bandwidth costs, deduplication, and more). Includes real-world examples from Netflix, Dropbox, Cloudflare, Figma, and GitHub.

---

### 📋 Outline

| # | Section | What You'll Learn |
|---|---------|-------------------|
| 1 | Why Blobs Need Different Storage | Why databases fail for binary data; object store design philosophy |
| 2 | Object Store Internals | How S3, GCS, and Azure Blob are built: flat namespaces, consistent hashing, erasure coding, strong consistency |
| 3 | The Upload Path | Direct upload vs server-proxied; presigned URLs; multipart upload; resumable uploads (TUS protocol) |
| 4 | The Download and Serving Path | Origin serving vs CDN; byte-range requests; streaming large files; signed URLs for access control |
| 5 | CDN Integration | How CDNs cache blobs; cache invalidation strategies; CDN origin shield; geo-routing |
| 6 | Access Control | Presigned URLs, bucket policies, signed cookies, token-vended access; expiry and scope |
| 7 | Media Processing Pipelines | Post-upload transcoding, thumbnail generation, image resizing; event-driven pipelines with queues |
| 8 | Deduplication and Content-Addressed Storage | SHA-256 fingerprinting, chunk-level dedup (Dropbox), storage savings |
| 9 | Storage Tiers and Lifecycle Policies | Hot/warm/cold/archive tiers; cost vs latency trade-offs; automatic tiering rules |
| 10 | Regular Scenarios | User avatar upload, video platform, document storage, backup system |
| 11 | Tricky Scenarios | Partial upload recovery, hot object problem, large file consistency, cross-region replication lag, cost explosion |
| 12 | Common Anti-Patterns | Storing blobs in Postgres, missing multipart for large files, serving via app server, no lifecycle policy |
| 13 | Reference Case Studies | Dropbox, Netflix, GitHub, Figma, Cloudflare |

---

## 1. Why Blobs Need Different Storage

A relational database stores structured rows in B-tree pages, typically 8KB each. Every row is indexed, transactionally protected, and carefully managed in a buffer pool. This machinery is exactly wrong for binary large objects (BLOBs):

| Property | RDBMS (Postgres/MySQL) | Object Store (S3/GCS) |
|----------|------------------------|----------------------|
| Unit of storage | 8KB pages, row-level locking | Arbitrary-size objects (bytes → terabytes) |
| Write model | In-place update with MVCC overhead | Immutable: write once, replace entirely |
| Read model | Query planner, index scan, row fetch | Key lookup → stream bytes |
| Cost per GB | $100–500/month (EBS/RDS storage) | $0.02–0.05/month (S3 Standard) |
| Max object size | ~1GB practical limit (BYTEA) | 5TB per object (S3), unlimited account |
| Concurrency | Row locks, MVCC snapshots | No locks — immutable objects have no concurrency conflicts |
| CDN integration | None — must proxy through app | Native — CDN pulls directly from bucket |
| Backup/replication | Database backup tools | Built-in multi-AZ replication, cross-region |

**The rule:** If the data is binary, unstructured, or larger than ~100KB — it belongs in an object store, not a database. Store only the metadata (filename, size, content type, storage URL, owner) in the database.

```python
# Correct pattern: metadata in DB, bytes in object store
class FileRecord(Model):
    id           = UUIDField(primary_key=True)
    owner_id     = ForeignKey("users")
    filename     = CharField(max_length=255)
    content_type = CharField(max_length=128)
    size_bytes   = BigIntegerField()
    storage_key  = CharField(max_length=512)  # e.g. "uploads/user_42/report.pdf"
    created_at   = DateTimeField(auto_now_add=True)
    # NOT: file_data = BinaryField() ← never do this
```

---

## 2. Object Store Internals

Understanding how object stores work lets you predict their behavior, avoid their pitfalls, and make correct design decisions.

### 2.1 Flat Namespace (Not a Real Filesystem)

An object store has no real directory hierarchy. The "path" `photos/2025/jan/img.jpg` is simply a string key — the `/` characters are cosmetic. The backend is a key-value store mapping `(bucket, key) → object bytes`.

```
Perceived:
  photos/
    2025/
      jan/
        img001.jpg
        img002.jpg

Reality (flat key space):
  "photos/2025/jan/img001.jpg" → [bytes]
  "photos/2025/jan/img002.jpg" → [bytes]
  "photos/2025/jan/"           → does NOT exist as an entity
```

**Consequence:** There is no atomic "rename directory" or "move folder" operation. Renaming `photos/2025/` to `archive/2025/` means copying every object with the new key prefix and deleting the old ones — O(N) operations for N files. This is why S3 "folder renames" are expensive and S3 itself doesn't natively support them.

### 2.2 Object Distribution via Consistent Hashing

Objects are distributed across storage nodes by hashing the object key. Early S3 used a prefix-based approach: objects with the same key prefix landed on the same physical partition.

```
Early S3 anti-pattern (prefix hot-spotting):
  All keys start with date: "2025-03-01-log-00001", "2025-03-01-log-00002"
  → Same prefix → same partition → hot spot → throttling (HTTP 503)

Fix: randomize prefix
  key = f"{hash(object_id)[:6]}/{object_id}/data"
  e.g. "a7f2c3/order-4892/receipt.pdf"
  → Distributed across partitions

Modern S3 (2018+): Handles sequential prefixes well (auto-partitions under the hood).
  But for very high-throughput workloads (>3,500 PUT/sec per prefix), still randomize.
```

### 2.3 Erasure Coding for Durability

Storing 3 full copies of every object (3x replication) costs 3x storage. Production object stores use **erasure coding** (Reed-Solomon) for a much better durability/cost ratio.

```
Erasure coding example (6+3 configuration):
  1. Split object into 6 equal data chunks
  2. Compute 3 parity chunks from the data chunks
  3. Store all 9 chunks on 9 different drives/nodes/AZs

Recovery:
  Lose any 3 chunks (any combination) → reconstruct perfectly from the remaining 6
  Storage overhead: 9/6 = 1.5x  (vs 3x for full replication)

S3's actual configuration (undisclosed, but effectively):
  Objects split across multiple AZs
  Can survive loss of entire AZ with no data loss
  11 nines (99.999999999%) durability — meaning ~1 object lost per 100 billion per year
```

The tradeoff: reconstruction from erasure coding requires reading from multiple nodes (higher read latency than full replication). Object stores use full replication for hot-tier objects and erasure coding for warm/cold tiers.

### 2.4 Strong Read-After-Write Consistency (S3 since December 2020)

Early S3 had eventual consistency — a `PUT` followed immediately by a `GET` could return a 404 (the `GET` hit a replica that hadn't applied the write yet). This caused subtle bugs in pipelines.

Since December 2020, S3 provides **strong read-after-write consistency** for all operations:

```python
# This is now SAFE on S3:
s3.put_object(Bucket="my-bucket", Key="report.pdf", Body=data)
obj = s3.get_object(Bucket="my-bucket", Key="report.pdf")  # Guaranteed to succeed
```

The mechanism: S3 uses a strongly-consistent metadata service (similar to a Paxos cluster) to track object state. A `GET` always queries this service to find the authoritative location before fetching bytes, ensuring it sees committed `PUT`s.

**Important:** This is for S3 specifically. Other object stores vary. Always check the consistency model for GCS (also strongly consistent for new objects), Azure Blob (strongly consistent), and self-hosted MinIO (strongly consistent).

### 2.5 Multipart Upload Internals

S3 splits large uploads into parts that are uploaded independently and assembled server-side.

```
Object: 1GB video file

Client:
  1. InitiateMultipartUpload(bucket, key) → upload_id="abc123"
  2. UploadPart(upload_id, part_number=1, data=chunk_1[0-100MB])   → ETag: "etag1" ┐ parallel
     UploadPart(upload_id, part_number=2, data=chunk_2[100-200MB]) → ETag: "etag2" ┤
     ...                                                                              ┘
     UploadPart(upload_id, part_number=10, data=chunk_10[900MB-1GB])→ ETag: "etag10"
  3. CompleteMultipartUpload(upload_id, [(1,"etag1"), (2,"etag2"), ...]) → final object

S3 side:
  Stores each part as a temporary object
  On CompleteMultipartUpload: assembles parts into final object atomically
  The final object is visible only after CompleteMultipartUpload succeeds
```

**Critical rule:** Always call `AbortMultipartUpload` if the upload is abandoned. Incomplete multipart uploads accumulate as orphaned parts and incur storage charges. S3 lifecycle policies can auto-abort stale uploads:

```json
{
  "Rules": [{
    "ID": "abort-incomplete-multipart",
    "Status": "Enabled",
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
  }]
}
```

---

## 3. The Upload Path

### 3.1 Anti-Pattern: Server-Proxied Upload

The naive implementation routes file bytes through your application server:

```
Client ──► App Server ──► Object Store
              ↑
        file bytes flow
        through here
```

**Why this is wrong:**
- Every upload byte passes through your app server — CPU, memory, and network bandwidth consumed for a task the app server adds no value to
- Large uploads occupy a thread/connection on your app server for the entire upload duration
- Upload throughput limited by your app server's network card, not the object store
- App server becomes a bottleneck; scaling requires scaling app servers proportionally with upload volume

### 3.2 Correct Pattern: Direct Upload via Presigned URL

The app server's only job is to generate a short-lived, scoped credential (presigned URL). The client uploads directly to the object store.

```
Step 1: Client requests upload permission from app server
        Client ──► POST /api/upload/request  ──► App Server
                                                     │
                                             generates presigned URL
                                             (valid for 15 minutes, scoped to one key)
                                                     │
                   Client ◄── { upload_url, file_key } ◄── App Server

Step 2: Client uploads directly to object store
        Client ──────────────── PUT presigned_url ──────────────► S3
                                    (raw file bytes, no app server involved)

Step 3: Client notifies app server upload is complete
        Client ──► POST /api/upload/complete { file_key, size, checksum }
                         App Server validates + saves metadata to DB
```

```python
# App server: generate presigned URL
import boto3, uuid

s3 = boto3.client("s3", region_name="us-east-1")

def request_upload(user_id, filename, content_type):
    file_key = f"uploads/{user_id}/{uuid.uuid4()}/{filename}"
    
    presigned_url = s3.generate_presigned_url(
        ClientMethod="put_object",
        Params={
            "Bucket": "my-uploads",
            "Key": file_key,
            "ContentType": content_type,
            # Enforce max file size server-side (prevents client sending huge files)
            "ContentLength": ...,  # passed in from client's declared size
        },
        ExpiresIn=900,  # 15 minutes
    )
    return {"upload_url": presigned_url, "file_key": file_key}

# Client (JavaScript):
async function uploadFile(file) {
    // 1. Get presigned URL from your API
    const { upload_url, file_key } = await fetch("/api/upload/request", {
        method: "POST",
        body: JSON.stringify({ filename: file.name, content_type: file.type })
    }).then(r => r.json());

    // 2. Upload directly to S3 — your servers never see the bytes
    await fetch(upload_url, {
        method: "PUT",
        body: file,
        headers: { "Content-Type": file.type }
    });

    // 3. Confirm completion
    await fetch("/api/upload/complete", {
        method: "POST",
        body: JSON.stringify({ file_key, size: file.size })
    });
}
```

### 3.3 Multipart Upload for Large Files

For files over ~100MB, a single-part upload is fragile — any network interruption requires starting over. Use multipart upload with parallel parts.

```python
import boto3, math, concurrent.futures

def upload_large_file(file_path, bucket, key, part_size_mb=50):
    s3 = boto3.client("s3")
    part_size = part_size_mb * 1024 * 1024  # 50MB per part

    # Step 1: Initiate
    mpu = s3.create_multipart_upload(Bucket=bucket, Key=key)
    upload_id = mpu["UploadId"]

    parts = []
    file_size = os.path.getsize(file_path)
    num_parts = math.ceil(file_size / part_size)

    def upload_part(part_number):
        offset = (part_number - 1) * part_size
        with open(file_path, "rb") as f:
            f.seek(offset)
            data = f.read(part_size)
        response = s3.upload_part(
            Bucket=bucket, Key=key,
            UploadId=upload_id,
            PartNumber=part_number,
            Body=data
        )
        return {"PartNumber": part_number, "ETag": response["ETag"]}

    # Step 2: Upload parts in parallel
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        futures = [executor.submit(upload_part, i) for i in range(1, num_parts + 1)]
        parts = [f.result() for f in concurrent.futures.as_completed(futures)]
        parts.sort(key=lambda p: p["PartNumber"])

    # Step 3: Complete
    s3.complete_multipart_upload(
        Bucket=bucket, Key=key,
        UploadId=upload_id,
        MultipartUpload={"Parts": parts}
    )
```

### 3.4 Resumable Uploads (TUS Protocol)

For mobile clients or unreliable networks, even multipart isn't enough — a crash mid-upload means re-uploading completed parts. The **TUS (Tus Resumable Upload) protocol** tracks byte offset server-side so any upload can resume from exactly where it stopped.

```
TUS Protocol:

1. POST /uploads           ← create upload session
   Upload-Length: 524288000  ← declare total file size (500MB)
   Response: Location: /uploads/a3f2b1

2. PATCH /uploads/a3f2b1   ← upload chunk starting at offset 0
   Upload-Offset: 0
   Content-Length: 52428800  ← 50MB chunk
   [bytes 0..50MB]
   Response: Upload-Offset: 52428800

   ... network drops ...

3. HEAD /uploads/a3f2b1    ← check where we left off
   Response: Upload-Offset: 52428800  ← server confirms 50MB received

4. PATCH /uploads/a3f2b1   ← resume from offset 50MB
   Upload-Offset: 52428800
   [bytes 50MB..100MB]
   Response: Upload-Offset: 104857600

... repeat until Upload-Offset = Upload-Length = 500MB
```

**TUS server implementations:** `tusd` (Go, official reference), `tus-node-server` (Node.js), `tus-ruby-server`. Backends: S3, GCS, local disk.

**Who uses it:** Vimeo (their entire upload infrastructure runs on TUS), Cloudflare Stream, GitHub LFS.

---

## 4. The Download and Serving Path

### 4.1 Never Serve Blobs Through Your Application Server

```python
# BAD: app server fetches from S3 and streams to client
@app.route("/files/<file_key>")
def serve_file(file_key):
    obj = s3.get_object(Bucket="my-bucket", Key=file_key)
    return Response(
        obj["Body"].read(),         # ← ALL bytes pass through app server RAM
        content_type=obj["ContentType"]
    )
```

**Why this is a problem:** Even a modest file (10MB) × 1000 concurrent downloads = 10GB in your app server's memory/network at once. Your app servers become a download bottleneck. Every download dollar is spent twice (S3 egress to app server + app server egress to client).

### 4.2 Correct Pattern: Presigned Download URLs

Generate a short-lived, signed URL that the client uses to download directly from S3.

```python
def get_download_url(file_key, expires_in=3600):
    return s3.generate_presigned_url(
        ClientMethod="get_object",
        Params={"Bucket": "my-bucket", "Key": file_key},
        ExpiresIn=expires_in,  # 1 hour
    )

# API response:
# { "download_url": "https://my-bucket.s3.amazonaws.com/uploads/user_42/report.pdf?X-Amz-Signature=..." }
# Client follows this URL directly — app server not involved in the transfer
```

**Controlling Content-Disposition (download vs inline):**
```python
# Force browser to download (not open in browser):
s3.generate_presigned_url(
    "get_object",
    Params={
        "Bucket": "my-bucket",
        "Key": file_key,
        "ResponseContentDisposition": f'attachment; filename="{original_filename}"',
    },
    ExpiresIn=3600,
)
```

### 4.3 Byte-Range Requests (Streaming and Seeking)

HTTP byte-range requests allow clients to request specific portions of an object — essential for video seeking, resumable downloads, and partial content delivery.

```
Client: GET /video.mp4
        Range: bytes=10485760-20971519  ← bytes 10MB to 20MB

S3 Response:
  HTTP/1.1 206 Partial Content
  Content-Range: bytes 10485760-20971519/524288000
  Content-Length: 10485759
  [10MB of video bytes]
```

S3 and GCS both support range requests natively. All CDNs pass range requests through to origin and cache the requested byte range (or the full object, depending on configuration).

**Application:** Video players (HLS, DASH) use byte-range requests to load video segments. A 4K movie stored as a single MP4 can be seeked without downloading the entire file.

```python
# Serve a specific byte range from S3:
def get_range(file_key, start_byte, end_byte):
    response = s3.get_object(
        Bucket="my-bucket",
        Key=file_key,
        Range=f"bytes={start_byte}-{end_byte}"
    )
    return response["Body"].read()
```

---

## 5. CDN Integration

### 5.1 How CDNs Cache Object Store Content

A CDN sits in front of the object store. The first request for an object goes to the CDN edge node closest to the user. On a miss, the edge fetches from origin (the object store), caches it locally, and serves all subsequent requests from cache — without touching origin.

```
User (Tokyo) requests /images/product-42.jpg:

  ┌─ Request 1 (cold) ─────────────────────────────────────────────────────────┐
  │ Tokyo Edge ──MISS──► CDN Shield (Singapore) ──MISS──► S3 Origin (US-East) │
  │            ◄─────────────────────────────────── bytes ◄─────────────────── │
  │ Tokyo Edge caches the object locally                                        │
  └────────────────────────────────────────────────────────────────────────────┘

  ┌─ Requests 2..1,000,000 (warm) ─────────────────────────────────────────────┐
  │ Tokyo Edge ──HIT──► Return cached bytes (5ms, S3 never involved)           │
  └────────────────────────────────────────────────────────────────────────────┘
```

**Cache-Control headers** — set these on every object you upload to control CDN TTL:

```python
# When uploading, set appropriate cache headers
s3.put_object(
    Bucket="my-bucket",
    Key="static/logo-v3.png",
    Body=image_bytes,
    ContentType="image/png",
    CacheControl="public, max-age=31536000, immutable",
    # ↑ Cache for 1 year. "immutable" tells browsers not to revalidate.
    #   Only safe for versioned/content-addressed URLs.
)

s3.put_object(
    Bucket="my-bucket",
    Key="users/42/avatar.jpg",
    Body=avatar_bytes,
    ContentType="image/jpeg",
    CacheControl="public, max-age=86400, stale-while-revalidate=3600",
    # ↑ Cache for 1 day. Serve stale for 1hr while revalidating in background.
)

s3.put_object(
    Bucket="my-bucket",
    Key="invoices/inv-2025-03.pdf",
    Body=pdf_bytes,
    ContentType="application/pdf",
    CacheControl="private, no-store",
    # ↑ Don't cache at CDN — user-specific private document
)
```

### 5.2 Cache Invalidation

When an object changes, stale copies may live on hundreds of CDN edge nodes worldwide.

**Strategy 1: Content-Addressed / Versioned URLs (best for static assets)**
Never invalidate. When content changes, change the URL.

```
/static/app.js?v=a3f2b1c4     ← hash of file contents in URL
/static/logo-v3.png            ← version in filename
```
Old URL cached forever = no cost. New URL auto-fetched on first request. This is how webpack, Vite, and modern CI/CD pipelines work.

**Strategy 2: CDN Purge API (for mutable objects)**
```python
import cloudflare

cf = cloudflare.Cloudflare(api_token=API_TOKEN)

# After updating user avatar:
cf.cache.purge(
    zone_id="my-zone-id",
    files=[f"https://cdn.myapp.com/users/{user_id}/avatar.jpg"]
)
# Takes 1-30 seconds to propagate to all edge nodes
```

**Strategy 3: Short TTL + stale-while-revalidate (for semi-dynamic content)**
Product thumbnails, article images: set `max-age=3600` (1hr). Stale for at most 1 hour after an update, without needing explicit purges.

### 5.3 CDN Origin Shield

Without an origin shield, a cache miss on a popular object triggers hundreds of simultaneous origin fetches from every CDN edge node globally. With an origin shield, all edge nodes funnel through a single regional "shield" node before hitting origin.

```
Without shield:
  500 CDN edges × 1 cache miss per region = 500 simultaneous S3 requests for 1 object

With shield (single regional POP):
  500 CDN edges ──► Shield (1 node) ──MISS──► S3 Origin (1 request)
                                    ──HIT──►  Return from shield cache

S3 origin request count reduced from 500 → 1
```

Enable origin shield in CloudFront:
```json
{
  "OriginShield": {
    "Enabled": true,
    "OriginShieldRegion": "ap-northeast-1"
  }
}
```

---

## 6. Access Control

### 6.1 Presigned URLs (Per-Object Short-Lived Access)

The most common pattern. The server generates a cryptographically signed URL that grants temporary access to exactly one object.

```python
# Read access: signed GET URL (1 hour)
download_url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "private-docs", "Key": f"user_{user_id}/contract.pdf"},
    ExpiresIn=3600
)

# Write access: signed PUT URL (15 minutes, for direct upload)
upload_url = s3.generate_presigned_url(
    "put_object",
    Params={
        "Bucket": "uploads",
        "Key": f"user_{user_id}/{uuid4()}/document.pdf",
        "ContentType": "application/pdf",
    },
    ExpiresIn=900
)
```

**Security properties:**
- URL is valid only until `ExpiresIn` seconds from generation
- Scoped to exactly one bucket + key
- Signature uses your AWS credentials — cannot be forged without your secret key
- Can be revoked indirectly by rotating AWS credentials (but affects all presigned URLs)

### 6.2 Signed Cookies (Access to Many Objects)

Presigned URLs work for single objects. For CDN access to a directory of objects (e.g., a user's entire media library, a paid video course), use **signed cookies**. The browser sends the cookie on every request under the protected URL prefix.

```python
# CloudFront signed cookie: grants access to all objects under /users/42/media/*
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
import base64, json, time

def create_cloudfront_signed_cookie(key_pair_id, private_key_pem, url_prefix, expires_at):
    policy = json.dumps({
        "Statement": [{
            "Resource": f"{url_prefix}*",
            "Condition": {
                "DateLessThan": {"AWS:EpochTime": int(expires_at.timestamp())}
            }
        }]
    }, separators=(",", ":"))

    private_key = serialization.load_pem_private_key(private_key_pem, password=None)
    signature = private_key.sign(policy.encode(), padding.PKCS1v15(), hashes.SHA1())
    
    return {
        "CloudFront-Policy": base64.b64encode(policy.encode()).decode(),
        "CloudFront-Signature": base64.b64encode(signature).decode(),
        "CloudFront-Key-Pair-Id": key_pair_id,
    }

# Set cookie on login or on course purchase:
cookies = create_cloudfront_signed_cookie(
    key_pair_id="APKAXXXXXXXXXX",
    private_key_pem=open("cf-private.pem", "rb").read(),
    url_prefix="https://cdn.myapp.com/courses/python-101/",
    expires_at=datetime.utcnow() + timedelta(hours=8)
)
response.set_cookie("CloudFront-Policy", cookies["CloudFront-Policy"], secure=True, httponly=True)
response.set_cookie("CloudFront-Signature", cookies["CloudFront-Signature"], secure=True, httponly=True)
response.set_cookie("CloudFront-Key-Pair-Id", cookies["CloudFront-Key-Pair-Id"], secure=True, httponly=True)
```

### 6.3 Bucket Policies and IAM (Service-Level Access)

For server-to-server access (your backend reading/writing objects), use IAM roles — not hardcoded credentials.

```json
// S3 bucket policy: allow only objects with specific prefix for this role
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456:role/upload-service-role" },
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::my-uploads/uploads/*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::my-uploads/internal/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::123456:role/admin-role"
        }
      }
    }
  ]
}
```

**Key rule:** Buckets should be **private by default**. Never use a public bucket for user data. Use presigned URLs or CDN for serving.

### 6.4 Protecting Against Presigned URL Abuse

Presigned URLs can be shared or leaked. Common defenses:

```python
# 1. Short expiry: minimum viable window
#    Avatar images: 24 hours (user won't re-request often)
#    Payment receipts: 15 minutes (sensitive)
#    API file downloads: 60 seconds (used immediately)

# 2. Bind URL to user identity (custom authorizer pattern)
#    Don't use raw S3 presigned URLs externally.
#    Instead: your API generates a short-lived token, which your CDN/Lambda
#    validates against user session before proxying the presigned URL redirect.

@app.route("/download/<file_id>")
@login_required
def download(file_id):
    record = FileRecord.get(file_id)
    if record.owner_id != current_user.id:
        abort(403)
    
    url = s3.generate_presigned_url("get_object",
        Params={"Bucket": BUCKET, "Key": record.storage_key},
        ExpiresIn=60  # 60 seconds — just enough for redirect
    )
    return redirect(url)  # Client follows redirect directly to S3
```

---

## 7. Media Processing Pipelines

### 7.1 The Problem With Synchronous Processing

```python
# BAD: processing during upload request
@app.route("/upload", methods=["POST"])
def upload():
    file = request.files["video"]
    s3.upload(file)
    transcode_to_720p(file)       # 5 minutes ← request times out
    transcode_to_1080p(file)      # 8 minutes
    generate_thumbnail(file)      # 10 seconds
    run_content_moderation(file)  # 30 seconds
    return {"status": "ready"}    # Never reaches here
```

Transcoding a 1GB video takes minutes. This blocks the HTTP connection, exhausts your web server threads, and times out for users.

### 7.2 Event-Driven Processing Pipeline

```
User uploads file ──► S3 ──► S3 Event Notification
                                      │
                                      ▼
                              SQS Queue (or SNS fan-out)
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                    Transcoder    Thumbnail    Moderation
                    Worker        Worker       Worker
                    (EC2/ECS)     (Lambda)     (Rekognition)
                         │            │            │
                         ▼            ▼            ▼
                    S3 (720p)    S3 (thumb)    Moderation
                    S3 (1080p)               result in DB
                    S3 (4K)
                         │
                         ▼
                    DB: file.status = "ready"
                    WebSocket push to client: "Your video is ready"
```

```python
# S3 event → SQS → worker
import boto3, json

sqs = boto3.client("sqs")
s3 = boto3.client("s3")

def process_upload_events():
    while True:
        messages = sqs.receive_message(
            QueueUrl=PROCESS_QUEUE_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20  # Long polling
        )

        for msg in messages.get("Messages", []):
            event = json.loads(msg["Body"])
            record = event["Records"][0]
            bucket = record["s3"]["bucket"]["name"]
            key = record["s3"]["object"]["key"]

            process_media(bucket, key)  # transcode, thumbnail, etc.

            sqs.delete_message(
                QueueUrl=PROCESS_QUEUE_URL,
                ReceiptHandle=msg["ReceiptHandle"]
            )
```

### 7.3 Image Resizing on the Fly (Transform Proxy Pattern)

For user-generated images (avatars, product photos) that need multiple sizes, resizing every image upfront into every possible dimension wastes storage. The alternative: resize on first request, cache the result.

```
Request: /images/product-42/300x300.webp
                │
                ▼
         CloudFront CDN ──HIT──► Return cached resized image (fast)
                │ MISS
                ▼
         Lambda@Edge / Cloudflare Worker (image transform function)
                │
                ├── Fetch original from S3: product-42/original.jpg
                ├── Resize to 300×300, convert to WebP
                ├── Return to CDN (CDN caches this resized version)
                └── Future requests for /images/product-42/300x300.webp → CDN hit

Storage: only ONE original per image in S3
         Derived sizes cached at CDN edge nodes indefinitely (no S3 storage for derivatives)
```

**Tools:** Imgix, Cloudflare Images, AWS Lambda@Edge + Sharp.js, Thumbor (open source).

---

## 8. Deduplication and Content-Addressed Storage

### 8.1 The Problem: Storage Amplification

In any system where users upload files, duplication is rampant:
- The same video is uploaded by 10,000 users
- A company document exists in 500 employees' personal drives
- Profile photos have identical crops across platforms

Without deduplication, each copy costs full storage.

### 8.2 File-Level Deduplication (SHA-256 Fingerprinting)

```python
import hashlib

def upload_with_dedup(file_bytes, metadata):
    # Compute SHA-256 of file content
    sha256 = hashlib.sha256(file_bytes).hexdigest()
    storage_key = f"content/{sha256[:2]}/{sha256[2:4]}/{sha256}"

    # Check if this exact content already exists in storage
    try:
        s3.head_object(Bucket=DEDUP_BUCKET, Key=storage_key)
        # Object already exists — no upload needed
    except s3.exceptions.ClientError:
        # New content — upload it
        s3.put_object(Bucket=DEDUP_BUCKET, Key=storage_key, Body=file_bytes)

    # Store metadata record pointing to the canonical storage key
    db.insert("files",
        owner_id=metadata.owner_id,
        filename=metadata.filename,
        sha256=sha256,
        storage_key=storage_key,  # ← many file records can point to same storage_key
        size_bytes=len(file_bytes)
    )
    return storage_key
```

**Benefit:** 10,000 uploads of the same video = 1 copy in S3. Only the metadata records differ.

**Security consideration:** Never reveal deduplication to users. If user A uploads file X and user B's upload is "instantly complete" (because the content was already there), B could infer A had the same file — a privacy leak. Show normal upload progress regardless of dedup.

### 8.3 Chunk-Level Deduplication (How Dropbox Works)

File-level dedup only works if files are identical. Dropbox uses **variable-size chunk deduplication** — more powerful and handles partial matches.

```
File: 500MB presentation.pptx (modified version of yesterday's 499MB file)

Without chunk dedup:  Upload 500MB ← entire file is "new"

With chunk dedup:
  Split into variable-size chunks using content-defined chunking (Rabin fingerprinting)
  Each chunk has a SHA-256 hash

  Yesterday's file: [chunk1|chunk2|chunk3|...|chunkN]
  Today's file:     [chunk1|chunk2|NEW_CHUNK|chunk3|...|chunkN]
                                   ↑ only this chunk is different

  Upload only NEW_CHUNK (perhaps 4MB) instead of 500MB
  All other chunks already exist on Dropbox's servers

Storage: Store chunks in a content-addressable store (key = SHA-256 of chunk)
         File manifest: ordered list of chunk hashes → reconstruct file by fetching chunks in order
```

```python
def upload_chunked_dedup(file_path):
    chunks = split_into_chunks(file_path)  # Rabin fingerprinting
    
    # Ask server which chunks it already has
    chunk_hashes = [sha256(chunk) for chunk in chunks]
    missing = server.check_chunks(chunk_hashes)  # Returns hashes not yet on server
    
    # Upload only missing chunks
    for chunk, h in zip(chunks, chunk_hashes):
        if h in missing:
            server.upload_chunk(h, chunk)
    
    # Upload manifest (tiny — just an ordered list of hashes)
    server.create_file(filename, chunk_hashes)
```

> 📖 **Reference:** [Dropbox's Magic Pocket and deduplication internals](https://dropbox.tech/infrastructure/magic-pocket-technical-deep-dive)

---

## 9. Storage Tiers and Lifecycle Policies

Object stores offer multiple storage tiers with different cost/latency trade-offs. Choosing the right tier is a major cost lever.

### 9.1 AWS S3 Tier Comparison

| Tier | Cost/GB-month | First Byte Latency | Min Storage Duration | Use Case |
|------|-------------|---------------------|---------------------|----------|
| S3 Standard | $0.023 | Milliseconds | None | Frequently accessed: active user files, CDN origin |
| S3 Standard-IA (Infrequent Access) | $0.0125 | Milliseconds | 30 days | Monthly reports, audit logs, backups accessed occasionally |
| S3 One Zone-IA | $0.01 | Milliseconds | 30 days | Reproducible data (can re-derive if lost) |
| S3 Glacier Instant | $0.004 | Milliseconds | 90 days | Archives that need occasional instant retrieval |
| S3 Glacier Flexible | $0.0036 | Minutes–hours | 90 days | Long-term archives: compliance, historical logs |
| S3 Glacier Deep Archive | $0.00099 | 12–48 hours | 180 days | 7-year compliance retention, disaster recovery |

**Note:** IA (Infrequent Access) tiers charge per-retrieval fee ($0.01/GB retrieved). If you access data frequently, Standard is cheaper overall.

### 9.2 Automatic Lifecycle Policies

```json
// S3 Lifecycle policy: auto-tier and expire objects
{
  "Rules": [
    {
      "ID": "user-uploads-lifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "uploads/" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"  // Move to IA after 30 days of inactivity
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER"  // Move to Glacier after 1 year
        }
      ]
    },
    {
      "ID": "tmp-uploads-expiry",
      "Status": "Enabled",
      "Filter": { "Prefix": "tmp/" },
      "Expiration": {
        "Days": 1  // Delete temporary files after 24 hours
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 1  // Abort orphaned multipart uploads after 1 day
      }
    },
    {
      "ID": "log-archive",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 7,    "StorageClass": "STANDARD_IA" },
        { "Days": 30,   "StorageClass": "GLACIER" },
        { "Days": 2555, "StorageClass": "DEEP_ARCHIVE" }  // 7 years → Deep Archive
      ]
    }
  ]
}
```

---

## 10. Regular Scenarios

### Scenario 1: User Avatar Upload and Serving

**Requirements:** Users upload profile pictures. Served globally with low latency. Multiple sizes needed (32px, 64px, 128px, 256px).

**Architecture:**
1. Frontend requests presigned PUT URL from API (scoped to `avatars/{user_id}/original.{ext}`)
2. Browser uploads original directly to S3
3. S3 event triggers Lambda → resizes to 4 standard sizes → stores to `avatars/{user_id}/32.webp`, `64.webp`, etc.
4. DB updated with avatar URL template: `https://cdn.myapp.com/avatars/{user_id}/{size}.webp`
5. CDN serves all size variants globally (long TTL, cache-busted by version suffix on update)

**Invalidation on update:** New upload uses `avatars/{user_id}/original_v{version}.jpg` — version incremented on each upload. CDN URL includes version → old URL cached forever, new URL fetched fresh.

---

### Scenario 2: Video Platform (YouTube-like)

**Requirements:** Users upload videos up to 5GB. Must support multiple quality levels (360p, 720p, 1080p, 4K). Adaptive bitrate streaming. Global delivery.

**Architecture:**
```
Upload:
  TUS resumable upload → S3 raw-uploads bucket
  S3 event → SQS → Transcoding fleet (Elastic Transcoder / MediaConvert)
  Transcoding: generates HLS segments (2-second .ts files) for each quality
  Output: S3 segments bucket
    videos/{video_id}/360p/index.m3u8
    videos/{video_id}/360p/segment_001.ts
    ...
    videos/{video_id}/master.m3u8  ← points to all quality manifests

Serve:
  CloudFront CDN → S3 segments bucket
  Video player fetches master.m3u8 → selects quality based on network speed
  Player fetches 2-second segments as needed (byte-range requests)
  Segments cached at CDN edge indefinitely (content-addressed filenames)
```

**Why HLS segments (not a single file):** A 2GB MP4 cannot be efficiently seeked or streamed — the player must buffer from the beginning. HLS/DASH splits the video into 2-second chunks. The player downloads only what it's about to play, selects quality in real time, and can seek by jumping to any chunk's URL. Each chunk is cached independently at the CDN.

---

### Scenario 3: Secure Document Storage (Google Drive-like)

**Requirements:** Users store sensitive documents. Documents should not be publicly accessible. Sharing generates time-limited links. Version history required.

**Architecture:**
- All objects stored in a **private bucket** (no public access)
- Every download goes through an authenticated API endpoint that checks permissions and generates a presigned URL (60s TTL) → redirects client
- Sharing: generate a `share_token` (UUID) stored in DB with `{file_id, permissions, expires_at}`. Share link = `https://app.com/share/{share_token}`. On click: validate token → generate presigned URL
- Versioning: enable **S3 Versioning** on the bucket. Each `PUT` creates a new version. DB stores version IDs. User can restore any version.

```python
# Enable versioning:
s3.put_bucket_versioning(
    Bucket="docs-bucket",
    VersioningConfiguration={"Status": "Enabled"}
)

# Upload creates a new version automatically
response = s3.put_object(Bucket="docs-bucket", Key="contracts/agmt.pdf", Body=new_bytes)
version_id = response["VersionId"]  # e.g., "BYT1234ABCD..."

# Restore to a previous version
s3.copy_object(
    Bucket="docs-bucket",
    Key="contracts/agmt.pdf",
    CopySource={"Bucket": "docs-bucket", "Key": "contracts/agmt.pdf", "VersionId": old_version_id}
)
```

---

## 11. Tricky Scenarios

### Tricky Scenario 1: Partial Upload Recovery Without TUS

**Problem:** User is uploading a 2GB file. At 1.9GB, their network drops. On reconnect, your system has no way to know how much was uploaded — starts from scratch. The user gives up.

**Fix without TUS:** Implement a lightweight offset-tracking mechanism.

```python
# Server: track upload state
class UploadSession(Model):
    upload_id    = CharField(primary_key=True)  # UUID
    storage_key  = CharField()
    total_size   = BigIntegerField()
    bytes_received = BigIntegerField(default=0)
    mpu_upload_id = CharField()  # S3 multipart upload ID
    parts         = JSONField(default=list)  # List of completed parts

# Client: on reconnect, check offset
GET /upload/{upload_id}/status
→ { "bytes_received": 1887436800 }  ← resume from byte 1.8GB

# Client sends next chunk starting at that offset
PATCH /upload/{upload_id}
Content-Range: bytes 1887436800-2147483647/2147483648
Body: [remaining bytes]
```

### Tricky Scenario 2: The Hot Object Problem

**Problem:** Your product's launch day. A single hero image (homepage banner) gets 500,000 requests in the first minute. CDN has not yet cached it globally. All 500k requests hit origin (S3) simultaneously.

**Symptoms:** S3 returns HTTP 503 (throttling). Page loads fail. Launch is ruined.

**S3 request rate limits:**
- 3,500 PUT/COPY/POST/DELETE per second per prefix
- 5,500 GET/HEAD per second per prefix

**Fixes:**
1. **Pre-warm the CDN before launch:** Send requests to the CDN URL (with `Cache-Control: no-cache` to force a miss and prime the cache) from servers in each CDN region, 30 minutes before launch.

```python
CDN_EDGE_REGIONS = ["https://cdn-us.myapp.com", "https://cdn-eu.myapp.com", "https://cdn-ap.myapp.com"]

def pre_warm_cdn(object_path):
    for cdn_base in CDN_EDGE_REGIONS:
        requests.get(f"{cdn_base}/{object_path}", headers={"Cache-Control": "no-cache"})
    print(f"Pre-warmed {object_path} across {len(CDN_EDGE_REGIONS)} regions")
```

2. **Multiple key prefixes:** Shard the hot object across multiple prefixes. CDN routes requests to one of N copies.

```python
# Store same image under 4 prefixes:
for i in range(4):
    s3.copy_object(
        Bucket=BUCKET, Key=f"hero/v{i}/banner.jpg",
        CopySource={"Bucket": BUCKET, "Key": "hero/banner.jpg"}
    )

# Serve via CDN with randomized prefix (distributes origin load across 4 keys):
def get_hero_image_url():
    shard = random.randint(0, 3)
    return f"https://cdn.myapp.com/hero/v{shard}/banner.jpg"
```

3. **Request collapsing at CDN (origin shield):** Enable origin shield. All edge nodes funnel through a single shield node. The shield makes 1 request to S3 (even if 1M edges miss simultaneously). The most effective solution.

---

### Tricky Scenario 3: Cross-Region Replication Lag Causes Read-After-Write Failure

**Problem:** User uploads a profile picture in US-East. Immediately navigates to their profile page served from EU-West CDN. CDN cache miss → EU CDN fetches from EU replica of S3 bucket (via S3 Cross-Region Replication). CRR hasn't completed yet (typically <1 minute, but can be seconds) → user sees old avatar.

**Solutions:**

1. **Use a single-region bucket as origin.** CDN serves from one origin (e.g., US-East). EU CDN miss fetches from US-East (slower first request, but always consistent).

2. **Always serve new uploads from origin for a grace period.**
```python
def get_avatar_url(user_id, last_upload_at):
    time_since_upload = time.now() - last_upload_at
    if time_since_upload < timedelta(minutes=2):
        # Too recent for CRR to guarantee — bypass CDN, serve directly from primary region
        return s3.generate_presigned_url("get_object",
            Params={"Bucket": PRIMARY_BUCKET, "Key": f"avatars/{user_id}/256.webp"},
            ExpiresIn=120
        )
    return f"https://cdn.myapp.com/avatars/{user_id}/256.webp"
```

3. **Version uploads into URL.** `avatars/{user_id}/256_v{timestamp}.webp`. Each upload is a new key — CRR replication creates a new key, not overwriting the old one. CDN always fetches new key from origin on first request.

---

### Tricky Scenario 4: Cost Explosion from Unexpected S3 Egress

**Problem:** Monthly S3 bill was $200 last month. This month it's $14,000. What happened?

**Common causes and diagnosis:**
```python
# Use S3 Storage Lens or Cost Explorer to break down costs:
# - PUT/COPY/POST/DELETE requests: charged per 1,000 requests
# - GET/SELECT requests: charged per 1,000 requests
# - Data Transfer OUT (egress): $0.09/GB

# Common culprits:
# 1. Application server proxying downloads (double egress: S3→app, app→user)
# 2. S3 serving directly to users without CDN (full S3 egress price vs $0.01/GB at CDN)
# 3. Inefficient thumbnail generation: fetching original 20MB image to create 100px thumbnail
# 4. Logging pipeline reading raw logs from S3 for every query (should use Athena/S3 Select)
# 5. Development/staging environment hammering production S3 bucket

# Immediate fixes:
# - Route all public traffic through CloudFront (CDN egress to users is free from S3 to CF)
# - Use S3 Transfer Acceleration only when explicitly needed (costs extra per GB)
# - Use S3 Select for partial reads of large CSV/JSON/Parquet files:
response = s3.select_object_content(
    Bucket=BUCKET, Key="large-log.csv",
    ExpressionType="SQL",
    Expression="SELECT s.timestamp, s.error_code FROM S3Object s WHERE s.status = '500'",
    InputSerialization={"CSV": {"FileHeaderInfo": "Use"}},
    OutputSerialization={"CSV": {}}
)
# S3 scans and filters server-side — you pay only for bytes returned, not full object
```

---

### Tricky Scenario 5: Ensuring Upload Integrity (Client-Side Corruption)

**Problem:** A file upload completes successfully (HTTP 200), but the stored file is corrupted — flipped bytes from a network error or client bug.

**Fix: Checksum validation with S3 Content-MD5 or SHA-256.**

```python
import hashlib, base64

def upload_with_integrity_check(file_bytes, bucket, key):
    # Compute SHA-256 of the file before upload
    sha256 = hashlib.sha256(file_bytes).digest()
    checksum_b64 = base64.b64encode(sha256).decode()

    # Send checksum with upload — S3 will verify and reject if mismatch
    s3.put_object(
        Bucket=bucket,
        Key=key,
        Body=file_bytes,
        ChecksumSHA256=checksum_b64,  # S3 validates server-side
    )

    # After upload, verify the stored ETag matches expected (for extra paranoia)
    stored = s3.head_object(Bucket=bucket, Key=key)
    assert stored["ChecksumSHA256"] == checksum_b64, "Upload integrity check failed!"
```

S3 will return HTTP 400 `BadDigest` if the received bytes don't match the declared checksum. This catches network corruption before the object is stored.

---

## 12. Common Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|---|---|---|
| **Storing blobs in Postgres BYTEA** | 8KB page size, B-tree overhead, buffer pool pollution, 1GB practical limit, $100+/GB | Store in S3; store only the S3 key in Postgres |
| **Serving files via app server** | Every byte passes through app RAM and CPU; app server becomes bottleneck; double egress cost | Presigned URLs for private files; CDN for public files |
| **Single-part upload for large files** | Any network drop = start over; upload threads tied up | Multipart (>100MB) or TUS protocol |
| **Public S3 buckets for user data** | Any leaked URL grants permanent access to any user's data | Private buckets + presigned URLs with short TTL |
| **No lifecycle policies** | Storage cost grows indefinitely; old tmp files and orphaned multipart uploads accumulate charges | Lifecycle rules: expire tmp, tier old data, abort stale multipart |
| **Resizing images on upload to all sizes** | Storage multiplied by N sizes; burst CPU on upload; sizes you didn't anticipate need re-processing | Store original; resize on-demand via transform proxy (Lambda@Edge, Imgix) |
| **Hardcoded AWS credentials in code** | Credential leak = full account access | IAM roles for EC2/Lambda; OIDC federation for CI/CD |
| **No checksum on upload** | Silently stored corrupted files; detected only when user opens the file | Always set `ChecksumSHA256` on `put_object` |
| **No S3 versioning on user documents** | Accidental overwrite or deletion is permanent | Enable bucket versioning + lifecycle to expire old versions after N days |
| **Infinite presigned URL TTL** | Leaked URL grants forever-access | Maximum 7 days for S3; use 15 minutes for sensitive documents |

---

## 13. Reference Case Studies

1. **Dropbox: Magic Pocket — Building an Exabyte-Scale Storage System**
   [https://dropbox.tech/infrastructure/magic-pocket-technical-deep-dive](https://dropbox.tech/infrastructure/magic-pocket-technical-deep-dive)

2. **Netflix: Blob Storage at Scale (Open Connect)**
   [https://netflixtechblog.com/open-connect-overview-d6a2b8e09e1d](https://netflixtechblog.com/open-connect-overview-d6a2b8e09e1d)

3. **Cloudflare R2 vs S3: Design Principles and Egress-Free Storage**
   [https://blog.cloudflare.com/introducing-r2-object-storage/](https://blog.cloudflare.com/introducing-r2-object-storage/)

4. **GitHub LFS: Large File Storage Architecture**
   [https://github.blog/engineering/large-file-storage-and-git/](https://github.blog/engineering/large-file-storage-and-git/)

5. **Figma: How Figma Stores Data at Scale**
   [https://www.figma.com/blog/how-figma-stores-data-at-scale/](https://www.figma.com/blog/how-figma-stores-data-at-scale/)

6. **TUS Resumable Upload Protocol (Open Standard)**
   [https://tus.io/protocols/resumable-upload.html](https://tus.io/protocols/resumable-upload.html)

7. **AWS: S3 Best Practices for Performance and Scale**
   [https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)

8. **Facebook Haystack: Finding a Needle in Haystack (image storage at Facebook scale)**
   [https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf)

9. **Google: GCS Consistency and Availability Guarantees**
   [https://cloud.google.com/storage/docs/consistency](https://cloud.google.com/storage/docs/consistency)

10. **AWS: S3 Strong Consistency (December 2020)**
    [https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/](https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/)
