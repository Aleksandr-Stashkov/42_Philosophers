# 🍝 Philosophers project at 42 Prague

![C Language](https://img.shields.io/badge/Language-C-blue.svg)
![Score](https://img.shields.io/badge/Score-125%20%2F%20100-success.svg)
![Topic](https://img.shields.io/badge/Topic-Concurrency%20%26%20Threads-orange.svg)

An elegant, highly optimized solution to the classic **Dining Philosophers Problem**, developed as part of the core curriculum at 42 Prague. This project serves as a deep dive into system-level multithreading, concurrent resource allocation, mutual exclusion primitives, and strict timing constraints.

---

## ⚠️ Integrity Notice
In compliance with 42's guidelines, the source code is **not publicly hosted** in this repository. 
* **Recruiters and Reviewers:** If you are a recruiter looking to review the source files, please reach out to me directly, and I will gladly provide access to the code upon request.

---

## 📌 Project Architecture

The simulation models a roundtable of philosophers who must alternatively **think**, **eat**, and **sleep**. To eat, a philosopher must acquire two forks. The core engineering challenge revolves around ensuring the table operates continuously without deadlocks and data races.

### Solution Components
* **Philosopher Threads (`pthread_t`)**: Each individual philosopher runs on an isolated lifecycle loop (`life`).
* **Dedicated Watcher Thread (`watcher`)**: An asynchronous monitoring system that constantly audits state parameters across the threads to detect death or verify meal count limits.
* **Localized State Primitives**: Resource allocation logic relies on mutex locks to separate data streams.

---

## 🚀 Key Engineering Accomplishments & Optimization

### 1. Asymmetric Resource Ordering (Deadlock Avoidance)
To break the circle of deadlocks, the initialization routine enforces an alternating fork assignments scheme based on the philosopher's parity:

```c
// Implementation excerpt from setup.c
if (id % 2)
{
  phil->fork2 = data->mutexes.forks[id - 1];
  if (id == data->n_phil)
    phil->fork1 = data->mutexes.forks[0];
  else
    phil->fork1 = data->mutexes.forks[id];
}
else
{
  phil->fork1 = data->mutexes.forks[id - 1];
  if (id == data->n_phil)
    phil->fork2 = data->mutexes.forks[0];
  else
    phil->fork2 = data->mutexes.forks[id];
}
```
Because the identifiers assigned to `fork1` and `fork2` are inverted dynamically between adjacent neighbors, philosophers sitting next to each other attempt to lock opposite physical resources first. This ensures a starvation cycle will not form by default.

### 2. High-Precision Microsecond Pacing

Standard UNIX timing functions like 'usleep()' suffer from substantial scheduling drift and overhead under heavy thread contention. To solve this, a custom time-tracking utility steps through the required duration using micro-intervals:

* It samples system time precisely via 'gettimeofday'.

* It executes minute 500-microsecond context switches via `usleep(500)`.

* It continuously verifies the global `should_stop` state flag on every mini-step, enabling **immediate simulation termination** if any thread experiences starvation.

### 3. Odd Configuration Desynchronization Pacing
In configurations containing an odd number of philosophers, a clean binary alternation of schedules is impossible. Without intervention, thread overlapping causes scheduling drifts that starve the final philosopher. This solution implements a pacing offset inside the thinking loop:

```c
// Implementation excerpt from routines.c
if ((phil->data->n_phil % 2) && !slept_well(phil->data,
    time_mark(time_shift) + (phil->data->time_to_eat * 0.65 * 1000), time_shift))
		return (0);
```

Delaying odd configurations by a calculated slice of the eating time `time_to_eat * 0.65` prevents synchronization collapse.

### 4. Zero Data Races via Granular Mutex Isolation
To achieve clean thread sanitizer metrics, each philosopher manages their own structural dependencies:

* `death_time_mtx`: Isolates updates to starvation thresholds.

* `times_ate_mtx`: Isolates checks on meal count goals.

* `should_stop_mtx`: Secures the teardown signaling loop.

### 5. Single Philosopher Edge Case Handling
If the configuration contains exactly one philosopher, eating is impossible due to the lack of a second fork. The project catches this scenario cleanly via `single_life()`, where the single thread claims its sole localized fork, waits for its scheduled starvation window to lapse safely without blocking, and exits immediately when intercepted by the `watcher` thread.

### 6. The Bonus Solution: Multi-Processing & POSIX Named Semaphores
While the mandatory part focuses on shared address spaces via threads, the bonus solution pivots entirely to an isolated process architecture using `fork()` and global signaling primitives:

* **Isolated Memory Address Spaces:** Each philosopher runs as a completely independent child process rather than an inline thread. This prevents any possibility of shared memory leaks across actors and isolates execution crashes.
* **Global Synchronization via Named Semaphores:** Instead of an array of localized mutexes, synchronization is managed globally using POSIX Named Semaphores (`sem_t`).
* **Centralized Fork Pool Allocation:** The table's forks are represented as a single, centralized counting semaphore initialized to the total number of philosophers. This naturally satisfies resource constraints without requiring index swapping.
* **Rigorous Resource Lifecycle Guarding:** Because named semaphores persist in the kernel even after process termination, the system implements a strict teardown sequence. The cleanup routine executes `sem_close` and `sem_unlink` across all registered primitives guaranteeing that zero stale semaphores clog the operating system after simulation exit.
