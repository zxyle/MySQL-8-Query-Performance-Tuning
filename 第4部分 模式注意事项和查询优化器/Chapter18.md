# 锁原理与监控

## Why Are Locks Needed?

## Lock Access Levels

## Lock Granularity

### User-Level Locks

### Flush Locks

### Metadata Locks

### Explicit Table Locks

### Implicit Table Locks

### Record Locks

### Gap Locks, Next-Key Locks, and Predicate Locks

### Insert Intention Locks

### Auto-increment Locks

### Backup Locks

### Log Locks

## Failure to Obtain Locks

### Metadata and Backup Lock Wait Timeouts

### InnoDB Lock Wait Timeouts

### Deadlocks

## Reduce Locking Issues

### Transaction Size and Age

### Indexes

### Record Access Order

### Transaction Isolation Levels

### Preemptive Locking

## Monitoring Locks

### The Performance Schema

### The sys Schema

### Status Counters and InnoDB Metrics

### InnoDB Lock Monitor and Deadlock Logging

## Summary