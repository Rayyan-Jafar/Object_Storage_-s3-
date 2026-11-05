# Object Storage: Buckets and Objects

**Course:** Big Data Analytics
**Tool:** Python (boto3) with a local S3-compatible server (moto)

## Overview

This lab works with object storage through the S3 API, using hospital scans, bank statements and photos as example data. It covers creating buckets, uploading and listing objects, reading metadata, overwriting and copying objects, versioning, delete markers, object lock, serving images in multiple sizes, and chunk-level deduplication.

Everything runs against a local in-memory server, so no cloud account or real credentials are needed.

## Key concepts

Object storage keeps data as objects in a flat pool rather than in folders. Each object has three parts: a **key** (its unique name), **metadata** (small facts such as owner, date and content type) and a **payload** (the file's bytes). Objects live in buckets and are accessed by name through a simple API: put, get, list and delete.

Objects are written whole and never edited in place. Changing an object means writing a new one under the same key, and a versioned bucket keeps the older copies. Keys that contain slashes only look like folders; "folder" views are built by listing keys that share a prefix. In real systems a database holds the searchable facts plus a pointer (the object key), and object storage holds the bytes.

## Requirements

- Python 3.9 or newer
- `boto3` and `moto[server]`
- The `sample_files` folder from the lab package

```bash
pip install boto3 "moto[server]"
```

## Setup

**1. Start the local S3 server** in its own terminal and leave it running for the whole lab:

```bash
moto_server -p 5000
```

The server keeps everything in memory, so stopping it erases all buckets and objects.

**2. Connect from Python**, starting from the folder that contains `sample_files`:

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="http://127.0.0.1:5000",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)
```

The `test` / `test` values are dummy credentials that only work with the local moto server. Never put real cloud keys in code or commit them to a repository.

## Repository structure

```
.
├── README.md
├── sample_files/            # Synthetic test files from the lab package
│   ├── xray-chest.png
│   ├── xray-chest-v2.png
│   ├── xray-wrist.png
│   ├── lab-report.pdf
│   ├── mri-brain.dcm
│   ├── statement-2026-03.pdf
│   ├── photo-original.jpg
│   ├── photo-feed.jpg
│   ├── photo-thumb.jpg
│   └── textbook.bin
├── downloads/               # Created in Question 8
└── object_storage_lab.ipynb # Solutions for Questions 1–15
```

All sample files are synthetic. They are not real medical or bank records.

## How to run

1. Start `moto_server -p 5000`.
2. Open the notebook (or script) from the project root so the `sample_files/` paths resolve.
3. Run the connection cell first, then run the questions **in order**. Later questions depend on buckets and objects created earlier.
4. To start over, restart the server, re-run the connection code and begin again from Question 1.

## What each part covers

| Part | Questions | Topic |
| --- | --- | --- |
| A: Create | 1–3 | Creating the `hospital-scans` bucket, uploading objects with keys, content types and custom metadata |
| B: Read | 4–8 | Listing objects, filtering by prefix, grouping with a delimiter, reading metadata with `head_object`, downloading and verifying with MD5 |
| C: Change an object | 9–11 | Overwriting and comparing ETags, "renaming" via copy + delete, enabling versioning and listing versions |
| D: Delete and protect | 12–13 | Delete markers and restoring deleted objects, object lock in COMPLIANCE mode on the `bank-statements` bucket |
| E: Photo sizes and chunking | 14–15 | Serving thumbnail / feed / original sizes from the `photo-app` bucket, fixed-size chunking and deduplication |

## Buckets created

| Bucket | Purpose | Features used |
| --- | --- | --- |
| `hospital-scans` | Patient scans under `patients/<patient id>/<date>/<file>` | Metadata, prefixes, versioning, delete markers |
| `bank-statements` | Monthly statements that must stay unchanged | Object lock (COMPLIANCE, 30 days) |
| `photo-app` | One photo stored in three sizes | Prefix listing, size-based retrieval |

## Notes

- Bucket names must be lowercase; a name like `Hospital_Scans` is rejected.
- `list_objects_v2` returns keys in alphabetical order.
- Versions written before versioning was enabled show a version id of `null`.
- A COMPLIANCE lock cannot be removed by anyone until it expires on a real cloud account. Only test object lock on the local server.
- In Question 14, the sample photos are scaled down (the original stands for 8 MB, the feed copy for 200 KB), but the size ratio matches a real app.

## Submission

Submitted as a single PDF or Word file named `RollNo_Name_ObjectStorage`, containing:

- Code for every question, pasted as text
- A one-line answer for each question (for example, "Objects in bucket = 4")
- Output screenshots for Questions 2, 4, 7, 9, 11, 12, 13, 14 and 15
- A written explanation for Question 15(b)
