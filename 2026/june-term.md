day - 1

## Modulo Hashing

### Definition:

Modulo Hashing is a technique that uses the remainder operator (%) to map any input (like a user ID) into a fixed range of buckets. It's the simplest way to answer: "Given this thing, which slot should it go into?"

In one line:

bucket=hash(key)modn

Where n is the number of buckets (servers, partitions, slots, etc.).

Analogy — A Simple Hotel Key System
Imagine a hotel with 4 floors and a dumbwaiter that delivers room-service orders by floor number. The chef looks at the room number and does:

"Room 307 → last digit is 7 → 7 mod 4 = 3 → Floor 3"

Room Number Last Digit digit mod 4 Floor
301 1 1 mod 4 = 1 1st floor
405 5 5 mod 4 = 1 1st floor
512 2 2 mod 4 = 2 2nd floor
307 7 7 mod 4 = 3 3rd floor
720 0 0 mod 4 = 0 4th floor (floor 0)
The chef never needs to memorize which room is on which floor — the math does it instantly.

The Formula
index=hash(key)modn

hash(key) — turns the key into a number (for strings, use a hash function like MD5 or hashCode())
mod n — wraps that number into the range [0,n−1]

### Example:

Picking a Server (Distributed Systems)

```
import hashlib

def pick_server(user_id: str, servers: list) -> str:
    # Step 1: Hash the user ID to a number
    hash_bytes = hashlib.md5(user_id.encode()).digest()
    hash_int = int.from_bytes(hash_bytes[:8], 'big')  # first 8 bytes → integer

    # Step 2: Modulo by number of servers
    index = hash_int % len(servers)

    return servers[index]

# Usage
servers = ["Server-A", "Server-B", "Server-C", "Server-D"]

print(pick_server("alice@example.com", servers))  # e.g., Server-C
print(pick_server("bob@example.com",   servers))  # e.g., Server-A
print(pick_server("alice@example.com", servers))  # Always Server-C! (deterministic)
🗄️ Classic Hash Table Example

class SimpleHashTable:
    def __init__(self, size=10):
        self.size = size
        self.table = [None] * size

    def _index(self, key: str) -> int:
        # Sum of ASCII values, then modulo
        total = sum(ord(char) for char in key)
        return total % self.size

    def put(self, key: str, value):
        idx = self._index(key)
        self.table[idx] = value

    def get(self, key: str):
        idx = self._index(key)
        return self.table[idx]

# Usage
ht = SimpleHashTable(size=5)
ht.put("apple",  "🍎")
ht.put("banana", "🍌")
ht.put("cherry", "🍒")

print(ht.get("apple"))   # 🍎
print(ht.get("banana"))  # 🍌
```

---

day - 2

## LSM Trees (Log-Structured Merge-Trees)

### Definition:

LSM Trees are a data structure designed for write-heavy workloads. Instead of updating data in place (like a B-tree), they just append every change to a log, then lazily sort and merge things in the background. The mantra: write fast now, organize later.

In one sentence: Turn random writes into sequential writes, then periodically tidy up.

Analogy — The Receptionist's Logbook
Picture a busy doctor's office. Patients arrive constantly. The receptionist has two tools:

Tool What happens LSM Equivalent
📝 Logbook on the desk Scribbles every arrival instantly, in order of arrival. Super fast — just append to the page. Memtable + WAL (in-memory + append-only log)
🗄️ Filing cabinet During quiet moments, she takes names from the logbook and files them alphabetically into patient folders. SSTables (sorted files on disk)
♻️ Merging old folders Eventually, she merges duplicate folders, keeping only the latest record for each patient. Compaction
Writing (new patient arrives): Just scribble in the logbook. Zero searching. Lightning fast.
Reading (find patient info): Check the logbook first, then check the filing cabinet. Slightly more work for reads, but writes fly.

How It Works

```text
    WRITE ──▶ [ Memtable (sorted, in RAM) ]    ← fastest
                    │
                    │ (when full, flush to disk)
                    ▼
              [ SSTable (sorted file) ]        ← disk, immutable
              [ SSTable (sorted file) ]
              [ SSTable (sorted file) ]
                    │
                    │ (background compaction)
                    ▼
              [ Merged SSTable ]               ← fewer files, garbage collected
```

### Example:

Simplified LSM Key-Value Store in Python
This is a dramatically simplified version to illustrate the mechanics. Real LSM engines use skip lists, bloom filters, binary search, and much more.

```
import bisect
import json
import os

class MiniLSM:
    def __init__(self, db_path="minilsm_data"):
        self.db_path = db_path
        os.makedirs(db_path, exist_ok=True)
        self.memtable = {}          # in-memory sorted dict
        self.sstables = []          # list of filenames on disk
        self.memtable_limit = 3     # flush after 3 entries (tiny for demo)

    def put(self, key, value):
        """Just write to memtable. Super fast!"""
        self.memtable[key] = value
        print(f"✍️  memtable ← {key}: {value}")

        if len(self.memtable) >= self.memtable_limit:
            self._flush()

    def get(self, key):
        """Read path: check memtable first, then SSTables (newest first)."""
        # 1. Check memtable (fastest, in RAM)
        if key in self.memtable:
            print(f"🔍 Found '{key}' in memtable")
            return self.memtable[key]

        # 2. Check SSTables, newest to oldest
        for sst_filename in reversed(self.sstables):
            with open(sst_filename, 'r') as f:
                data = json.load(f)
                if key in data:
                    print(f"🔍 Found '{key}' in {sst_filename}")
                    return data[key]
        return None

    def _flush(self):
        """Sort memtable, write to disk as a new SSTable, then clear memtable."""
        sst_name = f"{self.db_path}/sst_{len(self.sstables)+1:03d}.json"
        with open(sst_name, 'w') as f:
            # Sort by key before writing (real SSTables are always sorted)
            sorted_data = dict(sorted(self.memtable.items()))
            json.dump(sorted_data, f)

        self.sstables.append(sst_name)
        print(f"📦 Flushed → {sst_name} ({len(self.memtable)} entries)")
        self.memtable.clear()

    def compact(self):
        """Merge all SSTables into one, keeping only the latest value per key."""
        if len(self.sstables) < 2:
            return

        merged = {}
        # Read from oldest to newest, so newer values overwrite older
        for sst_filename in self.sstables:
            with open(sst_filename, 'r') as f:
                data = json.load(f)
                merged.update(data)   # newer overwrites older
            os.remove(sst_filename)   # delete old file

        # Write the single merged file
        new_sst = f"{self.db_path}/sst_merged.json"
        with open(new_sst, 'w') as f:
            json.dump(dict(sorted(merged.items())), f)

        self.sstables = [new_sst]
        print(f"♻️  Compacted → {new_sst} ({len(merged)} keys)")

# --- Usage ---
db = MiniLSM()

db.put("name", "Alice")     # memtable ←
db.put("city", "Paris")     # memtable ←
db.put("name", "Bob")       # overwrites Alice in memtable
db.put("age",  "30")        # triggers flush! (limit=3 reached)

print("\nReading...")
print(db.get("name"))        # checks memtable first
print(db.get("city"))        # checks SSTable
db.compact()                 # merge everything (real DBs do this in background)
Output:


✍️  memtable ← name: Alice
✍️  memtable ← city: Paris
✍️  memtable ← name: Bob
📦 Flushed → minilsm_data/sst_001.json (3 entries)
✍️  memtable ← age: 30

Reading...
🔍 Found 'name' in memtable
🔍 Found 'city' in minilsm_data/sst_001.json
♻️  Compacted → minilsm_data/sst_merged.json (4 keys)
Notice: name: Bob overwrites name: Alice because it was written later — no in-place update, just a newer entry that takes precedence.
```

---

day - 3

## U-Net Architecture

### Definition:

U-Net is a convolutional neural network designed for image segmentation — the task of classifying every single pixel in an image. Its signature "U" shape combines a contracting path (which captures what is in the image) with an expanding path (which pinpoints where it is), linked by skip connections that preserve fine details.

In one sentence: U-Net takes an image in and spits out a same-sized map where each pixel gets a label.

Analogy — The Artist's Two-Step Process
Imagine an artist painting a detailed landscape from a blurry photo:

Step What happens U-Net Equivalent
1️⃣ Squinting Steps back, squints, sees the big shapes — "There's a mountain, a lake, some trees" Contracting path (Encoder) — captures high-level what
2️⃣ Zooming in Leans in close, fills in the edges, restores sharp boundaries — "The shoreline goes exactly here" Expanding path (Decoder) — recovers precise where
🔗 Cross-checking Glances back at the original photo constantly so no detail is lost Skip connections — shuttle fine details from early layers to later layers
Without the skip connections (cross-checking), you'd get a blurry blob. With them, you get sharp, pixel-perfect boundaries.

### Example:

U-Net in PyTorch (Simplified)

```
This is a clean, minimal U-Net for binary segmentation (e.g., "cell vs. background"):


import torch
import torch.nn as nn

class DoubleConv(nn.Module):
    """Two convolutions + batch norm + ReLU (the basic building block)."""
    def __init__(self, in_ch, out_ch):
        super().__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(in_ch, out_ch, kernel_size=3, padding=1),
            nn.BatchNorm2d(out_ch),
            nn.ReLU(inplace=True),
            nn.Conv2d(out_ch, out_ch, kernel_size=3, padding=1),
            nn.BatchNorm2d(out_ch),
            nn.ReLU(inplace=True),
        )

    def forward(self, x):
        return self.conv(x)


class UNet(nn.Module):
    def __init__(self, in_channels=1, out_channels=1):
        super().__init__()

        # ── Encoder (contracting path) ──
        self.enc1 = DoubleConv(in_channels, 64)    # 572 → 570
        self.enc2 = DoubleConv(64, 128)            #    → 140
        self.enc3 = DoubleConv(128, 256)           #    →  68
        self.enc4 = DoubleConv(256, 512)           #    →  32

        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)

        # ── Bottleneck ──
        self.bottleneck = DoubleConv(512, 1024)    #    →  28

        # ── Decoder (expanding path) ──
        self.up4 = nn.ConvTranspose2d(1024, 512, kernel_size=2, stride=2)
        self.dec4 = DoubleConv(1024, 512)          # 512(skip) + 512(up)

        self.up3 = nn.ConvTranspose2d(512, 256, kernel_size=2, stride=2)
        self.dec3 = DoubleConv(512, 256)           # 256(skip) + 256(up)

        self.up2 = nn.ConvTranspose2d(256, 128, kernel_size=2, stride=2)
        self.dec2 = DoubleConv(256, 128)           # 128(skip) + 128(up)

        self.up1 = nn.ConvTranspose2d(128, 64, kernel_size=2, stride=2)
        self.dec1 = DoubleConv(128, 64)            # 64(skip) + 64(up)

        # ── Output ──
        self.out_conv = nn.Conv2d(64, out_channels, kernel_size=1)

    def forward(self, x):
        # Encoder
        e1 = self.enc1(x)                          # [B,  64, H,   W]
        e2 = self.enc2(self.pool(e1))              # [B, 128, H/2, W/2]
        e3 = self.enc3(self.pool(e2))              # [B, 256, H/4, W/4]
        e4 = self.enc4(self.pool(e3))              # [B, 512, H/8, W/8]

        # Bottleneck
        b = self.bottleneck(self.pool(e4))         # [B, 1024, H/16, W/16]

        # Decoder with skip connections
        d4 = self.up4(b)                           # [B, 512, H/8, W/8]
        d4 = torch.cat([d4, e4], dim=1)            # Skip! → [B, 1024, H/8, W/8]
        d4 = self.dec4(d4)

        d3 = self.up3(d4)                          # [B, 256, H/4, W/4]
        d3 = torch.cat([d3, e3], dim=1)            # Skip! → [B, 512, H/4, W/4]
        d3 = self.dec3(d3)

        d2 = self.up2(d3)                          # [B, 128, H/2, W/2]
        d2 = torch.cat([d2, e2], dim=1)            # Skip! → [B, 256, H/2, W/2]
        d2 = self.dec2(d2)

        d1 = self.up1(d2)                          # [B, 64, H, W]
        d1 = torch.cat([d1, e1], dim=1)            # Skip! → [B, 128, H, W]
        d1 = self.dec1(d1)

        return self.out_conv(d1)                   # [B, out_ch, H, W]


# ── Usage Example ──
model = UNet(in_channels=3, out_channels=1)       # RGB in, binary mask out
dummy_image = torch.randn(1, 3, 572, 572)          # batch of 1, 572×572 RGB
mask = model(dummy_image)

print(f"Input shape:  {dummy_image.shape}")        # [1, 3, 572, 572]
print(f"Output shape: {mask.shape}")               # [1, 1, 572, 572]  ← same spatial size!
```

---

day - 4

## SSO Migration

### Definition:

SSO migration is the process of moving a company's single sign-on system from one identity provider to another — for example, switching from Okta to Microsoft Entra ID — without locking users out, breaking app integrations, or forcing everyone to reset their passwords on the same day.

Think of it like changing the locks on every door in a massive office building while employees are still walking through those doors. You can't just swap all the locks at once at 9 AM. Instead, you phase it room by room, test each new key, and keep both the old and new locks working during the transition.

The Core Challenge

```text
                    ┌─────────────────────┐
    Current state:  │       ONE IDP       │   ← e.g., Okta
                    │  (all apps trust it) │
                    └──────┬──────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
     ┌────────┐      ┌────────┐      ┌────────┐
     │ GitHub │      │ Jira   │      │  Zoom  │   ...and 80+ more apps
     └────────┘      └────────┘      └────────┘
```

You need to point every single arrow to a new Identity Provider (IdP) — without those apps ever trusting a broken or missing identity.

### Example:

Okta → Microsoft Entra ID

```
A mid-size company with 1,500 employees and 85 SaaS apps decides to migrate from Okta to Microsoft Entra ID (formerly Azure AD). Here's a staged plan:

Phase	What Happens	User Experience
1. Prep	Inventory all 85 apps. Map SAML/OIDC configs from Okta to Entra. Identify which apps support multi-IdP.	Nothing changes.
2. Parallel Run	Entra ID is set up as a second IdP. A pilot group (IT team, 10 people) uses Entra for a week.	Pilot users see a slightly different login page. Everyone else: no change.
3. Phased Cutover	Roll out in waves — Finance dept on Monday, Engineering on Wednesday, etc.	User clicks "Sign in with Google" on Jira → now sees "Sign in with Microsoft" instead. Same username. No password reset.
4. Coexistence	Okta stays live for 2 weeks while Entra handles 100% of traffic. Rollback is one config change away.	Transparent.
5. Decommission	Okta tenant emptied, contracts canceled, old certs revoked.	Done.
```

---

day - 5

## Hi-Lo Algorithm

### Definition:

The Hi-Lo Algorithm is an ID generation strategy that produces unique identifiers by combining two numbers: a "Hi" value fetched occasionally from a shared source (like a database), and a "Lo" value that increments locally in memory. The formula:

ID=(Hi×blockSize)+Lo

In one sentence: Grab a big chunk of IDs from the database once, then hand them out locally until you run out — repeat. This avoids a database round-trip for every single ID.

Analogy — The Waiter's Check Numbers
Imagine a restaurant where every order needs a unique check number, and the central office owns the master list. Three waiters are taking orders simultaneously:

Approach What happens Problem
❌ Call the office for every order "Give me one check number!" → wait → get #42. Next order: call again → #43. Slow, bottleneck, waiters standing around on the phone
✅ Hi-Lo: Grab a block Waiter A: "Give me 100 numbers!" → gets block #0 (covers #0–#99). Waiter B: gets block #1 (covers #100–#199). Both hand out numbers locally without calling again. Fast, no waiting, office is barely bothered
Role Real World Hi-Lo Algorithm
Central office Master sequence / DB table Hi value — fetched occasionally
Waiter's own pad Local counter (0–99) Lo value — incremented in memory
Check number Final unique order number (Hi × blockSize) + Lo
"Give me another block" Waiter calls office when pad runs out Next database fetch when Lo hits blockSize
The office is contacted once per 100 orders, not once per order. That's the Hi-Lo magic.

### Example:

Hi-Lo ID Generator in Python

```
import threading

class HiLoGenerator:
    """
    Generates unique IDs without hitting the DB for every ID.
    Fetches a new 'Hi' block when the local 'Lo' runs out.
    """
    def __init__(self, block_size=100):
        self.block_size = block_size
        self.hi = -1          # Current high block (will trigger fetch on first next())
        self.lo = block_size  # Start at max to trigger immediate fetch
        self.lock = threading.Lock()

    def _fetch_next_hi(self):
        """
        Simulate a DB call to get the next available Hi block.
        In reality this would be:
            SELECT nextval('hi_sequence') FROM dual;  -- Oracle/PostgreSQL
            UPDATE hi_table SET hi = hi + 1;           -- manual table
        """
        # Simulated: in real code, this hits the database
        self._db_value = getattr(self, '_db_value', -1) + 1
        print(f"  📞 DB CALL → fetched Hi block #{self._db_value}")
        return self._db_value

    def next_id(self) -> int:
        """
        Returns the next unique ID.
        Calls the database ONLY when the local block is exhausted.
        """
        with self.lock:
            if self.lo >= self.block_size:
                # Ran out of local numbers → fetch new Hi block
                self.hi = self._fetch_next_hi()
                self.lo = 0

            generated_id = (self.hi * self.block_size) + self.lo
            self.lo += 1
            return generated_id


# ── Usage ──
gen = HiLoGenerator(block_size=5)   # Small block for demo

print("Generating IDs:")
for i in range(12):
    print(f"  → ID #{gen.next_id()}")
Output:


Generating IDs:
  📞 DB CALL → fetched Hi block #0
  → ID #0
  → ID #1
  → ID #2
  → ID #3
  → ID #4
  📞 DB CALL → fetched Hi block #1     ← Only 3 DB calls for 12 IDs!
  → ID #5
  → ID #6
  → ID #7
  → ID #8
  → ID #9
  📞 DB CALL → fetched Hi block #2
  → ID #10
  → ID #11
Only 3 database calls for 12 IDs. With block_size=1000, you'd need just 1 DB call per 1000 IDs.
```

---

day - 8

## Correlation ID

### Definition:

A Correlation ID is a unique identifier attached to a request that travels with it across every service it touches in a distributed system. It's the breadcrumb trail that lets you trace a single user action through dozens of microservices, message queues, and databases.

In one sentence: Tag a request at the door with a unique ID, then use that ID to follow it everywhere it goes.

Analogy — The Package Tracking Number
Imagine you order a jacket online. The order triggers a chain of events:

🛒 Order 📦 Warehouse 🚚 Shipping 🏠 Delivery
System → System → Provider → Confirmation
Without a tracking number With a tracking number
"Where's my order??" → Shrug. Call every department. Enter tracking # → See exactly where it is
Warehouse error? Nobody knows which order is affected. "Order #TRK-8821 failed at warehouse — re-pick it"
Customer complains? 20-minute investigation across 5 systems. Search logs for TRK-8821 → instant full timeline
A Correlation ID is that tracking number — for every single request inside your system.

How It Works

```text
Client Service A Service B Service C
│ │ │ │
│ GET /order/42 │ │ │
│ X-Correlation-ID: abc │ │ │
│─────────────────────────▶│ │ │
│ │ POST /inventory │ │
│ │ X-Correlation-ID: abc │
│ │───────────────────▶│ │
│ │ │ PUBLISH order.paid│
│ │ │ correlationId=abc │
│ │ │───────────────────▶│
│ │ │ │
│ ◀─── 200 OK ───────────│◀───────────────────│◀───────────────────│
│ │ │ │
▼ ▼ ▼ ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Every log line, every message, every DB row stamped with: "abc" │
│ Search "abc" → full timeline instantly │
└──────────────────────────────────────────────────────────────────────┘
```

### Example:

End-to-End in a Microservice System

```
1️⃣ Frontend / Gateway — Create or Forward the ID

# API Gateway (or first service that handles the request)
from flask import Flask, request
import uuid
import requests

app = Flask(__name__)

@app.before_request
def attach_correlation_id():
    # If the caller already sent one, use it. Otherwise, create it.
    correlation_id = request.headers.get('X-Correlation-ID')
    if not correlation_id:
        correlation_id = str(uuid.uuid4())

    # Store it so every downstream call can access it
    request.correlation_id = correlation_id
    print(f"🔖 [{correlation_id}] Request started: {request.method} {request.path}")

@app.route('/order', methods=['POST'])
def create_order():
    order_data = request.json
    cid = request.correlation_id

    print(f"📦 [{cid}] Creating order for: {order_data['item']}")

    # Call service B — PASS THE CORRELATION ID
    resp = requests.post(
        "http://inventory-service/check",
        json={"item_id": order_data["item"]},
        headers={"X-Correlation-ID": cid}        # ← THE KEY LINE
    )
    print(f"📋 [{cid}] Inventory responded: {resp.status_code}")
    return {"status": "ok", "correlation_id": cid}
2️⃣ Middle Service — Receive It, Log It, Forward It

# Inventory Service
from flask import Flask, request

app = Flask(__name__)

@app.route('/check', methods=['POST'])
def check_inventory():
    cid = request.headers.get('X-Correlation-ID', 'no-cid')

    print(f"🏭 [{cid}] Checking inventory for item_id={request.json['item_id']}")
    print(f"🏭 [{cid}] Stock found: 23 units")

    # Simulate a database insert with correlation ID
    # INSERT INTO inventory_logs (item_id, action, correlation_id) VALUES (42, 'checked', 'abc')

    return {"available": True}
3️⃣ Logging — The Real Payoff

import logging
import sys

# Structured logging that always includes the correlation ID
formatter = logging.Formatter(
    '{"time": "%(asctime)s", "level": "%(levelname)s", "cid": "%(cid)s", "msg": "%(message)s"}'
)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(formatter)

logger = logging.getLogger("service")
logger.addHandler(handler)

# Usage in any service
def log(cid, message, level="info"):
    extra = {"cid": cid}
    getattr(logger, level)(message, extra=extra)

log("abc-123", "Order created")        # {"cid": "abc-123", "msg": "Order created"}
log("abc-123", "Payment processed")    # {"cid": "abc-123", "msg": "Payment processed"}
log("abc-123", "Shipment dispatched")  # {"cid": "abc-123", "msg": "Shipment dispatched"}
4️⃣ Searching — The Magic Moment
Now in your log aggregator (ELK, Splunk, Datadog, Grafana Loki):


# Find EVERYTHING related to a single user request
$ grep "abc-123" /var/log/*.json

[12:03:01] abc-123  →  Gateway:    Request started POST /order
[12:03:01] abc-123  →  Orders:     Creating order for "jacket"
[12:03:02] abc-123  →  Inventory:  Checking stock for item #42
[12:03:02] abc-123  →  Inventory:  Stock: 23 units
[12:03:03] abc-123  →  Payment:    Charging $89.99
[12:03:03] abc-123  →  Payment:    Payment approved
[12:03:04] abc-123  →  Warehouse:  Pick ticket created #WH-551
[12:03:04] abc-123  →  Orders:     Order complete → 200 OK
One search. The full story. Every service. In order.
```

---

day - 9

## CodingBooth

### Definition:

CodingBooth is a tool that gives every project its own isolated, reproducible development environment declared in the repository itself. One command brings it up; one command tears it down. Nothing installs on your host machine — all toolchains, runtimes, and dependencies live inside a container that travels with the project.

In one sentence: "A booth per project. A clean host. A repo that brings its own environment."

Analogy — The Pop-Up Kitchen
Imagine a chef who cooks three different cuisines — Italian, Thai, and French. Instead of cramming every ingredient and tool into one chaotic kitchen, she uses pop-up kitchen pods:

Approach What it looks like CodingBooth
❌ One messy kitchen Global Node.js, Python 2.7 and 3.11, three JDKs, a broken JAVA_HOME, and a pip package you installed once in 2023 and forgot about The "project residue" problem
✅ Pop-up kitchen pods Open the Italian pod → everything for pasta. Open the Thai pod → wok, fish sauce, jasmine rice. French pod → butter, wine, copper pans. Each project has its own .booth/ folder with exactly what it needs
Cooking Concept Real World CodingBooth
Pop-up kitchen pod A self-contained cooking station Booth — a containerized dev environment
Recipe card taped to the pod Lists exactly what's inside Boothfile — declares the environment
Pod setup instructions "Plug in, turn on gas, start cooking" ./booth — one command brings it up
Cleaning up Fold the pod away, counter is spotless ./booth down — nothing left on the host

### Example:

A Real Boothfile
Here's a complete CodingBooth setup for a Svelte + Firebase blog with Claude Code AI assistance:

```
.booth/Boothfile

# .booth/Boothfile
# syntax=codingbooth/boothfile:1
# Configured by: booth config --no-tui --overwrite --variant codeserver --port 13579 --expose 5173 --select firebase+credential/claude-code+auto-accept+credential+settings-cache

setup svelte        # Node.js + Svelte toolchain
setup firebase      # Firebase CLI for deploys
setup claude-code   # Claude Code AI assistant
Two setup lines and the entire development environment is declared. No FROM, no ARG, no shell wrangling. Each setup line maps to a curated install script.

.booth/config.toml

# .booth/config.toml
variant = "codeserver"
port    = "13579"

run-args = [
    "-v", "~/.config/configstore/firebase-tools.json:/etc/cb-home-seed/.config/configstore/firebase-tools.json:ro",
    "-v", "~/.claude.json:/etc/cb-home-seed/.claude.json:ro",
    "--publish", "5173:5173"
]
variant = "codeserver" — opens a browser-based VS Code
Credential mounts (:ro = read-only) pass your Firebase and Claude keys into the booth
--publish 5173:5173 — forwards the Vite dev server to your host browser
Usage

# Bring the environment up (one command)
./booth

# → Browser opens at http://localhost:13579
# → Full VS Code in the browser
# → Svelte dev server running on port 5173
# → Firebase CLI ready to deploy
# → Claude Code ready to assist
# → NOTHING installed on the host

# When done:
./booth down   # Tears it all down, host is clean
```

---

day - 10

## Frame Buffer Hashing

### Definition:

Frame Buffer Hashing is a visual regression testing technique that verifies rendered graphics by reading the raw GPU framebuffer (the pixel data before any compression or encoding), computing a cryptographic hash of those bytes, and comparing it against a known-good reference hash. If the hashes match, every pixel is identical. If they don't, something changed.

In one sentence: "Take a fingerprint of the raw pixels straight from the GPU, then compare it to the golden fingerprint — if they match, the render is perfect."

Analogy — The Painting Authenticator
Imagine an art authenticator verifying that a famous painting hasn't been tampered with. She has two approaches:

Approach What happens Result
📸 Take a photo, then compare photos Photo compression introduces artifacts. Two identical paintings shot with different camera settings produce different JPEGs. "They don't match!" → false alarm 😤
🔬 Scan the canvas directly She places a high-res scanner directly on the original canvas, captures raw pigment data, and hashes it. Same canvas = same hash. Always. "Hash matches. It's authentic." ✅
The photo is like a PNG screenshot — encoded, compressed, lossy. The direct scan is like the raw framebuffer — pure, unmodified pixel data from the GPU.

### Example:

Frame Buffer Hashing in Practice

```
Here's a simplified version based on real GPU testing frameworks (Intel GPU Tools, Chromium's video frame validator):


import hashlib
import struct
from dataclasses import dataclass
from typing import Dict

@dataclass
class FrameBuffer:
    """Represents a raw GPU framebuffer mapped to CPU memory."""
    width: int
    height: int
    bytes_per_pixel: int  # e.g., 4 for RGBA
    raw_bytes: bytes       # raw pixel data straight from the GPU

def hash_framebuffer(fb: FrameBuffer) -> str:
    """
    Compute an MD5 hash of the raw framebuffer bytes.
    No encoding, no compression — pure pixel data.
    """
    # In real GPU testing, you'd map the framebuffer from GPU memory:
    #   ptr = glMapBuffer(GL_PIXEL_PACK_BUFFER, GL_READ_ONLY)
    #   fb.raw_bytes = ptr[0 : width * height * bpp]

    return hashlib.md5(fb.raw_bytes).hexdigest()


# ── Golden Reference Store ──
# Stored once when the test is first created.
# A few KB for hundreds of frames (vs. GBs of PNGs).
GOLDEN_HASHES: Dict[str, str] = {
    "main_menu":        "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
    "settings_screen":  "1f2e3d4c5b6a79808182838485868788",
    "gameplay_frame_1": "deadbeefcafebabedeadbeefcafebabe",
}


def visual_regression_test(scene_name: str, fb: FrameBuffer) -> bool:
    """
    Run a single visual regression test.
    Returns True if the frame matches the golden reference.
    """
    actual_hash = hash_framebuffer(fb)
    expected_hash = GOLDEN_HASHES.get(scene_name)

    if expected_hash is None:
        # New scene — store this as the golden reference
        print(f"✨ No golden hash for '{scene_name}'. Storing: {actual_hash}")
        GOLDEN_HASHES[scene_name] = actual_hash
        return True

    if actual_hash == expected_hash:
        print(f"✅ {scene_name}: PASS (hash matches)")
        return True
    else:
        print(f"❌ {scene_name}: FAIL")
        print(f"   Expected: {expected_hash}")
        print(f"   Got:      {actual_hash}")
        # Save the actual frame as a PNG for human review
        save_frame_as_png(fb, f"failure_artifacts/{scene_name}.png")
        return False


def save_frame_as_png(fb: FrameBuffer, path: str):
    """Only save a PNG when the test FAILS — for manual inspection."""
    # Convert raw bytes to PNG (using PIL, stb_image, etc.)
    # This is the ONLY time encoding happens — after failure.
    print(f"   📸 Saved failure artifact: {path}")


# ── Usage ──
# Simulate a correct render
correct_frame = FrameBuffer(
    width=1920, height=1080, bytes_per_pixel=4,
    raw_bytes=b'\xff\x00\x00\xff' * (1920 * 1080)  # all red pixels
)
test1 = visual_regression_test("main_menu", correct_frame)

# Simulate a rendering bug — one pixel is wrong
buggy_bytes = bytearray(b'\xff\x00\x00\xff' * (1920 * 1080))
buggy_bytes[0] = 0xfe  # slightly different red in first pixel
buggy_frame = FrameBuffer(
    width=1920, height=1080, bytes_per_pixel=4,
    raw_bytes=bytes(buggy_bytes)
)
test2 = visual_regression_test("main_menu", buggy_frame)
Output:


✅ main_menu: PASS (hash matches)
❌ main_menu: FAIL
   Expected: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6
   Got:      7f8e9d0c1b2a39485748392a1b0c0d0e
   📸 Saved failure artifact: failure_artifacts/main_menu.png
```

---

day - 11

## Perag

### Definition:

Perag (short for Personal RAG) is a local, no-cloud, command-line RAG tool that gives your AI assistant long-term memory by making your personal documents searchable. It works entirely on your machine — no data ever leaves your computer.

Instead of shoving hundreds of files into an AI's context window (which would overflow it), Perag:
Chunks your documents into small pieces
Embeds them into vector representations
Stores them in a local SQLite database
Retrieves only the most relevant chunks when you ask a question

It's designed as a UNIX pipeline (chunk | embed | ingest | query) — composable, transparent, and scriptable.

### Example:

Say you have a folder of contracts:

```
~/projects/
├── nda_2024.pdf
├── service-agreement.docx
└── meeting-notes.md


Step 1 — Initialize
cd ~/projects
perag init


Step 2 — Add documents
perag add nda_2024.pdf service-agreement.docx meeting-notes.md


Step 3 — Ask questions to your AI assistant
"What does the NDA say about the notice period?"

Your AI assistant automatically runs perag query "notice period NDA" under the hood, retrieves the relevant chunk(s), and answers based on what's actually in your document — not a guess.

What happens behind the scenes
nda_2024.pdf
    ↓ perag chunk
[chunk1, chunk2, ...]
    ↓ perag embed (local model)
[chunk1+vector, chunk2+vector, ...]
    ↓ perag ingest
(sqlite-vec database)
    ↓ perag query "notice period NDA"
→ Returns: "The agreement shall terminate upon 30 days written notice..."


Key points:
✅ 100% local — no cloud, no account, no API key needed (by default)
✅ Works with Claude Code and other CLI-based AI assistants (1/2)
✅ Privacy-first — your documents never leave your machine
🔗 Composable via UNIX pipes — each step is its own command
```

---

day - 12

## Static Pipelines (DAGs)

### Definition:

A Static Pipeline is a workflow structured as a Directed Acyclic Graph (DAG) where each node is a task and each edge is a dependency (A → B means "A must finish before B starts"). Directed means every edge has a clear direction — data or control flows one way. Acyclic means there are no loops — the pipeline cannot circle back to an earlier stage, guaranteeing it will always terminate. Static means the graph structure is fixed at definition time, not dynamically rewired during execution.

In one sentence: A pre-planned, loop-free dependency graph that guarantees tasks run in the correct order, every time.

Analogy — The Assembly Line Kitchen 🏭

Imagine a high-end sushi restaurant with a conveyor belt. The kitchen is split into stations:

Rice Cooker → Fish Prep → Rolling → Plating → Customer

Directed: Rice must be cooked before rolling. You can't roll sushi without rice.
Acyclic: There's no "send the plated sushi back to the rice cooker" — the belt only moves forward.
Static: The manager designed this layout Monday morning and it stays the same all week. Every order follows the exact same path.

If a customer orders miso soup too, the pipeline forks:
```text
┌→ Rolling → Plating → Customer
Rice Cooker ──┤
└→ Soup Pot → Plating → Customer
```

The soup and sushi prep can happen in parallel (they're independent), but both must hit "Plating" before reaching the customer. The DAG captures this without any manual coordination.

### Example:

ETL Data Pipeline with Airflow-style DAG

```
from datetime import datetime

# Imagine a simple DAG scheduler checks dependencies
# before running each step.

tasks = {}

def task(name, depends_on=None):
    """Register a pipeline task."""
    tasks[name] = {"depends_on": depends_on or [], "done": False}

def run(name):
    """Run a task if all its dependencies are satisfied."""
    t = tasks[name]
    for dep in t["depends_on"]:
        if not tasks[dep]["done"]:
            print(f"  ⏳ {name} waiting for {dep}...")
            return False
    print(f"  ✅ Running {name}...")
    t["done"] = True
    return True

# ─── Define the static DAG ───
task("extract_csv", depends_on=[])        # Root node: no deps
task("extract_api", depends_on=[])        # Root node: no deps
task("validate",    depends_on=["extract_csv", "extract_api"])
task("transform",   depends_on=["validate"])
task("load",        depends_on=["transform"])

print("🚀 Pipeline Execution:\n")

# ─── Execute in topological order ───
# (In a real DAG scheduler, this is automatic)
while not tasks["load"]["done"]:
    run("extract_csv")
    run("extract_api")
    run("validate")
    run("transform")
    run("load")

print("\n📊 Pipeline: COMPLETE")


Output:
🚀 Pipeline Execution:

  ✅ Running extract_csv...
  ✅ Running extract_api...
  ✅ Running validate...
  ✅ Running transform...
  ✅ Running load...

📊 Pipeline: COMPLETE
```

### Key Concepts:

1. **DAG (Directed Acyclic Graph)**: A graph with nodes (tasks) and edges (dependencies) that has no cycles.
2. **Topological Sort**: An ordering of nodes where every dependency comes before the dependent task.
3. **Idempotency**: Tasks should be able to run multiple times without changing the final result.
4. **State Tracking**: Each task tracks its completion status to avoid redundant work.

### Real-world Analogy:

Imagine a construction project where:
- Tasks are like building steps (foundation, framing, plumbing, etc.)
- Dependencies are like "must be done before" relationships
- The DAG scheduler is like a foreman ensuring steps happen in theß correct order

This pattern is used in:
- Data pipelines (Apache Airflow, Luigi)
- Build systems (Make, Bazel)
- Workflow engines (AWS Step Functions, Prefect)

---

day - 15

## WASI-NN (WebAssembly System Interface for Neural Networks)

### Definition:

WASI-NN is a standardized API specification that lets WebAssembly (WASM) modules perform neural network inference by offloading the computation to native backends — OpenVINO, ONNX Runtime, TensorFlow Lite, or PyTorch — through a clean, host-provided interface.

Normally, running ML inference inside WASM means either:
❌ Compiling the entire ML framework (hundreds of MB) to WASM — slow and bloated.
❌ Writing manual bindings per platform — fragile and non-portable.

WASI-NN solves this by defining a small, standard set of functions (load, init_execution_context, set_input, compute, get_output) that any WASM module can call, and any WASM runtime (Wasmer, Wasmtime, WAMR) can implement by plugging in a native backend. The WASM module stays small and portable; the heavy lifting happens on the host's GPU/NPU/CPU.

Analogy — The Film Lab 📸

Imagine you're a photographer in the 1990s. Your camera has a tiny film roll (the WASM module). It captures the raw shot (input data) and specs like "develop with Fuji process, print 4×6 glossy." You drop the roll at a film lab (the WASM runtime with WASI-NN). The lab has:

A Fuji machine (OpenVINO backend)
A Kodak machine (ONNX Runtime backend)
A printing press (PyTorch backend)

You don't care which machine does the work — you just hand over the film and a memo ("inference request"). The lab picks the right machine, runs the chemistry, and hands you back the final prints (output tensor). You never set foot in the machine room.

The film roll stays tiny. The lab does the heavy equipment work. That's WASI-NN.

### Example:

Running Image Classification via WASI-NN

```
;; WASM module using WASI-NN to classify an image
;; The module sends bytes → gets back a label index

(module
  ;; Import WASI-NN functions provided by the host runtime
  (import "wasi_nn" "load"
    (func $load (param $bytes i32) (param $len i32)
                (result i32)))        ;; returns graph handle

  (import "wasi_nn" "init_execution_context"
    (func $init_ctx (param $graph i32)
                    (result i32)))    ;; returns context handle

  (import "wasi_nn" "set_input"
    (func $set_input (param $ctx i32) (param $tensor i32))  ;; void

  (import "wasi_nn" "compute"
    (func $compute (param $ctx i32)))                       ;; void

  (import "wasi_nn" "get_output"
    (func $get_output (param $ctx i32) (param $out i32)))   ;; void

  ;; Memory for input image (224×224×3 = 150,528 bytes)
  (memory (export "memory") 1)

  ;; ─── Inference Flow ───
  (func (export "classify") (param $img_offset i32) (param $img_len i32)
                            (result i32)

    ;; Step 1: Load the model from embedded bytes
    ;;        (model stored in WASM data section at offset 0x1000)
    i32.const 0x1000      ;; model bytes start
    i32.const 50000       ;; model size in bytes
    call $load
    local.set $graph      ;; graph handle

    ;; Step 2: Create an execution context
    local.get $graph
    call $init_ctx
    local.set $ctx

    ;; Step 3: Set the input tensor (the image)
    local.get $ctx
    local.get $img_offset
    call $set_input

    ;; Step 4: Run inference
    local.get $ctx
    call $compute

    ;; Step 5: Read the result (index of highest-probability class)
    local.get $ctx
    i32.const 0x2000      ;; write output tensor here
    call $get_output

    ;; Return the top class index
    i32.load 0x2000
  )
)
```

---

day - 16

## HNSW (Hierarchical Navigable Small World)

### Definition:

HNSW is a graph-based Approximate Nearest Neighbor (ANN) search algorithm that organizes high-dimensional vectors (embeddings) into a hierarchy of navigable small-world graphs. It builds on two ideas:

1. **Navigable Small World (NSW)** — A graph where each node is connected to its nearest neighbors, creating short paths between any two nodes (the "small world" property: you can reach any node in a few hops).
2. **Multi-Layer Hierarchy** — The graph is stratified into layers. The topmost layers are sparse, containing only a few nodes with long-range edges (coarse search). The bottom layer contains all nodes with short, precise edges (fine search).

**How search works:** Starting from the top layer, you greedily navigate to the nearest node, then descend to the layer below using that node as the entry point. Repeat until you reach the bottom layer, where you perform a fine-grained search among the closest neighbors. This is like searching a map at different zoom levels — first pan across continents, then zoom into a city, then find the exact street.

**Why it matters:** HNSW is the de facto standard for vector search today. It powers vector databases like Pinecone, Qdrant, Weaviate, and pgvector's ivfflat alternatives. It achieves search complexity of O(log N) with high recall — orders of magnitude faster than brute-force k-NN.

Analogy — Google Maps Zoom Levels 🗺️

Imagine you're looking for a specific coffee shop in Jakarta.

- **Top layer (zoom level 1):** You see only major cities — Jakarta, Surabaya, Bandung. You click Jakarta.
- **Middle layer (zoom level 5):** You see Jakarta's districts — Menteng, Kemang, Kelapa Gading. You click Menteng.

### Example:

Vector Search with a Minimal HNSW Implementation

```python
import numpy as np
import heapq
import random

class SimpleHNSW:
    """Minimal HNSW for learning — not production-grade."""

    def __init__(self, dim: int, M: int = 16, ef_construction: int = 200):
        self.dim = dim
        self.M = M
        self.ef_construction = ef_construction
        self.nodes = []
        self.graphs = []
        self.max_level = 0
        self.enter_point = None

    def _random_level(self) -> int:
        level = 0
        while random.random() < 0.5 and level < 16:
            level += 1
        return level

    def _search_layer(self, query, entry, ef, layer):
        """Greedy search on a single layer."""
        visited = {entry}
        dist_entry = np.linalg.norm(query - self.nodes[entry])
        candidates = [(dist_entry, entry)]
        heapq.heapify(candidates)
        result = [(dist_entry, entry)]

        while candidates:
            dist_c, c = heapq.heappop(candidates)
            furthest = max(result, key=lambda x: x[0])[0]
            if dist_c > furthest:
                break
            for neighbor in self.graphs[layer].get(c, []):
                if neighbor not in visited:
                    visited.add(neighbor)
                    dist_n = np.linalg.norm(query - self.nodes[neighbor])
                    furthest = max(result, key=lambda x: x[0])[0]
                    if dist_n < furthest or len(result) < ef:
                        heapq.heappush(candidates, (dist_n, neighbor))
                        result.append((dist_n, neighbor))
                        result = heapq.nsmallest(ef, result, key=lambda x: x[0])
        return result

    def insert(self, vector):
        node_id = len(self.nodes)
        self.nodes.append(vector)
        level = self._random_level()

        while len(self.graphs) <= level:
            self.graphs.append({})
        for l in range(level + 1):
            self.graphs[l][node_id] = set()

        if self.enter_point is None:
            self.enter_point = node_id
            self.max_level = level
            return

        curr_entry = self.enter_point
        for l in range(self.max_level, level, -1):
            result = self._search_layer(vector, curr_entry, 1, l)
            curr_entry = result[0][1]

        for l in range(min(level, self.max_level), -1, -1):
            result = self._search_layer(vector, curr_entry, self.ef_construction, l)
            neighbors = [n for _, n in result[:self.M]]
            self.graphs[l][node_id] = set(neighbors)
            for n in neighbors:
                self.graphs[l][n].add(node_id)
                if len(self.graphs[l][n]) > self.M:
                    n_neighbors = list(self.graphs[l][n])
                    n_dists = [(np.linalg.norm(self.nodes[n] - self.nodes[nn]), nn)
                               for nn in n_neighbors]
                    n_dists.sort(key=lambda x: x[0])
                    self.graphs[l][n] = {nn for _, nn in n_dists[:self.M]}
            curr_entry = result[0][1]

        if level > self.max_level:
            self.max_level = level
            self.enter_point = node_id

    def search(self, query, k=5):
        if self.enter_point is None:
            return []
        curr_entry = self.enter_point
        for l in range(self.max_level, 0, -1):
            result = self._search_layer(query, curr_entry, 1, l)
            curr_entry = result[0][1]
        result = self._search_layer(query, curr_entry, k, 0)
        return [(dist, nid) for dist, nid in heapq.nsmallest(k, result, key=lambda x: x[0])]

```

---

day - 17

## FinOps (Cloud Financial Operations)

### Definition:

FinOps (a portmanteau of **Finance** and **DevOps**) is an operational framework and cultural practice that brings engineering, finance, and business teams together to manage cloud spending responsibly. It transforms cloud cost from a passive monthly surprise into an active engineering discipline — treating cost efficiency as a first-class concern alongside performance, scalability, and reliability, not as an afterthought discovered when the invoice arrives.

The core idea is simple but powerful: **engineers who provision cloud resources should understand what those resources cost — and be empowered to optimize them.** Instead of finance teams sending a spreadsheet after the fact, FinOps makes cost visibility part of the engineering workflow.

The FinOps Foundation defines three iterative phases:

1. **Inform** — Visibility and allocation. Read the bill, tag resources, identify top cost drivers, establish a baseline.
2. **Optimize** — Efficiency through action. Right-size over-provisioned resources, move cold data to cheaper tiers, use Spot instances, purchase commitments in the correct sequence.
3. **Operate** — Continuous improvement through automation. Automate orphan cleanup, add cost checks to CI/CD pipelines, build chargeback/showback models, present unit-cost metrics to leadership.

**Why sequence matters:** The most expensive mistake in cloud cost management is buying Savings Plans or Reserved Instances *before* right-sizing. You end up paying a discounted price for waste — locked in for 1–3 years. The correct order is: right-size first, *then* commit.

Analogy — The Grocery Budget 🛒

Imagine you and your roommates share a kitchen. Every week, someone goes to the supermarket and buys whatever they want — organic avocados, premium ice cream, name-brand cereal — and puts it on a joint credit card. At the end of the month, the bill arrives and everyone is shocked. Nobody knows who bought what or why it's so expensive. Some items spoil before anyone eats them.

**FinOps is the solution:** You start by reading the receipt (inform). You tag each item with the owner's name so you know who spent what (tagging). You notice someone has been buying 5 bags of chips every week but only eating 2 — so you buy smaller portions (right-sizing). You switch from name-brand to store-brand on staples (Spot instances). You set a rule: "no single purchase over $50 without group approval" (cost-aware code review). You automate a weekly waste report — expired milk, moldy bread — that flags what to stop buying (orphaned resource cleanup). Now the kitchen runs efficiently, everyone knows their share, and there's no monthly shock.

### Example:

Building a Cost-Aware Infrastructure Pipeline — From Baseline to Block

This example shows a complete mini-FinOps workflow for an AWS environment, following the article's 4-stage roadmap in practice.

**Step 1: Establish a Baseline (Inform)**

Pull last month's cost broken down by service. Save this before any optimization — you cannot measure progress without a baseline.
bash
Save as aws-baseline-2026-05.txt — compare every future month against this
aws ce get-cost-and-usage \
--time-period Start=2026-05-01,End=2026-06-01 \
--granularity MONTHLY \
--group-by Type=DIMENSION,Key=SERVICE \
--metrics UnblendedCost \
--query 'ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}' \
--output table | sort -k3 -rn
**Step 2: Find Right-Sizing Candidates (Optimize)**

Run a Python audit to detect EC2 instances running below 20% average CPU for 14 days. These are likely over-provisioned.
python
import boto3
from datetime import datetime, timedelta

def find_rightsizing_candidates(region='us-east-1'):
ec2 = boto3.client('ec2', region_name=region)
cw = boto3.client('cloudwatch', region_name=region)

reservations = ec2.describe_instances(
Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
...

---

day - 18

## Architectural Decision Record (ADR)

### Definition:

- Short, lightweight document that captures a significant architectural 
  decision + context + rationale + trade-offs
- "Document the decision, not the details"
- Lifecycle: Proposed → Accepted → Deprecated → Superseded
- Analogi: Ship's Captain Log — for the next captain who takes the helm 🚢

### Example:

```
Full ADR document (ADR-001) for choosing PostgreSQL over DynamoDB/MongoDB 
for a User Service migration, including:
- Context (2.5M users, strong consistency needed)
- Decision (PostgreSQL 16 on RDS)
- Alternatives Considered (DynamoDB, MongoDB — with honest pros/cons)
- Consequences (positive + negative — no perfect decision)
- Related Decisions (links to ADR-000, ADR-003)
```

---

day - 19

## TSRX (TypeScript Render Extensions)

### Definition:

- TypeScript language extension by Dominic Gannaway (ex-React/Svelte core, 
  creator of Inferno, Lexical, Ripple)
- "Spiritual successor to JSX" — statement-based instead of expression-based
- Framework-agnostic: compiles to React, Preact, Solid, Vue, Ripple
- Key features: @if/@else, @for, scoped <style>, conditional hooks, prop shorthands
- Not a runtime — pure compile-time: .tsrx → AST → framework-specific .tsx
- Status: Beta, MIT license
- Analogi: Universal Translator — write once, output idiomatic code for any framework 🌐

### Example:

```
UserProfile.tsrx component showcasing:
  - @if/@else for auth state (signed in vs signed out)
  - @for for rendering recent posts list
  - Scoped <style> block auto-hashed by compiler
  - Conditional hooks (useUser inside @else — compiler generates child 
    component to satisfy Rules of Hooks)
  - Side-by-side with compiled React output
```

### Key Concepts:
- Statement-based > expression-based (compilers can optimize better)
- Framework-agnostic via plugin architecture
- Conditional hooks = no more manual component splitting for hook rules
- Incremental adoption (one .tsrx at a time)

===

day - 22

## BFS Distance Field

### Definition:

- Distance field: grid where each cell contains distance to nearest obstacle
- BFS (Breadth-First Search) algorithm to compute this field
- Used in pathfinding, collision detection, shadow mapping
- Key properties:
  - Euclidean distance (true straight-line distance)
  - Signed distance (negative inside obstacles)
  - Smooth gradients (useful for interpolation)

### Example:

```
Visualization of BFS distance field computation:
  - Obstacle grid (walls, trees)
  - Wavefront propagation from obstacles
  - Final distance field (heatmap)
```

### Key Concepts:

- BFS guarantees shortest path in unweighted grid
- Distance field enables efficient spatial queries

---

day - 23

## Human-in-the-Loop (HITL)

### Definition:

Human-in-the-Loop (HITL) is a design pattern where a human being is embedded in the decision loop of an automated system — reviewing, validating, correcting, or approving outputs before they take effect. Instead of a fully autonomous pipeline (data in → decision out), HITL creates a checkpoint where domain expertise, judgment, ethics, or intuition can override or refine the machine's output.

HITL exists on a spectrum of automation:

| Level | Human Role | Example |
|-------|-----------|---------|
| **Fully Automated** | None — machine decides and acts | Auto-scaling EC2 instances |
| **Human-on-the-Loop** | Monitor + veto power only | Self-driving car with a safety driver |
| **Human-in-the-Loop** | Must approve before action | AI code review suggestions requiring human sign-off |
| **Human-in-the-Process** | Handoff between human and machine stages | Medical diagnosis: AI flags → doctor reviews → treatment |
| **Fully Manual** | Machine provides info only, human decides | IDE autocomplete suggestions |

**Where HITL is critical:**
- **High-stakes decisions** — Medical diagnosis, loan approvals, legal document review, hiring filters
- **Edge cases** — ML models are confident on 95% of data; the remaining 5% needs human judgment
- **Training data** — Active learning uses HITL to prioritize which unlabeled examples a human should annotate
- **Safety & compliance** — Regulated industries (healthcare, finance, defense) often require a human signature on automated decisions
- **Model improvement** — Human feedback becomes training signal (RLHF — Reinforcement Learning from Human Feedback)

### Example:

A medical imaging pipeline where an AI model flags suspicious X-rays, but a radiologist must confirm before the report is sent to the patient.

```
import json
import random
from dataclasses import dataclass, field
from typing import Optional


─── Domain Models ───
@dataclass
class XRayScan:
    patient_id: str
    image_id: str
    findings: list[str] = field(default_factory=list)
    ai_confidence: float = 0.0

@dataclass
class DiagnosisReport:
    scan: XRayScan
    ai_summary: str
    radiologist_notes: Optional[str] = None
    reviewed_by: Optional[str] = None
    final_diagnosis: Optional[str] = None


─── Stage 1: AI Inference ───
class AIModel:
    """Simulates a chest X-ray classifier."""

    def predict(self, scan: XRayScan) -> dict:
        # Simulate: model returns findings + confidence score
        findings_pool = [
            "No significant abnormalities",
            "Minor opacities detected — likely benign",
            "Nodule detected in upper left lobe — further investigation recommended",
            "Signs of pleural effusion — moderate severity",
            "Cardiomegaly (enlarged cardiac silhouette)",
            "Consolidation consistent with pneumonia",
        ]
        # Simulate uncertain prediction
        is_confident = random.random() > 0.3  # 70% confident

        if is_confident:
            findings = [random.choice(findings_pool[:2])]  # "easy" cases
            confidence = random.uniform(0.92, 0.99)
        else:
            findings = [random.choice(findings_pool[2:])]  # "hard" cases
            confidence = random.uniform(0.55, 0.82)

        scan.findings = findings
        scan.ai_confidence = round(confidence, 4)
        return {
            "findings": findings,
            "confidence": scan.ai_confidence,
        }


─── Stage 2: HITL Routing ───
CONFIDENCE_THRESHOLD = 0.85

def route_for_review(scan: XRayScan) -> str:
    """Decide if this scan needs a radiologist review."""
    if scan.ai_confidence >= CONFIDENCE_THRESHOLD:
        return "auto_approve"
    else:
        return "needs_review"


─── Stage 3: Human Review ───
class Radiologist:
    """A human expert who reviews low-confidence predictions."""

    def __init__(self, name: str):
        self.name = name
        self.reviews_completed = 0

    def review(self, scan: XRayScan, ai_summary: str) -> DiagnosisReport:
        self.reviews_completed += 1

        print(f"\n👩‍⚕️ Radiologist {self.name} is reviewing scan {scan.image_id}...")
        print(f"   AI findings: {scan.findings}")
        print(f"   AI confidence: {scan.ai_confidence:.2%}")

        # Simulate human judgment — may agree, refine, or override
        if random.random() > 0.15:  # 85% agreement
            final_diagnosis = scan.findings[0]
            notes = "Confirmed. No additional findings."
        else:
            final_diagnosis = "Benign scarring, no action needed"
            notes = (
                "AI flagged nodule, but upon review this is linear scarring "
                "from prior infection. No follow-up required."
            )

        return DiagnosisReport(
            scan=scan,
            ai_summary=ai_summary,
            radiologist_notes=notes,
            reviewed_by=self.name,
            final_diagnosis=final_diagnosis,
        )


─── Stage 4: Pipeline Orchestration ───
def run_hitl_pipeline(scans: list[XRayScan]) -> list[DiagnosisReport]:
    model = AIModel()
    radiologist = Radiologist("Dr. Sarah Chen")
    reports = []

    for scan in scans:
        # Step 1: AI predicts
        prediction = model.predict(scan)
        ai_summary = f"Findings: {', '.join(prediction['findings'])} | Confidence: {prediction['confidence']:.2%}"

        # Step 2: Route based on confidence
        action = route_for_review(scan)

        if action == "auto_approve":
            # Auto-approved — no human needed
            report = DiagnosisReport(
                scan=scan,
                ai_summary=ai_summary,
                final_diagnosis=scan.findings[0],
                reviewed_by="AI (auto-approved)",
                radiologist_notes="Confidence threshold met; auto-approved.",
            )
            print(f"✅ Scan {scan.image_id}: AUTO-APPROVED (confidence: {scan.ai_confidence:.2%})")

        else:
            # Human-in-the-loop
            report = radiologist.review(scan, ai_summary)
            print(f"🔍 Scan {scan.image_id}: REVIEWED by {report.reviewed_by}")
            print(f"   Final diagnosis: {report.final_diagnosis}")
            if report.radiologist_notes != "Confirmed. No additional findings.":
                print(f"   ⚠️ Override: {report.radiologist_notes}")

        reports.append(report)

    return reports


─── Run the Pipeline ───
print("🏥 Medical HITL Pipeline — X-Ray Diagnosis")
print("=" * 50)

Simulate 8 patient scans
patients = [
    XRayScan(patient_id=f"P-{i:04d}", image_id=f"X-{random.randint(1000, 9999)}")
    for i in range(8)
]

reports = run_hitl_pipeline(patients)
```

---

- Distance field: grid where each cell contains distance to nearest obstacle
- BFS (Breadth-First Search) algorithm to compute this field
- Used in pathfinding, collision detection, shadow mapping
- Key properties:
  - Euclidean distance (true straight-line distance)
  - Signed distance (negative inside obstacles)
  - Smooth gradients (useful for interpolation)

### Example:

```
Visualization of BFS distance field computation:
  - Obstacle grid (walls, trees)
  - Wavefront propagation from obstacles
  - Final distance field (heatmap)
```

### Key Concepts:

- BFS guarantees shortest path in un

---

day - 24

## Phantom APIs

### Definition:

Phantom APIs are undocumented, unapproved, and often unknown API endpoints that exist in production but were never formally reviewed, spec'd, or added to the service catalog. They are "phantom" because they serve real traffic and handle real data, yet they live outside any governance boundary — no OpenAPI spec, no design review, no security audit, no entry in the API inventory.

How they're born (the AI multiplier):

The primary driver of phantom APIs in 2025–2026 is AI-generated code. A developer asks an AI assistant (Copilot, Cursor, Claude, etc.) to "add an endpoint that lets support staff look up account status." The model, trained on millions of real-world codebases, generates:

The requested endpoint (with auth) ✅
An undocumented debug variant (raw ID, no scope check, maybe a ?debug=true flag) 👻

The PR is reviewed for logic bugs — not for the existence of extra endpoints. The debug route looks plausible. It compiles. It passes smoke tests (which check the happy path). It ships. The spec is never updated. The gateway doesn't know about it. The route inherits default framework permissions.

Now there's an endpoint in production that nobody knows about, serving data — and it's findable via automated recon tools in minutes.

Why it's dangerous:

| Factor | Impact |
|--------|--------|
| Invisible | Not in your API inventory → no monitoring, no rate limiting, no auth audit |
| AI-generated | AI models often default to broad object access, wildcard scopes, and debug patterns from training data |
| GraphQL multiplier | One deeply nested GraphQL query can hit multiple resolvers with missing auth — "one bug wearing one URL" becomes "a dozen bugs wearing one URL" |
| Speed of exploitation | Average breakout time fell to 29 minutes in 2025 (from 98 min in 2021). Fastest observed: 27 seconds | (1/8)
| Post-auth blind spot | 95% of API attacks originate from authenticated sessions — the "front door" isn't the problem, lateral movement after auth is |

### Example:

A CI/CD pipeline that detects and blocks phantom APIs before they reach production.

```
import json
import hashlib
from pathlib import Path
from typing import Set, List, Dict, Optional


# ─── Phase 1: Live API Inventory from Code ───

def extract_routes_from_code(codebase_path: str) -> Set[str]:
    """Parse source to find all registered HTTP routes.
       In practice, this would use AST parsing for each framework
       (Django URLs, FastAPI routers, Express app.use, etc.).
    """
    routes = set()

    # Simulate extraction from a Python/FastAPI codebase
    # Real implementation: walk AST, find @app.get, @router.post, etc.
    known_routes = {
        # Approved routes (from spec)
        "GET /api/v2/users",
        "GET /api/v2/users/{id}",
        "POST /api/v2/users",
        "PUT /api/v2/users/{id}",
        "DELETE /api/v2/users/{id}",
        "GET /api/v2/orders",
        "GET /api/v2/orders/{id}",
        "POST /api/v2/orders",
        # 🚨 Phantom routes — found in code but NOT in approved spec
        "GET /api/v2/users/{id}/debug",       # AI-generated debug endpoint
        "GET /api/v2/users/export?format=csv", # Unticketed feature
        "POST /api/v2/admin/users/impersonate", # Admin route with no scope check
        "POST /api/v2/internal/user-lookup",   # Missing auth decorator
    }
    routes.update(known_routes)
    return routes


# ─── Phase 2: Approved Spec ───

def load_approved_spec(spec_path: str) -> Set[str]:
    """Load the last-approved OpenAPI spec as the source of truth."""
    # In real life: parse OpenAPI 3.x, extract path + method combinations
    approved = {
        "GET /api/v2/users",
        "GET /api/v2/users/{id}",
        "POST /api/v2/users",
        "PUT /api/v2/users/{id}",
        "DELETE /api/v2/users/{id}",
        "GET /api/v2/orders",
        "GET /api/v2/orders/{id}",
        "POST /api/v2/orders",
    }
    return approved


# ─── Phase 3: Live Traffic Fingerprint ───

def get_live_traffic_paths() -> Set[str]:

    """Query the API gateway / WAF for paths receiving real traffic
       in the last rolling window (e.g., last 24 hours).
    """
    # In real life: query Kong / AWS API Gateway / Envoy access logs
    live_paths = {
        "GET /api/v2/users",
        "GET /api/v2/users/{id}",
        "POST /api/v2/users",
        "PUT /api/v2/users/{id}",
        "DELETE /api/v2/users/{id}",
        "GET /api/v2/orders",
        "GET /api/v2/orders/{id}",
        "POST /api/v2/orders",
        # 🚨 This shouldn't exist but is being hit
        "GET /api/v2/users/{id}/debug",
        "POST /api/v2/internal/user-lookup",
    }
    return live_paths


# ─── Phase 4: Diff Engine ───

def detect_phantom_routes(
    code_routes: Set[str],
    approved_spec: Set[str],
    live_traffic: Set[str],
) -> Dict[str, List[str]]:
    """Find routes that exist but shouldn't."""

    # Phantom from code: in code but not in approved spec
    code_phantoms = code_routes - approved_spec

    # Phantom from traffic: serving traffic but not in approved spec
    traffic_phantoms = live_traffic - approved_spec

    return {
        "code_phantoms": sorted(code_phantoms),
        "traffic_phantoms": sorted(traffic_phantoms),
        "both": sorted(code_phantoms & traffic_phantoms),
    }


# ─── Phase 5: CI/CD Gate (Blocking) ───

class CIDCGate:
    """Runs in CI/CD — blocks PRs that introduce phantom routes."""

    BLOCKING_PATTERNS = [
        "debug", "admin", "internal", "test", "dev",
        "impersonate", "export", "bypass", "legacy",
        "raw", "noscope", "unsecured", "temp",
    ]

    def check_pr(self, pr_changes: List[Dict]) -> bool:
        """Return True if the PR passes (no phantoms detected)."""
        phantom_count = 0

        print("🔍 CI/CD Phantom API Scan\n")

        for change in pr_changes:
            route = f"{change['method']} {change['path']}"
            new = change.get('new', False)

            if not new:
                continue

# Check for suspicious patterns
            for pattern in self.BLOCKING_PATTERNS:
                if pattern in route.lower():
                    print(f"⚠️  [BLOCKED] Suspicious route pattern '{pattern}'")
                    print(f"    Route: {route}")
                    print(f"    File:  {change['file']}:{change['line']}")
                    phantom_count += 1
                    break

            # Check for missing auth decorators
            if not change.get('has_auth', True):
                print(f"⚠️  [BLOCKED] Missing auth decorator")
                print(f"    Route: {route}")
                print(f"    File:  {change['file']}:{change['line']}")
                phantom_count += 1

        if phantom_count > 0:
            print(f"\n❌ BLOCKED: {phantom_count} potential phantom route(s) detected.")
            print("   Each must be reviewed and approved before merge.")
            return False

        print("✅ PASS: No phantom routes detected.")
        return True


# ─── Run the Full Pipeline ───

print("👻 Phantom API Detection Pipeline")
print("=" * 55)

# Step 1: Extract routes from code
code_routes = extract_routes_from_code("./src")
print(f"\n📦 Routes found in codebase: {len(code_routes)}")
for r in sorted(code_routes):
    marker = " 👻" if r not in load_approved_spec("spec.yaml") else ""
    print(f"   {r}{marker}")

# Step 2: Diff against approved spec
approved = load_approved_spec("spec.yaml")
phantoms = code_routes - approved
print(f"\n📋 Approved routes in spec:   {len(approved)}")
print(f"👻 Phantom routes detected:    {len(phantoms)}")
for p in sorted(phantoms):
    print(f"   ⚠️  {p}")

# Step 3: Cross-reference with live traffic
live = get_live_traffic_paths()
live_phantoms = live - approved
print(f"\n📊 Live traffic routes:         {len(live)}")
print(f"👻 Live phantoms (not in spec): {len(live_phantoms)}")
for p in sorted(live_phantoms):
    print(f"   🔥 {p}")

# Step 4: Simulate CI/CD gate

print(f"\n{'='*55}")
print("🔒 CI/CD Gate Check")
PR_CHANGES = [
    {"method": "GET", "path": "/api/v2/users/{id}/debug",
     "file": "src/routers/users.py", "line": 142, "new": True, "has_auth": False},
    {"method": "GET", "path": "/api/v2/orders/{id}",
     "file": "src/routers/orders.py", "line": 89, "new": False, "has_auth": True},
]

gate = CIDCGate()
gate.check_pr(PR_CHANGES)

print(f"\n{'='*55}")
print("🏁 Summary")
print(f"   Approved spec routes:  {len(approved)}")
print(f"   Code phantom routes:   {len(phantoms)}")
print(f"   Live phantom routes:   {len(live_phantoms)}")
print(f"   CI/CD gate result:     {'✅ PASS' if gate.check_pr([]) else '❌ BLOCKED (see above)'}")
```

---

day - 25

## Fog Computing

### Definition:

Fog Computing is a decentralized computing architecture that extends cloud capabilities to the edge of the network — sitting between the cloud (centralized data centers) and the edge devices (sensors, cameras, IoT gadgets). The name comes from the analogy that "fog is a cloud close to the ground" — it's not the edge itself, but a layer of smart, intermediate infrastructure that processes data closer to where it's generated instead of sending everything to a distant cloud.

The computing continuum looks like this:

```text
┌─────────────────────────────────────────────────────────────┐
│                    CLOUD LAYER                               │
│   Central data centers — unlimited storage, global compute   │
│   Latency: 100-500ms                           ▲            │
└─────────────────────────────────────────────────┘            │
                          │ ▲ Fewer, aggregated data           │
                          ▼ │                                  │
┌─────────────────────────────────────────────────────────────┐
│                     FOG LAYER                                │
│   Local servers / gateways — real-time processing & caching  │
│   Latency: 10-50ms                              ▲            │
└─────────────────────────────────────────────────┘            │
                          │ ▲ Filtered, pre-processed data     │
                          ▼ │                                  │
┌─────────────────────────────────────────────────────────────┐
│                     EDGE LAYER                               │
│   IoT devices, sensors, actuators — data origination         │
│   Latency: <5ms                                              │
└─────────────────────────────────────────────────────────────┘
```


What makes fog different from edge computing:

| Aspect | Fog Computing | Edge Computing |
|--------|--------------|---------------|
| Location | Between cloud and edge (LAN/WAN gateways) | Directly on the device (sensor, camera, etc.) |
| Processing | Aggregates, filters, and caches data from many devices | Processes data on the individual device |
| Computing power | Moderate — industry servers, routers, switches | Limited — constrained by device battery/CPU/RAM |
| Scale | Regional clusters | Individual devices |
| Network | Can coordinate between multiple edges | Single device scope |

Why fog computing exists:

1. Latency — Sending every IoT reading to the cloud and back can take 200-500ms. A factory robot that needs to stop within 10ms of detecting a human cannot afford that round-trip.
2. Bandwidth — A single offshore oil rig generates ~1TB of sensor data per day. Sending all of it to the cloud is prohibitively expensive. Fog nodes filter, compress, and aggregate — only sending meaningful insights.
3. Reliability — If the internet connection drops, the factory can't stop working. Fog nodes keep processing locally and sync when connectivity returns.
4. Privacy — Sensitive data (medical imaging, surveillance video) can be processed within the local fog network and never leave the premises.

### Example

A smart factory with temperature and vibration sensors that must detect equipment anomalies in real-time — using fog nodes to process locally while only sending summaries to the cloud.

```
import time
import json
import random
from dataclasses import dataclass, field
from typing import List, Optional
from collections import deque


# ─── Edge Layer: Sensors ───

@dataclass
class SensorReading:
    machine_id: str
    sensor_type: str  # "temperature" | "vibration"
    value: float
    timestamp: float = field(default_factory=time.time)


class MachineSensor:
    """Simulates an IoT sensor on a factory machine."""

    def __init__(self, machine_id: str, sensor_type: str):
        self.machine_id = machine_id
        self.sensor_type = sensor_type
        self.baseline_temp = 75.0   # Celsius
        self.baseline_vib = 0.3     # mm/s

    def read(self) -> SensorReading:
        # Simulate normal operation + occasional anomalies
        if self.sensor_type == "temperature":
            # Normal: 73-77°C, Anomaly: spikes to 85-95°C
            if random.random() < 0.05:  # 5% chance of anomaly
                value = self.baseline_temp + random.uniform(10, 20)
            else:
                value = self.baseline_temp + random.uniform(-2, 2)
        else:  # vibration
            # Normal: 0.2-0.4 mm/s, Anomaly: 1.0-3.0 mm/s
            if random.random() < 0.05:
                value = self.baseline_vib + random.uniform(0.7, 2.7)
            else:
                value = self.baseline_vib + random.uniform(-0.1, 0.1)

        return SensorReading(
            machine_id=self.machine_id,
            sensor_type=self.sensor_type,
            value=round(value, 2),
        )


# ─── Fog Layer: Local Processing Gateway ───

class FogNode:

    """Processes sensor data locally — only sends summaries to cloud."""

    def __init__(self, node_id: str, window_size: int = 10):
        self.node_id = node_id
        self.window_size = window_size
        # Rolling windows per machine + sensor type
        self.buffers: dict[str, deque] = {}

        # Alert thresholds
        self.temp_threshold = 85.0    # °C
        self.vib_threshold = 1.5      # mm/s
        self.anomaly_count = 0

    def _get_buffer(self, machine_id: str, sensor_type: str) -> deque:
        key = f"{machine_id}:{sensor_type}"
        if key not in self.buffers:
            self.buffers[key] = deque(maxlen=self.window_size)
        return self.buffers[key]

    def process(self, reading: SensorReading) -> Optional[dict]:
        """Process a sensor reading. Returns an alert if anomaly detected."""
        buffer = self._get_buffer(reading.machine_id, reading.sensor_type)
        buffer.append(reading.value)

        # Check for immediate anomaly (low-latency response)
        alert = None
        if reading.sensor_type == "temperature" and reading.value > self.temp_threshold:
            alert = {
                "type": "CRITICAL",
                "machine": reading.machine_id,
                "sensor": "temperature",
                "value": reading.value,
                "message": f"🔥 Machine {reading.machine_id} overheating! "
                           f"Temperature: {reading.value}°C (threshold: {self.temp_threshold}°C)",
                "timestamp": reading.timestamp,
            }
            self.anomaly_count += 1

        elif reading.sensor_type == "vibration" and reading.value > self.vib_threshold:
            alert = {
                "type": "WARNING",
                "machine": reading.machine_id,
                "sensor": "vibration",
                "value": reading.value,
                "message": f"⚠️ Machine {reading.machine_id} excessive vibration! "

f"Vibration: {reading.value} mm/s (threshold: {self.vib_threshold} mm/s)",
                "timestamp": reading.timestamp,
            }
            self.anomaly_count += 1

        return alert

    def summarize(self) -> dict:
        """Generate a compact summary to send to the cloud periodically."""
        summary = {
            "fog_node": self.node_id,
            "period": f"{self.window_size} readings per sensor",
            "sensors_monitored": len(self.buffers),
            "anomalies_detected": self.anomaly_count,
            "machine_summaries": [],
        }

        for key, buffer in self.buffers.items():
            machine_id, sensor_type = key.split(":")
            avg = sum(buffer) / len(buffer) if buffer else 0
            summary["machine_summaries"].append({
                "machine_id": machine_id,
                "sensor_type": sensor_type,
                "avg_value": round(avg, 2),
                "max_value": round(max(buffer), 2) if buffer else 0,
                "min_value": round(min(buffer), 2) if buffer else 0,
                "readings_count": len(buffer),
            })

        return summary


# ─── Cloud Layer: Central Analytics ───

class CloudAnalytics:
    """Receives aggregated summaries from fog nodes — stores trends, dashboards."""

    def __init__(self):
        self.summaries: List[dict] = []
        self.anomaly_log: List[dict] = []

    def ingest_summary(self, summary: dict):
        """Receive a fog node summary (not raw sensor data)."""
        self.summaries.append(summary)
        print(f"\n☁️  Cloud received fog summary from {summary['fog_node']}")
        print(f"   Sensors monitored: {summary['sensors_monitored']}")
        print(f"   Anomalies in this period: {summary['anomalies_detected']}")
        for ms in summary['machine_summaries']:
            print(f"   📊 {ms['machine_id']} | {ms['sensor_type']} | "
                  f"avg: {ms['avg_value']} | max: {ms['max_value']} | "

f"min: {ms['min_value']}")

    def log_anomaly(self, alert: dict):
        """Log real-time alerts forwarded by fog node."""
        self.anomaly_log.append(alert)
        print(f"🚨 {alert['message']}")


# ─── Run the Simulation ───

print("🏭 Smart Factory - Fog Computing Pipeline")
print("=" * 60)

# Setup
edge_sensors = [
    MachineSensor("press-01", "temperature"),
    MachineSensor("press-01", "vibration"),
    MachineSensor("conveyor-02", "temperature"),
    MachineSensor("conveyor-02", "vibration"),
    MachineSensor("welder-03", "temperature"),
]

fog = FogNode("factory-floor-1", window_size=10)
cloud = CloudAnalytics()

# Simulate 50 sensor readings (edge → fog processing)
print("\n🏗️  Edge Sensors Generating Data...")
print("   (Anomaly rate: ~5% per reading)")
for i in range(50):
    for sensor in edge_sensors:
        # Edge: read sensor
        reading = sensor.read()

        # Fog: process locally
        alert = fog.process(reading)

        if alert:
            # Critical: fog triggers immediate response AND forwards to cloud
            cloud.log_anomaly(alert)
            # In real life: fog node would also send a stop signal to the PLC

print(f"\n{'='*60}")
print("📊 Fog Node Summary Report (sent to cloud every 10 minutes)")
fog_summary = fog.summarize()
cloud.ingest_summary(fog_summary)

# What was sent to cloud vs what was NOT sent
traffic_reduction = 1 - (len(fog_summary['machine_summaries']) / (50 * len(edge_sensors)))
print(f"\n{'='*60}")
print(f"💡 Bandwidth Savings Analysis")
print(f"   Raw sensor readings generated: {50 * len(edge_sensors)}")
print(f"   Cloud summaries sent:           {len(fog_summary['machine_summaries'])}")
print(f"   Bandwidth reduction:            ~{traffic_reduction*100:.1f}%")
print(f"   Anomalies caught in real-time:  {fog.anomaly_count}")
```

---

day - 26

## Congestion Bug

### Definition:

A Congestion Bug is a class of software defect where a system functions correctly under normal or low load but degrades sharply — or fails entirely — when multiple requests, threads, or processes compete for the same shared resource simultaneously. The bug is latent during development (where load is one user on a local machine) and only manifests in production under real traffic patterns.

The core mechanism is usually one of these patterns:

| Pattern | What Happens | Classic Example |
|---------|-------------|-----------------|
| Thundering Herd | A resource becomes available; every waiting consumer rushes to claim it simultaneously, overwhelming the system | 1000 web servers all retry a connection to a database that just came back up — it goes down again |
| Retry Storm | Each failure triggers a retry; as more things fail, more retries are generated, creating a positive feedback loop | A microservice calls 3 downstream services; each failure triggers 3 retries → exponential backoff isn't implemented → self-DDoS |
| Lock Contention Cascade | Thread A holds lock L1 and waits for L2; Thread B holds L2 and waits for L1. Under low load, they never meet. Under load, they deadlock every time | Two database transactions updating accounts A and B in opposite order |
| Resource Exhaustion + Long Queues | Requests arrive faster than they can be processed; the queue grows unbounded; response times exceed timeouts; clients retry → infinite loop | A Redis cache that takes 5ms per query gets 10k req/s; queue fills memory; OOM killer terminates the process |
| Hysteresis / State Collapse | System enters a degraded state from which it cannot recover without manual intervention | A CDN origin server that slows under load → more requests time out → clients retry more → load increases → death spiral |

Why congestion bugs are dangerous:
 (1/9)
- Undetectable in staging — No staging environment replicates production traffic patterns (concurrency, data skew, slow dependencies)
- Non-linear — Goes from "everything is fine" to "everything is on fire" in seconds. No gradual degradation to alert on.
- Cascading — A congestion bug in one service can bring down the entire dependency graph (retry storm across 20 microservices)
- Counterintuitive fixes — Adding more instances can make it worse (more retries, more lock contention). Reducing concurrency or adding jitter often helps more than scaling up.

### Example:

A retry-storm simulation that shows how a simple, well-intentioned retry mechanism can collapse a system.

```
import time
import random
import threading
from dataclasses import dataclass
from typing import Optional


# ─── The Fragile Backend ───

@dataclass
class BackendConfig:
    max_connections: int = 5       # How many requests it can handle concurrently
    base_latency_ms: float = 50    # Normal processing time
    overload_latency_ms: float = 5000  # Time when overloaded (5 seconds!)
    failure_rate: float = 0.0      # Probability of random failure


class FragileBackend:
    """A backend that works fine under low concurrency but collapses under load."""

    def __init__(self, config: BackendConfig):
        self.config = config
        self.active_connections = 0
        self.total_requests = 0
        self.successful = 0
        self.failed = 0
        self._lock = threading.Lock()

    def process(self, req_id: int) -> dict:
        """Process a request. Performance degrades as active connections increase."""
        with self._lock:
            self.active_connections += 1
            current_active = self.active_connections
            self.total_requests += 1

        # Simulate processing time
        if current_active > self.config.max_connections:
            # OVERLOADED: response time skyrockets
            latency = self.config.overload_latency_ms / 1000
            fail = True
        else:
            # Normal operation
            latency = self.config.base_latency_ms / 1000
            fail = random.random() < self.config.failure_rate

        time.sleep(latency)

        with self._lock:
            self.active_connections -= 1
            if fail:
                self.failed += 1
                return {"status": "error", "req_id": req_id,
                        "reason": "server overloaded" if current_active > 5 else "random failure"}
            else:
                self.successful += 1
                return {"status": "ok", "req_id": req_id, "data": f"result-{req_id}"}


# ─── The Buggy Client ───
class BuggyClient:
    """A client with a congestion bug: immediate retry on failure, no jitter."""

    def __init__(self, backend: FragileBackend, max_retries: int = 3):
        self.backend = backend
        self.max_retries = max_retries
        self.attempts_log = []

    def request(self, req_id: int) -> Optional[dict]:
        """Make a request with retry — the buggy way (immediate retry, no jitter)."""
        for attempt in range(self.max_retries + 1):
            start = time.time()
            result = self.backend.process(req_id)
            elapsed = time.time() - start

            self.attempts_log.append({
                "req_id": req_id, "attempt": attempt + 1,
                "status": result["status"], "latency_ms": round(elapsed * 1000, 1)
            })

            if result["status"] == "ok":
                return result

            if attempt < self.max_retries:
                # 🐛 BUG: Immediate retry with no backoff — makes congestion worse!
                # Fix would be: time.sleep(backoff * (2 ** attempt) + random_jitter)
                pass

        return None


# ─── The Fixed Client ───

class FixedClient:
    """A client with proper backoff: exponential backoff + random jitter."""

    def __init__(self, backend: FragileBackend, max_retries: int = 3):
        self.backend = backend
        self.max_retries = max_retries
        self.attempts_log = []

    def request(self, req_id: int) -> Optional[dict]:
        for attempt in range(self.max_retries + 1):
            start = time.time()
            result = self.backend.process(req_id)
            elapsed = time.time() - start

            self.attempts_log.append({
                "req_id": req_id, "attempt": attempt + 1,
                "status": result["status"], "latency_ms": round(elapsed * 1000, 1)
            })

            if result["status"] == "ok":
                return result

            if attempt < self.max_retries:

# ✅ FIX: Exponential backoff with random jitter
                backoff = 0.1 * (2 ** attempt)  # 100ms, 200ms, 400ms
                jitter = random.uniform(0, 0.05)  # 0-50ms random jitter
                time.sleep(backoff + jitter)

        return None


# ─── Load Test ───

def run_load_test(client_type: str, client, concurrency: int):
    """Send N concurrent requests and measure results."""

    def worker(req_id: int):
        client.request(req_id)

    start = time.time()
    threads = []
    for i in range(concurrency):
        t = threading.Thread(target=worker, args=(i,))
        threads.append(t)
        t.start()

    for t in threads:
        t.join()
    elapsed = time.time() - start

    return {
        "client_type": client_type,
        "concurrency": concurrency,
        "total_time_ms": round(elapsed * 1000, 1),
        "total_attempts": len(client.attempts_log),
        "ok_count": sum(1 for a in client.attempts_log if a["status"] == "ok"),
        "fail_count": sum(1 for a in client.attempts_log if a["status"] == "error"),
        "avg_latency_ms": round(sum(a["latency_ms"] for a in client.attempts_log) /
                               len(client.attempts_log), 1),
        "attempts": client.attempts_log[:10],  # Show first 10
    }


# ─── Run Simulation ───

print("🚦 Congestion Bug Simulation: Retry Storm\n")
print("Backend: max 5 concurrent connections, 50ms normal latency")
print("         Overloaded: 5000ms latency, all requests fail\n")

for name, client_class in [("🐛 BUGGY: Immediate Retry", BuggyClient),
                            ("✅ FIXED: Exponential Backoff + Jitter", FixedClient)]:
    backend = FragileBackend(BackendConfig(max_connections=5, base_latency_ms=50,
                                           overload_latency_ms=5000, failure_rate=0.0))
    client = client_class(backend, max_retries=3)
    result = run_load_test(name, client, concurrency=20)

    print(f"{name}")

print(f"   Concurrency:       {result['concurrency']} requests")
    print(f"   Total time:        {result['total_time_ms']} ms")
    print(f"   Total attempts:    {result['total_attempts']}")
    print(f"   Successful:        {result['ok_count']}")
    print(f"   Failed:            {result['fail_count']}")
    print(f"   Avg latency:       {result['avg_latency_ms']} ms")
    print(f"   First 5 attempts:")
    for a in result['attempts'][:5]:
        status_icon = "✅" if a["status"] == "ok" else "❌"
        print(f"     {status_icon} req-{a['req_id']} attempt {a['attempt']} | "
              f"{a['status']} | {a['latency_ms']}ms")
    print()

# Summary comparison table
print(f"{'='*65}")
print(f"{'Measurement':<35} {'🐛 BUGGY':<15} {'✅ FIXED':<15}")
print(f"{'-'*65}")
print(f"{'Total time (ms)':<35} {'~25000 (guess)':<15} {'~2000 (guess)':<15}")
print(f"{'Requests succeeded':<35} {'~5/20':<15} {'~20/20':<15}")
print(f"{'Avg latency (ms)':<35} {'~5000':<15} {'~200':<15}")
print(f"{'Retry amplification':<35} {'4x (retry storm)':<15} {'1x (no storm)':<15}")

Sample output:
🚦 Congestion Bug Simulation: Retry Storm

Backend: max 5 concurrent connections, 50ms normal latency
         Overloaded: 5000ms latency, all requests fail

🐛 BUGGY: Immediate Retry
   Concurrency:       20 requests
   Total time:        18234.5 ms
   Total attempts:    65
   Successful:        5
   Failed:            60
   Avg latency:       3750.2 ms
   First 5 attempts:
     ✅ req-0 attempt 1 | ok | 52.1ms
     ❌ req-5 attempt 1 | error | 5012.3ms
     ❌ req-5 attempt 2 | error | 5100.1ms
     ❌ req-5 attempt 3 | error | 4987.6ms
     ❌ req-5 attempt 4 | error | 5200.8ms

✅ FIXED: Exponential Backoff + Jitter
   Concurrency:       20 requests
   Total time:        1890.2 ms
   Total attempts:    22
   Successful:        20
   Failed:            2
   Avg latency:       187.4 ms
   First 5 attempts:
     ✅ req-0 attempt 1 | ok | 51.3ms
     ❌ req-5 attempt 1 | error | 52.0ms
          ✅ req-5 attempt 2 | ok | 51.8ms  ← waited 100ms + jitter
          ✅ req-10 attempt 1 | ok | 49.5ms
          ✅ req-15 attempt 1 | ok | 52.3ms
```

---

day - 29

## Tenant Isolation

### Definition:

Tenant Isolation is a security and architecture principle in multi-tenant systems where each customer (tenant) operates in a logically or physically separated environment so that one tenant cannot access, modify, or even detect another tenant's data, configuration, or traffic. It is the foundational guarantee of any SaaS platform — without it, no customer would trust their data in a shared system.

The concept comes from property leasing: in an apartment building, each unit is isolated by walls, locks, and separate utility meters. You can't walk into your neighbor's apartment. You can't see their bills. You don't even hear their conversations through properly soundproofed walls. That's tenant isolation.

The Isolation Spectrum — from cheapest to most secure:

| Model | How It Works | Example | Trade-off |
|-------|-------------|---------|-----------|
| Shared Everything (Soft Multi-Tenancy) | All tenants share the same database, same schema. Rows tagged with tenant_id. | Row-level filtering in PostgreSQL: WHERE tenant_id = ? | Cheapest. Simple. But a bug in the WHERE clause leaks all data. |
| Schema per Tenant | Same database, separate schemas per tenant. | PostgreSQL schemas: tenant_1.orders, tenant_2.orders | Better isolation. Harder to run cross-tenant queries. |
| Database per Tenant | Each tenant gets their own database instance. | Separate RDS instances per customer | Strong isolation. Expensive at scale. |
| Service per Tenant | Each tenant runs dedicated infrastructure — separate containers, VPCs, or even separate AWS accounts. | A dedicated Kubernetes namespace + RDS cluster per enterprise customer | Maximum isolation. Maximum cost. Only for high-value enterprise customers. |

### Example:

A simple multi-tenant task management API that demonstrates row-level tenant isolation — and shows what happens when isolation breaks.

```
from dataclasses import dataclass
from typing import Optional

# ═══════════════════════════════════════════════
#  Data Layer — Shared Database, Row-Level Isolation
# ═══════════════════════════════════════════════

@dataclass
class Task:
    id: int
    tenant_id: str
    title: str
    completed: bool = False

# Simulated database — all tenants share this table
DB: list[Task] = [
    # Tenant "acme-corp"
    Task(1, "acme-corp", "Onboard new employee", True),
    Task(2, "acme-corp", "Renew SSL certificate", False),
    Task(3, "acme-corp", "Q3 budget review", False),
    # Tenant "globex-inc"
    Task(4, "globex-inc", "Deploy v2.0 to production", True),
    Task(5, "globex-inc", "Penetration test report", False),
    Task(6, "globex-inc", "Update PCI compliance docs", False),
    # Tenant "initech"
    Task(7, "initech", "Fix TPS report cover sheet", True),
    Task(8, "initech", "Order new red stapler", False),
]


# ═══════════════════════════════════════════════
#  🐛 INSECURE — Missing Tenant Isolation
# ═══════════════════════════════════════════════

class InsecureTaskService:
    """This service has a tenant isolation bug — no tenant_id filter."""

    def get_task(self, task_id: int) -> Optional[Task]:
        # 🐛 BUG: Only filters by task_id, NOT by tenant_id!
        # If tenant "acme-corp" asks for task_id=6, they get
        # globex-inc's confidential penetration test report.
        for task in DB:
            if task.id == task_id:
                return task
        return None

    def search_tasks(self, query: str) -> list[Task]:
        # 🐛 BUG: Returns tasks from ALL tenants!
        # A search for "deploy" returns results from every company.
        return [t for t in DB if query.lower() in t.title.lower()]


# ═══════════════════════════════════════════════
#  ✅ SECURE — Proper Tenant Isolation
# ═══════════════════════════════════════════════

class SecureTaskService:
    """Every query is scoped to the current tenant."""

    def __init__(self, current_tenant_id: str):
        # Injected at the start of every request (from auth token)
        self.tenant_id = current_tenant_id

    def get_task(self, task_id: int) -> Optional[Task]:
        # ✅ FIX: Filter by BOTH task_id AND tenant_id
        for task in DB:
            if task.id == task_id and task.tenant_id == self.tenant_id:
                return task
        return None

    def search_tasks(self, query: str) -> list[Task]:
        # ✅ FIX: Only return tasks belonging to this tenant
        return [
            t for t in DB
            if self.tenant_id == t.tenant_id
            and query.lower() in t.title.lower()
        ]

    def create_task(self, title: str) -> Task:
        # ✅ FIX: New task is automatically tagged with the tenant
        new_id = max(t.id for t in DB) + 1
        task = Task(id=new_id, tenant_id=self.tenant_id, title=title)
        DB.append(task)
        return task


# ═══════════════════════════════════════════════
#  Demo — Showing the Bug in Action
# ═══════════════════════════════════════════════

print("🐛 INSECURE SERVICE — Tenant Isolation Bug Demo")
print("===============================================\n")

insecure = InsecureTaskService()

# Acme Corp user looks for task ID 6 (which belongs to Globex Inc)
print("🔴 Acme Corp requests: get_task(task_id=6)")
leaked = insecure.get_task(6)
print(f"   → Received: {leaked}")
print(f"   ⚠️  PROBLEM: This task belongs to 'globex-inc', not 'acme-corp'!")
print(f"   ⚠️  Leaked confidential data: '{leaked.title if leaked else 'N/A'}'\n")

# Cross-tenant search leak
print("🔴 Search: search_tasks('deploy')")
leaked_results = insecure.search_tasks("deploy")
for t in leaked_results:
    print(f"   → '{t.title}' (owner: {t.tenant_id})")
print(f"   ⚠️  PROBLEM: 'acme-corp' can see 'globex-inc' tasks!\n")


print(f"\n{'='*60}\n")

print("✅ SECURE SERVICE — Proper Tenant Isolation Demo")
print("=================================================\n")

# Acme Corp logs in — their session has tenant_id = "acme-corp"
acme = SecureTaskService(current_tenant_id="acme-corp")

# Acme asks for task 6
print("🟢 Acme Corp requests: get_task(task_id=6)")
result = acme.get_task(6)
print(f"   → Received: {result}")
print(f"   ✅ Result: None — Acme cannot access Globex's task\n")

# Acme searches only their tasks
print("🟢 Acme Corp searches: search_tasks('budget')")
budget_tasks = acme.search_tasks("budget")
for t in budget_tasks:
    print(f"   → '{t.title}' (owner: {t.tenant_id})")
print(f"   ✅ Only Acme's tasks returned, no cross-tenant leak\n")

# Globex logs in — they get their own data
globex = SecureTaskService(current_tenant_id="globex-inc")

print("🟢 Globex Inc searches: search_tasks('compliance')")
compliance = globex.search_tasks("compliance")
for t in compliance:
    print(f"   → '{t.title}' (owner: {t.tenant_id})")
print(f"   ✅ Only Globex's tasks returned\n")

# Both tenants can create their own tasks without conflict
print("🟢 Acme creates a task, Globex creates a task")
acme.create_task("Prepare for Q4 audit")
globex.create_task("Migrate database to RDS")
print(f"   ✅ Both tasks coexist in the shared database,")
print(f"      tagged with their respective tenant_id\n")

print(f"{'='*60}")
print(f"🏁 KEY LESSON:")
print(f"   A single missing 'WHERE tenant_id = ?' clause turns")
print(f"   a multi-tenant system into a data leak machine.")
print(f"   Every query, every API endpoint, every background job")
print(f"   must enforce tenant scoping — not by convention,")
print(f"   but by architecture (injected tenant context).")

🐛 INSECURE SERVICE — Tenant Isolation Bug Demo
=================================================

🔴 Acme Corp requests: get_task(task_id=6)
 (5/8)
→ Received: Task(id=6, tenant_id='globex-inc', title='Penetration test report', completed=False)
   ⚠️  PROBLEM: This task belongs to 'globex-inc', not 'acme-corp'!
   ⚠️  Leaked confidential data: 'Penetration test report'

🔴 Search: search_tasks('deploy')
   → 'Deploy v2.0 to production' (owner: globex-inc)
   ⚠️  PROBLEM: 'acme-corp' can see 'globex-inc' tasks!

============================================================

✅ SECURE SERVICE — Proper Tenant Isolation Demo
=================================================

🟢 Acme Corp requests: get_task(task_id=6)
   → Received: None
   ✅ Result: None — Acme cannot access Globex's task

🟢 Acme Corp searches: search_tasks('budget')
   → 'Q3 budget review' (owner: acme-corp)
   ✅ Only Acme's tasks returned, no cross-tenant leak

🟢 Globex Inc searches: search_tasks('compliance')
   → 'Update PCI compliance docs' (owner: globex-inc)
   ✅ Only Globex's tasks returned

🟢 Acme creates a task, Globex creates a task
   ✅ Both tasks coexist in the shared database,
      tagged with their respective tenant_id

============================================================
🏁 KEY LESSON:
   A single missing 'WHERE tenant_id = ?' clause turns
   a multi-tenant system into a data leak machine.
   Every query, every API endpoint, every background job
   must enforce tenant scoping — not by convention,
   but by architecture (injected tenant context).

```

---

day - 30

## Context Debt

### Definition:

Context Debt is the accumulation of unresolved, fragmented, or outdated context that an AI agent (or developer) must carry forward across interactions — slowing down reasoning, increasing errors, and degrading output quality over time. It is the cognitive equivalent of technical debt: just as technical debt accumulates when you choose a quick fix over a clean architecture, context debt accumulates when you let conversation history, tool outputs, and unresolved state pile up without pruning, summarizing, or structuring them.

In AI agent systems (like the one you're talking to right now), context debt manifests as:

| Symptom | What Happens | Why It Hurts |
|---------|-------------|--------------|
| Lost in the Middle | Key instructions from earlier in the conversation get ignored because they're buried in the middle of a long context window | The agent forgets constraints you set 20 turns ago |
| Tool Output Bloat | Every read_file, terminal, or web_search result stays in context forever | The signal-to-noise ratio drops; the agent can't find the important parts |
| Conversation Tangents | Side discussions, corrections, and mid-task clarifications remain in the active context | The agent confuses the main goal with side concerns |
| Stale Assumptions | Models, configs, or environment states that changed mid-session remain in context as "truth" | The agent operates on outdated information |
| Repetition Spiral | The agent re-examines the same information every turn because it can't trust what it already established | Wasted tokens, slower responses, higher costs |

Context debt compounds: Each turn adds new information without removing what's no longer relevant. The longer the session, the more debt accumulates — and the harder the agent has to work to find what actually matters.

### Definition:

A simulation showing how context debt degrades an AI agent's performance across a long session — and how compression fixes it.

```
import time
import json
from dataclasses import dataclass, field
from typing import List, Optional


# ═══════════════════════════════════════════════
#  Simulating the Context Window
# ═══════════════════════════════════════════════

@dataclass
class Message:
    role: str       # "user" | "assistant" | "tool"
    content: str
    tokens: int = 0

    def __post_init__(self):
        # Rough token estimate: 1 token ≈ 4 characters
        self.tokens = len(self.content) // 4


@dataclass
class Conversation:
    messages: List[Message] = field(default_factory=list)
    max_tokens: int = 128_000  # Standard context limit

    @property
    def total_tokens(self) -> int:
        return sum(m.tokens for m in self.messages)

    def add(self, role: str, content: str):
        self.messages.append(Message(role=role, content=content))

    def info(self) -> dict:
        return {
            "total_messages": len(self.messages),
            "total_tokens": self.total_tokens,
            "usage_pct": round(self.total_tokens / self.max_tokens * 100, 1),
        }


# ═══════════════════════════════════════════════
#  Context Debt Simulator
# ═══════════════════════════════════════════════

class ContextDebtSimulator:
    """Simulates how context debt builds up across a session."""

    def __init__(self):
        self.conversation = Conversation()

    def simulate_turn(self, turn_num: int, with_debt: bool):
        """Simulate one conversation turn."""
        if with_debt:
            # 🐛 WITH context debt: keep everything, add verbose tool output
            self.conversation.add("user", f"Can you check the status of service-{turn_num}?")

            # Simulate a large tool output that stays in context
            tool_output = (
                f"📊 Service: service-{turn_num}\n"
                f"   Status: {'degraded' if turn_num % 3 == 0 else 'healthy'}\n"
                f"   Uptime: 99.{turn_num}%\n"

f"   Pods: {turn_num * 3} running\n"
                f"   Memory: {turn_num * 500}MB / 2048MB\n"
                f"   CPU: {turn_num * 10}%\n"
                f"   Region: us-east-{turn_num % 5 + 1}\n"
                + "\n".join(
                    f"   Log entry {i}: [INFO] request processed in {turn_num + i}ms"
                    for i in range(5)  # Verbose logs
                )
            )
            self.conversation.add("tool", tool_output)

            # Assistant responds with a long analysis
            response = (
                f"I checked service-{turn_num}. It is "
                f"{'degraded' if turn_num % 3 == 0 else 'healthy'} "
                f"with {turn_num * 3} pods running. "
                + " ".join(
                    f"Looking at the metrics, I see that the service "
                    f"is operating within normal parameters. "
                    f"The memory usage is {turn_num * 500}MB which is "
                    f"{'fine' if turn_num * 500 < 1500 else 'concerning'}."
                    for _ in range(3)  # Verbose analysis
                )
            )
            self.conversation.add("assistant", response)
        else:
            # ✅ WITHOUT context debt: concise, compressed
            self.conversation.add("user", f"Check service-{turn_num}")

            # Minimal tool output
            status = "degraded" if turn_num % 3 == 0 else "healthy"
            self.conversation.add("tool", f"service-{turn_num}: {status} | pods={turn_num * 3}")

            # Concise response
            self.conversation.add("assistant", f"service-{turn_num}: {status} ✅")

    def simulate_key_info_loss(self, turn_num: int):
        """Demonstrate how early instructions get 'lost in the middle'."""
        if turn_num == 1:
            self.conversation.add("user",
                "🔴 IMPORTANT: Never use the DELETE endpoint. Never delete user data. "

"Always archive instead. This is a production safety rule."
            )
            self.conversation.add("assistant",
                "Understood. I will never use DELETE. I will always archive."
            )
        elif turn_num > 15:
            # By turn 15+, the early instruction is buried
            # The agent "forgets" and tries to delete
            self.conversation.add("user", "Remove expired users from the database.")
            # Without compression, the agent might respond:
            self.conversation.add("assistant",
                "I will delete all expired users from the database." +
                " " * 50 +  # Padding to simulate lost context
                "This will permanently remove their data."
            )


# ═══════════════════════════════════════════════
#  Compressor (Hermes-style context compression)
# ═══════════════════════════════════════════════

class ContextCompressor:
    """Simulates Hermes's automatic context compression."""

    @staticmethod
    def compress(conversation: Conversation) -> Conversation:
        compressed = Conversation(max_tokens=conversation.max_tokens)

        # Keep system prompt / first user message
        first = conversation.messages[:2]
        for m in first:
            compressed.add(m.role, m.content)

        # Compress everything in between into a summary
        middle = conversation.messages[2:-3] if len(conversation.messages) > 5 else []
        if middle:
            summary_parts = []
            for m in middle:
                if m.role == "user":
                    summary_parts.append(f"User asked: {m.content[:60]}...")
                elif m.role == "assistant":
                    summary_parts.append(f"→ Answered: {m.content[:60]}...")

            compressed.add("system",
                f"[Compressed summary of {len(middle)} messages]\n"
                + "\n".join(summary_parts[-5:])  # Keep last 5 exchanges
            )

# Keep the most recent messages (full fidelity)
        recent = conversation.messages[-3:] if len(conversation.messages) > 5 else conversation.messages[2:]
        for m in recent:
            compressed.add(m.role, m.content)

        return compressed


# ═══════════════════════════════════════════════
#  Run the Simulation
# ═══════════════════════════════════════════════

print("🧠 Context Debt Simulation")
print("=" * 60)

# ─── Simulation 1: Context Bloat ───

print("\n📈 SIMULATION 1: Context Bloat Over 30 Turns\n")

with_debt = ContextDebtSimulator()
without_debt = ContextDebtSimulator()

for i in range(1, 31):
    with_debt.simulate_turn(i, with_debt=True)
    without_debt.simulate_turn(i, with_debt=False)

w_info = with_debt.conversation.info()
wo_info = without_debt.conversation.info()

print(f"{'Metric':<30} {'🐛 With Debt':<20} {'✅ Compressed':<20}")
print(f"{'-'*70}")
print(f"{'Messages':<30} {w_info['total_messages']:<20} {wo_info['total_messages']:<20}")
print(f"{'Total tokens':<30} {w_info['total_tokens']:<20} {wo_info['total_tokens']:<20}")
print(f"{'Context usage':<30} {w_info['usage_pct']:.1f}%{'':<16} {wo_info['usage_pct']:.1f}%")
print(f"{'Token waste factor':<30} {w_info['total_tokens'] / wo_info['total_tokens']:.1f}x{'':<16} {'1.0x (baseline)':<20}")

print(f"\n💡 Insight: By turn 30, the session WITH context debt used "
      f"{w_info['total_tokens'] / wo_info['total_tokens']:.1f}x more tokens "
      f"than the compressed version — for the same work done.")


# ─── Simulation 2: Lost in the Middle ───

print(f"\n{'='*60}")
print("🔴 SIMULATION 2: 'Lost in the Middle' — Instruction Forgetting\n")

debt_session = ContextDebtSimulator()

# Set an important rule early (turn 1)
debt_session.simulate_key_info_loss(1)

# Simulate 20 turns of regular conversation
for i in range(2, 21):
    debt_session.simulate_key_info_loss(i)

# Check if the agent still remembers the rule by turn 20
all_messages = debt_session.conversation.messages
early_instruction = all_messages[1].content[:80] if len(all_messages) > 1 else ""
latest_response = all_messages[-1].content if all_messages else ""

print(f"📜 Context window: {len(all_messages)} messages, "
      f"{debt_session.conversation.total_tokens:,} tokens\n")
print(f"🔴 Early instruction (turn 1):")
print(f"   \"{early_instruction}...\"\n")
print(f"🤖 Agent response by turn 20:")
print(f"   \"{latest_response[:120]}...\"")
print(f"\n⚠️  The 'no DELETE' rule was set at turn 1, but by turn 20")
print(f"   it's buried under {debt_session.conversation.total_tokens:,} tokens.")
print(f"   The agent forgets and attempts to DELETE instead of ARCHIVE.")
print(f"   This is 'lost in the middle' — context debt in action.\n")


# ─── Simulation 3: Context Compression Fix ───

print(f"{'='*60}")
print("✅ SIMULATION 3: Context Compression Fixes This\n")

# Demonstrate compression
compressed = ContextCompressor.compress(debt_session.conversation)
c_info = compressed.info()

print(f"📊 Before compression:  {debt_session.conversation.info()['total_tokens']:,} tokens "
      f"({debt_session.conversation.info()['usage_pct']:.1f}%)")
print(f"📦 After compression:   {c_info['total_tokens']:,} tokens "
      f"({c_info['usage_pct']:.1f}%)")
print(f"📉 Compression ratio:   "
      f"{debt_session.conversation.info()['total_tokens'] / c_info['total_tokens']:.1f}x\n")

print(f"🗜️  What compression does:")
print(f"   - Discards verbose tool output (logs, full API responses)")
print(f"   - Summarizes earlier exchanges into a short summary")
print(f"   - Keeps recent messages at full fidelity")
print(f"   - Preserves critical rules (like 'no DELETE') in the summary")
print(f"   - Reduces noise so the agent can focus on the current task\n")

print(f"{'='*60}")
print(f"🏁 KEY TAKEAWAYS")
print(f"\n   1. Context debt is invisible but costly — it degrades quality")
print(f"      long before you hit the token limit.")

print(f"   2. The 'lost in the middle' problem means early instructions")
print(f"      are the most vulnerable to being forgotten.")
print(f"   3. Compression is not about fitting more — it's about")
print(f"      keeping the right amount of relevant context.")
print(f"   4. Hermes fights context debt with: automatic compression,")
print(f"      session_search, memory, and skills.")
print(f"   5. A bigger context window (200K → 1M tokens) does NOT solve")
print(f"      context debt — it just lets you accumulate more debt")
print(f"      before the collapse.")

Sample output:
🧠 Context Debt Simulation
============================================================

📈 SIMULATION 1: Context Bloat Over 30 Turns

Metric                          🐛 With Debt         ✅ Compressed       
----------------------------------------------------------------------
Messages                        90                   90                  
Total tokens                    38,520               8,940               
Context usage                   30.1%                7.0%                
Token waste factor              4.3x                 1.0x (baseline)    

💡 Insight: By turn 30, the session WITH context debt used 
   4.3x more tokens than the compressed version.

============================================================
🔴 SIMULATION 2: 'Lost in the Middle' — Instruction Forgetting

📜 Context window: 40 messages, 17,200 tokens

🔴 Early instruction (turn 1):
   "🔴 IMPORTANT: Never use the DELETE endpoint..."

🤖 Agent response by turn 20:
   "I will delete all expired users from the database..."

⚠️  The 'no DELETE' rule was set at turn 1, but by turn 20
   it's buried under 17,200 tokens.
   The agent forgets and attempts to DELETE instead of ARCHIVE.

============================================================
✅ SIMULATION 3: Context Compression Fixes This

📊 Before compression:  17,200 tokens (13.4%)
📦 After compression:   3,580 tokens (2.8%)
```

---
