---
layout: post
title: SolidFire, vVols, and QoS Noisy Neighbor Issues
---

# Intro Note

This is a blog post I wrote back in September of 2018 to submit to VMware and NetApp after a pretty nasty issue we had with horendus disk latency when using NetApp SolidFire presented as vVols to ESXi with low IOPS limits enable, but I never had anywhere to post it.  Now that I have my blog, I figured it would be a good thing to publicize.  As far as I know, this is still an issue, but if it's not, there is still some really great information in this post that would be valuable to have out there in the world.

# The Problem

Several weeks ago, users of our platform started reporting periodic issues with what appeared to be high latencies on the storage we had allocated to a VMware based environment.  They frequently reported latencies in the 300-400 ms range.

Because of our use of IOPS QoS policies (more below) we assumed that the user had maxed out their IOPS policy.  However, we quickly realized the QoS policy was actually artificially introducing the noisy neighbor problem, and all VM's on the hypervisor (at least the ones we looked at) were experiencing high disk latencies by just a single VM exceeding their QoS policy.  By trying to solve the noisy neighbor problem, it appeared that we actually made it 10x worse.  On a cluster that was capable of doing over 200k IOPS, a single user doing 3000 4k IOPS was able to impact performance for all other VM's on the host, despite the storage barely being utilized.

To summarize, we had 2 problems:

1. A single user was maxing out their QoS policy (identify by #2).  I'll refer to this as the "Bad Neighbor" VM going forward.
2.  All VM's on the host that a maxed out IOPS policy VM was on experienced high disk latencies (300-400 ms).  I'll reference to this as the "Good Neighbor" going forward.

# The environment

The storage technology is NetApp SolidFire, which is an all-flash based software defined storage, which provides the capability to provide a minimum, maximum, and burst level of IOPS on a per volume setting.  Traditionally in a VMware environment, this would mean that you would be able to set QoS policies on the LUN that is presented for an entire datastore, and the aggregate of all VM's on that datastore would recieve the QoS policies.

However, couple this with VMware vVols and you have the ability for each VM "disk" to be a virtual volume on the SolidFire cluster. In our environment, we implemented this to prevent "noisy neighbor" situations in which a single VM (or a few) could consulme all the available IOPS for a given host/cluster/array.

In order to give our consumers options, we built 3 tiers of storage using SPBM profiles in vCenter:

| QoS Policy | Min IOPS | Max IOPS | Burst IOPS|
| ---- | ---- | ---- | ---- |
| Bronze | 25 | 1000 | 2000 |
| Silver | 50 | 1500 | 3000 |
| Gold | 100 | 3000 | 6000 |

The storage was presented to ESXi clusters over iSCSI, but was not currently set up with multi-pathing because:

1.  We only have 2 10GB NICs, and multi-pathing appears to require dedicated NICS for ESXi
2.  Our iSCSI initiator and targets are in different subnets, which [does not appear to be supported by iSCSI port binding](https://kb.vmware.com/s/article/2038869)

Additionally, no extra tuning had been performed.  All settings on VMware were out of the box.

# Troubleshooting

We quickly found that by migrating the "Good Neighbor" VM off the hypervisor, the issue went away for that VM.  However, it wasn't immediately apparent which VM was causing the problem.  We looked at all VM's and who was consuming the most IOPS, but we didn't see anything that was nearing a 2000-3000 max we had defined, so we kept searching.

In the meantime, another user reported the same issue to us, so we started looking in to their VM.  Assuming it was the same issue, we moved it to another hypervisor.  However, this time, it did not resolve the issue.  We then looked at the IOPS and noted that it was rather low and not hitting the QoS Policies.  We took a peak at what VMware was reporting the average block size of read/writes was, and noted it was doing an average of 512k block writes.  The total IOPS was not significant, so we weren't too concerned about it being a QoS policy at this point.

After a bit of testing with FIO, we realized (which we knew, but just had forgotten) that the SolidFire QoS policies scale with the block size.  In other words, a 6000 IOPs burst policy is defined at the 4k block size level.  When you go up to a 262k block size, it's only 154 IOPS:

![SolidFire IOPS](https://github.com/ryan-a-baker/ryanbakerio/blob/master/img/sfiops.png?raw=true){: .center-block :}

The "bad neighbor" VM had selected a bronze tier of storage, which granted them 2000 burst IOPS at 4k block size, which is somewhere around 25 IOPS at 512k block size.  Increasing their QoS policy to gold brought their performance inline with what they expected from a previous environment.  Great, problem #1 solved.

What about problem #2 though?  A bad neighbor exceeding their QoS policy should not be able to impact other VM's on the same hypervisor.

# Replicating the problem

The first thing we aimed to do was to be able to recreate the problem.  To do this, we went to our lab, which consisted of a single node SolidFire "cluster".

We created two VM's on a single hypervisor, on the same datastore, and used FIO and IOPING to attempt to recreate.

All the testing we had done on this had been around small block sizes.  On VM A, we ran the following FIO command:

```
fio --name=randwrite --ioengine=libaio --iodepth=1 --rw=randwrite --bs=4k --direct=1 --size=512MB --numjobs=1 --runtime=600 --unlink=0
```

Then, on VM B, we ran the following IOPING command:

```
ioping -c 100 .
```

This was unsuccessful recreated the problem despite hitting the max QoS policy.

Then, we jumped to a 512k block size with FIO:

```
fio --name=randwrite --ioengine=libaio --iodepth=1 --rw=randwrite --bs=512k --direct=1 --size=512MB --numjobs=1 --runtime=600 --unlink=0
```

Again, we were still unsuccessful at replicating.  Then, we ramped up the iodepth and number of jobs:

```
fio --name=randwrite --ioengine=libaio --iodepth=8 --rw=randwrite --bs=512k --direct=1 --size=512MB --numjobs=5 --runtime=600 --unlink=0
```

Bingo!

![Disk Latency](https://github.com/ryan-a-baker/ryanbakerio/blob/master/img/latency.png?raw=true){: .center-block :}

It's very obvious where we started the FIO test, and now we can replicate independently, without having to ask our end users to do it.  In my opinion, this should always be the first step in fixing a problem.  There is nothing worse than going back and forth with a consumer of my platform asking "Can you test again to see if that fixed it?"  I never felt that portrayed much confidence in your platform, and try to avoid it all costs.
