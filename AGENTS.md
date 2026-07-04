# Buddy Backup — AGENTS.md

> **Project scope:** This document covers the initial buddy-to-buddy backup mode.
> Future marketplace/credit system is mentioned briefly for architectural awareness
> but not detailed. The file will grow as the project evolves.

---

## Project Overview

Buddy Backup is a peer-to-peer encrypted backup system for home server users.
Two friends run the Docker container, pair via a buddy code, and back up
their data to each other's machines — fully encrypted, zero knowledge on the
receiving end.

**Long-term vision:** An optional cloud mode would provide a credit marketplace
where users can buy/sell backup storage in a decentralized network. This doc is
focused on getting the buddy-to-buddy mode working first.

**Team:** Carlos Fernandez Lockward (@carloslockward) + Adán Estévez (@rafacovez)

---

## Architecture

### Two-Port Container

The container exposes two separate ports. Two Flask apps (or two Waitress
instances) in the same process, sharing one `BackupServer` instance:

```
:8080  WebUI only — local/VPN access
:8081  Peer API only — public via reverse proxy or tunnel
```

**Port 8080 (WebUI)** serves:
- `/wizard/*` — setup flow
- `/dashboard` — status dashboard
- `/settings` — configuration
- `/login` — authentication
- `/static/*` — CSS, JS assets

**Port 8081 (Peer API)** serves **only**:
- `/api/v1/peer/*` — authenticated peer-to-peer endpoints
- `/healthz` — simple health check (no sensitive info)

No WebUI routes exist on port 8081. No peer routes exist on port 8080.

**Docker mapping (recommended):**
```bash
-p 127.0.0.1:8080:8080   # WebUI: localhost only
-p 8081:8081              # Peer API: exposed via reverse proxy
```

### Thread Model

```
Main process (Waitress, two server instances)
  ├── Thread pool A (port 8080) — Flask WebUI
  ├── Thread pool B (port 8081) — Flask Peer API
  └── Background thread — BackupServer.run() loop
        • Scheduler
        • Heartbeats
        • Sync jobs
        • Proof-of-storage
```

Both Flask apps receive a reference to the same `BackupServer` instance.
Communication is via direct method calls, no IPC overhead.

### Exposed via User's Reverse Proxy

The user exposes port 8081 to the internet via their own reverse proxy or tunnel
(Pangolin, Cloudflare Tunnel, Nginx Proxy Manager, etc.). The container does
not listen on privileged ports or handle TLS itself. Port 8080 is bound to
localhost only.

---

## Tools & Libraries

| Layer | Library | Why |
|---|---|---|
| **Web framework** | Flask + Flask-WTF | Carlos knows it well. Template rendering + forms. |
| **WSGI server** | Waitress | Thread-based (shares memory with engine). |
| **Database** | SQLite (via sqlite3 stdlib) | Runs in-process, no external DB needed. |
| **Validation** | Pydantic v2 | Config and model validation. |
| **Crypto** | PyNaCl (libsodium bindings) | X25519, XChaCha20-Poly1305, secretbox. |
| **Chunk hashing** | hashlib (stdlib) | SHA-256 for plaintext chunk identity. |
| **Content-defined chunking** | Custom (Buzhash) | Variable-size chunks for efficient delta sync. |
| **Key derivation** | pyca/cryptography HKDF | Or PyNaCl's key derivation. |
| **Templates** | Jinja2 (Flask default) | Server-rendered HTML. |
| **Background tasks** | threading (stdlib) + queue.Queue | Simple, no Celery/RQ needed. |
| **Password hashing** | argon2-cffi | Argon2id for webui passwords. |

---

## Security Architecture

### Key Separation

Three distinct key domains, never mixed:

| Key | Size | Purpose | Shared with buddy? |
|---|---|---|---|
| **node private key** | 32 bytes (X25519) | Identity, peer auth handshake | No |
| **peer shared secret** | 32 bytes | Derived per-buddy via ECDH, used for HMAC request signing | Derived independently (same value) |
| **backup root key** | 32 bytes (random) | Owner-only. Encrypts chunks, indexes, generates opaque chunk IDs | **Never** |

### Key Exchange

- Each node generates an **X25519 keypair** on first boot (via PyNaCl).
- **Private key** stored at rest as encrypted blob inside `/data/keys`.
- **Public key** is the node's identity. Base58-encoded for buddy codes.
- On pairing, each node computes:
  `peer_shared_secret = X25519(my_private_key, buddy_public_key)`
  Both sides arrive at the same value independently (standard ECDH).
- The peer shared secret is used **only** for HMAC request authentication.
- A separate **backup root key** (random 32 bytes) is generated on first boot
  and stored locally. The buddy never receives this key.

### Buddy Code Format

```
buddy://B6v3X8kPqR2mZXy...44chars@buddy-a.pangolin.example
        ↑ base58(X25519 pubkey)     ↑ user's public URL
```

Single pastable string. The wizard parses it on import.

### First-Run Setup Token

Even though the WebUI is local-only, a setup token prevents accidental
exposure if the user binds port 8080 publicly.

On first boot:
1. Generate `/data/setup-token` (16 random hex chars, formatted as `XXXX-XXXX-XXXX`)
2. Print token in container logs
3. Require `?token=<token>` on all `/wizard/*` routes
4. Delete/invalidate token after setup completes

Example:
```
First-run setup token: 7K4D-9Q2M-W8RA
Open: http://localhost:8080/wizard?token=7K4D-9Q2M-W8RA
```

### Out-of-Band Trust

Public keys (buddy codes) are exchanged manually by the users (Telegram, text,
email). **They are never transmitted over the network** between peers during
pairing. This means:

- Each side pre-registers the other's public key before any direct communication.
- A handshake request that doesn't match a pre-registered pubkey is rejected
  immediately — no information leaked to attackers.

### Canonical HMAC Request Authentication

Every public peer API request must include these headers:

```
X-Buddy-Id:        <peer id — first 16 hex chars of sha256(pubkey)>
X-Buddy-Timestamp: <unix epoch seconds>
X-Buddy-Nonce:     <random 128-bit value, hex-encoded>
X-Buddy-Signature: <HMAC-SHA256 of canonical string>
```

Canonical string format:

```
BUDDYBACKUP-HMAC-V1
method:POST
path:/api/v1/peer/chunks
query:
timestamp:1730000000
nonce:abc123def456...
content-type:application/octet-stream
body-sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

Verification:
1. Known peer ID (look up by X-Buddy-Id from paired list)
2. Timestamp within ±300 seconds (replay protection)
3. Nonce not reused within replay window (maintain a small LRU cache)
4. Signature valid — use `hmac.compare_digest()` (constant-time)
5. All fail → `401 Unauthorized` with no additional information

**All** `/api/v1/peer/*` endpoints require valid authentication.
`/healthz` returns minimal info: `{"status": "ok"}` only.

### Data Encryption (End-to-End)

All backup data is encrypted before transmission, independent of TLS. Keys are
derived from the owner-only **backup root key**:

```
backup_root_key = random 32 bytes (never leaves owner's node)

chunk_key  = HKDF(backup_root_key, "buddybackup chunk v1")
index_key  = HKDF(backup_root_key, "buddybackup index v1")
chunk_id_key = HKDF(backup_root_key, "buddybackup chunk id v1")
```

Encryption uses **XChaCha20-Poly1305** (via PyNaCl/libsodium):

```
For each chunk:
  nonce = random 24 bytes
  ciphertext = XChaCha20-Poly1305(chunk_key, nonce, plaintext, aad)

For each index/snapshot:
  index_nonce = random 24 bytes
  encrypted_index = XChaCha20-Poly1305(index_key, index_nonce, index_json, aad)
```

AAD (Authenticated Additional Data):

```
aad = "buddybackup chunk v1" || chunk_id || plaintext_size
```

The storing buddy gets opaque bytes. Even with full disk access, the data is
unreadable. Zero knowledge.

### Opaque Chunk IDs

Do not expose plaintext hashes to the buddy:

```
plaintext_hash = SHA256(plaintext_chunk)
chunk_id = HMAC-SHA256(chunk_id_key, plaintext_hash)
```

The remote buddy stores chunks indexed by `chunk_id` only. Only the owner
knows the mapping from plaintext hash to chunk_id. This prevents the buddy
from identifying duplicate content across different owners.

### Unattended Key Storage (V1 Tradeoff)

For V1, if backups should resume after container restart, the service needs
access to keys at boot.

V1 approach:
1. On first boot, generate a `secret_bundle` containing:
   - node private key
   - backup root key
2. Encrypt the bundle at rest with an auto-generated key
3. Store auto-unlock material in `/data/keys/` with strict file permissions

Documented honestly:

> If an attacker gains access to the Buddy Backup data volume, they may recover
> these keys. The storage buddy still cannot decrypt your backups — they would
> need to compromise your local node or data volume.
>
> A future optional mode will require manual password unlock after restart.

### WebUI Security

Even though the WebUI is local-only:
- Password login required (Argon2id hashing)
- CSRF protection (Flask-WTF or custom)
- Persistent Flask secret key in `/data/secret_key`
- Session cookies: `HttpOnly`, `SameSite=Lax`, `Secure` when HTTPS
- No WebUI routes on the public peer API port

---

## Backup Engine: Content-Defined Chunking

### How It Works

Files are split into variable-size chunks using **Buzhash** (rolling hash):

```
File: [---chunk1---][----chunk2----][--chunk3--]...
         ^           ^                ^
      boundary     boundary         boundary
      (hash mod    (hash mod        (hash mod
       N == 0)      N == 0)          N == 0)
```

- Chunk boundaries are determined by content, not position.
- Average chunk size configurable (default: 64 KB, min: 16 KB, max: 256 KB).
- Inserting a byte shifts only 1–2 boundaries → very efficient delta sync.

### Local Chunk Index (Owner Side)

```
SQLite table: chunks
  plaintext_hash: TEXT PRIMARY KEY (SHA-256 of plaintext)
  chunk_id:       TEXT (HMAC'd opaque ID sent to buddy)
  size:           INTEGER
  refs:           INTEGER (how many files reference this chunk)

SQLite table: files
  path:  TEXT PRIMARY KEY
  mtime: INTEGER
  size:  INTEGER

SQLite table: file_chunks
  file_path:      TEXT → files.path
  plaintext_hash: TEXT → chunks.plaintext_hash
  chunk_index:    INTEGER (ordering)
```

The index is local to the owner. The buddy stores only opaque chunks
indexed by `chunk_id` — no filenames, no structure.

### Encrypted Chunk Format (On-Disk at Buddy)

Each stored chunk has explicit metadata:

| Field | Size | Description |
|---|---|---|
| version | 1 byte | Format version (0x01) |
| chunk_id | 32 bytes | Opaque HMAC-based ID |
| nonce | 24 bytes | XChaCha20-Poly1305 nonce |
| ciphertext | variable | Encrypted chunk data |
| plaintext_size | 4 bytes | Original plaintext length (big-endian) |

AAD for encryption:
```
aad = "buddybackup chunk v1" || chunk_id || plaintext_size
```

### Sync Flow

```
1. Walk backup folders, stat each file
2. For changed files: re-chunk, compute plaintext hashes
3. Diff: which plaintext hashes are new?
4. HMAC plaintext_hash → chunk_id
5. Encrypt each new chunk (XChaCha20-Poly1305 with chunk_key)
6. Send framed chunks to buddy's peer API
7. Create versioned encrypted snapshot of file index
8. Upload snapshot to buddy
```

### Restore Flow

```
1. Fetch latest encrypted snapshot from buddy
2. Decrypt locally with index_key
3. For each file, look up chunk_ids
4. Request chunk_ids from buddy (batch endpoint)
5. Buddy returns framed encrypted blobs
6. Decrypt locally with chunk_key
7. Reassemble file
```

---

## Peer API Endpoints

All endpoints require canonical HMAC authentication (see Security section).
All IDs (chunk_id, snapshot_id) are strictly validated — fixed-length hex
or base64url only. Never used as raw filesystem paths.

### Framed Batch Format

Batch upload and download use a **framed binary format** (not comma-separated
headers with concatenated bodies). V1 uses a simple length-prefixed frame:

Each frame:

| Field | Size | Description |
|---|---|---|
| chunk_id | 32 bytes | Opaque HMAC-based ID |
| nonce | 24 bytes | XChaCha20-Poly1305 nonce |
| ciphertext_length | 4 bytes | Big-endian uint32 |
| ciphertext | ciphertext_length bytes | Encrypted data |
| plaintext_size | 4 bytes | Big-endian uint32 |

Frames are concatenated in the request/response body. No separator needed
since each frame is self-delimiting.

---

### Handshake (Pairing Completion)

```
POST /api/v1/peer/handshake
Body:
{
  "url": "https://buddy-a.pangolin.example",
  "display_name": "Alice"
}

Response 200:
{
  "status": "paired",
  "your_buddy_code": "buddy://B6v3...@buddy-b.example"
}
```

Pre-requisite: both users have already entered each other's buddy codes into
their respective wizards. The handshake proves the connection works and
exchanges display names/URLs.

### Health / Heartbeat

```
GET /api/v1/peer/health

Response 200:
{
  "status": "ok",
  "version": "0.1.0",
  "uptime_seconds": 12345,
  "disk_allocated_bytes": 536870912000,
  "disk_used_bytes": 123456789000
}
```

Called periodically (every 5 minutes) by the background thread to verify the
buddy is reachable and has capacity.

### Push Chunks (Framed)

```
POST /api/v1/peer/chunks
Content-Type: application/octet-stream
Body: [frame][frame][frame]...

Response 200:
{
  "stored": <count>,
  "errors": []
}

Response 413: quota exceeded
Response 422: invalid frame
```

### Push Snapshot (Versioned Index)

```
POST /api/v1/peer/snapshots
Content-Type: application/json
Body:
{
  "snapshot_id": "<unique id>",
  "created_at": "<ISO 8601>",
  "encrypted_index_nonce": "<24 bytes hex>",
  "encrypted_index_blob": "<base64>"
}

Response 200:
{
  "stored": true,
  "snapshot_id": "<same id>"
}
```

Snapshots are versioned — do not overwrite. Keep at least the last 7.

### Get Latest Snapshot

```
GET /api/v1/peer/snapshots/latest

Response 200:
{
  "snapshot_id": "...",
  "created_at": "...",
  "encrypted_index_nonce": "...",
  "encrypted_index_blob": "..."
}
```

### Get Snapshot by ID

```
GET /api/v1/peer/snapshots/<snapshot_id>

Response 200:
{
  "snapshot_id": "...",
  "created_at": "...",
  "encrypted_index_nonce": "...",
  "encrypted_index_blob": "..."
}

Response 404: not found
```

### Request Single Chunk (Restore)

```
GET /api/v1/peer/chunks/<chunk_id>

Response 200:
Content-Type: application/octet-stream
Body: <binary frame>

Response 404: chunk not found
```

### Batch Request Chunks (Restore)

```
POST /api/v1/peer/chunks/batch
Body:
{
  "chunk_ids": ["<id1>", "<id2>", ...]
}

Response 200:
Content-Type: application/octet-stream
Body: [frame][frame][frame]...  (in same order as requested)

Response 422: invalid chunk_id in request
```

### Proof of Storage

```
GET /api/v1/peer/proof/<chunk_id>

Response 200:
{
  "chunk_id": "...",
  "exists": true,
  "stored_bytes": 65536
}
```

Or with byte-range challenge:

```
POST /api/v1/peer/proof
Body:
{
  "challenges": [
    {"chunk_id": "<id>", "offset": 0, "length": 32}
  ]
}

Response 200:
{
  "responses": [
    {"chunk_id": "<id>", "data": "<base64 of requested bytes>"}
  ]
}
```

Periodic proof-of-storage ensures the buddy is actually storing your data and
hasn't silently deleted or corrupted it.

---

## Quotas and DoS Protections

The public peer API must enforce:

| Limit | Default | Notes |
|---|---|---|
| Max request body | 100 MB | Reject oversized requests early |
| Max chunk size | 1 MB | Single encrypted chunk cap |
| Max chunks per batch | 1000 | Per upload or download batch |
| Per-peer storage quota | Configurable | Default: 500 GB |
| Min free disk reserve | 5% | Reject writes below threshold |
| Request timeout | 60 s | Per-request timeout |

Storage writes must be atomic:
1. Write to a temporary file in the same directory
2. `os.fsync()` if practical (data integrity)
3. `os.rename()` to final path (atomic on Linux)

All IDs are strictly validated: `chunk_id` and `snapshot_id` must match
`^[a-f0-9]{64}$` (lowercase hex, 64 chars) or similar fixed-length pattern.
Never use raw user input as a filesystem path.

---

## WebUI: Setup Wizard

### Flow

| Step | Route | Purpose |
|---|---|---|
| 1 | `/wizard?token=<token>` | First-run check. Token required. If configured → redirect to `/dashboard` |
| 2 | `/wizard/password` | Set webui password (Argon2id) |
| 3 | `/wizard/network` | Enter your public URL |
| 4 | `/wizard/pairing` | Show your buddy code. Option to enter a buddy's code |
| 5 | `/wizard/folders` | Multi-select folders to back up |
| 6 | `/wizard/space` | Slider: allocate X GB for buddy's data |
| 7 | `/wizard/summary` | Review → "Start backing up!" → `/dashboard` |

### Dashboard

```
/dashboard
  - Buddy status (online/offline, uptime %)
  - Next scheduled sync
  - Last sync time + size
  - Backup size per folder
  - Allocated vs used buddy storage
  - "Sync Now" button
  - Recent activity log
```

---

## Sync Scheduler

Default: interval-based sync. No live/watch mode for v1.

- Configurable schedule (cron expression or presets: every 6h, daily, weekly).
- Default: daily at 3:00 AM (container local time — UTC recommended).
- Background thread checks schedule every 60 seconds.
- Manual "Sync Now" button on dashboard bypasses schedule.
- Concurrent syncs: one at a time. Queued if already running.

Future (v1.1+): inotify-based watch mode for near-live sync on selected folders.

---

## Data Flow Summary

```
User A                          Buddy B
  │                                │
  │ 1. Changed files detected      │
  │ 2. Re-chunk, compute hashes    │
  │ 3. HMAC hash → opaque chunk_id │
  │ 4. Encrypt chunks (XChaCha20)  │
  │ 5. POST framed chunks ────────► 6. Store by chunk_id
  │ 7. POST versioned snapshot ───► 8. Store encrypted index
  │                                │
  │     (later, restore)           │
  │ 9. GET /snapshots/latest ◄────── 10. Return encrypted index
  │11. Decrypt snapshot, list files │
  │12. POST /chunks/batch ◄──────── 13. Return framed chunks
  │14. Decrypt, reassemble files   │
```

Buddy B never sees plaintext. Buddy B never knows chunk_ids that weren't
specifically requested. Buddy B has no access to backup_root_key.

---

## Future: Marketplace / Credits (Briefly)

Long-term, the architecture supports:

- **Coordination server** (central, run by us): node registry, buddy matching,
  credit ledger, Stripe integration.
- **Credit economy**: storage providers earn credits by hosting others' data
  (weighted by size + uptime + bandwidth). Consumers buy credits to store
  beyond what they give.
- **Credit/cash conversion**: buy credits, cash out with small fee (the spread
  is revenue).

This AGENTS.md will be extended when that phase begins. For now, the
peer-to-peer mode is the entire product.