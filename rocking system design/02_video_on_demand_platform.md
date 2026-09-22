# System Design Session 02 — Video-on-Demand Platform

## Goal

Design a scalable Video-on-Demand platform similar to YouTube, Netflix, or Amazon Prime Video.

The session focuses on four user-facing features:

1. Upload video
2. Search videos
3. View / stream videos
4. Detect and flag adult / explicit content

And four main design considerations:

- Scalability
- Resilience / high availability
- Security
- Cost effectiveness

---

## 1. Start with Scope Before Architecture

A key interview habit from the session:

> Do not start drawing architecture immediately.

First clarify:

### Features in scope
- Upload
- Search
- View / stream
- Explicit-content detection

### Non-functional requirements
- Highly scalable
- Resilient / highly available
- Secure
- Cost effective

This prevents the interview from becoming too broad and gives the interviewer a clear boundary for the design.

---

# High-Level Architecture

```text
                         +----------------------+
                         |  Static Web Frontend |
                         |        S3            |
                         +----------+-----------+
                                    |
                                    v
                                  CDN
                                    |
                       +------------+------------+
                       |                         |
                       v                         v
                 API Gateway                Video Content
                       |                         |
                       v                         v
                    Lambda                     S3
                       |                         ^
                       v                         |
                 Elasticsearch /                |
                 Search Index                   |
                                                 |
User Upload                                      |
   |                                             |
   v                                             |
Original 4K MP4                                  |
   |                                             |
   v                                             |
   S3 -------------------------------------------+
   |
   +--> Step Functions
           |
           +--> MediaConvert --> 720p / 960p / 1080p / ...
           |
           +--> Rekognition --> moderation result
           |
           +--> Metadata indexing --> Elasticsearch
```

---

# 2. Upload Flow

Assume a user uploads:

- A 4K video
- MP4 format
- Title
- Description
- Tags

The video file and the metadata should not necessarily be stored in the same system.

---

## Video Storage — Amazon S3

The instructor chooses **Amazon S3** for video/object storage.

### Why S3?

The requirements are:

- Scalable
- Durable / resilient
- Secure
- Cost effective

Using attached block storage or local disks introduces operational overhead:

- Capacity planning
- Manual expansion
- Replication
- Backup / restore
- Failure handling
- Partition management

Object storage fits large media files much better.

```text
Uploader
   |
   v
Original Video
   |
   v
+------+
|  S3  |
+------+
```

### Interview point

For very large media objects, think **object storage**, not a relational database or local disk.

---

# 3. Metadata Storage and Search

Video metadata includes:

- Title
- Description
- Tags
- Video ID
- Locations / object keys for encoded versions
- Moderation / explicit-content status

The session uses **Elasticsearch** because the metadata needs text-search capability.

```text
Title
Description
Tags
   |
   v
+----------------+
| Elasticsearch  |
+----------------+
```

The search index allows users to search free-form text such as titles, descriptions, and tags.

---

# 4. Video Transcoding

A video may be uploaded in 4K, but users will consume it on many different devices and network conditions.

Possible output renditions:

- 720p
- 960p
- 1080p
- Other supported bitrates / resolutions

### Why not transcode on every view?

If transcoding were performed when a user presses Play:

- It would be CPU intensive
- Millions of simultaneous viewers could overload the system
- Playback startup latency would increase
- Users could experience buffering / stuttering

So the uploaded video is processed **ahead of time**.

```text
             Original 4K
                 |
                 v
          MediaConvert
        /      |       \
       v       v        v
    720p     960p     1080p
       \       |       /
        +------ S3 ----+
```

The AWS service used in the session is:

**AWS Elemental MediaConvert**

The generated renditions are stored back in S3.

Their object locations can be associated with the video metadata/search record.

---

# 5. Explicit-Content Detection

The session uses **Amazon Rekognition** for content moderation.

Conceptually:

```text
Original Video
     |
     v
Amazon Rekognition
     |
     v
Moderation Result
     |
     v
Metadata / Search Index
```

The system can update the video's record to mark or block content that violates policy.

The important system-design concept is that this moderation work can happen asynchronously after upload.

---

# 6. Parallel Processing with Step Functions

After the original file is uploaded, several jobs can happen independently:

- Encode 720p
- Encode 960p
- Encode 1080p
- Run content moderation
- Index metadata

The session uses **AWS Step Functions** as a workflow/orchestration engine.

```text
                    S3 Upload
                       |
                       v
                    Lambda
                       |
                       v
                Step Functions
          _________|___________
         |         |           |
         v         v           v
    MediaConvert  Rekognition  Metadata
      jobs        moderation   indexing
```

### Why parallelize?

Because transcoding, moderation, and indexing do not all need to wait for one another.

Parallel execution reduces total processing time.

---

# 7. Search Flow

Once the video is processed and indexed, users can search for it.

The whiteboard flow is approximately:

```text
User
 |
 v
Static Website
 |
 v
CDN
 |
 v
API Gateway
 |
 v
Lambda
 |
 v
Elasticsearch
```

The static frontend can itself be hosted from S3 and distributed through a CDN.

For dynamic search requests:

1. User enters a search term.
2. Request reaches API Gateway.
3. API Gateway invokes backend logic, represented by Lambda.
4. Lambda queries Elasticsearch.
5. Matching video metadata is returned.
6. The result may be cached closer to users where appropriate.

---

# 8. CDN for Global Delivery

The source video may be stored in one AWS region, for example Northern Virginia.

It is inefficient for every user worldwide to retrieve the same video directly from the origin S3 bucket.

A **Content Delivery Network (CDN)** is placed in front of the media origin.

```text
                 +----------------+
                 | Origin S3      |
                 | US Region      |
                 +-------+--------+
                         |
                         v
                      CDN
             _________|__________
            |         |          |
            v         v          v
          India     Europe      US
          Edge       Edge       Edge
```

### First request

If a video is not present at the edge:

```text
Viewer --> CDN Edge --> Origin S3
```

The CDN fetches it from origin and can cache it.

### Subsequent nearby requests

```text
Viewer --> CDN Edge --> Cached Video
```

Benefits:

- Lower latency
- Reduced origin traffic
- Better scalability
- Potentially lower transfer costs

---

# 9. Video Streaming with HLS

The transcript's \"ageless\" references are **HLS — HTTP Live Streaming**.

The idea is to divide media into small segments instead of sending the entire video as one monolithic object.

```text
Video
 |
 +--> Segment 1
 +--> Segment 2
 +--> Segment 3
 +--> Segment 4
 +--> ...
```

The player requests segments as playback progresses.

If the viewer stops after five minutes, the system does not need to deliver the rest of a long video.

Benefits:

- Faster playback start
- Reduced unnecessary bandwidth usage
- Better delivery over changing network conditions
- More cost effective

---

# 10. Static Frontend + Dynamic APIs + Media

The whiteboard separates the system into three major delivery paths:

### Static website

```text
S3 --> CDN --> Browser
```

### Dynamic API / search

```text
Browser --> CDN/API entry --> API Gateway --> Lambda --> Search Index
```

### Video playback

```text
Browser / Player --> CDN --> Video Origin in S3
```

This separation is an important architecture pattern.

The website, APIs, and large media content do not need to be served by the same backend component.

---

# 11. Cost Optimization with S3 Storage Classes

One of the stated design goals is **cost effectiveness**.

Not every uploaded video remains popular.

The session suggests moving older / less frequently accessed content to cheaper S3 storage classes using lifecycle policies.

Conceptual policy:

```text
Frequently accessed
      |
      v
S3 Standard
      |
      | low access after N days
      v
S3 Infrequent Access
      |
      | very low access after more time
      v
Archive / Glacier class
```

The exact thresholds are business decisions.

Example:

- New / popular video -> S3 Standard
- Older low-view video -> infrequent-access storage
- Rarely accessed archive -> archival storage

### Key design principle

Match **storage cost to access frequency**.

---

# 12. Security

The instructor divides security into:

1. Authentication and authorization
2. Data security

Data security is further divided into:

- Encryption at rest
- Encryption in transit

---

## Authentication / Authorization

Examples mentioned:

- Login / password
- Identity provider integration
- Amazon Cognito
- Active Directory

The system should decide:

- Who can upload?
- Who can view?
- Which content is public/private/restricted?

---

## Encryption at Rest

The session mentions encrypting services such as S3 and Elasticsearch using **AWS KMS-managed encryption keys**.

---

## Encryption in Transit

Use:

- HTTPS
- TLS

```text
Client
  |
 HTTPS/TLS
  v
CDN / API Gateway
  |
 TLS
  v
Backend
```

---

# 13. End-to-End Flow

## Upload

```text
User
 |
 v
Upload original video + metadata
 |
 v
S3
 |
 +--> Step Functions
        |
        +--> MediaConvert
        |      |
        |      +--> multiple renditions --> S3
        |
        +--> Rekognition
        |      |
        |      +--> moderation status
        |
        +--> metadata indexing
               |
               v
          Elasticsearch
```

## Search

```text
User
 |
 v
CDN / Website
 |
 v
API Gateway
 |
 v
Lambda
 |
 v
Elasticsearch
 |
 v
Matching video IDs / metadata
```

## Playback

```text
User selects video
       |
       v
      CDN
       |
       +--> Cache HIT --> Stream HLS segments
       |
       +--> Cache MISS --> S3 origin
                           |
                           v
                       CDN caches
                           |
                           v
                         User
```

---

# 14. Interview Mental Model for Video-on-Demand

When asked to design YouTube / Netflix / Prime Video, walk through:

```text
1. Define features and NFRs
        |
        v
2. Store original media
        |
        v
3. Store / index metadata
        |
        v
4. Transcode into multiple renditions
        |
        v
5. Moderate content
        |
        v
6. Orchestrate processing asynchronously
        |
        v
7. Deliver globally through CDN
        |
        v
8. Stream in segments
        |
        v
9. Optimize storage cost
        |
        v
10. Add auth + encryption
```

---

# 15. Important Speech-to-Text Corrections

Notable transcript corrections:

- \"on demand video\" -> **Video-on-Demand (VoD)**
- \"S3 / history / F3\" -> **Amazon S3**
- \"Elemental Media Convert\" -> **AWS Elemental MediaConvert**
- \"Amazon Recognition\" -> **Amazon Rekognition**
- \"ageless\" -> **HLS**
- \"C D N / citizen / Sydney\" -> **CDN**
- \"API G\" -> **API Gateway**
- \"Kamas\" -> **AWS KMS**
- \"data at transit\" -> **data in transit**
- \"Elastic Search\" -> **Elasticsearch / Amazon OpenSearch-style search layer**

---

# 16. Interview-Safe Technical Clarifications

The following points are useful to avoid repeating overly absolute statements from the lecture.

### Relational databases can scale horizontally

The lecture says relational databases cannot be horizontally scaled. That is too absolute.

A safer interview answer is:

> Relational databases can be horizontally scaled, but it is often more operationally complex because of sharding, distributed transactions, replication, and consistency requirements.

For this use case, the stronger reason to choose a search engine is that **full-text search is a first-class requirement**.

---

### Elasticsearch is a search/indexing engine, not a general replacement for the system of record

A robust production design would often keep durable metadata in a primary database and index searchable fields into Elasticsearch/OpenSearch.

Conceptually:

```text
Primary Metadata DB
        |
        v
Search Index
Elasticsearch / OpenSearch
```

For interview purposes, say:

> The database remains the source of truth, while the search engine is optimized for text search.

---

### MP4 vs HLS

The lecture says MP4 must be fully downloaded before viewing. That is too simplistic.

MP4 can support progressive download / byte-range delivery.

The stronger reason for HLS is:

- Segment-based streaming
- Adaptive bitrate streaming
- Easier CDN delivery
- Switching quality according to bandwidth/device conditions

---

### Adaptive Bitrate Streaming

A stronger design would not select only one fixed resolution based on device size.

Instead, the player can dynamically switch among multiple bitrate renditions depending on changing network conditions.

```text
High bandwidth  --> 1080p
Medium          --> 720p
Low             --> 480p
```

This is one of the most important ideas in real-world video streaming.

---

# 17. What to Remember for Interviews

The most important ideas from this session are:

1. **Scope the problem before drawing.**
2. Store large video objects in **object storage**.
3. Keep searchable metadata in a **search/indexing layer**.
4. **Pre-transcode** video into multiple renditions.
5. Run independent post-upload processing **asynchronously and in parallel**.
6. Use a **CDN** instead of serving every view from the origin.
7. Stream media in **segments**, typically using HLS/DASH-style delivery.
8. Use **lifecycle/storage-tiering** to reduce cost.
9. Cover both **authentication/authorization** and **encryption**.
10. In an interview, always explain **why** each component exists, not just name AWS services.
