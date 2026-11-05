# Object Storage Lab: Step-by-Step Guide

**Deadline:** today before 11
**Submit:** one PDF or Word file named `RollNo_Name_ObjectStorage` (e.g. `BSDSF23M031_YourName_ObjectStorage.docx`)

> Lab rule: *"Write the code yourself."* The code below is verified to produce the expected outputs on this machine. Use it as a reference: type it out and make sure you understand each call, because you may be asked about it.

---

## 0. Already done for you

- [x] Installed `boto3 1.43` and `moto[server] 5.2.4` (`pip install boto3 "moto[server]"`)
- [x] Checked that `moto_server` runs and that all 15 questions produce the expected outputs
- [x] Created `object_storage_lab.ipynb` with a connection cell and one empty cell per question

## 1. Start the server (Terminal 1, leave it open)

In VS Code, open a terminal (`Ctrl+``) and run:

```
moto_server -p 5000
```

You should see `Running on http://127.0.0.1:5000`. **Don't close this terminal.** If you restart it, everything is wiped, so you'll need to re-run from Q1.

## 2. Open the notebook

1. Open `object_storage_lab.ipynb` in VS Code.
2. Top right: **Select Kernel → Python 3.13**.
3. Run the **Setup** cell. If there's no error, you're connected.
4. Run each question cell **in order**. Every question depends on the ones before it.

> If something goes wrong partway through: stop the server (`Ctrl+C`), start it again, **Restart** the notebook kernel, and run from the Setup cell.

## 3. Questions

### Part A: Create

**Q1:** `create_bucket`, `list_buckets`
```python
s3.create_bucket(Bucket="hospital-scans")
for b in s3.list_buckets()["Buckets"]:
    print(b["Name"])
```
Expected: `hospital-scans` · One-line answer: *Bucket created = hospital-scans*

**Q2:** `upload_file`, `list_objects_v2` 📸
```python
B = "hospital-scans"
K = "patients/P-40231/2026-09-01/xray-chest.png"
s3.upload_file("sample_files/xray-chest.png", B, K)
for o in s3.list_objects_v2(Bucket=B)["Contents"]:
    print(o["Key"], o["Size"])
```
Expected: `patients/P-40231/2026-09-01/xray-chest.png 20000`

**Q3:** `ExtraArgs` sets the content type and metadata
```python
uploads = [
    ("lab-report.pdf", "patients/P-40231/2026-09-01/lab-report.pdf", "application/pdf",   "P-40231", "lab"),
    ("mri-brain.dcm",  "patients/P-40231/2026-09-15/mri-brain.dcm",  "application/dicom", "P-40231", "mri"),
    ("xray-wrist.png", "patients/P-40232/2026-09-03/xray-wrist.png", "image/png",         "P-40232", "xray"),
]
for f, key, ctype, patient, scan in uploads:
    s3.upload_file(f"sample_files/{f}", B, key,
                   ExtraArgs={"ContentType": ctype,
                              "Metadata": {"patient": patient, "scan-type": scan}})
print(s3.list_objects_v2(Bucket=B)["KeyCount"])
```
Expected: `4`

### Part B: Read

**Q4** 📸
```python
r = s3.list_objects_v2(Bucket=B)
print(r["KeyCount"])
for o in r["Contents"]:
    print(o["Key"])
```

**Q5:** `Prefix=`
```python
r = s3.list_objects_v2(Bucket=B, Prefix="patients/P-40231/")
print(r["KeyCount"])
for o in r["Contents"]:
    print(o["Key"], o["Size"])
```

**Q6:** `Delimiter="/"` returns `CommonPrefixes`
```python
r = s3.list_objects_v2(Bucket=B, Prefix="patients/", Delimiter="/")
for c in r["CommonPrefixes"]:
    print(c["Prefix"])
```

**Q7:** `head_object` reads metadata without downloading 📸
```python
h = s3.head_object(Bucket=B, Key="patients/P-40231/2026-09-15/mri-brain.dcm")
print(h["ContentLength"], h["ContentType"], h["Metadata"])
```
Expected: `50000 application/dicom {'patient': 'P-40231', 'scan-type': 'mri'}`

**Q8:** `download_file` + `hashlib.md5`
```python
os.makedirs("downloads", exist_ok=True)
s3.download_file(B, K, "downloads/xray-chest.png")

def md5(path):
    with open(path, "rb") as f:
        return hashlib.md5(f.read()).hexdigest()

print(md5("downloads/xray-chest.png") == md5("sample_files/xray-chest.png"))
```
Expected: `True`

### Part C: Change an object

**Q9:** compare `head_object` before and after the overwrite 📸
```python
before = s3.head_object(Bucket=B, Key=K)
s3.upload_file("sample_files/xray-chest-v2.png", B, K)
after = s3.head_object(Bucket=B, Key=K)
print(before["ContentLength"], "->", after["ContentLength"])
print("ETag changed:", before["ETag"] != after["ETag"])
```

**Q10:** no rename exists, so `copy_object` then `delete_object`
```python
old = "patients/P-40231/2026-09-01/lab-report.pdf"
new = "archive/P-40231/lab-report.pdf"
s3.copy_object(Bucket=B, Key=new, CopySource={"Bucket": B, "Key": old})
s3.delete_object(Bucket=B, Key=old)
for o in s3.list_objects_v2(Bucket=B)["Contents"]:
    print(o["Key"])
```

**Q11:** `put_bucket_versioning`, `list_object_versions` 📸
```python
s3.put_bucket_versioning(Bucket=B, VersioningConfiguration={"Status": "Enabled"})
s3.upload_file("sample_files/xray-chest.png", B, K)
for v in s3.list_object_versions(Bucket=B, Prefix=K)["Versions"]:
    print(v["VersionId"], v["IsLatest"], v["Size"])
```
Expected: `<long id> True 20000` then `null False 22000`

### Part D: Delete and protect

**Q12** 📸
```python
W = "patients/P-40232/2026-09-03/xray-wrist.png"

r = s3.delete_object(Bucket=B, Key=W)
print("(a)", r.get("DeleteMarker", False))

print("(b)", s3.list_objects_v2(Bucket=B)["KeyCount"])

r = s3.list_object_versions(Bucket=B, Prefix=W)
versions, markers = r.get("Versions", []), r.get("DeleteMarkers", [])
print("(c)", len(versions), "version and", len(markers), "delete marker")

# Deleting the delete marker (by its VersionId) brings the object back
s3.delete_object(Bucket=B, Key=W, VersionId=markers[0]["VersionId"])
print("(d)", s3.list_objects_v2(Bucket=B)["KeyCount"])
```
Expected: `True`, `3`, `1 version and 1 delete marker`, `4`

**Q13:** object lock must be enabled **when the bucket is created** 📸
```python
BB = "bank-statements"
SK = "statements/acct-771/2026-03.pdf"
s3.create_bucket(Bucket=BB, ObjectLockEnabledForBucket=True)

# (a)
until = datetime.now(timezone.utc) + timedelta(days=30)
with open("sample_files/statement-2026-03.pdf", "rb") as f:
    r = s3.put_object(Bucket=BB, Key=SK, Body=f,
                      ObjectLockMode="COMPLIANCE",
                      ObjectLockRetainUntilDate=until)
vid = r["VersionId"]

# (b)
h = s3.head_object(Bucket=BB, Key=SK)
print("(b)", h["ObjectLockMode"], h["ObjectLockRetainUntilDate"])

# (c) deleting the specific version is what the lock blocks
try:
    s3.delete_object(Bucket=BB, Key=SK, VersionId=vid)
except ClientError as e:
    print("(c)", e.response["Error"]["Code"])

# (d)
print("(d)", s3.list_objects_v2(Bucket=BB)["KeyCount"])
```
Expected: `COMPLIANCE <date 30 days ahead>`, `AccessDenied`, `1`

### Part E: Photo sizes and chunking

**Q14** 📸
```python
PB = "photo-app"
s3.create_bucket(Bucket=PB)
for f, name in [("photo-original.jpg", "original"), ("photo-feed.jpg", "feed"), ("photo-thumb.jpg", "thumb")]:
    s3.upload_file(f"sample_files/{f}", PB, f"photos/8831/{name}.jpg",
                   ExtraArgs={"ContentType": "image/jpeg"})

# (a)
print("(a)")
for o in s3.list_objects_v2(Bucket=PB, Prefix="photos/8831/")["Contents"]:
    print(o["Key"], o["Size"])

# (b)
def get_photo(photo_id, screen):
    size = {"list": "thumb", "feed": "feed", "fullscreen": "original"}[screen]
    return s3.get_object(Bucket=PB, Key=f"photos/{photo_id}/{size}.jpg")["Body"].read()

print("(b)")
for screen in ["list", "feed", "fullscreen"]:
    print(screen, len(get_photo(8831, screen)))

# (c) sizes from head_object, no download
feed_total = s3.head_object(Bucket=PB, Key="photos/8831/feed.jpg")["ContentLength"] * 50
orig_total = s3.head_object(Bucket=PB, Key="photos/8831/original.jpg")["ContentLength"] * 50
print("(c)")
print(f"feed copies: {feed_total} bytes")
print(f"originals: {orig_total} bytes")
print("ratio:", orig_total / feed_total)
```
Expected (c): `100000`, `4000000`, `ratio: 40.0`

**Q15(a)** 📸
```python
hashes = []
with open("sample_files/textbook.bin", "rb") as f:
    while True:
        chunk = f.read(4096)
        if not chunk:
            break
        hashes.append(hashlib.md5(chunk).hexdigest())

total, unique = len(hashes), len(set(hashes))
print("Total chunks:", total)
print("Unique chunks:", unique)
print(f"Saving: {(total - unique) / total * 100:.1f}%")
```
Expected: `12`, `8`, `33.3%`

**Q15(b): written answer.** Write this in your own words. Points to cover:
- The drive identifies each chunk by its hash, so identical content gives an identical hash.
- When 1,000 students upload the same textbook, every chunk hash already exists after the first upload, so only a reference is stored and no new bytes are written.
- Storage is roughly 1 copy instead of 1,000. Uploads after the first can also skip sending bytes, which saves bandwidth.
- Even inside one file, repeated chunks are stored once (that's the 33.3% here).

## 4. Build the submission (Word file)

For **every** question (1–15), add:
1. Heading: `Question N`
2. **Code pasted as text** (not as an image). Use a monospace font like Consolas.
3. **One-line answer**, e.g. *"Objects in bucket = 4"*

**Screenshots of the output are also required for Q2, 4, 7, 9, 11, 12, 13, 14, 15** (📸 above). Use `Win+Shift+S` and select the cell plus its output.

For Q15(b), write the paragraph instead of a one-line answer.

Save as `BSDSF23M031_<YourName>_ObjectStorage.docx` (or export to PDF) and submit.

## Quick fixes

| Problem | Fix |
|---|---|
| `Could not connect to the endpoint URL` | Terminal 1 isn't running `moto_server -p 5000` |
| `BucketAlreadyOwnedByYou` on Q1 | You ran Q1 twice. That's fine, or restart the server for clean output |
| `FileNotFoundError: sample_files/...` | Notebook isn't in the lab folder. Use `os.chdir(r"C:\Users\mrayy\OneDrive\Desktop\bsdsf23m031\Object_Storage_Buckets_and_Objects")` |
| Q11 shows 3 versions / Q12 counts wrong | Cells were re-run out of order. Restart the server and run from Q1 |
| `moto_server` not recognized | Use `python -m moto.server -p 5000` |
