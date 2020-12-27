# Overview
This repo is a rundown of real-world incidents (and ideally their remediations) as abridged by [me](https://github.com/mattforni) whenever I have a few moments to dig in.

# 2020
## 11/02/20
### Cloudflare - A Byzantine failure in the real world
[Incident](https://www.cloudflarestatus.com/incidents/9ggr0k6dwzwg) | [Postmortem](https://blog.cloudflare.com/a-byzantine-failure-in-the-real-world/)

This incident began with a misbehaving switch and ended with a full database replica rebuild. On 2020-11-02 at 14:43 UTC a switch began misbehaving, severing a server in the `etcd` cluster from the master node. This node then called for a leader election, but because the other nodes could still communicate with the elected master there was no new leader elected. Regardless of the outcome of the election, `etcd` uses [RAFT](https://raft.github.io/) to establish consensus, which forces the cluster into a read-only (RO) mode.  The orphaned node continued to call for a new leader election until the switch recovered effectively forcing the cluster into RO for an extended period of time.

While the `etcd` cluster was in RO mode, two database clusters were unable to communicate to the system that they had healthy primary nodes and thus the synchronous replica was promoted to primary. Due to an aggressive configuration this promotion kicked off a rebuild of all replicas in both clusters. One cluster was able to handle the rebuild with relative ease, but the other did not have such luck.

The second database cluster handles authn for all API and Dashboard operations and as such receives a significant amount of read traffic. To keep queries performant this load is normally shared with the replicas, but when the replicas were unavailable the primary was forced to take the full brunt of the load and was quickly overloaded.

Operators knew that the sheer size of the database meant that the rebuild would likely take several hours to complete. In an effort to alleviate the load during the rebuild operators began to manually divert traffic to a replicant in another region. Unfortunately, authn is tied to a user session, which is stored in a Redis cluster and cannot be ported from region to region. Thus, as traffic was shifted users were decoupled from their sessions, making for a poor end user experience.

Ultimately operators shifted all traffic back to the affected region and reduced non-critical function until the replica was rebuilt and put itself back into service.
