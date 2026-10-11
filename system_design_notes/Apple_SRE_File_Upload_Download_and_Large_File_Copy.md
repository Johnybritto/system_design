# Apple ASE SRE — File Upload, Download and Large File Copy

**Goal:** Practise practical Python coding, storage troubleshooting, and failure handling for the SRE interview. S3 is one example; the same fundamentals apply to internal object stores and mounted filesystems.

## 1. Common interview scenarios

| Scenario | Key concept |
|---|---|
| A 5 GB upload becomes slow | Client upload bandwidth, network, multipart transfers, throttling, retries |
| Upload fails at 80% | Multipart upload, retry only failed parts, persist progress if resuming after process restart |
| Download times out | Streaming, HTTP Range requests, timeout/retry handling |
| Application receives 403/404 | IAM/bucket policy, object key, signed URL expiry, KMS permissions |
| Need to copy a 20 GB file with little RAM | Read/write fixed-size chunks |
| Process crashes during copying | Write to a temporary file, rename only after completion |
| Disk has only 10 GB but file is 20 GB | Precheck free space where possible; handle ENOSPC during writes |
| Pod restarts mid-transfer | Understand ephemeral vs persistent storage and recoverable transfer state |
| NFS/EFS is slow | Mount health, I/O latency, network retransmissions, storage-side limits |

## 2. Python: copy a large file in chunks (the core interview exercise)

**Question:** Copy a 20 GB file without loading it into memory.

```python
source = "large.log"
dest = "backup.log"

with open(source, "rb") as src:
    with open(dest, "wb") as dst:
        while True:
            chunk = src.read(1024 * 1024)  # 1 MB
            if not chunk:
                break
            dst.write(chunk)
```

**Say aloud:**
1. `rb` reads bytes; `wb` creates/truncates and writes bytes.
2. Read and write 1 MB at a time until `read()` returns `b""` (EOF).
3. `with` closes files automatically, even if an exception occurs.
4. **Time: O(n)** for n bytes; **auxiliary space: O(1)** relative to file size (fixed 1 MB chunk).

**Follow-up:** What if the process crashes after copying 15 GB?

- The destination may contain a **partial 15 GB file** that looks like a completed backup.
- Improvement: copy to `backup.log.part` and rename only on success.

```python
import os

source = "large.log"
dest = "backup.log"
temp = dest + ".part"

with open(source, "rb") as src:
    with open(temp, "wb") as dst:
        while True:
            chunk = src.read(1024 * 1024)
            if not chunk:
                break
            dst.write(chunk)

os.replace(temp, dest)
print("Copy completed")
```

**Why `os.replace()`?** When the temp and destination are on the same filesystem, the final name switch is atomic: readers see the old completed file or the new completed file, not a partly written final file. A failed process can leave `.part` behind. Rename is *not* the same as guaranteeing persistence after sudden power loss; strict durability can require `fsync`. For concurrent copies, use unique temporary filenames rather than one shared `.part` path.

**Disk-full follow-up:** If the destination runs out of space, a write raises `OSError` (often `errno.ENOSPC`). The final rename is not reached. Handle the exception, alert/report failure, and clean up the incomplete temp file when safe. A preflight free-space check is useful but not sufficient, because available space can change.

## 3. Python: basic S3 upload and download
this is the only 4 line you need to remm
```python
import boto3

s3=boto3.client(“S3”)
Bucket = "my-bucket”
s3.upload_file(path, bucket, s3_key)

s3.download_file(bucket,s3key, path)
```

```python
import boto3
from botocore.exceptions import BotoCoreError, ClientError

s3 = boto3.client("s3")  # Prefer IAM role / managed credentials
bucket = "my-bucket"

def upload(local_path, key):
    try:
        s3.upload_file(local_path, bucket, key)
        print("Upload succeeded")
    except (ClientError, BotoCoreError, OSError) as exc:
        print(f"Upload failed: {exc}")

def download(key, local_path):
    try:
        s3.download_file(bucket, key, local_path)
        print("Download succeeded")
    except (ClientError, BotoCoreError, OSError) as exc:
        print(f"Download failed: {exc}")
```

Boto3's managed transfer functions can perform multipart transfers automatically for sufficiently large objects. For durable resume *across process restarts*, record the multipart upload ID and completed part metadata explicitly.

## 4. Large-file upload design (interview architecture)

```text
Client --(1. auth + request upload)--> API service
Client <--(2. presigned URL)-------- API service
Client --(3. file bytes, multipart)----------------> Object Storage (S3/Blob)
Object Storage --(4. completion event)--> Queue --> Worker (scan/process)
API / DB <--(5. verify and record metadata)------ Completion workflow
```

- API authenticates and authorizes the user, generates a short-lived URL for a controlled object key.
- Client transfers bytes **directly to object storage**, avoiding heavy API pod data-plane traffic.
- For multipart failures, retry individual parts with bounded backoff and jitter.
- After completion, validate object metadata, size/checksum as needed, then mark the upload available.
- On download, authorize access and issue a short-lived download URL; support streaming/range requests when appropriate.

## 5. SRE troubleshooting order

1. **Client:** file size, source disk read performance, upload bandwidth.
2. **Network:** DNS, TLS, packet loss, latency, connectivity to the correct storage endpoint.
3. **API / proxy (only if file bytes traverse it):** ingress limits, proxy timeout, CPU/memory, buffering.
4. **Storage:** HTTP status codes, request latency, throttling/503, multipart failures, storage limits.
5. **Recovery:** retries with backoff/jitter, resumable transfers, temporary files, integrity validation, useful logs/metrics.

Useful Linux checks: `df -h`, `df -i`, `iostat -xz 1 3`, `nfsstat -m` (for NFS). For S3 test a real authorized transfer; `curl -I` alone does not benchmark upload performance.

## 6. Rapid follow-up Q&A

**Q: S3 upload 80% complete; should we start over?**  
A: With multipart upload, retry the failed parts; on restart, resumption requires saved upload state. Abort abandoned multipart uploads.

**Q: API pod crashes after generating an upload URL?**  
A: Direct client-to-storage transfer can continue independently of that pod, subject to URL expiry and client/network availability.

**Q: Downloading pod crashes?**  
A: An ephemeral local partial file may be lost. Use durable destination storage or resumable downloads where required.

**Q: Two users use the same filename?**  
A: Allocate unique storage keys (e.g. UUID); retain original filenames in metadata.

**Q: A user gets 403?**  
A: Check identity, authorization, bucket/end-point policy, KMS access and signed URL validity. Missing S3 objects can sometimes surface as 403 if permission to list the bucket is absent.

**Q: What makes the copy code production-ready?**  
A: Unique same-directory temp paths, exception handling, cleanup, optional fsync for durability, free-space monitoring and integrity checks.

**Interview takeaway:** Explain the basic chunked copy first. Add a `.part` file plus `os.replace()` when the interviewer introduces crashes. Then discuss disk-full errors, retries, and cloud-object-storage specifics.
