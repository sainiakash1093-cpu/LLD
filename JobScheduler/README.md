# Job Scheduler LLD (C#)

## Problem Statement
Design a Job Scheduler that allows clients to schedule jobs and workers to execute them safely under concurrency.

---

## Functional Requirements
1. Create/Schedule a job
2. Support one-time and recurring jobs
3. Workers should fetch and execute due jobs
4. Retry on failure

---

## Non-Functional Requirements
1. Concurrency (multiple workers)
2. Consistency (no duplicate execution)
3. Low latency (fast pickup of due jobs)
4. Extensibility (new schedules/retry policies)

---

## Scheduling Supported
- Immediate
- One-time
- Recurring (interval / cron)

---

## Core Entities

### JobDefinition
- Stores job metadata and configuration
- Fields: JobId, JobType, Payload, Schedule, RetryPolicy, IsActive

### JobInstance
- Represents one execution
- Fields: InstanceId, JobId, ScheduledAtUtc, Status, AttemptCount, LeaseOwner, LeaseExpiryUtc

### JobSchedule
- Defines timing logic
- Fields: ScheduleType, StartAtUtc, Interval, CronExpression, NextRunAtUtc

### RetryPolicy
- Controls retry behavior
- Fields: MaxRetries, BackoffType, Delay

---

## Storage
- Persistent DB (source of truth)
- Stores JobDefinitions and JobInstances

---

## Concurrency & Consistency
- Use atomic claim (Scheduled → Claimed)
- Only one worker succeeds
- Use lease (LeaseOwner, LeaseExpiryUtc) for recovery

---

## Design Patterns
- Strategy (schedule, retry)
- Repository (data access)
- Facade (JobSchedulerService)
- State (job transitions)

---

## Interfaces

```csharp
public interface IJobSchedulerService
{
    Task<string> CreateJobAsync(CreateJobRequest request, CancellationToken ct);
}
```

```csharp
public interface ISchedulerPoller
{
    Task PollAndDispatchDueJobsAsync(CancellationToken ct);
}
```

```csharp
public interface IJobExecutor
{
    Task ExecuteAsync(string jobInstanceId, CancellationToken ct);
}
```

```csharp
public interface IScheduleStrategy
{
    DateTime? ComputeNextRunUtc(JobSchedule schedule, DateTime referenceUtc);
}
```

```csharp
public interface IRetryStrategy
{
    bool CanRetry(JobInstance instance, RetryPolicy policy);
    DateTime ComputeNextRetryUtc(JobInstance instance, RetryPolicy policy, DateTime nowUtc);
}
```

---

## Classes

### JobSchedulerService
- Creates job + first instance

### SchedulerPoller
- Fetches due jobs
- Claims and dispatches

### JobExecutor
- Executes job
- Handles retry/reschedule

### JobHandlerFactory
- Resolves handler by JobType

---

## Flow

### Create Job
Client → JobSchedulerService → DB → JobDefinition + JobInstance

### Poll
SchedulerPoller → Fetch due → TryClaim → Dispatch

### Execute
JobExecutor → Run → Success/Fail → Retry or Next Schedule

---

## Summary
- JobDefinition = what to run
- JobInstance = execution unit
- Atomic claim ensures no duplicates
- Strategy handles scheduling & retry
