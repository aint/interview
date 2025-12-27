# Java Virtual Machine (JVM)

The **Java Virtual Machine (JVM)** is an execution engine that provides a runtime environment to execute Java bytecode. It converts bytecode into machine-specific code and manages memory, threads, and execution.

## Table of Contents
1. [Core Components](#core-components)
2. [Runtime Data Areas](#runtime-data-areas)
3. [Garbage Collection](#garbage-collection)
4. [Java Memory Model (JMM)](#java-memory-model-jmm)
5. [Concurrency & Virtual Threads](#concurrency--virtual-threads)
6. [Performance Optimizations](#performance-optimizations)
7. [Quick Reference](#quick-reference)

---

## Core Components

The JVM consists of three main subsystems:

1. **Class Loader Subsystem**: Loads, links, and initializes classes
2. **Runtime Data Areas**: Memory areas used during program execution (heap, metaspace, etc)
3. **Execution Engine**: Executes bytecode (interpreter, JIT compiler, GC)
	- Interpreter
		- Reads and executes bytecode line by line
		- Fast interpretation, slower execution
		- Re-interprets code on every execution
	- JIT (Just-In-Time) Compiler
		- Compiles frequently executed bytecode to native code
		- Improves performance for hot methods
		- Uses profiling to identify hot code paths
		- Multiple compilation tiers (C1, C2 compilers)
	- AOT (Ahead-of-Time) Compilation
		- **Java 24+**: Pre-compiles classes before runtime
		- Reduces startup time (Project Leyden)
		- Trade-off: Less runtime optimization compared to JIT
		- Use case: Fast startup requirements (serverless, CLI tools)

---

## Runtime Data Areas

The JVM organizes memory into different runtime data areas. This section covers the structure and organization of these memory areas.

### Heap Structure

- **Young Generation**: New objects allocated here
  - **Eden**: New objects created here
  - **Survivor Spaces (S0, S1)**: Objects that survive minor GC
- **Old Generation**: Long-lived objects promoted from Young Generation

### Metaspace (Java 8+)

Replaced PermGen (removed in Java 8). Stores class metadata in **native memory** (not heap).

**Key differences from PermGen**:
- Grows automatically (no fixed size limit by default)
- Native memory instead of heap space
- Automatic cleanup of dead classes
- Configurable with `-XX:MetaspaceSize` and `-XX:MaxMetaspaceSize`

**Configuration**:
- `MetaspaceSize`: Initial size
- `MaxMetaspaceSize`: Maximum size (unlimited by default)
- `MinMetaspaceFreeRatio`: Minimum free percentage after GC
- `MaxMetaspaceFreeRatio`: Maximum free percentage to avoid space reduction

### GC Roots

Objects reachable from GC roots are considered alive. GC roots include:
- Local variables in active threads
- Static variables
- Active Java threads
- JNI references
- Monitors (synchronized objects)

**Important**: Objects unreachable from GC roots (even if self-referenced or cyclic) are eligible for garbage collection.

![roots](https://i.stack.imgur.com/IjZqR.png)

### Reference Types

- **Strong Reference**: Default reference type. Object cannot be GC'd while referenced.
- **SoftReference**: GC clears when memory is low. Useful for caches.
- **WeakReference**: GC clears immediately when no strong references exist. Useful for weak caches, listeners.
- **PhantomReference**: Used for cleanup actions. Must be manually cleared. GC doesn't automatically clean.

---

## Garbage Collection

### GC Phases

- **Mark**: Traverse object graph from GC roots, mark reachable objects as alive. Requires STW pause.
- **Sweep**: Release memory occupied by unmarked (dead) objects. Can cause fragmentation.
- **Compact**: Move live objects to eliminate fragmentation. Requires updating references (STW pause).

### Modern Garbage Collectors (2025)

#### Serial Collector
- Single-threaded collection
- STW pauses for both Minor and Full GC
- Uses mark-copy (Young) and mark-sweep-compact (Old)
- **Use case**: Small applications, single-core systems

#### Parallel (Throughput) Collector
- Multi-threaded collection
- STW pauses for both generations
- Uses mark-copy (Young) and mark-sweep-compact (Old)
- **Use case**: Throughput-focused applications, batch processing
- Enable: `-XX:+UseParallelGC` or `-XX:+UseParallelOldGC`

#### G1GC (Garbage First)
- Low-pause collector for large heaps
- Divides heap into fixed-size regions
- Collects regions with most garbage first
- Partial compaction during collection
- **Use case**: Large heaps (>4GB), low-latency requirements
- Enable: `-XX:+UseG1GC`

#### ZGC (Z Garbage Collector)
- Ultra-low-latency collector (sub-millisecond pauses)
- Concurrent marking, relocation, and compaction
- Scales to multi-terabyte heaps
- **Use case**: Large heaps, strict latency requirements
- Enable: `-XX:+UseZGC`
- **Status**: Production-ready since Java 15

#### Shenandoah
- Low-latency collector with concurrent evacuation
- Concurrent marking and compaction
- Pause times independent of heap size
- **Use case**: Low-latency applications, large heaps
- Enable: `-XX:+UseShenandoahGC`
- **Status**: Production-ready since Java 12

| Collector | GC Threads | STW (Young) | STW (Old) | Compaction | Primary Goal | Status |
|-----------|------------|-------------|-----------|------------|--------------|--------|
| Serial | Single | Yes | Yes | Yes | - | Production |
| Parallel | Multiple | Yes | Yes | Yes | Throughput | Production |
| G1 | Multiple | Short | Very short | Partial | Latency | Production |
| ZGC | Multiple | Minimal | Minimal | Concurrent | Ultra-low latency | Production (Java 15+) |
| Shenandoah | Multiple | Minimal | Minimal | Concurrent | Low latency | Production (Java 12+) |

**Note**: CMS (Concurrent Mark and Sweep) was deprecated in Java 9 and removed in Java 14. Use G1, ZGC, or Shenandoah for low-latency requirements.

### GC Triggers

- **Automatic**: When heap is full or thresholds are reached
- **Manual**: `System.gc()` (execution not guaranteed, generally discouraged)
- **Finalization**: `finalize()` method deprecated in Java 9, removed in Java 18. Use `Cleaner` API or try-with-resources instead.

---

## Java Memory Model (JMM)

The **Java Memory Model (JMM)** defines how threads interact through memory. It specifies the semantics of `volatile`, `synchronized`, and `final`, and ensures visibility and ordering guarantees in multi-threaded programs.

**Key distinction**: JMM is about **concurrency semantics** (visibility, ordering), not memory structure (heap/stack).

### Memory Visibility Problem

Without proper synchronization, changes made by one thread may not be visible to other threads due to:
- **CPU caches**: Each CPU core has its own cache
- **Compiler optimizations**: Reordering instructions for performance
- **CPU instruction reordering**: Out-of-order execution

**Example of visibility issue**:
	```java
// Thread 1
flag = true;  // May not be visible to Thread 2 immediately
value = 42;

// Thread 2
while (!flag) { }  // May loop forever
System.out.println(value);  // May print 0, not 42
```

### Happens-Before Relationship

The JMM defines **happens-before** relationships that guarantee ordering and visibility:

1. **Program order**: Actions in a thread happen in program order
2. **Monitor lock**: Unlock happens-before subsequent lock of same monitor
3. **Volatile**: Write to volatile variable happens-before subsequent read
4. **Thread start**: `Thread.start()` happens-before actions in started thread
5. **Thread join**: Actions in thread happen-before `Thread.join()` returns
6. **Final fields**: Constructor writes to final fields happen-before object publication

**Happens-before guarantees**:
- If A happens-before B, then A's results are visible to B
- If A happens-before B and B happens-before C, then A happens-before C (transitivity)

### Volatile

`volatile` provides:
- **Visibility**: Changes are immediately visible to all threads
- **Ordering**: Prevents reordering of volatile operations with other memory operations
- **No atomicity**: For compound operations (e.g., `count++`), use `AtomicInteger` or `synchronized`

**Volatile write happens-before volatile read**:
```java
volatile boolean flag = false;
volatile int value = 0;

// Thread 1
value = 42;      // Normal write
flag = true;     // Volatile write - establishes happens-before

// Thread 2
if (flag) {      // Volatile read - sees flag = true
    // Guaranteed to see value = 42 due to happens-before
    System.out.println(value);
}
```

**Use cases**:
- Simple flags (`volatile boolean running`)
- Single-writer, multiple-reader scenarios
- Publishing immutable objects

### Synchronized

`synchronized` provides:
- **Mutual exclusion**: Only one thread can execute synchronized block
- **Visibility**: All changes made inside synchronized block are visible to threads that acquire the same lock
- **Ordering**: Synchronized blocks establish happens-before relationships

**Synchronized visibility guarantee**:
```java
synchronized (lock) {
    shared = 42;  // Write
}  // Unlock - happens-before subsequent lock

synchronized (lock) {
    // Lock - guaranteed to see shared = 42
    System.out.println(shared);
}
```

### Final Fields

Final fields have special semantics:
- **Immutable after construction**: Cannot be modified after object construction
- **Safe publication**: Properly constructed object with final fields can be safely shared without synchronization
- **Freeze**: Final field writes "freeze" before constructor completes

**Final field guarantee**:
```java
class MyClass {
    final int value;  // Final field

    MyClass(int v) {
        this.value = v;  // Write to final field
    }
}

// Thread 1
MyClass obj = new MyClass(42);  // Constructor completes
shared = obj;  // Publish object

// Thread 2
MyClass obj = shared;  // May see partially constructed object
// But if obj is not null, obj.value is guaranteed to be 42
```

**Important**: For safe publication, the reference itself must be safely published (via `volatile`, `synchronized`, or `final`).

### Atomicity

**Atomic operations**:
- Reads and writes to primitive types (except `long` and `double`)
- Reads and writes to `volatile` variables (including `long` and `double`)
- Reads and writes to reference variables

**Non-atomic operations**:
- `long` and `double` reads/writes (64-bit) on 32-bit JVMs (use `volatile` for atomicity)
- Compound operations: `count++`, `x = x + 1` (use `AtomicInteger` or `synchronized`)

### Instruction Reordering

The JVM and CPU can reorder instructions for performance, but must respect **happens-before** relationships.

**Reordering example**:
```java
// Thread 1
x = 1;
y = 2;  // May be reordered before x = 1

// Thread 2
if (y == 2) {
    // x may still be 0 here (reordering allowed)
}
```

**Preventing reordering**:
- `volatile`: Prevents reordering across volatile operations
- `synchronized`: Prevents reordering across synchronized boundaries
- `final`: Ensures proper ordering in constructors

---

## Concurrency & Virtual Threads

### Platform Threads (Traditional)

- One-to-one mapping with OS threads
- Heavyweight (1MB+ stack per thread)
- Limited scalability (thousands of threads)
- Context switching overhead

### Virtual Threads (Project Loom - Java 21+)

- **Lightweight threads** managed by JVM
- Many-to-one mapping with OS threads (carrier threads)
- Minimal memory footprint (few KB per thread)
- Can create millions of virtual threads
- **Blocking is cheap**: No OS thread blocking

**Key benefits**:
- High concurrency with minimal resource usage
- Simplified code (no thread pools needed for I/O-bound tasks)
- Better scalability for I/O-intensive applications

**When to use**:
- I/O-bound tasks (network, file operations)
- High-concurrency scenarios
- Replacing thread pools for blocking operations

**When not to use**:
- CPU-intensive tasks (use platform threads or ForkJoinPool)
- Tasks requiring thread-local storage (limited support)

### Structured Concurrency (Java 25+)

- Manages groups of related tasks as a unit
- Simplifies error handling and cancellation
- Prevents thread leaks
- Better for parallel processing and AI workloads

---

## Performance Optimizations

### Compact Object Headers (Java 25+)

- Reduces per-object memory overhead
- Optimizes object header layout
- Improves performance for applications with many short-lived objects

### Class Data Sharing (CDS)

- Shares class metadata across JVM instances
- Reduces startup time and memory footprint
- Archive created with `-Xshare:dump`
- Enabled by default in recent Java versions

### Java Flight Recorder (JFR)

- Low-overhead profiling and diagnostics
- Built into JVM (no external tools needed)
- **Java 25+**: CPU-time-based profiling for multi-threaded applications
- Continuous recording with minimal performance impact

---


## Quick Reference

### Common Interview Questions

**Q: How does the JVM handle memory management?**
- Heap stores objects (Young/Old generations)
- Stack stores method calls and local variables
- Metaspace stores class metadata
- GC automatically reclaims unreachable objects

**Q: Explain the difference between JIT and AOT compilation.**
- **JIT**: Compiles bytecode to native code at runtime based on profiling. Better peak performance.
- **AOT**: Pre-compiles classes before runtime. Faster startup, less runtime optimization.

**Q: What are virtual threads and when should you use them?**
- Lightweight threads managed by JVM (many-to-one with OS threads)
- Minimal memory footprint, can create millions
- Ideal for I/O-bound, high-concurrency tasks
- Not ideal for CPU-intensive work

**Q: Compare modern garbage collectors (G1, ZGC, Shenandoah).**
- **G1**: Low-pause, good for large heaps, partial compaction
- **ZGC**: Ultra-low latency (sub-ms), concurrent compaction, scales to TB heaps
- **Shenandoah**: Low latency, concurrent evacuation, pause times independent of heap size

**Q: How do you identify and fix memory leaks?**
- Use heap dumps (`jmap`, `-XX:+HeapDumpOnOutOfMemoryError`)
- Analyze with tools (Eclipse MAT, VisualVM)
- Look for growing collections, unclosed resources, static references
- Fix: Proper resource management, correct equals/hashCode, cleanup listeners

**Q: What are GC roots and why are they important?**
- Starting points for reachability analysis
- Include: local variables, static variables, active threads, JNI references
- Objects reachable from GC roots are alive; unreachable objects are eligible for GC

**Q: What is the happens-before relationship?**
- Defines ordering and visibility guarantees between operations
- If A happens-before B, A's results are visible to B
- Established by: synchronized, volatile, thread start/join, final fields
- Transitive: If A happens-before B and B happens-before C, then A happens-before C

**Q: What's the difference between volatile and synchronized?**
- **volatile**: Visibility and ordering for single variable, no mutual exclusion
- **synchronized**: Mutual exclusion, visibility, and ordering for code blocks
- **volatile**: Lighter weight, but limited to single variable
- **synchronized**: Heavier, but provides atomicity for compound operations

**Q: Why might a thread not see updates from another thread?**
- CPU caches: Each core has its own cache
- Compiler optimizations: Reordering instructions
- Missing synchronization: No happens-before relationship established
- **Solution**: Use `volatile`, `synchronized`, or `Atomic*` classes

**Q: What're the intrinsic functions?**
Platform-specific optimizations for common operations:
- `Object.hashCode()`
- `Object.getClass()`
- `Object.clone()`
- `System.arraycopy()`
- Many `Math` and `String` operations
