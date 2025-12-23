# Lessons of 2025

<!-- 100 chars ------------------------------------------------------------------------------------>

## Lesson 1: Trade-offs are unavoidable

I'll start with the most relevant lesson of the year, not because it was the most impactful, but
because it lays the groundwork for many of the others. It is perhaps one of the universal truths of
system design.

__There are always trade-offs, and they will eventually cost you something.__

The trade-off in this case is not one I made, but one I devoted much time trying to minimise. This
is perhaps a sub-lesson: what is bad today, does not need to be as bad tomorrow. But for now, lets
focus on the main lesson.

I joined my team in August 2024 to work on their next-generation ObjectStorage offering. At the
time, it was in a Beta testing phase, but later reached General Availability (GA) in January 2025.
As befits a new product that was built relatively quickly, were a good number of trade-offs. But the
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
to create that dependency, without any redundancy, and letting it persist for so long without
results is a worse failure in my mind. Eventually, the responsibility was moved to someone else,
which is where I enter the story. You could argue that moving responsibility to another single point
of failure is recreating the same failure conditions. I'd more than likely agree.

I won't claim that the technical and human issues above were totally avoidable, mostly because I
don't think I would have done any better with what I knew at the start of the year. Making mistakes
is part of journey. But on reflection, it turns out trade-offs exist in both technical and human
domains. And they may just cost you the same thing.

[Ceph]: https://ceph.io/en/

## Lesson 2: Sunk Cost Fallacy

I've worked on codebases with problems. I've worked on codebases that have yet to see a release. But
taking over a unreleased project where no one can tell you if it even works is a novel experience
for me. Initially, there was a perception that the project could be made good. It just needed some
polish and we could ship it. This turned out to be incorrect.

__Recognise when starting over is better than continuing on.__

I learned about sunk-cost fallacy a while ago, likely from a post on Hacker News or Reddit. It felt
deceptively simple to recognise. But I'd never really experienced it first-hand.

When I started working on this project, the first task I was assigned was to evaluate what has been
built already. Figure out if it was salvageable, bring it up to the standards set by existing
microservices if so. I didn't subscribe to the idea that anything is truly irredeemable. If code can
do anything, surely it can be made better. If my recommendation was to throw away what was already
built, I felt I needed a strong, objective rationale. So I created an objective tool for evaluating
it, or indeed any microservice.

In a previous job, I'd been introduced to a "maturity model". It was a simple checklist, full of
flexible yet unambiguous requirements, grouped into levels. The higher the level, the more mature
the service was, and the less human intervention was required to support it. This is the tool I
replicated for my evaluation, with a few changes to match the environment I currently found myself
in. Using the tool, I built myself a backlog of tasks, mostly around improving serviceability,
making sure we had enough information to debug in production.

As I dug through the code, I found the real problem: I couldn't tell if it satisfied the
requirements. There was a design for the service. But it lacked not only a good description of the
problem, but also the depth to show it solved whatever problem it wanted to solve. While the design
was reviewed, there was no recording of that discussion. There was a recorded demo of the service in
action, which cast yet more doubt on the correctness of the solution.

Eventually, I concluded that starting over from scratch was the only option. I went back to gathering
requirements, defining the scope, documenting the various implementation options. All work that took
time, but meant that I could rationalise the decisions made. Some parts of the original
implementation remained, there were some good ideas within. Others got replaced, simplified, or
refined. And after all that, we started building again.

I suppose the main takeaway for me is figuring out what to care about. Code structure is an
implementation detail. Serviceability issues are fixable. But building on the wrong requirements was
only fixed by starting the development process again.

## Lesson 3: It ain't what you know for sure that gets you in trouble

When you design a system, you naturally build in rules. For example, action A must happen before
action B. So if A fails, B is never attempted. These rules help you reason about the system's
behaviour and how it can fail. So when something supposedly impossible happens on a Monday, it's
gonna be a bad week.

This particular week started with an escalation. I work on ObjectStorage, and much of the system is
built upon sharding. We have multiple [Ceph] clusters, with buckets existing in multiple clusters,
and objects existing in just 1. We have a database that tracks where an object may exist. I use
"may" here because of our rules. We write the location to the database before we write the object to
the cluster. So if the latter step fails, we have an orphaned database record. Similarly, if the
object gets deleted from the bucket, the database record is orphaned again. In both cases this is
acceptable, and we have processes to periodically clean up those orphans.

The escalation had come through a couple of layers of the Support team. The customer reported that
they could list and object, but could not download it. One of these Support engineers had noticed
the cause for this: the object was not where it was meant to be. Listing objects went directly to
each the Ceph clusters, listing the objects and aggregating the results. But downloading the object
hit the database first, then went to the wrong Ceph cluster and got an error back.

Customers tend to react badly to their data being unavailable, so a number of us gathered to
brainstorm our next steps. There was much debate about the cause of the failure. There were
thankfully few components involved, but we questioned everything about them. Anything that was
remotely possible was scrutinised. Obscure knowledge was dusted off and considered. At this point,
we didn't know the scale of the problem. Did it affect 1 customer or 100? Did it affect all
production regions, or just 1? 

The issue turned out to be in our code. We use [TiKV] to store this data, but do so without
transactions because they were too slow for our performance requirements. As such, 2 writes could
conflict with one another. We thought this risk to be solved because we used mutexes to prevent
concurrent writes. Under the hood, TiKV uses gRPC streams, keeping connections open like a regular
SQL database. This meant that whilst our database client might time out, the database server would
keep going. Another request might come into the service with the database client, which places the
object somewhere else and writes the decision to the database. This decision appears to get
persisted, but the first write is what actually succeeded. We never checked what was in the
database, and returned the result of the most recent decision.
<!-- 100 chars ------------------------------------------------------------------------------------>
