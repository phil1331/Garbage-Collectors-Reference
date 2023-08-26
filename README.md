# Garbage Collection in Java (Reference)
## (Automatic) Memory Management
* In C memory has to be managed manually (on the heap) </br>
&rarr; It has to be allocated (see [malloc(3)](https://www.man7.org/linux/man-pages/man3/malloc.3.html), [calloc(3p)](https://man7.org/linux/man-pages/man3/calloc.3p.html))</br>
&rarr; And it has to be deleted (see [free(3)](https://man7.org/linux/man-pages/man3/free.3p.html)) </br>
&rarr; Which is why concepts such as RAII came up (C++, std::unique_ptr, std::shared_ptr, ...)
* In Java the Garbage Collector does the job
* The garbage collector runs as a daemon
* It cleans up unused (unreachable) memory (also referred to as garbage) and even manages the heap space
## Garbage Collectors
### Possible operations are mark, sweep and compact
1. mark all reachable memory (meaning memory that is referenced by the program)
2. sweep the unused memory 
3. order the fragmented memory into contiguous chunks ("erase" / arange the unused space between memory chunks)
### Generational Collectors
* Based on the perception, that most objects are meant to be alive short-term
* The heap is divided into 2 parts, the young and the old generation
* A generation is a set of objects of similar age
#### Young generation
* Principle: All new and newer objects are living in the young generation part in heap memory
* Garbage Collection: A minor garbage collection cleans the young generation up, i.e. it deletes ("sweeps") unreachable objects
* The young generation itself consists in the basic generational collector model of 3 parts
1. Eden
2. Survivor 1
3. Survivor 2
* Eden is where newly allocated objects are stored in
* When a minor gc happens, first reachable objects are marked, second moved into survivor 1 and given a survival-counter, third unreachable objects in eden are sweeped
* The next minor gc would do the same just with the newly created objects in eden and older objects in survivor 1 and a destination of survivor 2, so the compacting step doesn't have to be made (it is quite expensive to arange the memory the right way)
* This process is being repeated after every minor gc with a switching destination space between survivor 1 and 2
* This means that the minor gc is actually not doing any compacting at all
#### Old ([tenured](https://dictionary.cambridge.org/dictionary/english/tenured)) generation
* After the survival-counter has reached a certain value, objects are moved ("promoted") into the old generation space
* The old generation gets cleaned up along with the young generation by the so called major gc which will perform the mark, sweep and compact step on the whole heap
* This means that the major gc is expensive and might cause a serious pause in the program
* When the old generation space is full a `java.lang.OutOfMemoryError` is being thrown
#### Metaspace / Permanent block
* Here JVM metadata and objects that are expected to exist for the entire duration (time the jvm runs) are stored
* Class names, method names, etc.
### Key Performance Indicators
1. [Latency](https://dictionary.cambridge.org/dictionary/english/latency) (time that the gc pauses the application (STW); useful for applications that require constant responsiveness)
2. Throughput (the percentage of how much of the runtime was actually spend by the program and not the garbage collector; useful for needed complex computations)
3. Footprint (Memory, CPU-Usage)
### Print the information (logs) about the garbage collector in your application
* Add following parameter to your JVM executable: `-Xlog:gc*:file={MYFILENAME}` where `{MYFILENAME}` is for example equal to `gc.log`
### Info
* `System.gc()` causes either nothing or a full STW collecting process which is why most you shouldn't call this in most applications
* the object alignment $k$ determines the size $s$ of each object by the equation $s\equiv 0 \pmod{k}$ or $s = nk$ where $n\in\mathbb{N}_0$, where the default $k$ of most jvm's is 8 (object alignment of 8)
* coops (compressed ordinary object pointers) are pointers to objects with 32 bit size on 64 bit systems with object alignment of 8 bytes for heap sizes less than 32 GiB
### Common Types of Garbage Collectors in Java
#### Stop-The-World (STW)
* Both minor and major garbage collectors are stopping the world/program when they are executed, the minor one just collects on a smaller amount of memory
#### Serial Collector
* The most classic/basic collector
* Performs garbage collecting sequentially after the memory has gotten full where the program is being stopped
* Runs in a single thread
#### Parallel Collector / Throughput-Collector
* is basically the serial collector but collects in parallel/on multiple cores when the respective generations or sub-generations get full
* was the default collector from Java 1.5 to 1.8
* stops-the-world when it runs
* usage: mainly for applications that value throughput the most
#### Concurrent Mark-Sweep (CMS) Collector
* Cleans up memory after some time, doesn't let the generations get full
* Stops the program if and only if the mark operations takes place
* Runs in parallel with the program
* Problem: leaves heap fragmanted, doesn't do the compacting step
* Deprecated since Java 9, removed since Java 14
* Usage: [Responsiveness](https://dictionary.cambridge.org/dictionary/english/responsiveness) / low latency (time that the garbage collector pauses) as no significant pauses are happening while the program is being executed
#### G1 (Garbage first) Collector
* Runs concurrently
* Default garbage collector from Java 9 until today
* User can provide a flag to the jvm which specifies the latency which the G1 collecotor should *trie* to not overstep (STW time): 200ms normal, 50 ms maybe, 10 ms no
* Heap memory is fragmented into equally sized chunks where young generation chunks and old generation chunks are specially marked
* When minor garbage collection runs all the young generation chunks are marked in parallel and the ones with the most unreachable objects are collected
* Combines the CMS and parallel collector, but focuses more on the latency KPI
* G1 is the only GC compatible with string deduplication which may save over $10\%$ of the programs heap usage! (might be an indicator to use G1 for low-footprint-needing applications)
* Usage: Keep a balance between the CMS and parallel GC with optimizations
#### Shenandoah GC
* goal: minimize GC pauses
* also runs concurrently and minimizes GC pauses
* adapts advantages of CMS and G1
* does fix CMS by introducing concurrent compacting, so the heap doesn't get fragmented
* GC pauses down to 1ms (very low latency)
* saves survival information in the object header (mark)
* use for low-latency applications (and consider prefering this for applications where the heap size is expected to be less than 32 GiB)
#### Z Garbage Collector (ZGC)
* the goal is the same as the one from shenandoah (reducing latency)
* runs concurrently, does compacting step
* It is similar to G1
* It uses heap chunks of different sizes
* saves information about the objects in the pointers (colored pointers) itself and reserves bits for that purpose (coops cannot be used!)
* that's why is is rather great with larger heaps
* does not stop the application for more than 10ms
#### Epsilon GC
* Does *nothing*
* Experimental, for short applications where the developer is conscious about the respective footprint, for performance tests or some other kind of testing
* Throws a `java.lang.OutOfMemoryError` when the heap is full
#### Other space that is being used for the program outside the main thread heap space
1. Garbage Collection Thread
2. Other threads
3. Code Generation (JIT compilation for example to create native machine code from jvm instructions (byte code))
4. Socket Buffers
5. JNI (Java Native Interface)
### Useful parameters for the JVM (for GC Tuning)
* New generation heap space for each core: `-Xmn{size}`
* Old generation heap space for each core: `-Xmx{size}`
* Print java command line flags: `-XX:+PrintCommandLineFlags`
* Initiate string deduplication (only G1): `-XX:+UseStringDeduplication`
* Use Shenandoah (most Oracle installations don't support this): `-XX:+UseShenandoahGC`
* Use ZGC: `-XX:+UseZGC`
* Use parallel GC (for pure throughput applications): `-XX:+UseParallelGC`
* Use epsilon GC `-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC`
* Metaspace heap size for each core: `-XX:MetaspaceSize{size}`
* Disable `System.gc()` calls action: `-XX:+DisableExplicitGC`
* Use compress OOP's (available by default from approximately Java 6) `-XX:+UseCompressedOops`
* Set maximum parallel threads `-XX:ParallelGCThreads{size}`
* Set maximum pause time that the GC *tries* to correspond to `-XX:MaxGCPauseMillis{size}`
* Set threads for ZGC `XX:ConcGCThreads`