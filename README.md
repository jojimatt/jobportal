# Job Portal — Spring Boot Backend

A full-featured job portal REST API built with Spring Boot as part of Madan Reddy's Udemy coursework. The backend handles multi-role authentication, job postings, applications, company management, and user profiles.

---
## Acknowledgements
This project was developed as part of the Master Spring 7, Spring Boot 4, REST, JPA, Security course by Madan Reddy (completed on May 2026)
The implementation follows the course curriculum.
Course completion certificate : https://www.udemy.com/certificate/UC-1ad93fe8-82dd-4d2a-ab9d-db9cabc292ce/

## Table of Contents

- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Features](#features)
- [Roles & Access Control](#roles--access-control)
- [API Endpoints](#api-endpoints)
- [Database Schema](#database-schema)
- [Configuration](#configuration)
- [Running Locally](#running-locally)
- [Caching](#caching)
- [Observability](#observability)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 25 |
| Framework | Spring Boot 4.0.1 |
| Security | Spring Security + JWT (JJWT 0.13.0) |
| Persistence | Spring Data JPA + Hibernate |
| Database | MySQL |
| Cache | Caffeine (in-memory) |
| Validation | Spring Validation (Bean Validation) |
| API Docs | SpringDoc OpenAPI / Swagger UI |
| Monitoring | Spring Actuator |
| Observability | OpenTelemetry (traces, metrics, logs) |
| Build | Maven |
| Dev Tools | Spring DevTools, Spring Docker Compose |

---

## Project Structure

```
jobportal/
└── src/main/java/com/eazybytes/jobportal/
    ├── auth/               # Login & registration
    ├── company/            # Company management
    ├── contact/            # Contact form submissions
    ├── job/                # Job postings & applications
    ├── user/               # User profiles, saved jobs
    ├── entity/             # JPA entities
    ├── dto/                # Request/response DTOs
    ├── repository/         # Spring Data repositories
    ├── security/           # JWT filters, security config, CORS
    ├── cache/              # Caffeine cache configuration
    ├── audit/              # JPA auditing
    ├── constants/          # Application constants
    ├── otel/               # OpenTelemetry initializer
    └── JobportalApplication.java
```

---

## Features

- **JWT Authentication** — stateless login with configurable token expiry
- **Role-based Authorization** — separate access for Job Seekers, Employers, and Admins
- **Job Postings** — employers can create, update status, and view applications for their jobs
- **Job Applications** — job seekers can apply, withdraw, and track application status
- **Saved Jobs** — job seekers can bookmark jobs for later
- **User Profiles** — profile creation with picture and resume file uploads (stored as BLOBs)
- **Company Management** — admin CRUD for companies; public read for active-job companies
- **Contact Form** — public submission with admin review, sorting, and pagination
- **Caching** — Caffeine cache on jobs, companies, and roles with configurable TTLs
- **JPA Auditing** — `createdAt`, `createdBy`, `updatedAt`, `updatedBy` on all entities
- **Compromised Password Check** — HaveIBeenPwned API integration via Spring Security
- **OpenAPI Docs** — auto-generated Swagger UI
- **Actuator** — health, env, and metrics exposed on port `9090`

---

## Roles & Access Control

| Role | Description |
|---|---|
| `ROLE_JOB_SEEKER` | Default role on registration. Can manage profile, apply/save jobs. |
| `ROLE_EMPLOYER` | Assigned by admin. Can post jobs and manage applications. |
| `ROLE_ADMIN` | Full access. Manages companies, users, contacts. |

Path suffixes reflect role restrictions:
- `/public` — no authentication required
- `/jobseeker` — `ROLE_JOB_SEEKER` only
- `/employer` — `ROLE_EMPLOYER` only
- `/admin` — `ROLE_ADMIN` only

---

## API Endpoints

All endpoints are prefixed with `/api`.

### Auth

| Method | Path | Description |
|---|---|---|
| POST | `/auth/register/public` | Register a new user (default role: JOB_SEEKER) |
| POST | `/auth/login/public` | Login and receive a JWT token |

### Users

| Method | Path | Description |
|---|---|---|
| GET | `/users/search/admin` | Search user by email |
| PATCH | `/users/{userId}/role/employer/admin` | Elevate user to EMPLOYER role |
| PATCH | `/users/{userId}/company/{companyId}/admin` | Assign a company to an employer |
| PUT | `/users/profile/jobseeker` | Create or update profile (multipart: JSON + picture + resume) |
| GET | `/users/profile/jobseeker` | Get authenticated user's profile |
| GET | `/users/profile/picture/jobseeker` | Download profile picture |
| GET | `/users/profile/resume/jobseeker` | Download resume |
| POST | `/users/saved-jobs/{jobId}/jobseeker` | Save a job |
| DELETE | `/users/saved-jobs/{jobId}/jobseeker` | Remove a saved job |
| GET | `/users/saved-jobs/jobseeker` | List all saved jobs |
| POST | `/users/job-applications/jobseeker` | Apply for a job |
| DELETE | `/users/job-applications/{jobId}/jobseeker` | Withdraw an application |
| GET | `/users/job-applications/jobseeker` | List user's applications |

### Jobs

| Method | Path | Description |
|---|---|---|
| GET | `/jobs/employer` | Get all jobs posted by the authenticated employer |
| POST | `/jobs/employer` | Create a new job posting |
| PATCH | `/jobs/{jobId}/status/employer` | Update job status |
| GET | `/jobs/applications/{jobId}/employer` | Get applications for a specific job |
| PATCH | `/jobs/applications/employer` | Update an application's status and notes |

### Companies

| Method | Path | Description |
|---|---|---|
| GET | `/companies/public` | List companies with active job postings |
| GET | `/companies/admin` | List all companies |
| POST | `/companies/admin` | Create a company |
| PUT | `/companies/{id}/admin` | Update company details |
| DELETE | `/companies/{id}/admin` | Delete a company |

### Contacts

| Method | Path | Description |
|---|---|---|
| POST | `/contacts/public` | Submit a contact form message |
| GET | `/contacts/admin` | Get all new contact messages |
| GET | `/contacts/sort/admin` | Get messages with custom sorting |
| GET | `/contacts/page/admin` | Get messages with pagination and sorting |
| PATCH | `/contacts/{id}/status/admin` | Update contact message status |

---

## Database Schema

### Entities & Relationships

```
JobPortalUser
 ├── role          → Role             (many-to-one)
 ├── company       → Company          (many-to-one, nullable)
 ├── profile       → Profile          (one-to-one)
 ├── savedJobs     → Job[]            (many-to-many, join table: saved_jobs)
 └── jobApplications → JobApplication[] (one-to-many)

Company
 └── jobs          → Job[]            (one-to-many, cascade all)

Job
 └── jobApplications → JobApplication[] (one-to-many)

JobApplication
 ├── user          → JobPortalUser    (many-to-one)
 └── job           → Job             (many-to-one)
```

### Job Application Status Flow

`PENDING` → `IN_REVIEW` → `INTERVIEW` → `HIRED` / `REJECTED`

### Audit Fields (on all entities)

All entities extend `BaseEntity` which provides:
- `createdAt` — set on insert
- `createdBy` — set on insert
- `updatedAt` — set on update
- `updatedBy` — set on update

---

## Configuration

Key properties in `application.properties`:

```properties
# Database
DATABASE_HOST=localhost
DATABASE_PORT=3306
DATABASE_NAME=jobportal
DATABASE_USERNAME=root
DATABASE_PASSWORD=root

# JWT
JWT_SECRET=<your-secret-key>

# Actuator (runs on port 9090)
management.server.port=9090
management.endpoints.web.base-path=/jobportal/actuator

# CORS
cors.allowed-origins=http://localhost:5173,https://dev.jobportal.eazybytes.com
```

All database and JWT values can be overridden via environment variables.

---

## Running Locally

### Prerequisites

- Java 25+
- Maven 3.8+
- Docker for running MySQL. Docker starts automatically with services mentioned in Spring

---
## Steps
- Change file path in compose.yml for Sql database persistent volume
- Connect to database and create all tables in jobportal-schema.sql and jobportal-data.sql
- Open jobporatl ui folder. Run 'npm install' followed by 'npm run dev'. Open the localhost:5173 web to test out the applications
- Before proceeding. Register three accounts with the same credentials as the demo credentials in the UI.
- Change the roles id of the three users to the respective roles in Users database to test out all functionalities

## Caching

Caffeine in-memory caching is applied to frequently read, rarely changed data:

| Cache Name | TTL | Max Size | Eviction |
|---|---|---|---|
| `jobs` | 10 minutes | 5000 | On company create/update/delete |
| `companies` | 10 minutes | 500 | On company create/update/delete |
| `roles` | 1 day | 100 | — |

---

## Observability

OpenTelemetry is configured to export to an OTLP collector at `http://localhost:4318`:

| Signal | Endpoint |
|---|---|
| Traces | `/v1/traces` |
| Metrics | `/v1/metrics` |
| Logs | `/v1/logs` |

Sampling probability is set to 100% by default. Connect any OTEL-compatible collector (e.g., Jaeger, Grafana Tempo, OpenTelemetry Collector) on port `4318`.

---

## Related

- **Frontend:** `job-portal-ui/` (separate Vite/React app)

