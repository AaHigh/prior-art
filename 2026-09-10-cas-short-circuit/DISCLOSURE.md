# Content-Addressed Local Short-Circuit for System Downloads

**Public priority date:** 2026-09-10T23:48:40Z (2026-09-10 16:48:40 PDT)  
**Author:** Aaron Michael Hightower  
**Status:** Frozen first public draft. Refinements go in later dated files.  
**Form:** Defensive publication / technical disclosure

---


This document steelmans a simple claim:

> Before a device transfers a file over the network, it should ask whether those exact bytes already exist anywhere it is allowed to look. If they do, the "download" should resolve locally.

The claim is not that a sorted list of hashes is the best database. The claim is that the **lookup-before-transfer** primitive is sound, that current clients still miss it (including repeat Chrome downloads), and that Apple can expose it below applications so even unmodified browsers benefit.

---

## 1. Hypothesis

Let every persisted file that the system is allowed to index carry a 256-bit cryptographic digest of its contents, for example SHA-256.

Let \( H \) be the set of those digests on the device.

When a download of payload \( x \) is requested, and a digest \( h(x) \) is known or can be obtained cheaper than transferring \( x \):

- if \( h(x) \in H \), do not transfer \( x \)
- materialize the new name or destination with a copy-on-write clone of the existing extents
- verify later only if the source of \( h(x) \) was untrusted

Even a naive index is enough to prove the point:

- store the hashes sorted
- binary-search before each download
- one million SHA-256 values occupy 32 MiB
- a lookup is about twenty comparisons

That is a lookup, not a rewrite of the operating system. It is a pre-transfer existence check.

The hypothesis does **not** say this eliminates first-time uploads to iCloud, or that two nearby phones reduce unique bytes that must reach Apple. It says **repeat acquisition of the same bytes is waste**.

---

## 2. What is already true, and why the problem remains

Browsers and operating systems already cache by **URL**, **ETag**, and **Cache-Control**. That is not the same as caching by **content**.

| Key | What it answers | Failure mode |
|---|---|---|
| URL | Have I fetched this address before? | Same bytes, different URL: miss |
| ETag / validator | Has this URL changed? | New host, same ISO: miss |
| Filename | Is there a file with this name? | Collision, rename, " (1).pdf" |
| Content digest | Do I already have these bytes? | Only works if the digest is known early |

That is why downloading the same installer twice from Google Chrome still often writes a second copy. Chrome is not asking the device, "do you already hold this exact payload?" It is asking, "is this URL in my HTTP cache?" Different question.

The steelmanned idea is to put a **content-addressed store** under the download path, not to make Chrome smarter by itself.

Prior art exists and should be named honestly:

- rsync and content-defined chunking
- Git objects
- IPFS CIDs and magnet infohashes
- Subresource Integrity (SRI)
- HTTP `Content-Digest` / `Repr-Digest` (RFC 9530)
- APFS clones and copy-on-write
- Windows Delivery Optimization and BranchCache
- Dropbox LAN sync (historical)
- Photos duplicate detection and iCloud exact-duplicate merge

The gap is not the hash function. The gap is that **consumer download APIs are still location-addressed**, and the OS does not offer applications a default "resolve by digest, clone if present" path.

---

## 3. Steelman

### 3.1 The primitive

Treat local durable storage as a content-addressable layer:

```
digest -> {
  inode or file-id,
  size,
  first-seen time,
  allowed-use policy,
  still-resident original? (yes/no)
}
```

On inbound transfer:

1. Obtain or compute a candidate digest \( d \).
2. Query the index.
3. On hit, clone. On miss, transfer, then hash-on-write and insert.

On iOS and macOS, "clone" should mean an APFS clone, not a full copy. The user sees a file in Downloads. The disk holds one set of extents until one copy is mutated.

### 3.2 The chicken-and-egg problem (must be stated)

A local hash index cannot skip a transfer unless **someone supplies the digest before the body**.

If the only way to learn \( h(x) \) is to read \( x \) off the wire, the download already happened. The steelman therefore splits into three strengths:

**A. Same URL, same bytes (weak, already partly solved)**  
HTTP cache and "resume" logic. Still fails for "Save As" and "Download again."

**B. Digest known before body (strong)**  
Server or catalog sends `Content-Digest: sha-256=:...:`.  
App Store package hashes.  
iCloud blob identifiers.  
Magnet / CID style links.  
A signed system catalog of popular objects.

**C. Digest discovered after first acquisition (medium, still useful)**  
Hash-on-write every completed download.  
The **second** request for those bytes, from any URL, can short-circuit.

Chrome repeating a download of a file already in Downloads is case A plus C. It does not require Chrome to invent a protocol. It requires the OS download manager to consult a content index and the Downloads folder as first-class members of that index.

### 3.3 Why a sorted list is an acceptable proof, not the product

A sorted array of 256-bit hashes is enough to show:

- existence checks are cheap compared with radio transfer
- memory cost is linear and small
- correctness does not depend on AI, peers, or iCloud

A production index would use a hash map or LSM tree, a Bloom filter in front, and incremental hash-on-write. That is engineering. It does not change the hypothesis.

### 3.4 What "anywhere on the device" must mean

Not a scan of every sandbox on every download. That is slow and not permitted.

It means:

- files the user already downloaded
- files in locations the user authorized
- blobs the system itself stored (iCloud originals still resident, Mail attachments the user saved, Files app, the shared Downloads container)
- optionally, same-account peer devices that advertise have-lists

Photos "Optimize iPhone Storage" is the important negative case. A local derivative does **not** match the digest of the original. The index must record whether the **original bytes** are resident. A preview hit must not suppress an original fetch.

---

## 4. Proposed OS interface (apps need not opt in)

The point of putting this in the operating system is that Chrome, Safari, Mail, Podcasts, and a game updater can all benefit from the same path.

### 4.1 Transparent path (no app change)

Intercept at the system URL loader / download manager (`URLSession` downloads, `WKDownload`, the Files download pipeline):

1. If the response headers include a content digest, query the system store before opening a long body read.
2. If the destination name and size match a hashed local file and the user is repeating a download, offer or auto-complete as a clone.
3. On completion of any download, hash-on-write into the store.

On iOS, third-party browsers use WebKit. A CFNetwork / WebKit-level short-circuit can improve Chrome on iPhone **without a Chrome patch**, provided the digest is present or the file is a repeat of a completed download already indexed.

Desktop Chrome is different. It has its own network stack. The OS can still help when Chrome saves into the user Downloads directory if the save path goes through a file coordinator that consults the store. That is weaker than a kernel- or NSURL-level hook, but it still catches "I already have this installer."

### 4.2 Explicit path (apps that want to do it well)

```
CASLookup(digest) -> Miss | Hit(fileID)
CASClone(fileID, destinationURL) -> File
CASRegister(fileURL) -> digest
CASAdvertise?(digest) -> policy-controlled peer announce
```

An app that already knows a SHA-256 (CDN manifest, torrent, iCloud asset record) calls `CASLookup` first. If the OS has the bytes, the app never opens a socket.

### 4.3 Minimum viable behavior for "Chrome downloaded it twice"

No new protocol required:

1. Hash every file that lands in Downloads.
2. On a new download, if headers give digest, look it up.
3. Else if the suggested filename plus size plus last-modified match an indexed file, compare digest after a short probe or prompt.
4. If equal, clone into `file (2)` or replace the in-progress transfer with the existing object.

That single policy would have saved every user who has two copies of the same PDF sitting in Downloads.

---

## 5. Operations-research trajectory

This section is a suggested research and product path, not a claim that a sorted list is optimal.

### 5.1 Decision problem

At request time the system has:

- optional digest \( d \)
- URL \( u \), size \( s \) if known
- network class (Wi-Fi, cellular, hotspot, peer)
- battery and thermal state
- local index \( H \)
- optional nearby same-account peers with have-lists \( H_p \)

Choose an action:

- \( A_0 \): local clone
- \( A_1 \): fetch from nearby peer
- \( A_2 \): fetch from WAN
- \( A_3 \): fetch from WAN and verify against \( d \)
- \( A_4 \): do nothing / ask the user

Expected cost:

\[
C = \alpha B_{\text{wan}} + \beta E_{\text{radio}} + \gamma L + \delta R_{\text{privacy}} + \varepsilon P_{\text{integrity}}
\]

where \( B \) is metered bytes, \( E \) is radio energy, \( L \) is latency, \( R \) is information leaked by hash queries, and \( P \) is probability of serving the wrong bytes.

The steelman policy is obvious when \( d \in H \) and originals are resident: choose \( A_0 \). Everything else is a tradeoff. That is the OR problem Apple would actually ship: **when to query peers, when to trust a server digest, when hashing is worth the flash reads**.

### 5.2 Index cost model

Let \( n \) be indexed objects.

| Design | Lookup | Memory | Notes |
|---|---|---|---|
| Sorted array of SHA-256 | \( O(\log n) \) | \( 32n \) bytes | Enough to prove the hypothesis |
| Hash map | amortized \( O(1) \) | higher | Production default |
| Bloom filter + confirm | \( O(1) \) with FPs | a few bits each | Avoids disk on misses |
| Chunk index (CDC) | higher | higher | Needed for partial hits, not for the base claim |

Hash-on-write of a completed 10 MB file is cheap next to the cellular transfer of that file. The incremental maintenance cost is not the obstacle. Policy and security are.

### 5.3 Phased Apple trajectory

**Phase 0. Downloads container only.**  
Hash-on-write for the user Downloads and Files "Recents" set. Repeat download of the same bytes becomes a clone. Ships the Chrome example.

**Phase 1. Digest-aware URLSession.**  
Honor `Content-Digest` / `Repr-Digest`. If the digest is in the store, complete the download from disk. Verify if the digest is not authenticated.

**Phase 2. System Content Store API.**  
One index used by Safari, Mail, iCloud Drive local cache, and App Store. Apps may register files they own.

**Phase 3. Original-resident bit for iCloud Photos and iCloud Drive.**  
Do not treat optimized derivatives as hits for original fetches. Do treat a still-resident original as a hit for a second device or a second app.

**Phase 4. Same-account have-list over AWDL.**  
A nearby signed-in device may answer "I hold digest \( d \)" and serve the bytes locally. This is the proximity design. It saves **repeat downloads**, not unique first uploads.

**Phase 5. iCloud blob store becomes content-addressed end to end.**  
Once a digest exists in the account, any device with originals evicted can pull from a sibling device or from iCloud without a second logical object. Checksums become the cross-device name of the bytes.

Phase 0 and Phase 1 do not require Chrome's cooperation on iOS. Phase 4 is the earlier two-iPhone discussion, now stated correctly: peers help when the bytes already exist somewhere in the account's possession.

---

## 6. Security and policy (not optional)

A content index is also a possession oracle.

If any process may ask "do you have SHA-256 \( d \)?", a hostile page can test whether the user holds a known confidential file, a leaked document, or contraband. That is the main reason this must live in the OS with a policy kernel, not in every app.

Required controls:

- **No raw cross-app hash probes from the web.** A page in Chrome must not enumerate \( H \).
- **Digest from the network is untrusted until verified.** A server can lie about `Content-Digest`. Clone on header, then either trust a signed catalog or hash the clone and compare. Better: only short-circuit on authenticated digests (App Store, iCloud, signed CDN).
- **Scope.** Default index is Downloads, user-authorized folders, and system blobs. Not Messages attachments, not photos marked hidden, not other apps' containers.
- **Peer have-lists.** Advertise only to devices signed into the same Apple Account, over an authenticated local channel. Do not broadcast the user's hash list on the LAN.
- **Deletion.** Removing a file must remove or refcount the digest. A clone is not a second independent secret once extents are shared; deletion policy must follow APFS refcounts.
- **Legal process.** A hash index is metadata. Treat it as sensitive.

None of these objections kill the hypothesis. They constrain the interface.

---

## 7. Claims and non-claims

**Claimed**

1. Existence of exact bytes on a device is information the download path should use.
2. A 256-bit digest plus a cheap index is sufficient to make that decision when the digest is known early, or on the second acquisition after hash-on-write.
3. The operating system is the right layer, because then applications that do nothing special still benefit.
4. Repeat Chrome downloads of a file already on disk are evidence that current stacks are location-addressed, not content-addressed.
5. Nearby same-account devices can extend the same index. That is a download-side win.

**Not claimed**

1. That a linear or sorted list is the production database.
2. That hashing every sandbox on every request is low-overhead.
3. That two nearby iPhones reduce unique bytes that must be uploaded to iCloud the first time.
4. That a local derivative (optimized photo) should suppress an original download.
5. That this document is a filed patent. It is a dated public technical disclosure.

---

## 8. One-sentence proof

If the cost of asking "do I already have digest \( d \)?" is small next to the cost of moving the file, and if \( d \) can be known without reading the whole body, then skipping the transfer is a strict improvement for that request.

Current download UIs fail that test for the ordinary case of "I already downloaded this." An OS-level content store is how you fix it once, for every app that writes a file.

---

## 9. Suggested GitHub layout

```
content-addressed-download-short-circuit/
  README.md          (this file, or a short pointer)
  DISCLOSURE.md      (this document)
  LICENSE            (choose one; CC BY 4.0 or MIT for text)
```

If this is published to establish a public date, leave the text unchanged after the first push except for typographical fixes, and record the commit hash and date in the repo description.

---

## 10. License note

Copyright 2026 Aaron Michael Hightower.  
This disclosure is published so the idea is citable. Reuse of the text should preserve authorship. Implementation of the idea in software is not restricted by this file.
