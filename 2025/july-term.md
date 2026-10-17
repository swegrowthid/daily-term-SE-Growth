day - 1

## Deadlock

### Definition:

A deadlock is a situation where two or more processes are blocked indefinitely, each waiting for the other to release a resource that they need to continue execution. None of the processes can proceed, creating a circular dependency.

### Example:

- Banking System Scenario

```
# Process A needs Account1 and Account2
# Process B needs Account2 and Account1

Process A:
1. Lock Account1 ✅
2. Wait for Account2 ⏳ (held by Process B)

Process B:
1. Lock Account2 ✅
2. Wait for Account1 ⏳ (held by Process A)

Result: Both processes wait forever! 🔄
```

- Traffic Intersection Analogy
  Imagine four cars at a 4-way intersection:

1. Each car occupies one lane
2. Each needs the lane occupied by the next car
3. No car can move forward
4. Result: Complete gridlock!

---

day - 2

## Behavioral Design Patterns

### Definition:

Behavioral design patterns are a category of design patterns that focus on the interactions and communication between objects. They help define how objects collaborate and distribute responsibility among them, making it easier to manage complex control flow and communication in a system.

### Example:

- Chain of Responsibility
- Command
- Iterator
- Mediator
- Memento
- Observer
- State
- Strategy
- Template Method
- Visitor

reference: [Link-1](https://refactoring.guru/design-patterns/behavioral-patterns)
[Link-2](https://www.geeksforgeeks.org/system-design/behavioral-design-patterns)

---

day - 3

## Database Normalization

### Definition:

Database Normalization is the process of organizing data in a database to reduce redundancy and improve data integrity by dividing large tables into smaller, related tables and defining relationships between them.
**Normalization Forms:**
1NF: Each column contains atomic values, no repeating groups
2NF: 1NF + All non-key attributes fully depend on primary key
3NF: 2NF + No transitive dependencies
BCNF: 3NF + Every determinant is a candidate key

### Example:

**Student Database Normalization**

Before Normalization:

```
Students Table:
+----+----------+-------------+------------------+----------------+
| ID | Name     | Courses     | Instructors      | Departments    |
+----+----------+-------------+------------------+----------------+
| 1  | John     | Math, CS    | Smith, Johnson   | Math, CS       |
| 2  | Sarah    | Physics     | Brown            | Physics        |
+----+----------+-------------+------------------+----------------+
```

After 1NF(First Normal Form):

```
Students Table:
+----+----------+
| ID | Name     |
+----+----------+
| 1  | John     |
| 2  | Sarah    |
+----+----------+

Student_Courses Table:
+----+---------+------------+------------+
| ID | Course  | Instructor | Department |
+----+---------+------------+------------+
| 1  | Math    | Smith      | Math       |
| 1  | CS      | Johnson    | CS         |
| 2  | Physics | Brown      | Physics    |
+----+---------+------------+------------+
```

After 2NF & 3NF:

```
Students Table:
+----+----------+
| ID | Name     |
+----+----------+
| 1  | John     |
| 2  | Sarah    |
+----+----------+

Courses Table:
+-----------+-------------+----------------+
| Course_ID | Course_Name | Department_ID  |
+-----------+-------------+----------------+
| 101       | Math        | 1              |
| 102       | CS          | 2              |
| 103       | Physics     | 3              |
+-----------+-------------+----------------+

Departments Table:
+---------------+-----------------+
| Department_ID | Department_Name |
+---------------+-----------------+
| 1             | Mathematics     |
| 2             | Computer Science|
| 3             | Physics         |
+---------------+-----------------+

Instructors Table:
+---------------+----------+---------------+
| Instructor_ID | Name     | Department_ID |
+---------------+----------+---------------+
| 1             | Smith    | 1             |
| 2             | Johnson  | 2             |
| 3             | Brown    | 3             |
+---------------+----------+---------------+

Enrollments Table:
+------------+-----------+---------------+
| Student_ID | Course_ID | Instructor_ID |
+------------+-----------+---------------+
| 1          | 101       | 1             |
| 1          | 102       | 2             |
| 2          | 103       | 3             |
+------------+-----------+---------------+
```

reference: [Link-1](https://www.geeksforgeeks.org/dbms/introduction-of-database-normalization/)
[Link-2](https://www.datacamp.com/tutorial/normalization-in-sql)

---

day - 4

## Test Driven Development

### Definition:

Test Driven Development (TDD) is a software development methodology where you write tests before writing the actual code. It follows a simple cycle: Red → Green → Refactor.
TDD Cycle:

- Red: Write a failing test
- Green: Write minimal code to make the test pass
- Refactor: Improve the code while keeping tests passing

### Example:

Calculator Addition Function
**Step 1: Red - Write Failing Test**

```
import unittest

class TestCalculator(unittest.TestCase):
    def test_add_two_numbers(self):
        calc = Calculator()
        result = calc.add(2, 3)
        self.assertEqual(result, 5)

    def test_add_negative_numbers(self):
        calc = Calculator()
        result = calc.add(-1, -2)
        self.assertEqual(result, -3)

if __name__ == '__main__':
    unittest.main()
```

**Step 2: Green - Write Minimal Code**

```
class Calculator:
    def add(self, a, b):
        return a + b
```

**Step 3: Refactor - Improve Code**

```
class Calculator:
    def add(self, a: int, b: int) -> int:
        """Add two numbers and return the result."""
        if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
            raise TypeError("Both arguments must be numbers")
        return a + b
```

reference: [Link-1](https://martinfowler.com/bliki/TestDrivenDevelopment.html)

---

day - 7

## Lazy Loading

### Definition:

Lazy Loading is a design pattern that delays the initialization or loading of resources until they are actually needed, rather than loading everything upfront. This improves performance by reducing initial load time and memory usage.

**Benefits vs Trade-offs**

- Benefits: Faster initial load, Lower memory usage, Better user experience
- Trade-offs: Potential delay when accessing, More complex implementation, Possible loading indicators needed

### Example:

- Web Development - Image Lazy Loading

```
<!-- Images load only when they come into viewport -->
<img src="placeholder.jpg" data-src="actual-image.jpg" loading="lazy" alt="Example">
```

- JavaScript - Module Lazy Loading

```
// Module is loaded only when function is called
async function loadFeature() {
    const module = await import('./heavy-feature.js');
    return module.default();
}
```

---

day - 8

## Black Box Testing

### Definition:

Black Box Testing is a testing technique that mainly focuses on testing the system functionalities without knowing the actual implementations.

### Example:

There are many testing techniques that actually implement Black Box Testing:

- E2E Testing (End-to-End Testing)
- System Testing
- User Acceptance Testing (UAT)

These are the tool examples for Black Box Testing

| **Tool**   | **Description**                                                                                                                |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Postman    | E2E API tool. Support other protocols like graphQL and gRPC                                                                    |
| Playwright | E2E Web automation testing tool. Using JS / TS as a primary language                                                           |
| Katalon    | E2E Multi-platform automation tool. Suitable for testing across different platforms like API, web and mobile. Free for 30 days |

---

## Eager Loading

### Definition:

Eager Loading is a technique where data or resources are loaded immediately when they are declared or referenced, rather than waiting until they are actually needed. This approach prioritizes upfront loading to ensure data is readily available when accessed.

- **Eager Loading vs Lazy Loading**
  |Aspect |Eager Loading |Lazy Loading |
  |--- |--- |--- |
  |Loading Time |Load immediately/upfront |Load when needed |
  |Memory Usage |Higher initial memory consumption |Lower initial memory usage |
  |Performance |Faster access after loading |Slower first access, faster subsequent |
  |Network Requests |More upfront requests |Fewer initial requests |
  |Use Case |Critical data needed immediately |Optional/conditional data |

### Example:

- Database Query

```
# Eager Loading Example - SQLAlchemy
from sqlalchemy.orm import joinedload

# Load user with all their posts immediately
user = session.query(User)\
    .options(joinedload(User.posts))\
    .filter(User.id == 1)\
    .first()

# Posts are already loaded - no additional query needed
print(user.posts)  # ✅ Data already available
```

- Javascript

```
// Eager Loading Example - JavaScript
class ImageGallery {
    constructor(imageUrls) {
        this.images = [];
        this.loadAllImages(imageUrls); // Load immediately
    }

    loadAllImages(urls) {
        urls.forEach(url => {
            const img = new Image();
            img.src = url; // Load all images upfront
            this.images.push(img);
        });
    }
}

// All images start loading immediately
const gallery = new ImageGallery(['img1.jpg', 'img2.jpg', 'img3.jpg']);
```

---

day - 9

## CARL Interview Method

### Definition:

CARL is a structured interview technique used to evaluate candidates' past experiences and problem-solving abilities through specific storytelling components:

- Context - The situation or background
- Action - What the candidate specifically did
- Result - The outcome achieved
- Learning - What was learned from the experience

### Example:

**Leadership Challenge**

Question: "Tell me about a time when you had to lead a difficult project."

Candidate's CARL Response:

🔍 Context:

"In my previous role as a software team lead, we had a critical e-commerce platform that was experiencing 40% performance degradation during peak hours, affecting customer sales."

⚡ Action:

"I assembled a cross-functional team, implemented daily standups, analyzed the codebase to identify bottlenecks, prioritized database optimization, and coordinated with DevOps for infrastructure scaling."

📊 Result:

"We reduced page load times by 60%, improved system stability to 99.8% uptime, and recovered $2M in potential lost sales within 3 weeks."

📚 Learning:

"I learned the importance of proactive monitoring and that effective leadership requires both technical problem-solving and clear communication across departments."

---

day - 10

## RAFT algorithm

### Definition:

RAFT (Replicated And Fault Tolerant) is a consensus algorithm designed to manage distributed systems by ensuring all nodes agree on a shared state, even when some nodes fail. It's simpler and more understandable than Paxos.

Key Concepts

- Leader Election: One node becomes the leader to coordinate operations
- Log Replication: Leader replicates entries to follower nodes
- Safety: Ensures consistency across all nodes

### Example:

Banking System

Scenario: 5-node banking cluster processing account transfers

```
Initial State:
Node A (Leader) | Node B | Node C | Node D | Node E
Balance: $1000  | $1000  | $1000  | $1000  | $1000
```

Step 1: Client Request

```
Client → Transfer $200 from Account X to Account Y
```

Step 2: Leader Processing

```
Node A (Leader):
1. Receives transfer request
2. Creates log entry: "Transfer $200 X→Y"
3. Sends to followers for replication
```

Step 3: Consensus

```
Node A → Node B: "Append log entry #5"
Node A → Node C: "Append log entry #5"
Node A → Node D: "Append log entry #5"
Node A → Node E: "Append log entry #5"

Responses:
✅ Node B: "Committed"
✅ Node C: "Committed"
❌ Node D: "Failed" (network issue)
✅ Node E: "Committed"

Result: 3/5 nodes = Majority achieved ✅
```

Step 4: Commit

```
Final State (majority nodes):
Node A | Node B | Node C | Node E
Balance: $800   | $800   | $800   | $800
(Node D will sync when reconnected)
```

---

day - 11

## Currying (Functional Programming Technique)

### Definition:

Currying is a technique where a function with multiple arguments is transformed into a sequence of functions, each taking a single argument.

### Example:

```
def multiply(x):
    def inner(y):
        return x * y
    return inner

# Usage:
mul_by_3 = multiply(3)
result = mul_by_3(5)   # result is 15

# Or, in one line:
result = multiply(3)(5)  # result is 15

```

Explanation:

```
multiply(3)
```

returns a function that multiplies its argument by 3.

---

day - 14

## Replica Technique in System Design

### Definition:

Replica Technique is a design pattern used in distributed systems to enhance reliability, availability, and performance by creating multiple copies (replicas) of data or services across different nodes.

Key Benefits
⚡ High Availability - System remains operational even if some replicas fail
🚀 Performance - Reduced latency through geographical distribution
🛡️ Fault Tolerance - Data redundancy prevents single point of failure
📈 Scalability - Load distribution across multiple replicas

Types of Replication:

- **Master-Slave Replication**: One master node handles writes, while multiple slave nodes handle reads. Use case: Read-heavy applications like web servers.
- **Master-Master Replication**: Multiple nodes can handle both reads and writes. Use case: High write availability systems where any node can accept writes.
- **Peer-to-Peer**: All nodes are equal, and each can read/write data. Use case: disributed systems like blockchain.

### Example:

E-commerce Database Replication(scenario: Amazon-like online store)

```text
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Master    │    │   Slave 1   │    │   Slave 2   │
│  Database   │───▶│  (US East)  │    │  (Europe)   │
│   (Main)    │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
   All Writes         Read Queries        Read Queries
   (Orders, Users)    (Product Info)     (Product Info)
```

Implementation Strategy

- Master Database 🎯

Handles all write operations (new orders, user registration)
Located in primary data center

- Read Replicas 📖

Handle read queries (product searches, user profiles)
Distributed globally for reduced latency

- Replication Process 🔄

```
-- Master receives write
INSERT INTO orders (user_id, product_id, amount)
VALUES (123, 456, 99.99);

-- Changes replicated to slaves asynchronously
-- Users in different regions read from nearest replica
```

---

day - 15

## Functional Programming

### Definition:

Functional Programming is a programming paradigm that treats computation as the evaluation of mathematical functions, emphasizing immutability, pure functions, and avoiding side effects.

Key Principles:

- Pure Functions - Same input always produces same output
- Immutability - Data cannot be changed after creation
- No Side Effects - Functions don't modify external state
- Higher-Order Functions - Functions can take other functions as arguments or return them

Popular Functional Languages: Haskell, Lisp, Clojure, F#, Erlang, Scala

### Example:

JavaScript - Pure vs Impure Function

```
// IMPURE - Modifies external state
let counter = 0;
function increment() {
    counter++; // Side effect
    return counter;
}

// PURE - No side effects, same input = same output
function add(a, b) {
    return a + b;
}
```

Python - Functional Approach

```
# Functional style with map, filter, reduce
numbers = [1, 2, 3, 4, 5]

# Using higher-order functions
squared = list(map(lambda x: x**2, numbers))
evens = list(filter(lambda x: x % 2 == 0, numbers))

from functools import reduce
sum_all = reduce(lambda x, y: x + y, numbers)

print(f"Original: {numbers}")
print(f"Squared: {squared}")
print(f"Evens: {evens}")
print(f"Sum: {sum_all}")

```

---

day - 16

## Event Sourcing

### Definition:

Event Sourcing is a software architectural pattern where state changes are stored as a sequence of events, rather than just the current state. Each event represents a change in the system, allowing for complete reconstruction of the application's state at any point in time.

Key Concepts:

- Events - Immutable, timestamped records of state changes
- Event Store - Database for persisting events
- Event Handlers - Process events to update application state

- CQRS - Often used with Command Query Responsibility Segregation for separating read and write operations

### Example:

Bank Account with Event Sourcing

```
# Event Store
events = [
    {"type": "AccountCreated", "account_id": 1, "timestamp": "2025-07-16T10:00:00"},
    {"type": "Deposit", "account_id": 1, "amount": 100, "timestamp": "2025-07-16T10:01:00"},
    {"type": "Withdrawal", "account_id": 1, "amount": 50, "timestamp": "2025-07-16T10:02:00"}
]

# Event Handler
def apply_events(events):
    balance = 0
    for event in events:
        if event["type"] == "Deposit":
            balance += event["amount"]
        elif event["type"] == "Withdrawal":
            balance -= event["amount"]
    return balance

# Get current balance
current_balance = apply_events(events)
print(f"Current Balance: ${current_balance}")  # Output: $50
```

---

day - 17

## Message Persistence

### Definition:

Message Persistence is the ability to store messages durably so they survive system failures, restarts, or crashes. It ensures messages are not lost and can be retrieved and processed even after unexpected interruptions.

Key Features:
✅ Recursive Design: Efficiently traverses nested structures
✅ Early Return: Stops searching once target is found
✅ Error Handling: Returns -1 for non-existent documents
✅ Flexible Structure: Works with any depth of nesting
✅ Clean Code: Well-documented and readable

### Example:

RabbitMQ Persistent Queues

```
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Declare persistent queue
channel.queue_declare(queue='persistent_queue', durable=True)

# Send persistent message
channel.basic_publish(
    exchange='',
    routing_key='persistent_queue',
    body='Hello World!',
    properties=pika.BasicProperties(
        delivery_mode=2,  # Make message persistent
    )
)
```

Database Message Store

```
import sqlite3
from datetime import datetime

class MessageStore:
    def __init__(self):
        self.conn = sqlite3.connect('messages.db')
        self.create_table()

    def create_table(self):
        self.conn.execute('''
            CREATE TABLE IF NOT EXISTS messages (
                id INTEGER PRIMARY KEY,
                content TEXT,
                timestamp DATETIME,
                processed BOOLEAN DEFAULT FALSE
            )
        ''')

    def store_message(self, content):
        self.conn.execute(
            'INSERT INTO messages (content, timestamp) VALUES (?, ?)',
            (content, datetime.now())
        )
        self.conn.commit()

    def get_unprocessed_messages(self):
        cursor = self.conn.execute(
            'SELECT * FROM messages WHERE processed = FALSE'
        )
        return cursor.fetchall()
```

---

day - 18

## Types of Failures in Distributed Systems

### Definition:

Types of failures in distributed systems refer to the various ways components can malfunction or become unavailable, affecting the system's ability to provide consistent and reliable service across multiple nodes.

Major Types of Failures:

- Fail-stop: A node halts and remains halted permanently. Other nodes can detect that the node has failed (i.e., by communicating with it).
- Crash: A node halts, but silently. So, other nodes may not be able to detect this state. They can only assume its failure when they are unable to communicate with it.
- Omission: A node fails to respond to incoming requests.
- Byzantine: A node exhibits arbitrary behavior: it may transmit arbitrary messages at arbitrary times, take incorrect steps, or stop.
  Byzantine failures occur when a node does not behave according to its specific protocol or algorithm. This usually happens when a malicious actor or a software bug compromises the node.
  To cope with these failures, we need complex solutions. However, most companies deploy distributed systems in environments that they assume to be private and secure.
  Fail-stop failures are the simplest and the most convenient ones from the perspective of someone that builds distributed systems. However, they are not very realistic. This is because there are many cases in real-life systems where it’s not easy for us to identify whether another node crashes or not.

---

day - 21

## Batch Processing

### Definition:

"A batch processing system takes a large amount of input data, runs a job to process it, and produces some output data.

Designing Data-Intensive Applications, M Kleppmann"

There are three important parts to this definition:

Takes a large amount of input data: This means that it does not make sense to run batch processing on top of a small amount of data that can be easily handled by request-response pattern.
Runs a job: This means it could be run periodically or based on some external trigger. In the context of batch processing, every time the system processes a bunch of data, we call that the system is running a job. This job runs for a few minutes to hours, or even for days.
Produces some output data: This means there is an outcome when the batch job finishes. Generally, the outcome is the generation of a set of output data.
One important aspect of batch processing is the execution of the job by using many processes, typically using a cluster of nodes.

### Example:

- Payroll Processing System

```text
Monthly Salary Calculation:
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Employee DB   │───▶│  Batch Processor │───▶│  Salary Reports │
│ (10M records)   │    │  (Multiple nodes) │    │   & Payments    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

- E-commerce Analytics:

```
# Daily sales report generation
Input: 24 hours of transaction data
Process: Calculate metrics across multiple servers
Output: Sales dashboard, inventory updates, recommendations
```

---

day - 22

## Cloud Exit

### Definition:

Cloud Exit is the strategic process of migrating applications, data, and IT infrastructure away from cloud services back to on-premises systems or alternative cloud providers to regain control, reduce costs, or address specific business requirements.

Key Characteristics

- Data Migration - Moving workloads out of cloud platforms
- Cost Optimization - Reducing ongoing cloud expenses
- Control Recovery - Regaining direct infrastructure management
- Vendor Independence - Avoiding cloud provider lock-in

### Example:

- Basecamp(2022):

```
Migration: AWS → On-premises
Reason: Cost savings (~$7M over 5 years)
Result: Built own data centers, reduced operational costs
```

- Spotify:

```
Migration: Google Cloud → Multi-cloud + On-premises
Reason: Better control over music streaming infrastructure
Result: Hybrid approach with custom data centers
```

- Financial Institution Example:

```
Migration: Public Cloud → Private Cloud
Reason: Regulatory compliance and data sovereignty
Result: Enhanced security and compliance control
```

---

day - 23

## Pessimistic Concurrency Control (PCC)

### Definition:

PCC is a database locking strategy that prevents conflicts by locking data while it's being read or modified, ensuring only one transaction can access it at a time.

**Use Case**
Banking systems use PCC to prevent two users from withdrawing the same money simultaneously.

### Example:

```
-- Transaction A locks the row for update
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- No other transaction can read or update this row until A commits
```

---

## Optimistic Concurrency Control (OCC)

### Definition:

OCC allows multiple transactions to proceed without locks, assuming conflicts are rare; it detects collisions only at commit-time using version tags.

**Use Case**
E-commerce carts: many users can edit simultaneously; only the first commit wins, others retry.

### Example:

JSON + HTTP

```
PUT /inventory/42
If-Match: "v5"
{
  "stock": 12
}
```

---

day - 24

## Paxos Algorithm

### Definition:

Paxos is a consensus algorithm used in distributed systems to achieve agreement among multiple nodes on a single value, even when some nodes fail or network partitions occur. It guarantees safety (correctness) but not liveness (progress) under all conditions.

Key Characteristics
✅ Fault-tolerant: Works with up to

```
(n-1)/2
```

node failures
✅ Safety guaranteed: Never produces incorrect results
⚠️ Liveness not guaranteed: May not always make progress
🎯 Single-value consensus: Agrees on one value per instance

Roles
The Paxos algorithm defines three different roles:

- Proposers
- Acceptors
- Learners
  Every node in the system can potentially play multiple roles.

### Example:

Distributed Database Leader Election

Scenario: 5 database replicas need to elect a leader

```
Nodes: A, B, C, D, E
Goal: Elect one leader from candidates A, B, C
```

Phase 1 (Proposers):

```
Node A (Proposer): "Prepare request #1, I want to propose myself as leader"
→ Sends to nodes B, C, D, E
```

Phase 2 (Acceptors):

```
Node B: "Promise #1, I haven't seen higher proposals"
Node C: "Promise #1, I haven't seen higher proposals"
Node D: "Promise #1, I haven't seen higher proposals"
Node E: "Promise #1, but I previously accepted Node B as leader"
```

Phase 3 (Learners):

```
Node A: "Accept Node B as leader" (uses highest previous value from promises)
→ Majority (B, C, D) accept
→ Node B becomes the leader
```

---

day - 25

## Internationalization (i18n) Frontend

### Definition:

Internationalization (i18n) is the process of designing and developing frontend applications to support multiple languages, regions, and cultures without requiring code changes. It enables applications to adapt to different locales by externalizing text, formatting dates/numbers, and handling cultural preferences.

Key Features
🔤 Text Translation - Dynamic language switching
📅 Date/Time Formatting - Locale-specific formats
💰 Number/Currency Formatting - Regional standards
🔄 RTL/LTR Support - Text direction handling
🎨 Cultural Adaptation - Colors, images, layouts

### Example:

- React with react-i18next

```
// i18n.js - Configuration
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

const resources = {
  en: {
    translation: {
      "welcome": "Welcome",
      "login": "Login",
      "users_count": "{{count}} user",
      "users_count_plural": "{{count}} users"
    }
  },
  es: {
    translation: {
      "welcome": "Bienvenido",
      "login": "Iniciar Sesión",
      "users_count": "{{count}} usuario",
      "users_count_plural": "{{count}} usuarios"
    }
  }
};

i18n.use(initReactI18next).init({
  resources,
  lng: 'en',
  fallbackLng: 'en'
});
```

```
// Component Usage
import { useTranslation } from 'react-i18next';

function App() {
  const { t, i18n } = useTranslation();

  const changeLanguage = (lng) => {
    i18n.changeLanguage(lng);
  };

  return (
    <div>
      <h1>{t('welcome')}</h1>
      <button>{t('login')}</button>
      <p>{t('users_count', { count: 5 })}</p>

      <select onChange={(e) => changeLanguage(e.target.value)}>
        <option value="en">English</option>
        <option value="es">Español</option>
      </select>
    </div>
  );
}
// jsx
```

---

day - 28

## Logical Clocks(Distributed Systems)

### Definition:

Logical Clocks are mechanisms used in distributed systems to establish a partial ordering of events across different nodes without relying on synchronized physical clocks. They assign timestamps to events to determine causality relationships between operations happening on different machines.

### Example:

- Scenario: Three nodes (A, B, C) exchanging messages

```
Node A: [1] Send msg to B → [2] Receive from C → [4] Internal event
Node B: [3] Receive from A → [4] Send msg to C → [5] Internal event
Node C: [2] Send msg to A → [5] Receive from B → [6] Internal event
```

Lamport Clock Rules:

- Internal Events:

```
LC = LC + 1
```

- Send Message:

```
LC = LC + 1
```

- Receive Message:

```
LC = max(LC, received_timestamp) + 1
```

---

day - 29

## N-Tier Architecture

### Definition:

N-Tier Architecture is a software design pattern that separates an application into multiple logical layers (tiers), where each tier has a specific responsibility and communicates only with adjacent tiers. This promotes separation of concerns, scalability, and maintainability.

**Key Characteristics**
🔄 Layered Structure: Each tier serves a specific function
📡 Inter-tier Communication: Tiers communicate through well-defined interfaces
🎯 Separation of Concerns: Business logic, data access, and presentation are separated
📈 Scalability: Each tier can be scaled independently

### Example:

- 3-Tier Architecture:

```text
┌─────────────────────┐
│   Presentation      │ ← User Interface (Web/Mobile/Desktop)
│      Tier           │
├─────────────────────┤
│    Business         │ ← Business Logic & Rules
│   Logic Tier        │
├─────────────────────┤
│    Data Tier        │ ← Database & Data Storage
└─────────────────────┘
```

- 4-Tier Architecture:

```text
┌─────────────────────┐
│   Presentation      │ ← Frontend (React, Angular)
├─────────────────────┤
│   Application       │ ← Web Server (APIs, Controllers)
├─────────────────────┤
│   Business Logic    │ ← Business Rules & Processing
├─────────────────────┤
│    Data Tier        │ ← Database (MySQL, PostgreSQL)
└─────────────────────┘
```

Real-World Example: E-Commerce Application
| Tier | Technology | Responsibility |
| --- | --- | --- |
| 🖥️ Presentation | React.js, HTML/CSS | Product catalog, shopping cart UI |
| ⚙️ Application | Node.js/Express API | Handle HTTP requests, routing |
| 🧠 Business Logic | Java/Spring Boot | Order processing, payment validation |
| 💾 Data | PostgreSQL | Store products, users, orders |

---

day - 30

## Big-O Notation

### Definition:

Big-O notation describes the upper bound of an algorithm's time or space complexity as input size grows. It expresses the worst-case scenario performance, helping developers compare algorithm efficiency.

### Example:

- Common Big-O Examples
  | Notation | Name | Description | Example Algorithm |
  | --- | --- | --- | --- |
  | O(1) | Constant | Same time regardless of input size | Array access by index |
  | O(log n) | Logarithmic | Time increases slowly | Binary search |
  | O(n) | Linear | Time grows proportionally | Linear search |
  | O(n log n) | Log-linear | Efficient sorting algorithms | Merge sort, Quick sort |
  | O(n²) | Quadratic | Time grows exponentially | Bubble sort, nested loops |
  | O(2ⁿ) | Exponential | Extremely slow growth | Recursive Fibonacci |

---

## Vertical Sharding Database

### Definition:

Vertical Sharding is a database partitioning technique where columns of a table are split across multiple databases or servers, rather than splitting rows. Each shard contains a subset of columns from the original table, typically grouping related columns together based on access patterns or data characteristics.

**Key Characteristics**
✅ Column-based partitioning
✅ Reduces table width
✅ Improves query performance for specific column groups
✅ Enables specialized storage optimization

### Example:

- E-commerce User Database

```
users_table:
- user_id (Primary Key)
- username, email, password_hash
- first_name, last_name, phone, address
- profile_image, bio, preferences
- last_login, created_at, updated_at
- account_balance, subscription_plan
```

After Vertical Sharding
| Shard | Database | Columns | Use Case |
| --- | --- | --- | --- |
| Authentication | `auth_db` | `user_id`, `username`, `email`, `password_hash` | Login/Security |
| Profile | `profile_db` | `user_id`, `first_name`, `last_name`, `phone`, `address` | User Management |
| Content | `content_db` | `user_id`, `profile_image`, `bio`, `preferences` | Social Features |
| Billing | `billing_db` | `user_id`, `account_balance`, `subscription_plan` | Payment Processing |

---

day - 31

## Principle of Inclusion and Exclusion(PIE)

### Definition:

PIE is a combinatorial counting technique that calculates the size of the union of multiple sets by:

- Including individual set sizes
- Excluding pairwise intersections
- Including triple intersections
- And so on, alternating signs

**Formula:**

```
|A ∪ B ∪ C| = |A| + |B| + |C| - |A ∩ B| - |A ∩ C| - |B ∩ C| + |A ∩ B ∩ C|
```

### Example:

- Students and Subjects
  Problem: In a class of 100 students:

- 60 study Math (M)
- 50 study Physics (P)
- 40 study Chemistry (C)
- 30 study both Math and Physics
- 20 study both Math and Chemistry
- 15 study both Physics and Chemistry
- 10 study all three subjects

Find: How many students study at least one subject?

**Solution using PIE:**

```
|M ∪ P ∪ C| = |M| + |P| + |C| - |M ∩ P| - |M ∩ C| - |P ∩ C| + |M ∩ P ∩ C|

= 60 + 50 + 40 - 30 - 20 - 15 + 10
= 150 - 65 + 10
= 95
```

Answer: 95 students study at least one subject.

---
