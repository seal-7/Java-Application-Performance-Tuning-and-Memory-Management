# Java Application Performance Tuning and Memory Management

## Just in time compliation and code cache.

* Generally java code is converted into byte code understood by JVM.
* JIT Converts a block of code into native code which is understood directly by the operating system where the program is running.
* JVM automatically chooses the methods which are most frequently running and are doing CPU heavy operation and uses JIT to convert them to native code.
* VM agrument helps pring the log for type of compier used and the level of compilation.
  * -XX:+PrintCompilation 
  * -XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation
* There are 4 level of compilations, higher the number deeper the compilation.
  * 1,2,3 - C1 compiler (AKA Client Compiler)
  * 4 - C2 Compiler. (AKA Server Compiler)
  * if we have a % sign means the code is running in the most optimal way possible by jvm optimisation.
  * if we have n means code was converted to native code.
* If C2 level 4 compilation is very frequent, java moves that code into code cache.
  * -XX:+PrintCodeCache [prints the size of code cache and amount used]
  * -XX:ReservedCodeCacheSize=240m [configures the java program to use maximum of metioned size for code cache]
  * -XX:InitialCodeCacheSize=140m [configures the java program to metioned size for code cache when initialisaing.]
  * Jconsole can add an extra use of roughly 2MB of code cache when connected.
  
## Selecting the JVM

* If running 32 bit OS use 32 bit jvm, if running 64 bit jvm, you can use both 32 bit and 64 bit jvm.
* 32 bit vs 64 bit
  * 32 Bit
    * If application heap size is < 3 GB, 32 bit will be faster. As all pointers will be 32 bit and faster to manipulate
    * Max Heap Size must not exced 4GB
  * 64 Bit
    * If application is using a lot of long/double variables 64 bit JVM will be more suitable.
    * Necessary if heap size is greater then 4GB
* -XX:CICompilerCount=n is the number of threads given to the compiler for compiling.
  * You can run `java -XX:+PrintFlagsFinal | grep -i CICompilerCount` to check the number of thread configured. Default for 32 bit jvm it's 1 and for 64 bit it's 2.
* --XX:CompileThreshold=n is the threshold number of time after which jvm converts that method to native code for faster compilations.

## How memory works - the stack and the heap.

* Stack vs Heap
  * Stack
    * Every thread has it's own stack.
    * All the variables declared in the local method are added to the top of stack, when the method execution completes, this variables are popped out.
    * Stack only store premitive variables, it cannot store complex variables like Objects
  * Heap
    * Heap is common for all the threads.
    * Heap can store complex variables like Objects, Strings
    * The pointer to the objects created/stored on the heap are kept in the stack.
* There is no difference in the way public and private objects are stored on stack and heap, it just controlls the access level in classes.
  
* Java Memory Rules
    * Objects are stored on the heap
    * Variables are a reference to the object which are stored on the stack.
    * Local variables are stored on the stack.
    * premitive variables are entirely stored on stack (int, long, float etc)


## Metaspace and internal JVM Memory Management

* Static premetive variables are entirely stored on metaspace.
* Static Objects are stored in Heap and reference is store on metaspace.
* Variable in the metaspace are permanent. They are not garbage collected.
* Metaspace is accesible by any thread, hence static variables are accesible across the application.

## JVM Tuning

### String pool/ String Table and Tuning.
* String pool is stored on heap. So 
* As string are immutable, if we create two strings one = "hello", two = "hello" JVM will not create two different objects on the heap, instead both of this will have the same reference and will be a single object on the heap. This only happens with litral constant strings ("").
* JVM puts all the littral string on string poll so it can reuse them and use the above optimisation. 
* String pool is implemented with creating a list of buckets and assiging strings into relevant bucket using a hash function. There can be multiple strings in a bucket because of collisions which is okay, but ideally our string pool should be big enough to have minium string per bucket overall.
* `-XX:+PrintStringTableStatistics` flag will help list the statistics for string pool.
  * There is a bug in one of the versions of OpenJDK 11 causing Fatal error upon using this flag, but it was fixed later on.
  * Returns Number of Buckets, Number of entries, Average bucket size, Maximum bucket size and other few parameters.
  * Try to reduce the Average bucket size by increasing the number of buckets.
  * When increase the number of buckets, try to use a prime number which is double the current size.
* `-XX:StringTableSize=<PrimeNumber>` flag will help in configuring the StringTable Size.
  * Average guidance is if your average bucket size is around 35-50 or more, you should consider tuning your String pool size.
  * When setting the Size of the StringTable, we should consider the heap size configued as well. As the string table is stored on Heap.

### Heap size and Tuning.
* `-XX+UnlockDiagnosticVMOptions -XX:+PrintFlagsFinal` will give a lot of data including the max heap size and initial heap size.
* `-XX:MaxHeapSize=<1024m>` or (`-Xmx1024m`) will set the max heap size to 1024 mb.
* Application will return OutOfMemoryError when your heap size is not sufficient or you have a memory leak.
  * Imidiate fix would be to increase the heap size
  * You can reduce the heap size in non prod to reproduce the issue and debug.
* `-XX:InitialMaxHeapSize=<1g>` or (`-Xms1g`) will set the initial size of the heap when starting the application.
  * If your application is resizing the heap, this could cause performace impact. So check your heap foot print and set the initial heap size accordingly.
  * You can also consider setting the initial and max heap size same to avoid resizing.

##  Garbage Collection
* Garbage Collection eligibility
  * When the objects on the heap don't have any reference on stack or metaspace.
  * There can also be circular reference on heap where more than 1 objects are referencing to each other, but again if none of the objects have reference on stack/metaspace, they are eligible for garbage collection.
  * Objects on metaspace are never removed as they are static variables, hence static variables are not eligible for garbage collection.
* Garbage collection is supposed to be responsibility of Garbage Collector in JVM and programmers should avoid interferaing/manually triggering GC, although it is possible to manually trigger GC using System.gc();
  * System.gc() is just a suggestion to JVM it is not promissed that GC will occur.
  * When garbage collecting, it can reduce the system performance or even pause the system.
* In Java 11 there was an optimisation where after GC, JVM returns unused memeory to the operating system. This optimisation was not there in Java 8.
  * This optimisation can actually cause issue as later again asking memory from OS can cause a slight performace impact.
  * This can be controller by setting the initial heap size, so in Java 11 GC will never let the heap memory go below the initial heap size.
* Rather then figureing out which object need to be GC, GC Saves the objects that have references and sweeps all. This is called Mark and Sweep.
  * Marking cannot work properly when other thread are creating objects, hence GC has to stop the world or pause application.
  * GC also move the saved object to continues block of memory after GC. This make it easier for JVM to find and allocate future memory.
  * Hence the garbage collection works well when there is more garbage.
* Heap has two generation sections
  * Young and Old
  * Young
    * New Objects are creaetd on Young gen.
    * Most of the objects here are short lived hence they are quickly garbage collected. (AKA Minor GC)
    * We don't notice the application freezing when GC for young GC
    * Young is further split into 3 generations. Eden, s0 and s1.
    * s0 and s1 are survivor spaces, there is no priority between s0 and s1. They are used alternativly to swap surviging object.
    * New Objects are created in Eden Space, after GC all the surviing Objects are moved to s0. After next GC, the survinging Objects from eden are moved to s1 and also s0 objects are moved to s1.
      * This improves performance as marking only happens at Eden + s0 or s1.
      * Moving objects to continues block is also fast as s0 or s1 continues block is always reserved.
      * Although slight downside is, one of s0 or s1 is always empty.
    * Surviving objects are moved to Old Generation after they have survivied a threshold of young GC. This is configurable.
  * Old
    * Runs only if needed, for eg when full.
    * It's slow as most objects on Old GC are within reference hence marking takes time.
    * Area is more compared to young gen, and hence move objects to continues block of memory after GC also takes a little time.
    * Pauses the whole application, hence can cause user impact.
  * Old gc's can cause intermediate latency Spikes in your APIs as the full application is paused.
* You can use VisualVM application with Visual GC plugin to see the distribution and live usage of your heap and garbage collection across different spaces.
* You can also use `-verbose:gc` flag 

## Garbage Collection Tuning
* `-XX:NewRatio=n` this flag denotes how many time bigger does the old gen have to be compared to new gen.
  * if ratio is 2, and young gen is 10 mb. Old gen will be 20 mb.
* `-XX:SurvivorRatio=n` how much of young gen has to be taken by s0 and s1 space, and remaining will be taken by Eden space.
  * If Ration is 8, s0 and s1 will each be 1/8 of young generation.
  * JVM would resize this space if it think it can run our program in optimum way even if we have provided the ratio.
* `-XX:MaxTenuringThreshold=n` how many how many generation does the variable have to survive to be part of old generation.
  * Max value is 15

## Choosing the Garbage Collector
* Serial
  * Uses a single thread to perform GC.
  * `-XX:+UseSerialGC`
* Parallel
  * Will peform GC on Young gen in parallel using multiple threads.
  * `-XX:+UseParallelGC`
* Mostly Concurrent
  * Minimum stop the world time, pauses application when marking while performing GC. And resumes application while collecting objects.
  * We have 2 Mostly Concurrent GC's. MarkSweep GC and G1GC
    * `-XX:+UseConcMarkSweepGC`
    * `-XX:+UseG1GC`

### How does G1 GC work
* The complete heap is broken down into region, 2048 default.
* When doing a minor GC on Young gen. G1GC can decide an reallocate regions between eden s0 and s1 spaces.
* When a full GC happens on Old gen, it will first collect the regions having most garbage. That's why it G1, garbage first
  * Hence, if it only clears few region with major garbage and if it's sufficient, it won't have to do GC for all the regions.
  * Hence performance of G1GC is better as it can resize and reallocate regions and it can do partial collection in full GC.

### Tuning G1GC
* Most of the time you don't have to tune G1GC.
* `-XX:ConcGCThreads=n` number of concurrenct threads available for smaller regional collection.
  * This is generally used to reduce the GC threads in case you want other application running on same CPU to perfrom better.
* `-XX:InitiatingHeapOccupancyPercent=n` configure the heap full percent after which GC should run. Default 45%
  * GC other then G1 perform GC when the young or old gen are full. G1 does it when 45% is full.
* `-XX:UseStringDeDuplication` make more space if duplicate string on heap.
  * availble since java 8 update 20 help increase more space on heap.
  * removes duplicate references of same strings when performing GC. (More explanation above in string pool section.)
  * Only use if there are many string de duplications as enabling this flag adds extra overhead to GC.

## Using a Profiler to anaylise your application.
* Will add a little additional overhead on your application.
* 

### Memory Leaks
* Soft leak - When the object is no longer needed but we always have a reference on stack for it, hence it will never be garbage collected.


## IO vs CPU work
* Understanding the power of Non Blocking IO 
  * https://www.youtube.com/watch?v=wB9tIg209-8

## Quick Tips and Tricks
* Command to get pid of all java process - `jps`
* Print the value of flag used for a running application, `jinfo -flag <FlagName> <pid>`
* New Virtual machine are clever, they can create the entire objects on stack instead of heap as part of some rare optimisaion.
* https://chriswhocodes.com/

## Imp links and References
* Understading Non Blocking IO: https://www.youtube.com/watch?v=wB9tIg209-8
* http://isuru-perera.blogspot.com/2016/03/specifying-custom-event-settings-file.html?m=1
* https://www.baeldung.com/spring-webflux-concurrency
* https://stackoverflow.com/questions/61552078/resilience4j-bulkhead-thread-behaviour
* https://www.baeldung.com/netty
* https://dzone.com/articles/thread-local-allocation-buffers
* https://blog.gceasy.io/2020/06/02/simple-effective-g1-gc-tuning-tips/
* https://filia-aleks.medium.com/microservice-performance-battle-spring-mvc-vs-webflux-80d39fd81bf0



* Scale up redis
* Reduce pod count from 66 -> 55
* Reduce the heap size and overall memory from 6GB to 4GB
* Reduced DDS pods from 30 to 7
* Using Ambassador instead of ALB

