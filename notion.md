# Notion System Design (Simplified)

A beginner-friendly system design for a Notion-like workspace app. Users
can create pages, organize content into blocks, view database entries in
different ways, collaborate in real time, and share pages with others.


## Table of Contents

1.  [Phase 1: Requirements and Scope]
2.  [Phase 2: High-Level Architecture]
3.  [Phase 3: Data Model and API
    Contracts]
4.  [Phase 4: Real-Time Collaboration]
5.  [Scaling and Storage Strategy]
6.  [Important Trade-Offs]
7.  [Summary]

------------------------------------------------------------------------

## Phase 1: Requirements and Scope

### 1.1 Functional Requirements

The system should allow users to:

-   **Create and nest pages:** A page can contain other pages.
-   **Build content with blocks:** Text, headings, checklists, images,
    and other content are represented as blocks.
-   **Use databases:** Users can store structured entries and view them
    as tables, boards, or lists.
-   **Edit together:** Multiple users can edit the same page at the same
    time.
-   **Share content:** Page owners can grant access to other users and
    control what they can do.
-   **Read and update content:** Users can load pages, add blocks, edit
    blocks, move blocks, and delete blocks.

### 1.2 Non-Functional Requirements

  -----------------------------------------------------------------------
  Requirement                         Meaning
  ----------------------------------- -----------------------------------
  Low typing latency                  Aim for under 50 ms for local
                                      keystroke response. The UI should
                                      update immediately without waiting
                                      for the server.

  Eventual consistency                Different users' screens may
                                      briefly differ, but should converge
                                      to the same result after updates
                                      are exchanged.

  Large pages                         Support pages containing 10,000 or
                                      more blocks.

  Large scale                         Design for millions of users and
                                      billions of blocks by distributing
                                      data and adding service capacity.

  Availability                        A temporary failure in one service
                                      should not take down the entire
                                      system where possible.

  Security                            Check a user's permissions before
                                      returning or changing content.
  -----------------------------------------------------------------------


### 1.3 Out of Scope

This design does not cover:

-   Native mobile applications
-   Offline synchronization across devices
-   Public developer APIs
-   A template marketplace

The browser client may still use temporary local storage for fast
updates and recovery during a short connection interruption. Full
offline editing and later synchronization are out of scope.

<img width="1012" height="542" alt="image" src="https://github.com/user-attachments/assets/7d61e122-cd25-4262-9c2c-5e86d706fc9e" />


------------------------------------------------------------------------

## Phase 2: High-Level Architecture

### 2.1 Architecture Diagram

<img width="765" height="422" alt="image" src="https://github.com/user-attachments/assets/b4751e5a-17b1-4df7-be6e-8e9f74782477" />

The diagram is logical: in a production system, services may use
internal APIs or messaging to communicate, and the Database Service is
the controlled path for database access.

### 2.2 Main Components

#### Browser / Web Client

-   Shows pages, blocks, and database views.
-   Updates the screen immediately when a user types.
-   Sends normal requests over HTTPS.
-   Keeps a WebSocket connection open for live collaboration.
-   Uses a local cache and transaction queue to improve responsiveness.

#### API Gateway

-   Receives client requests.
-   Authenticates users and applies basic request limits.
-   Routes block operations to the Block Service.
-   Routes collaboration connections to the Collaboration Service.
-   Helps enforce a consistent entry point for the backend.

#### Block Service

-   Creates, reads, updates, moves, and deletes blocks.
-   Validates block properties and parent-child relationships.
-   Checks whether a user can access the target page.
-   Requests persistent reads and writes through the Database Service.

#### Collaboration Service

-   Maintains WebSocket connections for users currently editing pages.
-   Receives text and block updates.
-   Sends updates to other users editing the same page.
-   Helps resolve concurrent edits and manages reconnection recovery.
-   Does not need to store every live update permanently itself; durable
    changes are persisted through the backend.

#### Database Service

-   Provides a controlled layer for database reads and writes.
-   Maps a workspace to its logical shard and physical PostgreSQL
    instance.
-   Runs queries and transactions.
-   Works with Redis to cache frequently requested data.

#### Sharded PostgreSQL

-   Stores users, workspaces, pages, blocks, permissions, and database
    entries.
-   Uses transactions for related changes that must be saved together.
-   Is divided into multiple shards as data and traffic grow.

#### Redis Cache

-   Stores frequently accessed page data, block data, permission
    results, and other short-lived information.
-   Reduces repeated database reads.
-   Uses expiration and invalidation so old data does not remain cached
    indefinitely.

### 2.3 Read and Write Flow

**Read a page**

1.  The browser sends an HTTPS request to the API Gateway.
2.  The gateway authenticates the user and routes the request.
3.  The Block Service checks access and asks for page data.
4.  The Database Service checks Redis first.
5.  If the cache misses, it reads from the correct PostgreSQL shard and
    may fill the cache.
6.  The page data is returned to the browser.

**Write a block**

1.  The browser updates the visible content immediately.
2.  It sends a create or update request through HTTPS, or sends a
    collaboration update over WebSocket.
3.  The backend validates the change and checks permissions.
4.  The change is saved to PostgreSQL.
5.  Relevant caches are invalidated or refreshed.
6.  The Collaboration Service broadcasts the accepted update to other
    users.

    <img width="1018" height="657" alt="image" src="https://github.com/user-attachments/assets/eea498f2-9ac1-4896-b360-7b78aa3732c9" />
    <img width="1018" height="362" alt="image" src="https://github.com/user-attachments/assets/27847c59-72d4-4a38-ad9b-fe1cc59bcb39" />



------------------------------------------------------------------------

## Phase 3: Data Model and API Contracts

### 3.1 Core Data Model

The main idea is to represent many types of content using one common
**Block** structure. A heading, paragraph, checklist item, and child
page can all be blocks with different types and properties.

A simplified relational model:

``` text
User
- id
- name
- email

Workspace
- id
- name

WorkspaceMember
- workspace_id
- user_id
- role

Page
- id
- workspace_id
- title
- root_block_id
- created_at
- updated_at

Block
- id
- workspace_id
- page_id
- parent_id
- type
- properties (JSONB)
- child_order (ordered list of child block IDs)
- version
- created_at
- updated_at

Permission
- id
- workspace_id
- page_id
- user_id
- role

DatabaseEntry
- id
- workspace_id
- database_id
- properties (JSONB)
- created_at
- updated_at
```

These tables are illustrative. A production implementation may normalize
ordering, permissions, and database metadata into additional tables.

### 3.2 Database Views

A database is a collection of structured entries. Each entry has
properties such as name, status, date, or assignee.

The same entries can be displayed in different ways:

-   **Table:** Rows and columns
-   **Board:** Cards grouped by a property such as status
-   **List:** A simple list of entries

The view is mostly a saved configuration (filters, sorting, grouping,
and visible properties). It does not need a separate copy of every
entry.

### 3.3 REST API Contracts

All endpoints require authentication. The backend must check permissions
for the requested workspace, page, or block.

  -----------------------------------------------------------------------
  Method and endpoint                 Purpose
  ----------------------------------- -----------------------------------
  `GET /blocks/:id`                   Fetch one block and its properties.

  `POST /blocks`                      Create a block under a page or
                                      parent block.

  `PATCH /blocks/:id`                 Update block properties, parent, or
                                      ordering.

  `DELETE /blocks/:id`                Delete a block, following the
                                      chosen child-deletion policy.

  `GET /pages/:id`                    Fetch a page and its nested blocks.

  `POST /databases/:id/query`         Query database entries with
                                      filters, sorting, and pagination.
  -----------------------------------------------------------------------

<img width="1035" height="337" alt="image" src="https://github.com/user-attachments/assets/fb966c75-8531-4367-b233-453ee6b6c62e" />

<img width="1028" height="631" alt="image" src="https://github.com/user-attachments/assets/ca052f21-98c0-4c9d-8fca-921b843cf2fb" />

<img width="1028" height="657" alt="image" src="https://github.com/user-attachments/assets/d0e2c898-5549-47af-8e57-bb913f89f10c" />


------------------------------------------------------------------------

## Phase 4: Real-Time Collaboration

### 4.1 Why Collaboration Is Difficult

Two people may edit the same paragraph at almost the same time. If the
system simply saves whichever full paragraph arrives last, one person's
typing can overwrite the other person's work.

The design therefore uses two conflict-resolution methods for different
types of changes.

### 4.2 Hybrid Conflict Resolution

#### Block-level Last-Writer-Wins (LWW)

LWW is suitable for many block properties, such as a checkbox state, a
title, or a selected color.

-   Each update has a logical timestamp or revision.
-   The system compares competing updates.
-   A deterministic rule selects the winning value.

A server-assigned revision or logical clock is safer than trusting
client wall-clock timestamps, because device clocks can be wrong.

**Trade-off:** LWW is simple, but one conflicting property update can
replace another.

#### Character-level CRDT for Text

A Conflict-free Replicated Data Type (CRDT) is a data structure designed
so concurrent text changes can be merged consistently.

-   Each text insertion or deletion has an operation identity and
    ordering information.
-   The system exchanges operations rather than replacing the entire
    paragraph each time.
-   Concurrent edits are merged according to deterministic rules.
-   Replicas eventually converge when they receive the same operations.

For example, if two people insert text at different positions in the
same paragraph, a CRDT can preserve both insertions instead of losing
one.

A CRDT requires extra metadata and implementation complexity. Use a
well-tested text CRDT library rather than inventing a new algorithm.

### 4.3 Optimistic Local Updates

To keep typing responsive:

1.  The user types a character.
2.  The browser immediately updates the visible text.
3.  The client records the operation in its local transaction queue.
4.  The operation is sent to the server.
5.  The server validates, persists, and broadcasts the operation.
6.  The client reconciles the server response with its local state.

The user should not have to wait for a database round trip for every
keystroke.

<img width="1018" height="190" alt="image" src="https://github.com/user-attachments/assets/0f1be92c-9689-40a5-aab3-36bcdc28be30" />

------------------------------------------------------------------------

## Scaling and Storage Strategy

### 5.1 Application-Level Sharding

Billions of blocks cannot comfortably live on one database server
forever. Split workspaces across logical shards and physical PostgreSQL
instances.

Use `workspace_id` as the partition key because most content and
permissions belong to a workspace.

Example:

``` text
workspace_id
     |
     v
Shard Mapping Service
     |
     +---- workspace A -> PostgreSQL shard 1
     +---- workspace B -> PostgreSQL shard 2
     +---- workspace C -> PostgreSQL shard 3
```

The mapping service can use a stable hash or a shard directory to find
the correct shard.

**Why this helps**

-   Workspaces can be distributed across database servers.
-   Most queries stay within one shard.
-   More shards can be added as traffic grows.
-   One very large workspace can be identified and handled specially if
    it becomes a hot spot.

**Important trade-offs**

-   Queries spanning multiple workspaces are more difficult.
-   Moving a workspace to another shard requires a controlled migration.
-   All tables that commonly join together should use compatible shard
    keys.
-   A single very active workspace may need further partitioning or
    dedicated capacity.

### 5.2 PostgreSQL Storage

PostgreSQL stores the durable source of truth.

Recommended basics:

-   Index `workspace_id`, `page_id`, `parent_id`, and frequently queried
    fields.
-   Use JSONB for flexible block properties, but keep frequently
    filtered fields indexed or modeled separately.
-   Use transactions for changes that must be atomic.
-   Use backups, replication, and tested recovery procedures.
-   Use pagination instead of loading every block or database entry at
    once.

For pages with 10,000+ blocks, return data in manageable chunks where
practical, and avoid re-rendering the entire page for every small edit.

### 5.3 Two-Tier Caching

#### Tier 1: Browser Local Cache

The browser uses local storage or an appropriate browser database for
temporary state.

-   **RecordCache:** Holds recently loaded pages and blocks so the UI
    can render quickly.
-   **TransactionQueue:** Holds pending operations until they are sent
    and acknowledged.

This is a responsiveness and short-interruption recovery mechanism, not
a promise of full offline support. The queue should be bounded and
should surface an error if changes cannot be saved.

#### Tier 2: Server-side Redis Cache

Redis stores frequently requested data, for example:

-   Page and block read results
-   Workspace metadata
-   Permission lookup results
-   Short-lived collaboration metadata

A typical cache read is:

1.  Look for the requested data in Redis.
2.  If found, return it after the required access checks.
3.  If not found, read PostgreSQL.
4.  Put an appropriate result in Redis with a time-to-live (TTL).
5.  Return the result.

### 5.4 Cascading Permission Invalidation

Permissions can be inherited from a workspace or parent page. If an
owner removes access, cached permission decisions must not keep granting
access.

A simple approach:

1.  Every relevant permission change increments a workspace or
    permission revision.
2.  Cache keys include that revision, or the system invalidates the
    affected cache entries.
3.  Child-page permission caches are invalidated when an inherited
    permission changes.
4.  The backend checks authorization before returning sensitive data or
    accepting writes.
5.  If cache invalidation fails, short TTLs and revision checks limit
    how long stale decisions can remain valid.

Do not rely on cache invalidation alone as the security boundary. The
authorization layer must remain authoritative.

### 5.5 Growth Plan

  -----------------------------------------------------------------------
  Stage                               Strategy
  ----------------------------------- -----------------------------------
  Early stage                         One PostgreSQL primary, Redis, and
                                      a small set of stateless services.

  Growing usage                       Add read replicas, indexes,
                                      metrics, and more service
                                      instances.

  Large scale                         Shard by workspace, use a shard
                                      directory, and tune hot workspaces
                                      separately.

  Very large pages                    Paginate reads, batch writes,
                                      optimize block ordering, and avoid
                                      unnecessary full-page reloads.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Important Trade-Offs

  -----------------------------------------------------------------------
  Design choice           Benefit                 Cost or limitation
  ----------------------- ----------------------- -----------------------
  One unified block model Supports many content   Some properties are
                          types consistently      less convenient to
                                                  query inside JSONB

  PostgreSQL + JSONB      Reliable transactions   Indexes need careful
                          with flexible           planning
                          properties              

  Redis cache             Faster repeated reads   Cache invalidation adds
                                                  complexity

  Block-level LWW         Simple conflict         A competing update can
                          handling for many       win and replace another
                          properties              value

  Text CRDT               Merges simultaneous     More metadata and
                          text edits              implementation
                                                  complexity

  Workspace-based         Keeps most workspace    Cross-workspace queries
  sharding                data together           and shard movement are
                                                  harder

  WebSockets              Efficient live updates  Requires connection
                                                  management,
                                                  authentication, and
                                                  recovery logic
  -----------------------------------------------------------------------

  <img width="1026" height="465" alt="image" src="https://github.com/user-attachments/assets/e8ac4c51-3e62-4d18-82b6-ae656a2746fb" />
  <img width="1017" height="410" alt="image" src="https://github.com/user-attachments/assets/c7faf1d6-86f9-4a20-8753-e0e2a8966b49" />



## Summary

The proposed system uses a browser client, an API Gateway, a Block
Service, a Collaboration Service, and a Database Service. PostgreSQL
stores durable workspace content, Redis accelerates common reads, and
workspace-based sharding distributes data as usage grows.

Blocks share one recursive structure with JSONB properties and ordered
children. REST endpoints support normal page operations, while
WebSockets carry live collaboration updates. Block-level LWW handles
many property conflicts, and a text CRDT merges concurrent text edits.
Optimistic UI updates keep typing responsive, while state vectors help
recover after reconnecting.

The key design principle is to keep the system understandable at first,
then add sharding, caching, and collaboration optimizations as traffic
and data volume increase.
