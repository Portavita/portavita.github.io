---
layout: post
title:  "Caveats for running Kafka on K8s"
date:   2021-01-15 11:14:56 +0200
categories: Kafka
author: Fabio Pardi
---




<img src="https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/kafka_head_prague.jpg" width="320" height="460"><div>Franz Kafka Head Sculpture on display in Prague</div>



# Intro

As times change, also technology does.


Many of us started their career or studies at what I call the 'Hardware Times'. Software was running straight on the OS running straight on the Hardware.

After that, the Virtual Machines came along. A new layer on top of the hardware allowed multiple machines to run on the OS which runs on the Hardware.

More recently, we witnessed the containerization revolution. Microservices running on a VM which runs on the Hardware.

The last arrived in house is nowadays Kubernetes.

Every technology brings new challenges and in my opinion there is no silver bullet that solves all the problems.

When it comes to data in particular, the requirements sometimes do not match with what the technology has to offer, or maybe there is not much gain but only pain.

That is the case of running data intensive PostgreSQL installations on K8s. But for today I will spare you this story...


What I would like to talk to you about this time is Kafka on K8s.

Kafka can run in a cluster and allows horizontal scaling without much effort. If you do your homework properly then you will be able to design a proper architecture which keeps the worries at the door.

Since it is not possible to describe in one single article how to properly setup your Kafka on K8s, I would like to give you a few tips coming from my experience on setting up a Kafka cluster serving data for several Million patients in the Netherlands in a multi TB setup.


# Sharing is Caring

First of all my recommendation is to work in a devops fashion way involving your developers in the architecture. Kafka brokers, the consumers and the producers are all part of an ecosystem.



<img src="https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/sharing_caring.jpg" width="640" height="426"><div>Let's get our hands dirty together!</div>



How data is produced and consumed depends very much from the kind of data and the use you make of it.

When it comes to topics, partitions, compression, deletion and offsets, the decisions must be very much taken together.

My recommendation is also to properly document which decisions were taken and the reasons behind every single choice. 

Document them in your internal wiki page and do not leave it only to your git history.

Internal wiki is more accessible and quickly shows the last snapshot of the situation allowing a shared contribution.

Did you spend months learning Kafka and now your developers colleague want to take decisions you do not agree upon? Share your knowledge and grow your team stronger! Sharing is caring!



# Rolling out updates

If your Kafka cluster is composed by more than one broker, and it should, then I recommend you to put extra care on how updates are rolled out. K8s can roll out updates but the way it does that might not be convenient for your setup.

K8s updates can be rolled out pod by pod. As soon as a pod is marked 'available', the next pod is then terminated and updated. Nice, eh?

No.

Not nice when it comes to data and High Availability. That way you might end up with several pods rebalancing data or maybe even worse out of sync.

And if you have too many pods out of sync (actual In Sync Replicas < min.insync.resplicas Kafka setting) then your HA cluster will magically be Highly Unavailable.

One way to work around it is to wait for all In Sync Replicas to catch up before updating the next pod.

Another solution, quick and dirty, is to allow some time between one pod is marked available and the next one is terminated.

If you follow this simple piece of advice you will be able to roll out updates in the middle of the day. No more planned maintenance windows to roll out new changes to your cluster.



# Nodes Maintenance

Another very important aspect to take care of is the cluster availability during K8s nodes maintenance.  

K8s has been designed to allow maintenance on the nodes without compromising services availability. That assumption is not entirely true when it comes to data.

For instance you cannot do a care free maintenance on a node hosting the master database serving your customers.

Similarly, Kafka needs to have a certain number of nodes always available (and in most setups: in sync) to be able to serve data.

For that reason it is important to make sure your pods are spread over multiple nodes, and different from each other. 

The way to achieve this goal is to make use of a K8s feature called 'podAntiAffinity', you can read more about it [here](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

What it does is basically making sure that not 2 pods run on the same node but are scheduled on different nodes instead. That way if one node goes down, we have the guarantee that only one Kafka broker will not be available.

Following this idea you will then be able to spread your Kafka brokers over multiple nodes, or hardware, or racks.. I invite you to also look at the Kafka feature [rack awareness](https://kafka.apache.org/documentation/#basic_ops_racks)

 

# Conclusion

Of course this is not an exhaustive guide on how to run Kafka on K8s but I hope it will save you from some hassle and provide inspiration to think in the right direction.

Ideas or comments? You can find me on [Linkedin](https://www.linkedin.com/in/fabiopardi/)
