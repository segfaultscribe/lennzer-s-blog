---
# layout: ../../layouts/BlogLayout.astro
title: 'Scheduling in Operating systems'
date: '2025-07-25'
hero: ''
tags: ["operating systems", "scheduling"]
---

An <span style="color: #E4004B">Operating System</span> is much like a conductor in an orchestra. The OS, instead of managing musicians, orchestrates the processes in a computer system. It aims for two things: Efficiency and Control. One of the major responsibilities that the OS carries is deciding which program to run and when, a concept known as <span style="color: #E4004B">scheduling</span>. 

When you boot up your PC, launch VS Code, open GPT in the browser, and play Spotify in the background, it’s the OS that keeps all these programs running smoothly together, so you don’t feel the heat. You see all these programs running at once and that's partly true, since modern computers have multiple CPUs and can run things in parallel. But on a single-CPU system, the OS must carefully schedule all processes to share that one CPU.

So, the OS creates an elaborate illusion: it makes it seem like multiple programs are running at once, even though only one is running at any given moment. I suppose it is fairly common knowledge for anyone reading this blog about what <span style="color: #E4004B">concurrency</span> is. 

It means running small chunks of each program so quickly that it feels like everything is happening at once. That's exactly what an operating system does. 

In short, the OS runs one process for a few milliseconds, stops it, saves its state, figures out the next process to run, loads its state, and <span style="color: #E4004B">context switches</span> into it. Then it repeats this cycle over and over again.

 > A **context switch** is when the OS saves the state of one process and loads the state of another, like pausing one show and playing another from where you left off.

There are many scheduling algorithms that make this possible and understanding them forms the foundation of how operating systems work. 

NOTE: The term `jobs` and `processes` in this blog often refer to the same thing unless specified otherwise.

### Metrics

**Turn-Around-Time (TAT)** : The amount of time it takes the OS to complete a process after it has arrived.

Arrival Time(to the process queue): AT
Completion Time: CT

`Turn Around Time (TAT)` = `CT` - `AT`

**Response Time** : the time from when the job arrives in a system to the first time it is scheduled
(sometimes it might also mean from the time a job arrives to the time when it produces a 'response')

`Response Time` = `T_sheduled_first_run `- `T_arrival`

<hr>

## FCFS | Non-preemptive
First-Come, First-Served (FCFS) is about as straightforward as it gets: whoever comes first, gets served first. A process that arrives early gets run to its completion. 

You would realize very early that this isn't a very good approach. If the first process that arrives decides it'll take forever to execute... well, good luck to all other processes in the queue. 

This, in turn, increases the <span style="color: #E4004B">Turn Around Time (TAT)</span> for all other processes drastically. To solve this, we might try to run the _shortest_ job first instead of just the _first_ one.

## Shortest Job First(SJF) | Non preemptive
Exactly what it sounds like. All arriving processes are sent into a queue and the OS selects the process with the shortest total CPU burst time and runs it to completion. 

But here’s the catch.

Let’s say **Job A** shows up at time **t = 0**. The OS starts running it immediately. But if A turns out to be a long-winded process that takes _seconds_ to finish, we’ve got a problem.

Why? Because now, at **t = 1**, **t = 2**, and so on, **shorter jobs could be arriving** but they’ll have to wait. Just like in good old FCFS. So despite trying to be smarter, we’re still punishing short jobs that show up a bit late.

But what if we were allowed to interrupt jobs so shorter ones could "preempt" longer ones?
## Shortest Time to Completion First(STCF) | Preemptive
STCF is the cooler, preemptive cousin of SJF. A preemptive scheduler means that it doesn’t require the current job to finish, in order to run another one.

When a new job enters the system/queue the STCF scheduler looks at which of the remaining processes have the least amount of time required to complete and runs it i.e. if a shorter job shows up, it politely (more often not) kicks the current job off the CPU. Hence this is SJF but with preemption. Its great for cutting down on that all-important <span style="color: #E4004B">TAT</span>.

Of course the system achieves this through a <span style="color: #E4004B">Context switch</span> initiated by an interrupt.
## Round Robin | Preemptive
Round Robin runs each process for a specific <span style="color: #E4004B">scheduling quantum</span> (time slice) and after each time slice, <span style="color: #E4004B">Context switches</span> into the next process in the queue. The previous process, if not finished, is moved to the back of the queue. If the time slice is 2ms and three process enters the system: 

p1 @ t = 0 with burst time 2ms,  
p2 @ t = 1 with burst time 4ms and 
p3 @ t = 2 with burst time 3ms

- queue at t = 0ms <span style="color: #E4004B">[ p1(2ms) ]</span>
	- scheduler runs p1 for 2ms and finishes it

- queue at t = 2ms <span style="color: #E4004B">[p2(4ms), p3(3ms)]</span>
	- scheduler runs p2 for 2ms → p2 has 2ms remaining

- queue at t = 4ms <span style="color: #E4004B">[p3(3ms), p2(2ms)]</span> 
	- scheduler runs p3 for 2ms → p3 has 1ms remaining

- queue at t = 6ms <span style="color: #E4004B">[p2(2ms), p3(1ms)]</span>
	- scheduler runs p2 for 2ms and finishes it

- queue at t = 8ms <span style="color: #E4004B">[p3(1ms)]</span>
	- scheduler runs p3 for 1ms and finishes it

@ t = 9ms: All processes complete.

```
Now as you turn back and look at the algorithms you might 
note that SJF, STCF are optimized for turn around time 
while being bad at response time. Round Robin on the other 
hand performs well on response time but sucks at turn around time
```

<hr>


 The scheduling algorithms we’ve explored so far FCFS, SJF, STCF, and Round Robin are foundational _policies_ that define how the OS decides which process runs next. However, real world schedulers are more complex systems that combine these policies with mechanisms like time-slicing, dynamic priorities, and I/O heuristics to ensure fairness, responsiveness, and efficiency under real workloads.

To understand how actual operating systems manage scheduling in the wild, let’s now explore strategies like MLFQ and Linux’s Completely Fair Scheduler (CFS), which are designed for real-time constraints, dynamic workloads, and system interactivity.

#### A very important thing to note

For schedulers like SJF and STCF, they are pretty efficient systems but have you noticed that this requires the OS to know beforehand how long the process is? The OS has no way of knowing this. Of course we can estimate how long a process might run but that is not very efficient. So how are we to implement SJF and STCF without knowing how long a process might run?

## The Multi-Level Feedback Queue(MLFQ)

Before we begin with MLFQ, do note that we're entering flexible, adaptive territory. The algorithms, strategies and plans discussed here are blueprints and actual implementation can be modified for specific purposes. However they are pretty solid and not to be slept on.

A multi-level feedback queue has a set of priority queues where each queue contains processes of equal priority. The topmost queue is seen as most important and the importance deteriorates as we move down the queues. Think of 'importance' as a metric where more importance means the process will be selected **earlier** and be run for a **short** amount of time. The most priority level has the shortest time slice but chosen first and the lowest level has the highest time slice but is chosen last.

The idea behind this is that we want to implement a shortest job first type scheduler without actually knowing how long a process takes. Now let's look at how a basic MLFQ works.
 
 An MLFQ generally has
 
- `n` number of priority queues each queue having priority (top: high -> bottom: low)
- If process 1 has priority greater than process 2, then process 1 runs first
- if process 1 and process 2 have equal priority, they run in round robin fashion
- allotment time: the time for which a process is allotted a specific priority queue
	- if the process exceeds this time, its brought down to a lower priority queue

Let's look at a job that has a long execution time arriving in the system. The system will first assign it the highest priority and it will start getting executed. Once the allotment time is over, it moves down to a lower priority queue. This is repeated again for all processes in the system. 

Eventually this leads to the longer process reaching the lowermost priority queues and in a way we have achieved SJF type of scheduling without knowing how long each process takes. If it was a short process, it would still get executed and finish in the top most priority itself and long running jobs will fall down to the least priority queues.

But it also brings two concerns
1.  What about processes that have a number of I/O calls? This means the process is interactive and requires top priority.
2.  What about long running processes that are doomed because a flood of short-lived processes keep occupying the top queues?

***1*** : I/O processes that are interactive, a quick fix would be to state that if a process gives up the CPU due to I/O, then we consider it as interactive and the allotment time is refreshed for that process thereby enabling it to stay at the same priority. 

But this also brings about a **security problem**. A malicious code can be used to call I/O within allotment time to ensure one single process stays in the top priority and starves the rest of the processes in lower priority. This could be a well written code that might eventually monopolize CPU time for itself.

***2*** : This is called starvation and for a busy system, the long running processes might never get CPU if the top priority levels are always filled. This also brings into consideration an issue when long running processes after a certain time convert into I/O interactive processes which should make it top priority.

A "sort-of" solution: **`Priority Boost`**, A priority boost can be done in many ways but simply we can take all existing processes and push them periodically into the top priority queue and therefore ensure that every process has a chance to execute. This also makes sure long running processes that turn into I/O have a chance to turn into top priority processes.

> Variants of MLFQ have influenced real-world schedulers, including early BSD and Windows NT systems.

To solve the issue of malicious code that might monopolize the CPU by using the I/O merit, we can simply rule that any program irrespective of whether it makes I/O calls or not, will move down to lower priority if the allotment time is finished. Hence we do not give any more special considerations to interactive processes that frequently call I/O and yield the CPU. This ensures security of the system. 

<img src="https://github.com/The-Lennzer/images/blob/main/tech%20images/MLFQ.png?raw=true" alt="MLFQ visual">

This is a very surface level and simple explanation of an MLFQ. Feel free to explore this further or even try implementing it yourself.

## Proportional Share Schedulers

So far, we’ve looked at schedulers that decide **which single process** should run at a given moment based on estimated efficiency or priority.

But what if the goal wasn’t just to decide _who runs next_, but to make sure that, over time, **each process (or user) gets a <span style="color: #E4004B">fair share</span>** of CPU time?

> If the CPU were a cake, the pieces would be divided based on assigned weights not appetite. More weight? Bigger slice. But no one starves.

That’s the central idea behind **Proportional Share Schedulers** schedulers that treat CPU time like a divisible resource.

Enter:

- `Lottery Scheduling`
- `Stride Scheduling`

---

#### Lottery Scheduling

The most important aspect of lottery scheduling is... <span style="color: #E4004B">tickets</span> (how obvious was that?).

Tickets represent a process's share of the CPU. If two processes, say **p1** and **p2**, arrive in the system and **p1** should have more priority, we can give more tickets to **p1** and fewer to **p2**.

For example:

- `p1` → 75 tickets
- `p2` → 25 tickets

This means **p1** has a 75% chance of being chosen to run at each decision point, and **p2** has a 25% chance. Over time, this translates to **p1** receiving ~75% of CPU time, and **p2** the remaining ~25%.

This percentage is achieved _probabilistically_, not in a fixed or ordered fashion. That is, when the OS needs to choose a process to run, it picks one at random weighted by the number of tickets.

To visualize this:  

Imagine the scheduler randomly picks a number between 1 and 100. If **p1** owns ticket numbers 1–75 and **p2** owns 76–100, then **p1** has a 75% chance of being picked.  

**Simple, right?**

This randomness actually comes with some nice properties:

- Easy to implement
- Lightweight
- Fast (just a random number generation)
- Smoothly handles fairness over time
- Avoids tricky edge cases that more rigid schedulers can run into

```
NOTE: The exact 75%-25% CPU split doesn't happen immediately. Over a short time span, the actual usage may fluctuate. But over the long run, the averages tend to converge. This is the nature of probability-based systems.
```

This kind of system also introduces the idea of **ownership**. Think of it like owning shares in a company: more shares = more influence. In our case, **tickets are the shares**, and processes "buy" influence over the CPU.

This leads to two interesting mechanisms:

**Ticket Transfer** : A process can donate its tickets to another process to boost its CPU share.  
A common case: a client process temporarily transfers tickets to a server process that’s handling a request to ensure quicker turnaround and a better user experience.

**Ticket Inflation** : A process can temporarily increase its number of tickets to gain more CPU time.  
While this might help in boosting performance for time-sensitive tasks, it’s also a **potential security risk**:  

> A greedy or malicious process could continually inflate its ticket count, causing **other processes to starve**  monopolizing the CPU unfairly.

Lottery scheduling is easy to implement (assuming you’re okay with the random number generation logic), and quite elegant in systems where **fairness and flexibility** are more important than strict determinism.

#### Stride Scheduling

If you replace the randomness in lottery ticket scheduler with something more <span style="color: #E4004B">deterministic</span>, you get stride scheduling.

The processes still get tickets, but now we're concerned about strides. A stride is **inversely proportional** to the number of tickets a process has. More tickets means a smaller stride, which means more frequent access to the CPU. If that doesn't make any sense, let's look at an example.

Let's say we have three processes, **p1**, **p2** and **p3** with 200, 100 and 50 tickets each. To calculate strides, we pick a large constant (say, **10000**) and divide it by the number of tickets each process has.

| Process | Tickets | Stride (10000 / tickets) |
| ------- | ------- | ------------------------ |
| P1      | 200     | 50                       |
| P2      | 100     | 100                      |
| P3      | 50      | 200                      |

For each process, we'll now maintain a counter, all initialized to 0.

- p1_counter = 0
- p2_counter = 0
- p3_counter = 0

Now every time a process runs, we'll increment the counter by the stride value, so if `p1` runs first, we'll increase `p1_counter` by 50 and so on. 

To choose a process to run is simple, at any given point of time, if the scheduler has to choose a process to run, simply choose the process with the lowest <span style="color: #E4004B">pass value</span>. 

> Pass value is the value held by the counter for each process.

So in a dry run

|Step|Pass Values (P1, P2, P3)|Chosen Process|Reason|
|---|---|---|---|
|1|0, 0, 0|P1|Lowest pass (tie)|
|2|50, 0, 0|P2|Now lowest (0)|
|3|50, 100, 0|P3|Now lowest (0)|
|4|50, 100, 200|P1|Lowest (50)|
|5|100, 100, 200|P2|Tie with P2/P1, pick P2|
first `P1` ran, then `p2` and `p3` had lowest pass values, so `p2` was chosen, then `p3` had 0 pass value, and at step 4 you can see that all process ran at least once and now we're back to choosing the lowest pass value which is `p1` with 50. Repeat.

Lottery scheduling gets the proportions probabilistically over time, but stride gets it exactly right at the end of each scheduling cycle but before you rejoice on finding an extremely neat algorithm know that stride scheduling brings with it an overhead of keeping track of the stride as well as the pass values of each processes. 

Lottery scheduling on the other hand has got no need to maintain state, just add the new process's ticket to the total ticket and you got your `CPU%`

So while stride scheduling requires more bookkeeping than lottery scheduling, it rewards you with precise, predictable fairness which can be essential in systems where consistent resource allocation matters.

<hr>

So far, we’ve seen:

- Classic schedulers like FCFS and Round Robin that focus on **“who’s next”**
- MLFQ trying to guess process behavior and adjust dynamically
- Lottery and Stride Scheduling, introducing **proportional fairness**

Schedulers like **Lottery** and **Stride** bring us into the world of **proportional fairness** where CPU time is divided _not equally_, but based on how important or resource-hungry a process is.

This concept is often referred to as **Weighted Fair Share Scheduling** where each process is given a weight, and over time, receives CPU time proportional to that weight.

- Lottery implements this <span style="color: #E4004B">probabilistically</span> using tickets.
- Stride implements this <span style="color: #E4004B">deterministically</span> using stride values and pass counters.

And this concept is the very idea at the core of the **Completely Fair Scheduler (CFS)** used in modern Linux systems.
## Linux Completely Fair Scheduler

A highly effective and scalable design for a scheduler and honestly the full extent of this scheduler is out of the scope of this scripture[^1]. 

CFS employs a counting technique called <span style="color: #E4004B">virtual runtime (vruntime)</span> to fairly divide the CPU time across all processes. Similar to the stride mechanism, every process accumulated **vruntime** as it runs and when the scheduler needs to pick up a new process, it chooses the one with the lowest **vruntime**.  

CFS also has a few control parameters which make it efficient

**sched_latency** : This value determines how long a process must run before switching, but not in the way you think. CFS, takes in the number of processes currently in the system and divides the **sched_latency** by it to get the **time quantum** for each processes. 

```
If there are 12 processes in the system and sched_latency = 120ms, then each process would get a time slice of 120/12 = 10ms
```

But... think about it. If there are too many processes in the system, say 120 (considering the above example). Then each process would get a time slice of 1ms, which is pretty weird and inefficient. The amount of context switch overheads would hurt the performance of the system. How do we handle this?

**min_granularity** : This is a minimum value which is set to keep time slice for each process under control. Let's say we kept this value to 6ms. The even if we have 120 process in the system with a **sched_latency** of 120ms, each process will run for 6ms i.e. **min_granularity** acts as a cap for how low the time slice for each process could go. 

CFS also tries to be **fair** to the processes by applying a concept known as <span style="color: #E4004B">nice</span> level of a process. This is a parameter associated with the priority of a process and can range from -20 to +19 (default: 0), with a negative value implying more priority. The nicer a process is the less priority it has. 

Much like in the real world where a patient and kind customer is taken advantage of to treat the customer that might cause a ruckus if not served on time 😏.

CFS associates certain weights to each nice level and uses this to calculate how much time slice a process might get. 

**Choosing the next process**

In order to choose the next process to run, CFS makes uses of a data structure called red black trees, which is much more efficient than a linear data structure. It maintains low depth and most operations can be carried out in logarithmic time. 

CFS also ensures that only the current process that are ready are maintained on this structure and not idle processes that are waiting for I/O or for any other process to happen. When the number of processes increase, maybe in the range of 1000s then logarithmic operations are more noticeable.

**Idle processes**

If a process goes to sleep or goes idle for any reason for a long time, its **vruntime** might be very small compared to the other running processes. If two process A and B start running and B goes idle after a vruntime of 10, A might still be running. When B comes back, it might have a far less vruntime compared to A and can hence monopolize the CPU for itself. 

> CFS chooses the process with lowest `vruntime` to run next, so you might be wondering what about the **nice** weights? The nice level affects how quickly a process’s `vruntime` increases i.e., processes with lower priority (higher nice values) accumulate vruntime _faster_, so they run _less often_.

So how can we make sure processes that 'sleep' for a long time do not monopolize the CPU. CFS handles this by setting the vruntime of a process that wakes up to the minimum value of the vruntime in the data structure (remember that the structure holds only running or ready processes)

CFS is a really complex and we haven't even grazed a percent of its capabilities, but this is still a good introduction to get you started with the scheduler, and so for the other ones.

## Beyond 

Beyond the classic and general-purpose schedulers discussed above, there are **many other scheduling algorithms** used in specialized systems, including embedded devices, real-time applications, and experimental operating systems. 

- <span style="color: #E4004B">Rate Monotonic Scheduling (RMS) </span>: Static-priority scheduling based on how frequently a task runs. Shorter period = higher priority.
- <span style="color: #E4004B">Earliest Deadline First (EDF)</span> : Dynamic priority: the task with the nearest deadline is scheduled first. Optimal for single-CPU systems under ideal conditions.
-  <span style="color: #E4004B">Least Laxity First (LLF)</span> : Chooses the task with the smallest slack (deadline - remaining execution time).

**some others**

- Cyclic Executive Scheduler
- Fixed Priority Preemptive Scheduler
- FreeRTOS Scheduler
- Windows NT Scheduler
- BSD ULE Scheduler (used in freeBSD)

and much more...

Feel free to explore them as you like. 

But here's one you might be tempted to check out : <span style="color: #E4004B">Linux BFS</span> (Brain Fuck Scheduler)

In this scripture we've covered the basics of some general schedulers that go beyond the normal sources that somehow conveniently stop at round robin. Even though we haven't discussed these schedulers in detail, now you know they exist and are waiting for you to explore them. 

*In a system where hundreds and thousands of processes demand attention, the OS must act like a fair and efficient ruler not just making decisions, but making them fast and fair in real time. Scheduling, at its heart, is about choosing wisely under pressure and that might just be the most human thing a computer ever does.*

*👋🏼 Bye bye now, and have a wonderful rest of your life!* 



<hr>

[^1]: *Yes, we’re calling it a scripture. Because scheduling is sacred (and maybe a little confusing).*