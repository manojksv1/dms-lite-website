# Technical Architecture Deep Dive: DMS Enterprise

This document provides a comprehensive technical breakdown of the core systems powering **DMS Enterprise**. It focuses on security, data integrity, and enterprise-grade features like Content Addressable Storage (CAS), Encryption, Search, IAM, and Disaster Recovery.

---

## 1. Core Storage Engine (CAS)
The system employs a **Content Addressable Storage (CAS)** architecture to ensure data integrity, deduplication, and efficient storage usage.

### 1.1 Architecture
-   **Addressing**: Files are not stored by their filename. Instead, the system calculates the **SHA-256 hash** of the file content. This hash becomes the unique identifier for the blob.
-   **Sharding**: To prevent file system limits on the number of files per directory, the storage is sharded into a 2-level directory structure based on the hash:
    ```
    storage/
    ├── a1/
    │   ├── b2/
    │   │   └── a1b2c3d4e5f6... (The actual file blob)
    ```
-   **Deduplication**: Before writing a new file, the system checks if the hash already exists. If it does, the upload is skipped, and the new document version simply points to the existing blob. This saves massive amounts of space for duplicate uploads.
-   **Reference Counting**: The system tracks how many document versions reference a single physical blob. A blob is only physically deleted when its reference count drops to zero (via the `cleanup_expired_trash` background task).

### 1.2 Pipeline (Write Path)
When a file is uploaded, it passes through a **Streaming Pipeline** to handle large files (e.g., 5GB+) without loading them entirely into RAM:
1.  **Hashing**: The stream is read chunk-by-chunk to calculate the SHA-256 hash.
2.  **Compression**: The stream is compressed using **Zstandard (Zstd)** (Level 3) for high-performance compression.
3.  **Encryption**: The compressed stream is encrypted on-the-fly (see Section 2).
4.  **Atomic Write**: The data is written to a temporary file. Once complete and verified, it is atomically moved to its final sharded location.

---

## 2. Security & Cryptography

### 2.1 Encryption at Rest
All files stored in the CAS are encrypted using **AES-256-GCM** (Galois/Counter Mode), which provides both confidentiality and integrity.
-   **Key Management**: The system uses a 32-byte master key loaded from the `ENCRYPTION_KEY` environment variable (Base64 encoded).
-   **File Format**:
    ```
    [Nonce (12 bytes)] + [Auth Tag (16 bytes)] + [Ciphertext (Variable)]
    ```
-   **Streaming Decryption**: The system supports seeking and range requests on encrypted files. It uses a generator-based approach to decrypt chunks on-the-fly, allowing instant streaming of videos or large PDFs without full decryption.

### 2.2 Encryption in Transit
-   **TLS**: All traffic is terminated at the **Nginx** reverse proxy using TLS 1.2/1.3.
-   **Internal Network**: Communication between containers (Backend, DB, Redis, ClamAV) occurs over an isolated Docker bridge network (`dms-network`) and is not exposed to the host or public internet.

### 2.3 Virus Scanning
-   **ClamAV Integration**: The system integrates with a ClamAV daemon (`clamd`) running in a sidecar container.
-   **Stream Scanning**: Uploads are streamed directly to the ClamAV socket for inspection before they are finalized in storage.
-   **Policy**: If a virus is detected, the transaction is aborted, and the file is rejected with an HTTP 400 Security Alert.

---

## 3. IAM & Permission Policy Engine
The system uses a robust **Role-Based Access Control (RBAC)** system combined with a **Hierarchical Permission Resolver**.

### 3.1 Policy Evaluation (`PolicyEvaluator`)
Permissions are not just simple boolean flags. They are evaluated based on JSON policies attached to users and groups, similar to AWS IAM.
-   **Statement Structure**:
    ```json
    {
      "Effect": "Allow",
      "Action": ["dms:document:Read", "dms:document:Write"],
      "Resource": "dms:folder:123/*"
    }
    ```
-   **Evaluation Logic**:
    1.  **Action Match**: Checks if the requested action (e.g., `WRITE`) matches the policy action (supports wildcards like `*`).
    2.  **Resource Match**: Checks if the target resource ARN matches (supports wildcards).
    3.  **Effect**: `Deny` statements always take precedence over `Allow`. An "Implicit Deny" applies if no statement matches.

### 3.2 Hierarchical Resolution (`PermissionResolver`)
For enterprise folders (Cabinet > Drawer > Folder), permissions are inherited and reduced down the tree.
-   **Ancestry Traversal**: The system resolves the full path of a document (e.g., `/1/5/12`) to identify all parent containers.
-   **Reduction Rule**: A child container can never *expand* permissions granted by a parent, but it can *restrict* them (Deny). However, explicit assignments at a lower level (e.g., "User A has Read on Folder X") are additive.
-   **Effective Permissions**: Calculated by unioning explicit assignments and inherited permissions, then subtracting inherited denials.

---

## 4. Search & Indexing Strategy
The system uses **PostgreSQL Full-Text Search** for powerful and efficient retrieval.

### 4.1 Indexing Pipeline
When a document is uploaded, a background **Celery** task (`indexing.process`) is triggered:
1.  **Text Extraction**:
    -   **PDF**: Uses a hybrid approach. First, it attempts standard text extraction via `PyPDF2`. If the text density is low (indicating a scanned doc), it falls back to OCR.
    -   **Office (DOCX/XLSX/PPTX)**: Extracts XML text content and unzips the file to find embedded images for OCR.
    -   **Images**: Processed directly via OCR.
2.  **OCR Engine**: The system uses **PaddleOCR** (optimized for CPU) with angle classification to handle rotated scans.
3.  **Vector Storage**: The extracted text is processed into a `TSVECTOR` (Text Search Vector) and stored in the `search_vector` column of the `DocumentVersion` table.
4.  **Indexing**: A **GIN (Generalized Inverted Index)** is maintained on the `search_vector` column, allowing for sub-millisecond search queries even with millions of rows.

---

## 5. Workflow Automation Engine
The workflow engine is a state machine driven by database events and user actions.

### 5.1 Components
-   **Workflow Definition**: A template defining the steps (e.g., "Manager Review" -> "Director Approval").
-   **Workflow Instance**: A running execution of a workflow on a specific document.
-   **Workflow Task**: An actionable item assigned to a user or group.

### 5.2 Execution Logic
1.  **Triggering**: Workflows are triggered automatically (e.g., on `UPLOAD` to a specific folder) or manually by a user.
2.  **Step Processing**:
    -   When a step starts, the system identifies assignees (User, Group, Initiator, or Document Owner).
    -   It creates `WorkflowTask` records for all assignees.
    -   **Notifications**: Sends real-time WebSocket alerts and emails to assignees.
3.  **Transition**:
    -   **Approval**: If the required number of approvals is met, the system advances to the next step.
    -   **Rejection**: If rejected, the workflow transitions to a "Rejected" state, and the document status is updated.
4.  **Locking**: Documents under active workflow are **locked** to prevent edits by non-assignees.

---

## 6. Retention & Lifecycle Management
Automated governance of document lifecycles to ensure compliance.

### 6.1 Policy Execution
A background worker (`process_retention_policies`) runs periodically:
1.  **Identification**: Scans for documents where `created_at` + `duration` < `now`.
2.  **Action**:
    -   **Soft Delete**: Moves the document to the Recycle Bin.
    -   **Audit**: Logs a `RETENTION_DELETE` event.
3.  **Trash Cleanup**: A separate process (`cleanup_expired_trash`) permanently removes items from the Recycle Bin after a grace period (e.g., 30 days). This triggers the physical blob deletion in CAS if no other versions reference the file.

---

## 7. Real-Time Notifications (WebSockets)
The system maintains a persistent connection with online users to push updates instantly.

### 7.1 Architecture
-   **Connection Manager**: Tracks active `WebSocket` connections, mapping `user_id` to a list of sockets (handling multiple tabs/devices).
-   **Events**:
    -   `notification`: General alerts (Workflow tasks, mentions).
    -   `workflow_update`: Live updates on workflow progress.
-   **Fallback**: If a user is offline, the notification is stored in the database (`Notification` table) and displayed upon next login.

---

## 8. Backup & Disaster Recovery
A comprehensive backup strategy ensures business continuity.

### 8.1 Backup Process (`BackupManager`)
1.  **Database Dump**: Uses `pg_dump` with custom format (`-Fc`) to capture the entire database schema and data. Crucially, it bypasses `PgBouncer` to ensure reliability during long operations.
2.  **Storage Snapshot**: Copies the entire CAS storage directory.
3.  **Compression**: Bundles the DB dump and storage into a single `.zip` archive.

### 8.2 Restore Process
1.  **Validation**: Verifies the integrity of the zip file.
2.  **Database Restore**: Uses `pg_restore` with `--clean` to wipe the existing DB and restore the backup state.
3.  **Storage Restore**: Replaces the current storage directory with the backed-up version.
4.  **Schema Migration**: Automatically runs `alembic upgrade head` after restore. This allows restoring an older backup (e.g., from v1.0) into a newer system (v2.0), with the system automatically migrating the old schema to the new format.

---

## 9. Office Integration (WOPI)
The system implements the **WOPI (Web Application Open Platform Interface)** protocol to integrate with **Collabora Online** (LibreOffice) for real-time editing.

### 9.1 Protocol Implementation
-   **CheckFileInfo**: Returns metadata, user permissions, and dynamic watermark configuration (e.g., `{User} - {Date}`).
-   **GetFile**: Streams the decrypted file content to the Office server.
-   **PutFile**: Receives the updated file stream from the Office server.
    -   **Version Coalescing**: If a user saves multiple times within 10 seconds, the system intelligently updates the latest version instead of creating a new one, preventing version history spam.
    -   **Locking**: Implements `X-WOPI-Lock` handling using Redis to prevent concurrent edits.

---

## 10. Scalability & Performance

### 10.1 Asynchronous Task Queue
The system uses **Celery** with **Redis** as the broker to handle background workloads:
-   **Worker-Heavy**: Dedicated queue for CPU-intensive tasks like OCR and Video Transcoding. Concurrency is limited to prevent CPU starvation.
-   **Worker-Fast**: High-concurrency queue for I/O bound tasks like email notifications and webhooks.

### 10.2 Database Performance
-   **Connection Pooling**: **PgBouncer** is deployed in `transaction` pooling mode. This allows hundreds of concurrent API requests to share a small pool of actual database connections (e.g., 20), significantly reducing overhead.
-   **Stateless Backend**: The **FastAPI** application is stateless, allowing it to be horizontally scaled (e.g., `docker-compose up --scale backend=3`) behind the Nginx load balancer.

### 10.3 Caching
-   **Redis**: Used for:
    -   Celery Message Brokering.
    -   WOPI Locks.
    -   Session Caching (if applicable).
    -   Rate Limiting counters.
