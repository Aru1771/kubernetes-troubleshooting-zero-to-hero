OOM KILL with Code:137 due to JVM Heap Memory is equal or Greater Than Pod resource memory Limit:
===================================================================================================

JVM Heap is the part of RAM that the Java Virtual Machine (JVM) uses to store objects created by Java applications while they run.
Think of it as the application's working memory for Java objects.

1. Simple example

Suppose Jenkins is running on your server. Jenkins is a Java application.

When Jenkins creates objects to manage jobs, builds, plugins, and pipeline execution, those objects generally use JVM heap memory.

Server RAM — 16 GB

Jenkins JVM process

JVM Heap
Java objects, job data, and application objects

The JVM also uses memory outside the heap, such as thread stacks and class metadata.


2. What happens when heap memory is insufficient?
For example, suppose Jenkins has a maximum heap of 2 GB.
- Heap has enough space: Jenkins can create objects and process jobs normally.
- Heap is filling up: The JVM's garbage collector tries to reclaim memory from objects that are no longer needed.
- Heap is exhausted: If the JVM cannot free enough memory, it may throw OutOfMemoryError: Java heap space. Jenkins may become slow or fail.
3. How to check JVM heap for Jenkins
If Jenkins is running on Linux, find its process:
ps -ef | grep jenkins


To inspect JVM heap settings, if supported by your Java installation:
jcmd <PID> VM.flags


Replace <PID> with the Jenkins Java process ID. Look for -Xms and -Xmx:
- -Xms = initial heap size.
- -Xmx = maximum heap size.
For example:
-Xms512m -Xmx2g


This configures an initial heap of 512 MB and a maximum heap of 2 GB.
Remember: JVM heap is not all the RAM used by Java. It is specifically the memory area used for Java objects, managed by the garbage collector.

 
