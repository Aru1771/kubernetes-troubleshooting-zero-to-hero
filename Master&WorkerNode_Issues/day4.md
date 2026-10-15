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



Lesson 3: Probes and application health
-----------------------------------------

* Imagine you're the on-call DevOps engineer supporting a Spring Boot application on Amazon EKS.

* kubectl get pods


Output:

    NAME                 READY   STATUS    RESTARTS
    empower-app-abc123   0/1     Running   0


* The Pod is Running, but READY is 0/1. Users report that the application is unavailable.
* You investigate the Pod and see:

      Readiness probe failed:
      HTTP probe failed with statuscode: 503

 * What does this tell us?

 Running: The container is running; this does not guarantee that the application is healthy.

Ready 0/1: The container is not ready to receive normal Service traffic.

# HTTP 503: The readiness endpoint returned “Service Unavailable.” The application may still be starting, may be unhealthy, or may have a dependency problem.

then check:

* Check Pod details and events, inspect application logs, and investigate why the health endpoint returns 503.

what if Probes and application health:

1. check the logs of the pod.
2. check the app is accessble inside the pod at the this if you get the responce like below.
   
         {
          "status": "DOWN",
          "components": {
            "db": {
              "status": "DOWN"
            }
          }
        }
3. Now you have evidence that the health endpoint reports the database as unhealthy. The next step is to investigate database connectivity and configuration—not to change the Kubernetes Service type.

solution: B. Investigate database availability, endpoint, port, credentials, and network connectivity while keeping the unready Pod out of normal Service traffic.
