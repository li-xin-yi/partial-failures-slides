3. Let's start from an example: In a running `Zookeeper` system, the serializer inside the leader processor gets stuck when dumping snapshot to another instance due to a transient network error. However, the serializer still holds the lock, which blocks the request processor module. Since the fail detector module is not affected, it still keeps sending heartbeats to other processes. Thus, this process appears to be working properly. From the view of an on-call engineer who maintains the `ZooKeeper` cluster, he/she finds that some services relying on `ZooKeeper` are experiencing a lot of time-outs. When he/she tries to inspect the issue, the whole service looks normally, he can still read from the server but read or update operations suffer time-outs. No obvious error can be inferred from log files.

3. The definition for partial failure is **a fault which doesn't crash process but causes safety and liveness violations, or serve slowness For some but not all functionalities.**
    - The scope of their study is specified as process level instead of service level which might be caused by crash.

4. They conducted a study of 100 real-word partial failures from 5 large-scale open-source systems. They studied 20 cases for each system. Interestingly, 54% of them occur in the most recent 3 years' software release, which suggests that partial failure has a trend occurring in part as software evolves.
   - [Apache Zookeeper](https://zookeeper.apache.org/): A service for distributed systems offering a hierarchical key-value store, which is used to provide a distributed configuration service synchronization service, and naming registry for large distributed systems.
   - [Apache Cassandra](https://cassandra.apache.org/): A free and open-source, distributed, wide column store, NoSQL database management system.
   - [Hadoop Distributed File System(HDFS)](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html): A distributed file-system that stores data on commodity machines
   - [Apache Mesos](http://mesos.apache.org/) An open-source project to manage computer clusters (abstract)

6. The 1st finding is partial failure do not have a uniform or dominating root cause. The tops three root causes which occupy 48% are uncaught errors (like runtime exceptional code anywhere), indefinite blocking (this occurs when some functions get stuck), and buggy handling (e.g. empty handlers)

7. For consequences, they found in 48% of study cases, some system features get stuck (here, stuck means some part of the program was not making any progress but the process was still not completely unresponsive), and 17% of partial failures caused a certain operation to take a long time to complete. The slow in the graph means the performance is severely degraded and the service is barely usable.

8. They found that most studied cases would explicitly violate liveness requirements or trigger errors. But some partial failure are completely silent. For example, if the server uses the unclean buffer to handle a new session and return a random response. 
   - Such partial failures are usually hard to detect without detailed correcting specifications. About 71% of partial failures are triggered by some specific environment conditions or special input in the production.   
  - They also found diagnosing partial failures is always a painful experience for developers. One reason is mysterious symptoms mislead the troubleshooting direction and the system does not expose enough runtime information. So developers have to enable debug log, analyze heap or instrument codes.

9. The study shows that partial failures are common and severe so how to deal with them? From the study, partial failure are over exposed with only the unique production workloads so it is hard to eliminate them in static analysis approaches. But the current dynamic detectors, they are too shallow.
    - How about asking developers to manually write more defensive checks? But it is unrealistic to ask developers to check for each operation in the system
    - So what the developers want is to systematically generate such checkers to erase their burden. 
    - An automate solution to cover all cases is different because partial failures have diverged through the causes.
    - But most of partial failures do not rely on deep semantic understanding to detect, such checkers can potentially be automatically constructed.
    - So how to generate an effective checker?
  
10. Existing checker is ineffective because it is very disjoint with the main program. An effective checker should be intersect with the execution of a monitored module. 
11. They proposed an intrinsic software watchdog abstraction. 
    - Their watchdog abstraction has three main characteristics
13. First, each checker should be customized to inspect a certain system module for issues specific to that module. Previously, others are some software like Linux watchdog or `http` watchdogs but there are very simple and generic. Their checkers' logic should be tailored to monitor module.
14. Second, to accurately reflect the monitored execution, The checkers should be stateful, which means the checkers need to use the lastest program state since checking. In this design, it is called context.
15. Third, checkers run concurrently with the main program. Essentially they decouple runtime with the main execution. In this way, watchdogs can provide isolation for both performance and safety.
16. The way they build tailored checker is mimic checking. The mimic checker performed a similar operation and a check at the target. It has a good accuracy because it shares the same environment with main execution and it can pinpoint 40 operations. For the previous example, the mimic checker will perform a similar snapshot as what the serializer did, and it also got stuck and the last watchdog can oberserve the liveness issue.
17. They designed a tool `OmegaGen` who automates the checker construction. For a given system,  `OmegaGen` will analyze the source code, generate customized checkers, instrument the main execution and finally package the generated checkers and driver back to the original software. The core technique used to generate mimic checkers is called program reduction.
18. The goal of program reduction is to produce a reduced version of a main program even with that it still allows a reduced version to expose the failures that the original main program may have. So why they want to do a reduction here? Because they should not everything into the checker because it is desired to narrow down the cause and some functionalities' correctness are logically deterministic (e.g. sort) that can be tested before running. But some operations could be more vulnerable in run-time.
19. The whole work-flow takes 5 steps, we will see them in detail in the following slides.
20. First, the system could have very large code regions but we only care about the part that may be executed continuously. 
22. And for each such function, we are interested in maintaining the operations that are worthy of monitoring. It will recursively analyze each function and look for vulnerable operations.
23. Their current criteria for in selecting such vulnerable operations are based on heuristics which by default includes
    - synchronization
    - resource allocation
    - vent polling
    - async waiting
    - invocations using external arguments
    - file or network 
    - complex while loop conditional
    - it also allows developers to customize this criteria
    - Once found it will mark on the vulnerable operations
24. And then construct a checker by extracting all vulnerable operations in a function and remove similar operations and add safety and liveness checkers
25. Checkers at this point cannot be directly executed because there are uninitialized Parameters or variables. So it inserts the context hooks in the main program to synchronize states.
27. Errors reported by generated checker could be transient or horrible. By default, it just simply re-execute the checker and compare for transient errors. It also allows developers write their own user-defined validation tasks to check some entry functions and automate the part of deciding which validation task to invoke depending on which checker failed.
28. One danger from watchdog checkers is they can accidentally modify the main program state. `OmegaGen` will analyze all the context reference in the checkers and replicate the context for checking usage. So any modification will only affect the replicated context and the original main program state is safe.
29. To prevent blind replicating imposing too much overhead, it will only replicate mutable part of the context based on the immutability analysis and it will lazily replicate the context only when the checker is invoked.
30. But lazy replication brings one issue that the context might be different between the time the operation invoked and the context replicated. For this issue it will calculate hash code for context object and skip checking if the hash codes do not match.
31. They also addressed I/O side effect. The main idea is using IO redirection and idempotent wrappers
32. They applied the tool to 6 popular distributed systems and successfully generated hundreds of watchdogs for all 6 systems. Note that there are not all watchdogs will be activated during run-time.
33. To evaluate the effectiveness of generated watchdogs, they reproduced 22 real-world partial failures.
35. To compare, they implement 4 types of advanced detectors as baseline checkers.
    - The client checker is based on the observers in state-of-the-art work.
    - The probe checker represents `Falcon`, which periodically invokes some APIs, may miss many failure.
    - Signal and Resource are industry practice
    - Signal checker monitor some health indicator, susceptible to environment noise and usually have poor accuracy
36. Overall, the watchdogs detected 20 of 22 cases with a median detection time of 4.2s
- In general, watchdog is effective for liveness issues like deadlock, indefinite blocking, and safe issues but inefficient for silent correctness issues.
- For baseline checkers, even the combination of all checkers they can only detect 14 of 22. Notice there are some negative detect time because the checker testing the operation before the main program and observed the failures.
37. There are different localization levels:
    - watchdogs can pinpoint 14 fault instruction in 22 cases, while others always can only pinpoint fault processor.
38. They further evaluated the false alarm of the watchdogs and baseline detectors on different set-ups. Watchdog almost did not report false alarm in a stable set-up but during a loaded scenario they incur around 1% due to the socket connection errors or result contention. The false alarms can be reduced by the very detailed mechanism. Signal in general can detect most failures among baselines but it also has a high false alarm rate.
39. On system throughput, watchdogs impose a 5-6.6% overhead and the main overhead comes from the hooks rather than the concurrent execution. For memory usage, watchdogs are found not to incur a significant memory overhead because they only lazily replicate the objects upon checking. Checking frequence is much smaller compared to main programs execution.
40. Interesting, the generated watchdogs exposed a new bug in the lastest `zookeeper` during their fault injection testing. The author reported the bug to the community and it is confirmed and fixed. 
   


