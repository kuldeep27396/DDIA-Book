Three Counter-Intuitive Truths About Building Software Systems That Last

Introduction

As software engineers, we build applications on the shoulders of giants. We assemble them from a suite of standard, powerful components: databases to store data, caches to speed up reads, and search indexes to make information discoverable. Most of us wouldn't dream of writing a new data storage engine from scratch—the existing tools are excellent.

But the moment you connect that cache to your database, you’ve made a choice. You've stopped being a pure application developer and have become a data system designer, often without realizing it. This new role is critical, and it comes with a host of complex challenges. How do you ensure the system remains correct and performs well, even when individual parts fail? How do you scale to handle a sudden increase in load? How do you build something that another engineer can understand and maintain a year from now?

The answers are rarely obvious. To build systems that are reliable, scalable, and maintainable requires a deeper understanding of the forces at play. This article shares some of the most surprising and impactful principles for designing the data-intensive systems of today.

To Build a Reliable System, You Must Try to Break It

Conventional wisdom tells us that reliability is about prevention. We think of it as building a fortress, meticulously trying to prevent every possible fault. While important, this is an incomplete and ultimately fragile strategy for system availability.

The more robust, counter-intuitive approach is fault tolerance: designing systems that can cope with faults rather than assuming they can all be prevented. It's crucial to distinguish between a fault—one component of the system deviating from its spec—and a failure, which is when the system as a whole stops providing its service to the user. The goal of a fault-tolerant system is to prevent faults from causing failures.

Here is the surprising truth: deliberately increasing the rate of faults within your system can actually make it more reliable over time.

This works because it forces the system's fault-tolerance machinery to be constantly exercised and tested in a controlled environment. Many critical bugs are not in the primary logic but in the error-handling code paths that rarely execute under normal conditions. By deliberately inducing faults, you uncover weaknesses in that logic before a real, uncontrolled event triggers a catastrophic failure. The most famous example is the Netflix Chaos Monkey, a tool that intentionally and randomly terminates instances in a production environment to test the system's resilience.

Counterintuitively, in such fault-tolerant systems, it can make sense to increase the rate of faults by triggering them deliberately—for example, by randomly killing individual processes without warning. Many critical bugs are actually due to poor error handling; by deliberately inducing faults, you ensure that the fault-tolerance machinery is continually exercised and tested...

Your 'Average' User Doesn't Exist—So Stop Measuring for Them

As accidental system designers, our first instinct is to measure system performance using the "average response time." This is a trap. While the arithmetic mean is easy to calculate, it provides a dangerously poor picture of the actual user experience because it doesn't tell you how many users actually experienced that specific delay.

A seasoned designer thinks about performance using percentiles. By sorting response times from fastest to slowest, you can identify key thresholds that reveal the true user experience:

* Median (p50): The halfway point. If your median is 200ms, it means half your requests return in less than 200ms, and half take longer than that. This is a far better indicator of a typical experience than the mean.
* 99th percentile (p99): The point at which 99% of requests have finished. The remaining 1% of requests take longer than this value. These high percentiles represent your tail latencies.

Focusing on tail latencies is critical because the users experiencing the slowest requests are often your most valuable customers. Amazon's internal services, for example, use the 99.9th percentile (p999) for their service level objectives. Why? Because "the customers with the slowest requests are often those who have the most data on their accounts because they have made many purchases—that is, they’re the most valuable customers." It’s a business imperative to keep them happy; Amazon has also observed that a mere 100ms increase in response time can reduce sales by 1%.

This problem is compounded in modern microservice architectures due to tail latency amplification. If a user's request requires calls to several backend services, it only takes one of those calls to be slow for the entire user-facing request to be slow. As dependencies increase, the probability that a user will hit a slow path rises dramatically. The average doesn't matter; the tail is what the user feels.

The Biggest Threat to Your System Isn't a Failing Hard Drive. It's Your Team.

When we imagine system outages, our minds often jump to hardware failures: a crashed disk, faulty RAM, or a network switch giving up. While hardware faults happen, they are not the primary cause of system failure.

The surprising truth comes from a study of large internet services, which found that configuration errors by operators were the leading cause of outages. Hardware faults were a factor in only 10-25% of those disruptions. This happens because humans, even with the best intentions, are unreliable. The challenge, then, is to design systems that minimize both the opportunity for human error and the impact when an error inevitably occurs. The best systems combine several approaches to mitigate this risk:

* Demand and build well-designed abstractions and APIs that guide engineers to the right solution and make the wrong one difficult.
* Provide fully-featured non-production sandbox environments where people can explore and experiment safely, using real data, without affecting real users.
* Prioritize quick and easy recovery. Fast rollbacks of configuration changes are more valuable than the hope of flawless deployments.
* Set up detailed and clear monitoring (telemetry). Clear performance metrics and error rates act as early warning signals, helping to diagnose issues and validate that the system is operating correctly.

Conclusion

You became a system designer the moment you connected a database to a cache. The question is no longer if you are a designer, but how good of one you will be. Building systems that last requires moving beyond surface-level assumptions and challenging the conventional wisdom we often take for granted.

By embracing failure to build resilience, measuring what truly matters to the user experience, and designing for the reality of human fallibility, we engage in a more thoughtful and durable form of engineering. These principles are not quick fixes, but a fundamental shift in how we approach our work.

So, which "obvious" assumption about your own system might be worth challenging this week?
