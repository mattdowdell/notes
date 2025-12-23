# Lessons of 2025

<!-- 100 chars ------------------------------------------------------------------------------------>

## Lesson 1: Trade-offs

I'll start with the most relevant lesson of the year, not because it was the most impactful, but
because it lays the groundwork for many of the others. It is perhaps one of the universal truths of
system design.

__There are always trade-offs, and they will eventually cost you something.__

The trade-off in this case is not one I made, but one I devoted much time trying to minimise. This
is perhaps a sub-lesson: what is bad today, does not need to be as bad tomorrow. But for now, lets
focus on the main lesson.

I joined Akamai in August 2024 to work on their next-generation ObjectStorage offering. At the time,
it was in a Beta testing phase, but later reached General Availability (GA) in January 2025. As
befits a new product that was built relatively quickly, were a good number of trade-offs. But the
one that I've orbited the most this year was performance at the cost of availability.

Our entire architecture was predicated upon sharding. We took a number of [Ceph] clusters, and
sharded buckets and objects across them. This solved the problems of the initial ObjectStorage
offering because we could scale horizontally capacity by simply adding more Ceph clusters. From
there we added a number of microservices running on Kubernetes to create the illusion of a single
S3 endpoint. Each bucket existed on multiple Ceph clusters, but each object and all its versions
existed on just 1. Deciding where to place each object was the job of the object placement
microservice.

The placement algorithm is entirely random, with the result written to a key-value database. This
random quality meant that 2 concurrent placement requests could result in 2 different answers. The
easy answer here is to leverage database transactions, which were supported. However, using
transactions for all writes was unacceptably slow. Especially as concurrent writes are relatively
rare. Instead, we moved the isolation into the microservice itself. This took the form of a hashmap
of mutexes, where each placement would lock the mutex for a particular object key, make a decision,
write the result, and release the lock.

Microservices scale horizontally, adding more replicas to handle more requests, and object placement
was no exception. But the in memory locks create a stateful workload, which is a little tricker to
scale. To ensure that locks were never duplicated, we sharded object keys across 8 replicas. Placing
any object went to 1 of the 8 replicas. Herein comes the availability cost: if 1 replica goes
down we lose an 8th of our availability, or 12.5%. To make the sharding logic a bit simpler, we used
Kubernetes StatefulSets, where each Pod in the StatefulSet owned 1 of the shards.

For those that have never used StatefulSets, updating them consists of deleting the old Pod and
recreating it with the updates. Each Pod in the StatefulSet is deleted and recreated separately and
in order. We made changes almost every week, and so each week would bring unavailability. For a
time, our customers didn't notice. It was a new product, and while Akamai has many customers that
expect stability and availability, not many of them had moved over. However, the new ObjectStorage
was far more popular and so eventually it became one of the highest profile problems, if not the
highest.

This design decision, which was entirely reasonably given the time constraints and performance
requirements, was fast becoming a dealbreaker. Escalations started being raised by the Support team,
incidents were declared. Each time, we had to explain this was unfortunate, but expected behaviour.
Fortunately, the impact of the trade-off was recognised long before GA, and someone was assigned to
lessening its impact. Unfortunately, that solution never materialised.

Blame could be assigned to that person, but truthfully it was more complicated than that. The lack
of delivery persisted over many months. The SRE team would ask for updates and be told it was merely
weeks away. The QA team later told me that they never got to the point where they could effectively
test what code had been written. I think the reality is that giving one person responsibility does
not always produce consistent results. People are different, with different skillsets, motivations
and experience. In this case, it was almost certainly the wrong person to depend upon. But choosing
to create that dependency, and letting it persist for so long without results is a failure of
management. Eventually, the responsibility was moved to someone else, which is where I enter the
story.

I won't claim that the technical and human issues above were totally avoidable, mostly because I
don't think I would have done any better with what I knew at the start of the year. Making mistakes
is part of journey. But on reflection, it turns out trade-offs exist in both technical and human
domains. And they may just cost you the same thing.

[Ceph]: https://ceph.io/en/

## Lesson 2: Sunk Cost Fallacy

Taking over a failing unreleased project is an experience. Starting on a codebase that's running in
production and being used by customers is easy by comparison. With enough telemetry, you can see
what's not working and work towards fixing it. Nonetheless, first task I was given was getting it up
to the standards of our other microservices. In hindsight, this was a misstep, and leads nicely into
the second lesson.

__Recognise when starting over is better than doubling down.__

To be clear, I knew what sunk-cost fallacy is. I browse Reddit and Hacker News and read the sage
wisdom of those with more experience. But I clearly didn't know how to put that into practice.

Unsurprisingly, bringing anything up to a standard is far easier when that standard is documented.
In this case, it was not, so I set about creating one. I was fortunate to have seen good standards
for building an operating microservices in a previous job. After drawing from that and adding in
some tweaks to better suit my current team, I had something to work from. There were 2 minor
learnings for me here:

- Leaving a job means leaving behind their documentation. I can remember the broad strokes, but not
  the details. Rebuilding it from scratch was much harder than I expected.
- Code itself makes very little difference when measuring one microservice against another.



<!-- 100 chars ------------------------------------------------------------------------------------>

Bringing something up to a standard is easier if that standard is documented. Perhaps
unsurprisingly, it did not. So I set about making one. In doing so, I learned something that still
fascinates me months later: the implementation is entirely subjective. Instead, I found myself
caring about qualities like serviceability, observability, security, availability. I ended up with a
non-functional checklist, where each item could be easily validated. These validations were almost
entirely manual on the basis that anything you could automate would be expressed through the config
file of some tool instead.

Herein lies another lesson: keep hold of the good ideas you've seen in the past. For the avoidance
of doubt, this doesn't include anything proprietary. In a previous job, I was fortunate to have come
across the concept of a Maturity Model. This one was targeted at deploying microservices to customer
datacenters where you might not have easy access. It was therefore a guide for how to create a
microservice that could survive with negligible human interaction. A quick text to a former
colleague and friend got me what I needed and with a bit of tweaking I had my own checklist.




<!-- 100 chars ------------------------------------------------------------------------------------>
