### ClassLoader

- **BootStrap ClassLoader** responsible for loading classes from the bootstrap classpath, nothing but **rt.jar**. Highest priority will be given to this loader.
- **Extension ClassLoader** responsible for loading classes which are inside ext folder (**jre\lib**)
- **Application ClassLoader** rLoads jar files from path specified in the CLASSPATH environment variable.


**Dynamic Class Loading** allows the loading of java code that is not known about before a program starts. Many classes rely on other classes and resources such as icons which make loading a single class unfeasible. For this reason the ClassLoader (java.lang.ClassLoader) is used to manage all the inner dependencies of a collection of classes. The Java model loads classes as needed and doesn't need to know the name of all classes in a collection before any one of its classes can be loaded and run.

**NoClassDefFoundError** caused when there is a class file that your code depends on and it is present at compile time but not found at runtime. Look for differences in your build time and runtime classpaths.
NoClassDefFoundError is a fatal error. It occurs when JVM can not find the definition of the class while trying to:
- Instantiate a class by using the new keyword
- Load a class with a method call

Meanwhile **ClassNotFoundException** occurs when an application tries to load a class through its fully-qualified name and can not find its definition on the classpath.
This occurs mainly when trying to load classes using `Class.forName()`, `ClassLoader.loadClass()` or `ClassLoader.findSystemClass()`.


### Execution Engine

- **Interpreter** reads the bytecode, interprets it and executes it one by one. The interpreter interprets the bytecode faster but executes slowly. The disadvantage of the interpreter is that when one method called multiple times, every time interpretation is required.
- **JIT Compiler** when Interpreter found repeated code it uses JIT compiler which compiles the entire bytecode and changes it to native code.  This native code will be used directly for repeated method calls which improve the performance of the system.
- **Garbage Collector** collects/removes the unreferenced objects. GC can be triggered by calling “System.gc()”, but the execution is not guaranteed. Collects only those objects that are created by new keyword. So if you have created any object without new, you can use finalize method to perform cleanup.



### Memory leaks

Example: not closed resources (streams, connections), HashSet/HashMap  with key's incorrect equals/hashCode, substring before some java 7 update, string concatination - not leak but cause memory pressure.

How to avoid memory leaks:
- Store only the necessary data in your **HttpSession**.
- Release the session when it is no longer needed. Use the `HttpSession.invalidate()` to do this.
- Avoid using string concatenation


**Reference** - abstract base class for reference objects that defines the operations common to all reference objects:
- **SoftReference**: GC is required to clear all **SoftReference** objects when memory runs low.
- **WeakReference**: when GC senses a weakly referenced object, all references to it are cleared and ultimately taken out of memory.
	```java
	Counter counter = new Counter();
	WeakReference<Counter> weakCounter = new WeakReference<Counter>(counter);
	counter = null; // now Counter object is eligible for garbage collection
	```
- **PhantomReference**: GC would not be able to automatically clean up **PhantomReference** objects, you would need to clean it up manually by clearing all references to it.


### JVM

If any object is not reachable from the GC roots(even if it is self-referenced or cyclic-referenced) it will be subjected to garbage collection

![Alt description](https://i.stack.imgur.com/IjZqR.png)


#### Phases:
- **Mark**: marking phase is about traversing the whole object graph, starting from GC Roots. When GC visits the object, it marks it as accessible and thus alive. All the objects which are not reachable from GC Roots are garbage. Marking requires stop-the-world (STW) pauses, because the running application threads could interfere.
- **Mark-sweep:** after marking phase, we have the memory space which is occupied by visited and unvisited objects. Sweep phase releases the memory fragments which contains unreachable objects. It is simple, but because the dead objects are not necessarily next to each other, we end up having a fragmented memory.
- **Mark-sweep-compact:** this algorithm fixes the problem with fragmented memory. After all alive objects are marked, they are moved to the beginning of the memory space. That helps to avoid *OutOfMemoryError* caused by too fragmented memory, but compacting the heap isn’t for free. Copying objects and updating all references to them take time and it all happens during STW pause.

#### Types:
- **Serial Collector**
The collection is performed by only one thread. Stop-the-world (STW) pauses are necessary during both Minor and Full GC.
This collector uses the *mark-copy *algorithm for the Young Generation, whereas the Old Generation is cleaned up using the *mark-sweep-compact* algorithm.
- **Parallel (Throughput) Collector**
The Young collection is parallelized by multiple threads which makes Minor GC much faster. As a result, this collector leads to shorter, but more frequent Young collection STW pauses.
Both `-XX:+UseParallelGC` and `-XX:+UseParallelOldGC` flags enable Throughput Collector with parallel processing of both the Old and Young Generations.
This collector also uses the mark-copy algorithm in the Young Generation and mark-sweep-compact in the Old Generation, but both copy and compact phases are executed by multiple threads.
- **Concurrent Mark and Sweep (CMS)**
Minor GC is performed with multiple threads using the parallel mark-copy algorithm. All application threads are stopped then. The Old Generation is mostly collected concurrently – application threads are paused for very short periods of time when the background GC thread scans the Old Generation. The actual algorithm used during Major GC is concurrent mark-sweep -  the collector doesn’t compact the Tenured space and thus the memory can be left fragmented.
Due to lack of heap compaction, when GC is not able to fit new objects into the memory, JVM fallbacks to the serial mark-sweep-compact algorithm to defragment and compact the Old Generation. That’s when performance degradation comes – all application threads are stopped and just one single thread is responsible for cleaning and compacting the Tenured space.
- **G1GC**
Garbage First (G1) is a new low-pause garbage collector designed to process large heaps with minimal pauses. The heap is broken down into several regions of fixed size (while still maintaining the generational nature of the heap). That kind of design allows us to get rid of long STW pauses when the entire Young or Old Generations are processed. Now, each region can be collected separately which leads to shorter, but more frequent STW pauses. G1 copies objects from one region into another, which means that the heap is at least partially compacted.

| Collector | Multiple GC Threads | STW (Young Generation) | STW (Old Generation) | Heap Compaction | Primary Goal |
|-----------|---------------------|------------------------|----------------------|-----------------|--------------|
| Serial | no | yes | yes | yes | - |
| Parallel | yes | yes | yes | yes | throughput |
| CMS | yes | yes | only during scan | no | latency |
| G1 | yes | yes | very short ones | partially | latency |

#### PermGen vs Metaspace

- PermGen is a special **heap space** separated from the main memory heap.
The JVM keeps track of loaded class metadata in the PermGen. Additionally, the JVM stores all the static content in this memory section. This includes all the static methods, primitive variables, and references to the static objects. Furthermore, it contains data about bytecode, names and JIT information.
With its limited memory size, PermGen is involved in generating the famous OutOfMemoryError.
- Metaspace is the **native memory** region that grows automatically by default.
	- *MetaspaceSize* and *MaxMetaspaceSize* – we can set the Metaspace upper bounds.
	- *MinMetaspaceFreeRatio* – is the minimum percentage of class metadata capacity free after garbage collection
	- *MaxMetaspaceFreeRatio* – is the maximum percentage of class metadata capacity free after a garbage collection to avoid a reduction in the amount of space

	Additionally, the garbage collection process also gains some benefits from this change. The garbage collector now automatically triggers cleaning of the dead classes once the class metadata usage reaches its maximum metaspace size.


### JIT

An intrinsic function is a fucntion which can be used by JIT to replace some code for a better performance. An intrinsic function can be implemented with some platform-specififc optimisations.

```
_hashCode             java.lang.Object.hashCode()
_getClass             java.lang.Object.getClass()
_clone                java.lang.Object.clone()
```

To log used intrinsic functions:
```
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintIntrinsics
```