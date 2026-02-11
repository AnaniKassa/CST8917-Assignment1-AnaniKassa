## Header: 
### Anani Kassa, 041140713, CST8917, Serverless Computing - Critical Analysis, 2026-02-10


---

## Part 1: Paper Summary (≈500 words)

In their paper *Serverless Computing: One Step Forward, Two Steps Back*, Hellerstein et al. (2019) take a hard look at early serverless platforms, especially Functions-as-a-Service (FaaS), and basically argue that while these platforms make some things easier, they actually make it harder to build data-heavy and distributed applications.

> The main point of the paper is pretty straightforward: today's serverless platforms are kind of a mixed bag.

On one hand, they make cloud computing easier by automatically scaling your app and only charging you for what you actually use. But on the other hand, they create limitations that actually set us back in terms of what we can build.

When the authors talk about "one step forward, two steps back," the forward step is the automatic scaling and not having to worry about servers. The two steps back are all the problems these platforms create for modern apps, especially ones that need to work with lots of data or run across multiple machines.

The authors point out several big problems with first-generation FaaS platforms.

### Execution Time Limits

First, there are strict time limits on how long your code can run. For example, AWS Lambda cuts you off after 15 minutes, which means if you need to do something that takes longer, you have to chop it up into smaller pieces and manage all that complexity yourself. This makes serverless basically unusable for things like machine learning training, processing streams of data, or any algorithm that needs to run for a while.

### Function Communication Limitations

Second, there are major issues with how functions talk to each other. Functions can't communicate directly over the network—they're not even addressable, and they can't keep connections open. Instead, if one function needs to share data with another, it has to go through storage systems like object stores or databases, which are way slower than just sending messages directly. The authors show that this creates huge bottlenecks and adds a ton of latency, making it really impractical to coordinate fine-grained distributed tasks.

### The "Data Shipping" Anti-Pattern

Third, the paper talks about what they call the "data shipping" anti-pattern. Basically, in current serverless setups, data keeps getting moved back and forth between storage and short-lived compute instances, instead of just running the computation where the data already lives. This wastes money on data transfer, hurts performance, and doesn't make good use of the cloud's memory hierarchy.

### Lack of Specialized Hardware

Another big limitation is that you can't access specialized hardware. FaaS platforms only give you basic CPUs and limited memory—no GPUs, TPUs, or other accelerators. Since machine learning and data processing increasingly depend on specialized hardware, this is a major blocker for innovation.

### Difficulty Building Distributed and Stateful Applications

Finally, the authors emphasize how hard it is to build distributed and stateful applications. Because functions are stateless and have to coordinate through slow storage systems, it's extremely difficult to implement common distributed systems patterns like leader election, consensus, or shared mutable state.

To fix these problems, the authors suggest several directions for the future of cloud programming. They argue for platforms that support long-running, stateful services that can still autoscale, allow efficient direct communication between compute units, and let you run code close to where your data lives. They also want access to specialized hardware and better programming abstractions that go beyond just chaining simple functions together. The goal is to actually unlock what the cloud can do instead of limiting it to narrow use cases.

---

## Part 2: Azure Durable Functions Deep Dive

### Orchestration Model (≈120 words)

Azure Durable Functions build on top of regular FaaS by adding an orchestration model that includes client functions, orchestrator functions, and activity functions (Microsoft, n.d.-a). The client function kicks off the workflow, the orchestrator defines the control flow, and activity functions do the actual work. Unlike basic FaaS where each function is independent and short-lived, the orchestrator coordinates everything across multiple steps in a reliable way. This directly responds to the paper's criticism that FaaS platforms don't support structured distributed workflows. That said, while the orchestrator makes coordination better, it still works at a logical level rather than enabling fast, peer-to-peer communication between compute instances, which is one of the fundamental problems the paper identifies.

### State Management (≈120 words)

Durable Functions handle state through event sourcing, checkpointing, and deterministic replay (Microsoft, n.d.-b). The orchestrator's state gets automatically saved to Azure Storage, so workflows can pick up where they left off after failures without you having to manage databases yourself. This partially addresses the paper's critique that serverless functions are stateless by design. However, the state is still stored externally rather than kept in memory close to where the computation happens. So while Durable Functions improve reliability and make things easier for developers, they don't get rid of the storage-centric coordination model that Hellerstein et al. (2019) criticized. The fundamental assumption that "state lives in storage" is still there.

### Execution Timeouts (≈110 words)

Durable orchestrator functions get around standard Azure Function time limits by pausing and resuming through checkpoints (Microsoft, n.d.-c). This lets workflows run for hours or even days, which addresses one of the paper's major complaints about short execution lifetimes. But activity functions still have normal timeout limits. This means long-running computations still need to be broken into smaller chunks, which reinforces the authors' concern that serverless platforms aren't designed for sustained, compute-heavy workloads. Durable Functions help with this issue but don't completely solve it.

### Communication Between Functions (≈120 words)

In Durable Functions, communication happens indirectly through the orchestrator, which schedules activity functions and records events in Azure Storage (Microsoft, n.d.-d). This saves developers from having to manually set up queues or work with blob storage, but it doesn't enable direct network communication between functions. So while Durable Functions are easier to use, they don't fundamentally solve the paper's criticism that FaaS relies on slow storage intermediaries instead of direct messaging. The underlying architecture still prioritizes reliability over raw performance.

### Parallel Execution (Fan-Out/Fan-In) (≈120 words)

Durable Functions support fan-out/fan-in patterns, which let orchestrators run multiple activity functions at the same time and wait for all of them to finish (Microsoft, n.d.-e). This pattern improves scalability for embarrassingly parallel tasks and partially addresses concerns about distributed execution. However, as the paper argues, parallelism without efficient communication isn't enough for many distributed systems. Fan-out/fan-in works great for independent tasks but doesn't support tightly coordinated workloads that need frequent low-latency communication between workers.

---

## Part 3: Critical Evaluation (≈500 words)

Despite their improvements, Azure Durable Functions still don't solve several of the core problems raised in the paper.

### Data Locality and Communication

First, efficient data locality and communication remain mostly unaddressed. Durable Functions still depend on Azure Storage for persisting state and coordinating between functions, which reinforces the "data shipping" anti-pattern that Hellerstein et al. (2019) identified. While orchestrators make workflow logic simpler, they don't move computation closer to data or enable direct communication between compute units. This limitation sticks around because Azure prioritizes fault tolerance and scalability over raw performance, making storage-mediated coordination a deliberate design choice rather than an oversight.

### Limited Hardware Access

Second, limited hardware access isn't fixed. Durable Functions don't give you access to GPUs, large amounts of shared memory, or custom accelerators. As the paper emphasizes, modern innovation in machine learning and data systems increasingly depends on specialized hardware. Durable Functions inherit the same hardware abstraction as standard Azure Functions, which means they're not suitable for performance-critical workloads that need accelerators. This limitation persists because serverless platforms aim to provide a uniform execution environment, even if that means sacrificing expressiveness.

Overall, Azure Durable Functions represent incremental progress rather than a fundamental shift toward the future that the authors envision. They successfully work around some operational constraints—like execution time limits and manual state handling—but they do this by adding orchestration logic on top of an unchanged serverless foundation. The fundamental architectural issues identified in the paper, including storage-centric communication, lack of data locality, and absence of hardware specialization, all remain.

Therefore, Durable Functions align more with practical engineering trade-offs than with the transformative cloud programming model that Hellerstein et al. (2019) proposed. They make serverless more usable for workflows, but they don't unlock the cloud's full potential for data-rich, distributed innovation.

---

## References

Hellerstein, J. M., Faleiro, J., Gonzalez, J. E., Schleier-Smith, J., Sreekanti, V., Tumanov, A., & Wu, C. (2019). Serverless computing: One step forward, two steps back. In CIDR 2019, 9th Biennial Conference on Innovative Data Systems Research. https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf

Microsoft. (n.d.-a). Durable Functions overview. Azure Documentation. Retrieved from https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview

Microsoft. (n.d.-b). Orchestrations in Durable Functions. Azure Documentation. Retrieved from https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-orchestrations

Microsoft. (n.d.-c). Checkpointing and replay in Durable Functions. Azure Documentation. Retrieved from https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-checkpointing-and-replay

Microsoft. (n.d.-d). Task hubs in Durable Functions. Azure Documentation. Retrieved from https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-task-hubs

Microsoft. (n.d.-e). Fan-out/fan-in scenario in Durable Functions. Azure Documentation. Retrieved from https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-cloud-backup
