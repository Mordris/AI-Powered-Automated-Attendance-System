# AI-Powered Automated Attendance System

This project implements a robust, scalable, and efficient automated attendance system using facial recognition, designed for educational institutions or organizational settings. It features a Python FastAPI backend, asynchronous operations, GPU-accelerated machine learning via Celery, and real-time feedback through WebSockets.



## Core Features

*   **Course and Enrollment Management:** Create courses, enroll students in courses, and manage class sessions specific to each course.
*   **Student Biometric Enrollment:** Securely enroll students by processing multiple facial images to generate robust embeddings.
*   **GPU-Accelerated Face Recognition:**
    *   **Detection:** Utilizes DeepFace with configurable backends (e.g., MTCNN, RetinaFace, YuNet) running on NVIDIA GPUs via Celery workers for high performance.
    *   **Encoding:** Employs Facenet (via DeepFace) on GPUs to generate 128-dimensional facial embeddings.
*   **Efficient Similarity Search:** Integrates FAISS for fast and accurate matching of detected faces against a global database of enrolled student embeddings.
*   **Real-time Attendance Processing:**
    *   Frames submitted to an active class session are processed by Celery workers.
    *   Recognized faces are filtered against the specific course's enrollment list.
    *   Attendance is logged asynchronously to a PostgreSQL database.
*   **WebSocket Real-time Feedback:** Provides instant feedback to connected clients (e.g., camera feed) showing:
    *   Recognized and enrolled students.
    *   Recognized students not enrolled in the current course.
    *   Unknown faces.
*   **Distributed Architecture:**
    *   **FastAPI Backend:** Handles API requests, WebSocket connections, and task dispatching. Runs with multiple Uvicorn workers.
    *   **Celery Workers:** Execute computationally intensive ML tasks on dedicated GPUs.
    *   **Redis:** Serves as a message broker for Celery, a Pub/Sub mechanism for WebSocket updates, and a cache for active session state and enrolled student lists.
    *   **Celery Beat:** Periodically schedules tasks, such as rebuilding the FAISS index from the database.
*   **Database:** PostgreSQL with SQLAlchemy ORM and Alembic for schema migrations.
*   **Containerized:** Fully containerized using Docker and Docker Compose for consistent environments and deployment.
*   **Configurable ML Pipeline:** Face detector backend, detection confidence thresholds, and image pre-processing steps are configurable.
*   **Liveness Detection Placeholder:** Architecture includes a placeholder for future integration of robust liveness detection.

## Technology Stack

*   **Backend:** Python 3.10, FastAPI, Uvicorn
*   **Machine Learning:** TensorFlow, Keras, DeepFace (for MTCNN, RetinaFace, YuNet, Facenet), FAISS, OpenCV, Pillow
*   **Database:** PostgreSQL, SQLAlchemy, Alembic, AsyncPG
*   **Task Queue & Messaging:** Celery, Redis
*   **Real-time Communication:** WebSockets
*   **Containerization:** Docker, Docker Compose
*   **Dependency Management (Development):** Poetry
*   **Core Libraries:** NumPy, Pydantic

## System Workflow Overview

The system operates through a series of interconnected processes to deliver automated attendance:

1.  **Setup & Enrollment:**
    *   Administrators or instructors define courses and create class sessions.
    *   Students are enrolled into the system by submitting facial images. These images are processed asynchronously by GPU-accelerated Celery workers to generate and store unique facial embeddings in a PostgreSQL database.
    *   Students are then formally enrolled into specific courses.

2.  **Attendance Session Initiation:**
    *   An instructor starts an attendance session for a particular class via an API call.
    *   The system activates the session, primarily by setting flags and caching relevant course enrollment data in Redis for quick access.

3.  **Live Frame Processing:**
    *   A camera client (e.g., a webcam connected to a script or a dedicated device) captures video frames.
    *   These frames are sent via HTTP POST to the FastAPI backend.
    *   FastAPI validates the request and dispatches an image processing task to a Celery task queue (managed by Redis).

4.  **Celery Worker - ML Pipeline Execution:**
    *   A Celery worker (running on a machine with GPU access) picks up the task.
    *   The image frame undergoes pre-processing (e.g., resizing, contrast enhancement).
    *   Face detection is performed using the configured DeepFace backend (e.g., MTCNN, RetinaFace) on the GPU.
    *   Detected faces are filtered based on a confidence threshold.
    *   For each confident detection, a (placeholder) liveness check is performed.
    *   If live, a facial embedding is generated using Facenet on the GPU.
    *   This embedding is compared against a global FAISS index (containing embeddings of all enrolled students) to find potential matches.

5.  **Contextual Verification & Logging:**
    *   If a match is found in FAISS, the system verifies if the matched student is enrolled in the specific course for the active session (using data from Redis, with a DB fallback).
    *   If enrolled and not already marked present for this session (checked against Redis), an attendance record is asynchronously logged to the PostgreSQL database.
    *   The student's "present" status for the session is updated in Redis.

6.  **Real-time Feedback via WebSockets:**
    *   The Celery worker publishes the detailed processing result (including detected faces, matched student info, and enrollment status) to a Redis Pub/Sub channel.
    *   The FastAPI backend, subscribed to this channel, receives the result.
    *   It then broadcasts this information via WebSockets to all connected clients for that specific attendance session.
    *   Clients (like the camera client script) display this information, providing real-time visual feedback.

7.  **FAISS Index Maintenance:**
    *   A Celery Beat scheduled task periodically triggers a Celery worker to rebuild the global FAISS index. This task reads all current embeddings from the PostgreSQL database and overwrites the FAISS index file, ensuring new enrollments are incorporated into the searchable index.

8.  **Cache Coherency:**
    *   When a student's enrollment in a course is modified (e.g., added or removed), the system invalidates the cached list of enrolled students in Redis for any active sessions associated with that course. This forces subsequent attendance tasks for those sessions to re-fetch the updated enrollment list from the database.

This distributed and asynchronous architecture ensures that the API remains responsive, ML tasks are efficiently processed using GPUs, and the system can scale to handle multiple concurrent sessions and a large number of students.

## Future Plans

The project has a roadmap for further enhancements to achieve full production-readiness and expand capabilities:

### Phase A: Solidify Core and Add Basic Production Readiness

1.  **Implement Nginx as a Reverse Proxy:**
    *   Serve the FastAPI application via Nginx.
    *   Configure SSL/TLS termination using Let's Encrypt for HTTPS.
    *   Implement rate limiting and basic security headers to protect the application.
2.  **Combined Frontend - Admin Panel:**
    *   Develop a comprehensive web interface using a modern JavaScript framework like React, leveraging a component library such as Material UI.
    *   Key features will include: user authentication, student management (CRUD, biometric enrollment guidance), course management (CRUD), class session administration (create, start/stop), student-course enrollment management, live attendance monitoring via WebSockets, and attendance report generation/download.
    *   Utilize React Router for navigation and Fetch API or Axios for communication with the backend REST APIs.
3.  **Automated Testing (Foundational):**
    *   **Unit Tests:** Implement unit tests for critical functions and classes across the backend modules (e.g., ML components, core logic, database CRUD operations) using Pytest.
    *   **Integration Tests:** Develop integration tests for API endpoints to verify interactions between routers, services, and a test database/Redis instance.
    *   **CI Setup (Basic):** Establish a basic Continuous Integration pipeline (e.g., using GitHub Actions) to automatically run tests on code changes.

### Phase B: Enhance ML Pipeline and Monitoring

1.  **Data Augmentation for Enrollment Images:**
    *   Integrate libraries like `albumentations` or custom OpenCV scripts to augment student enrollment images (e.g., slight rotations, brightness/contrast adjustments, minor blurring) before generating embeddings. This aims to create more robust reference embeddings, improving recognition accuracy under varied conditions.
2.  **ML Model Optimization (TensorRT):**
    *   Explore converting TensorFlow-based models (used by DeepFace for detection and encoding) to NVIDIA TensorRT (TF-TRT) engines. This can provide significant inference speedups on NVIDIA GPUs, further reducing latency in Celery workers.
3.  **Monitoring and Observability (Free Tier Focus):**
    *   **Celery Flower:** Deploy Flower for real-time monitoring and administration of Celery tasks and workers.
    *   **Prometheus and Grafana:** Investigate integrating Prometheus for collecting key application and system metrics (e.g., API request rates/latency from FastAPI using `starlette-prometheus`, Celery task statistics, basic resource usage). Visualize these metrics using Grafana dashboards.

### Phase C: Further Production Hardening (If Time/Need Arises)

1.  **Web Application Firewall (WAF):**
    *   Implement an Nginx-based WAF, potentially utilizing ModSecurity with the OWASP Core Rule Set, to add a robust layer of defense against common web application vulnerabilities and attacks.
2.  **Database Read Replicas (PostgreSQL):**
    *   If the system scales to a point where database read operations become a bottleneck (e.g., for extensive reporting or high API read traffic), explore setting up PostgreSQL read replicas to distribute the read load and improve overall database performance.
3.  **Robust Liveness Detection:** Integrate a production-ready passive liveness detection model.
4.  **Full OAuth2 Authentication & Granular RBAC:** Implement a more robust and standard authentication/authorization mechanism across the system.
5.  **Advanced CI/CD and Deployment Strategies:** Mature CI/CD pipelines for blue/green or canary deployments, and further refine production deployment strategies for high availability and resilience.

---

This README provides a comprehensive overview of the system's architecture, functionality, and future direction.
