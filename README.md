# TaskRunner — Background Job Processing System

A lightweight Java-based job queue with worker thread pooling, retry logic, and status tracking. Built with Spring Boot so it just works out of the box.

---

## Problem

Most background job systems are either over-engineered (Kubernetes + RabbitMQ + a dozen microservices) or too primitive (a cron script that silently fails). I wanted something in the middle — a standalone Java service that can receive jobs via HTTP, process them asynchronously, handle failures gracefully, and let me check on them later.

Real scenarios this covers:
- Trigger a long-running data export without blocking your main API
- Retry a flaky third-party integration a few times before giving up
- Queue up report generation so it doesn't crush your app server
- Offload CPU-heavy work (image processing, PDF generation) to a separate process

You spin it up once and it handles the queuing, worker management, and retry bookkeeping so you don't have to.

---

## Design Decisions

**Why in-memory instead of a real database?**

For a self-contained, single-process deployment, an in-memory `ConcurrentHashMap` plus a `LinkedBlockingQueue` is actually the right call. No external dependencies, no Docker Compose files, no "it worked on my machine but the Redis connection failed in prod." If the process restarts, jobs are lost — but that's documented and acceptable for a first version. Swapping the `JobStore` interface to Redis or Postgres later is straightforward.

**Why Spring Boot?**

It's boring, it works, and every Java developer knows it. The built-in `ScheduledExecutorService` gives us worker threads and retry scheduling without pulling in Quartz or another scheduler dependency. The HTTP layer comes for free. Less ceremony than a custom Netty server, less magic than some reactive framework.

**Why exponential backoff for retries?**

Linear retry (wait 5 seconds, try again) is fine until you have a downstream service that's actually down. Then you hammer it every 5 seconds and make things worse. Exponential backoff (500ms → 1s → 2s → 4s) gives the failing service breathing room and keeps your queue from getting clogged with dead jobs.

**Why a handler registry instead of a big switch statement?**

A `HandlerRegistry` mapping job types to `JobHandler` implementations means adding a new job type is just writing a class and annotating it. No central file to edit, no merge conflicts on a giant switch block. It also makes unit testing trivial — instantiate the handler, pass it a job, assert the result.

---

## What I'd Do Differently at Scale

**Persistence layer.** At any real scale, in-memory storage is a non-starter. I'd swap `JobStore` for a Postgres table with proper indexing on `status` and `createdAt`. That gives you durability across restarts, the ability to query job history, and a foundation for metrics.

**Separate the queue from the workers.** Right now the queue lives in the same process as the HTTP API. If a worker pegs the CPU, your API latency spikes. I'd split into two services: a lightweight API that writes jobs to the database, and a fleet of worker processes that poll for jobs. This lets you scale workers independently.

**Use a real message broker.** Postgres works as a queue up to a few hundred jobs per second, but after that you want Redis (Redisson) or a proper broker like RabbitMQ or AWS SQS. That gives you features like delayed jobs, priority queues, and dead-letter channels for permanently failed jobs.

**Add observability.** At scale you need metrics — queue depth over time, average processing duration, retry rate per job type, worker utilization. I'd wire in Micrometer + Prometheus, and add structured logging with correlation IDs so you can trace a job from submission to completion across services.

**Distributed locking for workers.** With multiple worker instances, you need a way to prevent two workers from picking up the same job. A simple `SELECT FOR UPDATE SKIP LOCKED` on Postgres handles this, or a Redis distributed lock if you're using that.

**Schema evolution for job payloads.** Right now payloads are just JSON strings. In production you'd want versioned schemas so you can change a job's parameters without breaking in-flight jobs.

---

## Known Limitations

- **No persistence.** If the JVM process exits, all queued and in-flight jobs are lost. This is a single-process, in-memory system by design.
- **No horizontal scaling.** Multiple instances would each have their own isolated queue and job store. You can't run two copies of this service and expect them to share work.
- **No authentication.** The HTTP endpoints are wide open. In production you'd want at least an API key or internal network segregation.
- **Fixed retry count.** Retries are hardcoded to 3 attempts with exponential backoff. There's no per-job override or dead-letter queue — after 3 failures, the job just sits in FAILED status forever.
- **No priority queue.** Jobs are processed in FIFO order. An urgent job waits behind a batch of low-priority work.
- **No cancellation.** Once a job is submitted, there's no way to stop it. A RUNNING job will complete even if it's no longer needed.
- **No payload validation.** The API accepts arbitrary JSON payloads. A malformed job won't be caught until a worker tries to process it and fails.
- **Console-only monitoring.** There's no web UI, dashboard, or admin endpoint beyond the raw JSON status check. You hit the API or you don't know what's happening.

---

## Quick Start

```bash
./mvnw spring-boot:run
