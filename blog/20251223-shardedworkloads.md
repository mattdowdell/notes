# ShardedWorkloads

<!-- 100 chars ------------------------------------------------------------------------------------>

## Notes

_Working notes, to be removed_

- Come up with better section titles (and title).

## Background

The original ObjectStorage product at Akamai was inherited as part of their acquisition of Linode.
It was basically a bunch of storage and compute, with [Ceph] installed. There were multiple
independent Ceph clusters and customers interacted with each of them separately. This implementation
has limits on the number of objects, the amount of data it could store and the throughput it could
support. I guess there's a limit to how many nodes and disks you can get in a Ceph cluster.

Akamai later acquired Ondat, and tasked the team with coming up with an ObjectStorage offering that
could scale beyond the limits. The new offering became known internally as Gen 2, whilst the
original was imaginatively named Gen 1. The idea was relatively simple: join multiple Ceph clusters
together and shard buckets and objects across them. To add more capacity, we simply add more Ceph
clusters. To present the illusion of a single Ceph cluster to customers, a number of microservices
running on Kubernetes were created. One service created and managed buckets, another tracked where
objects were located, and so on.

I joined Akamai in August 2024, a few months before their new ObjectStorage product reached General
Availability (GA) in January 2025. Onboarding naturally included learning the architecture of the
product. And to my pleasant surprise, it was more or less what I would have designed myself given
the same requirements. However, as with almost every software design task I've ever seen, there were
trade-offs.

[Ceph]: https://ceph.io/en/

## Problem

Our main trade-off was gaining performance at the cost of availability. Specifically, we would lose
writes if a specific microservice had any downtime. As we have a weekly release schedule, this
happened every week. The reason was down to how we used StatefulSets in Kubernetes.

The core problem being solved was deciding where each new object would be placed. A bucket was
sharded across multiple Ceph clusters, but an object existing in just one of those shards. We select
a Ceph cluster at random, although a deterministic placement algorithm would probably have worked in
hindsight. We persisted the placement decision in a key-value database, specifically [TiKV]. TiKV is
a distributed database, and offers transactions which would ensure that 2 concurrent placements
could not result in an object in 2 places at once. But the transactions were deemed too slow to meet
our performance requirements. Instead, we moved the synchronisation into the microservice making the
decision. This was basically a big map of mutexes. If you wanted to place an object, you'd lock the
mutex for the object's key, make the decision, and then release the lock. Because the locks were in
memory, each Pod had to be able to assume that it had sole ownership of the lock. This is where
StatefulSets came in.

The idea of a StatefulSet is that you have a number of Pods, each with a unique identity. This
identity provides a stable and unique network address. Whenever we needed to place a new object, we
would calculate a hash of the key, use that to select the appropriate Pod, and send the request to
said Pod. All this meant that if any single Pod in the StatefulSet was down, we lost the ability to
write to the objects that belong to that particular shard. When an update is made to a StatefulSet,
it destroys each Pod and then recreates it, thus maintaining the unique identity and creating our
weekly downtime.

Fortunately, this issue was recognised as a significant problem well before GA and someone was
assigned to improving it. This improvement took the form of an operator that was meant to reduce the
downtime between updates. Unfortunately, that person never delievered it, despite working on it for
around 9 months. There were a few theories why, but at this point I'm not sure it matters. What does
matter is that the SRE team would repeatedly ask when the solution was going to arrive, and each
time be told it was merely weeks away. Eventually, management had to claim defeat and moved the
project to someone else: me.

[TiKV]: https://tikv.org/

## Evaluating

Taking over a failing unreleased project is an interesting experience. Starting on a codebase that's
running in production and being used by customers is relatively easy by comparison. With sufficient
telemetry, you can see what's working and what's not, and work towards fixing the latter. If you
don't have the telemetry, you can add it. For something that was never released to production, I
later learned the first question is whether it works. For the record, it didn't. But instead, the
first task I was given was getting it up to the standards of our existing microservices.

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
