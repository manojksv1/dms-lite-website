# Product Specification Document: DMS Enterprise

## 1. Introduction
**DMS Enterprise** is a robust, scalable, and secure Document Management System designed for enterprise environments. It facilitates the storage, management, retrieval, and collaboration of digital documents while ensuring data security and compliance.

### 1.1 Purpose
The purpose of this document is to define the functional and non-functional requirements, system architecture, and technical specifications of the DMS Enterprise system.

### 1.2 Scope
The system encompasses:
-   **Identity & Access Management (IAM)**: Secure authentication and role-based access control.
-   **Document Lifecycle Management**: Upload, versioning, metadata management, and archival.
-   **Search & Discovery**: Advanced full-text and metadata search capabilities.
-   **Collaboration**: Real-time editing, sharing, and commenting.
-   **Workflow Automation**: Approval workflows and task management.
-   **Security & Compliance**: Audit logging, retention policies, and virus scanning.
-   **Physical Records Integration**: Scanning and digitization support.

---

## 2. User Roles and Personas

| Role | Description | Key Responsibilities |
| :--- | :--- | :--- |
| **Administrator** | System super-user. | User management, system configuration, retention policies, audit log review. |
| **Manager** | Department or team lead. | Workflow approvals, folder management, team oversight. |
| **Contributor** | Standard employee. | Uploading, editing, and collaborating on documents. |
| **Viewer** | Read-only user. | Viewing and downloading documents without edit permissions. |
| **External User** | Guest/Client. | Accessing shared documents via public links (if permitted). |

---

## 3. Functional Requirements

### 3.1 Identity & Access Management (IAM)
-   **Authentication**:
    -   Username/Password login with bcrypt hashing.
    -   Multi-Factor Authentication (MFA) using TOTP (Google/Microsoft Authenticator).
    -   Account locking after failed attempts.
-   **User Management**:
    -   Registration workflow (with optional admin approval).
    -   Password reset via email token.
    -   Profile management (Avatar, Bio).
-   **Authorization**:
    -   Role-Based Access Control (RBAC) with predefined roles (Admin, User, Guest).
    -   Granular permissions at Cabinet, Drawer, and Folder levels.

### 3.2 Document Management
-   **File Operations**:
    -   Support for PDF, DOCX, XLSX, PPTX, Images, and Video.
    -   Large file upload support (>100MB).
    -   Drag-and-drop bulk upload.
    -   Duplicate file handling (Version/Rename/Skip).
-   **Versioning**:
    -   Automatic version incrementing (v1 -> v2).
    -   Version history view with restore and download capabilities.
-   **Previewing**:
    -   In-browser preview for PDF, Images, Videos.
    -   WOPI integration for high-fidelity Office document rendering.
-   **Organization**:
    -   Hierarchical structure: **Cabinet > Drawer > Folder**.
    -   Personal Workspace ("My Documents").
    -   Move, Copy, Rename, and Delete operations.
-   **Recycle Bin**:
    -   Soft delete with restore capability.
    -   Configurable retention period for deleted items.

### 3.3 Search & Discovery
-   **Full-Text Search**:
    -   Content extraction from PDF (OCR via PaddleOCR), DOCX, and Text files.
    -   Indexing for rapid retrieval.
-   **Metadata Search**:
    -   Filter by Author, Date Range, File Type, Tags, and Location.
-   **Advanced Queries**:
    -   Boolean operators and combined filters.

### 3.4 Collaboration & Editing
-   **Online Editing**:
    -   Integration with **Collabora Online** for real-time co-authoring of Word, Excel, and PowerPoint files.
-   **Sharing**:
    -   Internal sharing with specific permissions (View/Edit).
    -   Public links with password protection and expiry dates.
-   **Social Features**:
    -   Comments and threaded replies on documents.
    -   User mentions (@user) and notifications.

### 3.5 Workflows & Automation
-   **Approval Workflows**:
    -   Multi-step approval processes.
    -   Task assignment and tracking.
    -   Status transitions (Draft -> Pending -> Approved/Rejected).
-   **Notifications**:
    -   In-app and email notifications for tasks and mentions.

### 3.6 Security & Compliance
-   **Virus Scanning**:
    -   Real-time scanning of uploads using **ClamAV**.
-   **Watermarking**:
    -   Dynamic watermarking (User + Date) on previews and downloads.
-   **Audit Trails**:
    -   Comprehensive logging of all user actions (Login, Upload, Delete, Share).
    -   Exportable audit logs for compliance reviews.
-   **Retention Policies**:
    -   Automated data retention and disposal rules based on document type or location.

### 3.7 Scanning & Physical Records
-   **Desktop Agent**:
    -   Integration with local scanners (TWAIN/WIA).
    -   Direct upload to DMS.
    -   Basic image manipulation (Crop, Rotate, Reorder).

---

## 4. Technical Architecture

### 4.1 Technology Stack
-   **Frontend**:
    -   **Framework**: React 19 (Vite).
    -   **Language**: TypeScript.
    -   **UI Library**: Material UI (MUI).
    -   **State Management**: React Hooks / Context.
-   **Backend**:
    -   **Framework**: FastAPI (Python 3.11+).
    -   **ORM**: SQLAlchemy 2.0 (Async).
    -   **Migrations**: Alembic.
    -   **Task Queue**: Celery with Redis.
-   **Database**:
    -   **Primary**: PostgreSQL 15.
    -   **Connection Pooling**: PgBouncer.
-   **Infrastructure**:
    -   **Containerization**: Docker & Docker Compose.
    -   **Reverse Proxy**: Nginx.
    -   **Caching**: Redis.
    -   **Office Suite**: Collabora CODE.
    -   **Security**: ClamAV.

### 4.2 System Components
1.  **Nginx Load Balancer**: Entry point, handles SSL termination and routing.
2.  **FastAPI Backend**: Stateless API server, scalable horizontally.
3.  **PostgreSQL**: Persistent relational data storage.
4.  **Redis**: In-memory data store for caching, session management, and Celery message brokering.
5.  **Celery Workers**: Background processing for OCR, file conversion, email sending, and virus scanning.
    -   *Worker Heavy*: CPU-intensive tasks (OCR, Video processing).
    -   *Worker Fast*: Quick tasks (Notifications, simple updates).
6.  **Collabora Online**: WOPI host for document editing.

### 4.3 Database Schema (High-Level)
-   **Users & Auth**: `users`, `roles`, `permissions`, `refresh_tokens`.
-   **Documents**: `documents`, `document_versions`, `tags`, `metadata`.
-   **Structure**: `cabinets`, `drawers`, `folders`.
-   **Workflows**: `workflow_instances`, `tasks`, `workflow_definitions`.
-   **Audit**: `audit_logs`.

---

## 5. Non-Functional Requirements

### 5.1 Performance
-   **Response Time**: API response < 200ms for standard operations.
-   **Search**: Search results returned within 1 second for < 1M records.
-   **Concurrency**: Support for 100+ concurrent active users.

### 5.2 Reliability
-   **Availability**: 99.9% uptime target.
-   **Data Integrity**: ACID compliance via PostgreSQL transactions.
-   **Backup**: Automated daily backups with point-in-time recovery.

### 5.3 Security
-   **Encryption**:
    -   In-transit: TLS 1.2/1.3.
    -   At-rest: Database volume encryption (recommended deployment config).
-   **Input Validation**: Strict sanitization to prevent XSS and SQL Injection.
-   **Protection**: CSRF tokens, Rate limiting on sensitive endpoints.

### 5.4 Scalability
-   **Horizontal Scaling**: Backend services can be scaled behind Nginx.
-   **Database**: Connection pooling allows high throughput; read replicas can be added.
-   **Storage**: Compatible with S3-compatible object storage (future proofing) or scalable block storage.

---

## 6. Deployment & Environment

### 6.1 Prerequisites
-   Docker Engine & Docker Compose.
-   Minimum 4GB RAM, 2 vCPUs.

### 6.2 Configuration
-   Environment variables managed via `.env` file.
-   Sensitive secrets (Database passwords, API keys) must be protected.

### 6.3 Maintenance
-   **Zero-Downtime Migrations**: Alembic configured for auto-patching.
-   **Monitoring**: Health check endpoints for all services.
