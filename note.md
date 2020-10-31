3. Let's start from an example: In a running `Zookeeper` system, the serializer inside the leader processor gets stuck when dumping snapshot to another instance due to a transient network error. However, the serializer still holds the lock, which blocks the request processor module. Since the fail detector module is not affected, it still keeps sending heartbeats to other processes. Thus, this process appears to be working properly. From the view of an on-call engineer who maintains the `ZooKeeper` cluster, he/she finds that some services relying on `ZooKeeper` are experiencing a lot of time-outs. When he/she tries to inspect the issue, the whole service looks normally, he can still read from the server but read or update operations suffer time-outs. No obvious error can be inferred from log files.

3. The definition for partial failure is **a fault which doesn't crash process but causes safety and liveness violations, or serve slowness For some but not all functionalities.**
    - The scope of their study is specified as process level instead of service level which might be caused by crash.

4. They conducted a study of 100 real-word partial failures from 5 large-scale open-source systems. They studied 20 cases for each system. Interestingly, 54% of them occur in the most recent 3 years' software release, which suggests that partial failure has a trend occurring in part as software evolves.

5. The 1st finding is partial failure do not have a uniform or dominating root cause. The tops three root causes which occupy 48% are uncaught errors (like runtime exceptional code anywhere), indefinite blocking (this occurs when some functions get stuck), and buggy handling (e.g. empty handlers)

6. For consequences, they found in 48% of study cases, some system features get stuck (here, stuck means some part of the program was not making any progress but the process was still not completely unresponsive), and 17% of partial failures caused certain operation to take a long time to co

   


