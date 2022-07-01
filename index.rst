:tocdepth: 1

Abstract
========

This technote gathers information about the history, current state and contemplates future evolutions of the Vera Rubin Observatory Control System (Rubin-OCS) Middleware.
The middleware is the backbone of the Rubin-OCS.
The highly distributed nature of the Rubin-OCS places tight constraints in terms of latency, availability and reliability for the middleware.
Here we gather information to answer some common questions regarding technology choices, describe some of the in-house work done to obtain a stable system and document some of the concerns with the current state of the system, potential impacts for near-future/commissioning and future/operations.
We also cover some of the work we have been doing to investigate alternative technologies and suggest some potential roadmaps for the future of the system.

Introduction
============

"Middleware" is the term used to describe software applications used for communication in sofware systems.
These technologies became popular with the advent of *distributed systems*, wich come as a solution to the problem of parallel computation.
As software and hardware systems became more and more complex, it become impractical to develop and execute them in a single process and, eventually, a single node.
With that, software evolved from *monolith* applications, where a single program executes in a single process or node, to *distributed* applications, where the system is divided into a number of smaller applications; each running on their own process or node.
For these applications to work together coherently they must be able to comunicate with each other, thus giving origin to middleware technologies.

How a *distributed systems* is broken down into smaller pieces is heavily dependent on the problem.
Some systems are only broken down into a small number of components each still in charge of large contexts, others are broken down into several small applications that are in charge only of small simple tasks.
The later has gained substantial popularity recently and is commonly referred to as *microservices*.
These systems are behind many of popular large services in use today like Google and Amazon. 

The architecture of *distributed systems* can take many shapes and forms.
For instance, some systems are designed to mimick or emulate *monolith* applications.
The application is composed of a number of smaller applications but there is a hierarchical organization, with components at the top, in charge of communicating and operating components at the bottom.
The advantage of these systems is that they are easier to understand and to maintain.
Since each component is isolated from the rest of the system, and only communicates with a components on the top and bottom of the hierarchical chain, adding new components is realatively easy and have minimal impact in the system.
The disadvantage is that the system is more vulnarable to outages, if a component on the top of the hierarchical chain becomes unavailable those components at the bottom also become unavailable.

A popular alternative to this architecture are *event-driven reactive systems*.
In these systems each component is designed as an independent entity that reacts to input data, be it from other components or external data services.
Since these system are desiged with separation of concerns in mind, each component must be able to act independently, *reactive systems* are usually extremely resilient.
If one component becomes unavailable, the others are expected to continue to operate, taking precautious to deal with the missing agent.
At the same time, it can become quite burdensome to update and grow these systems as their complexity increases exponentially with the number of different components.

In any case, the middleware plays a crucial part in any kind of *distributed system*, acting as the glue that binds the system together.

The Vera Rubin Observatory Control System (Rubin-OCS) is designed following an *event-driven*, *reactive*, *distributed* architecture.
The system is composed of a number of independent components that work together to execute cohesive operations.

The middleware is encapsulated with a layer of abstraction known as the Service Abstraction Layer (SAL), which uses the ADLink-OpenSpliceDDS implementation of the Data Distribution Service (DDS) message passing system.

With this high-level overview of the system in mind, we will now focus on SAL and the different aspects of the current middleware technology (ADLink-OpenSpliceDDS).

The Past
========

Probably the most important thing for us to consider when speaking about the past, is trying to understand why DDS was selection in the first place and then why the ADLink-OpenSpliceDDS implementation was adopted.

To put it in perspective, the firt commit to the current `SAL code repository`_ dates back to August 2014, some 8 years from the time of this writting and almost a whole year before the construction `first stone`_.
For comparisom, the Apache Kafka message system first stable release dates back to January 2011, whereas the `DDS version 1.0 standard` dates back to December 2004.

.. _SAL code repository: https://github.com/lsst-ts/ts_sal
.. _first stone: https://www.nsf.gov/news/news_summ.jsp?cntn_id=134805&org=NSF&from=news
.. _DDS version 1.0 standard: https://www.omg.org/spec/DDS/1.0

At the time when decision was being made about selecting a middleware technology, DDS was a mature standard.
The technology defines a powerfull real-time message system protocol with high inter-operability between platforms and programming languanges.
In fact, DDS has most of the important features we recognize as crucial for the Rubin-OCS.
Some of the most important ones are, for example:

-  Real-time message transfer capabilities.

   DDS is a brokerless messaging system with small overhead.
   As such, it is capable of high-throughput and low-latency data transfer.
   For example, in our `internal benchmarks`_, DDS reached transfer rates of the order of >16kHz with millisecond latency.
   This is certainly way beyond the requirements of our system, which are mostly constrained by the throughput of M1M3 (REQ?) and the throughput/latency required for tracking (LTS-TCS-PTG-0008 and LTS-TCS-PTG-0001 in :lts:`583` specify lead-times between 50-70ms with standard deviation of 3ms for tracking demands).

-  Durability service.

   In messaging systems, "durability" refer to the capability of a system to store published data and serve it to components that join the system afterwards.
   This service is crucial for a distributed system like Rubin-OCS as it guarantees that components comming online at any time are able to determine the state of the system by accessing previously published information.

   A surprising number of systems do not provide any type of durability service, especially those that are deemed "real-time".

   This mostly boils down from the fact that most real-time capable systems are brokerless (like DDS).
   Nevertheless, in order to provide a durability service, a system must have some kind of broker, that can store published messages and distribute them when needed.

   DDS provides a rather elegant solution to this problem.
   Basically, each independent node can be configure to act as a broker for durability service.
   One of those systems is elected as the "master" node, which will be in charge of actually distributing the data.
   If the master node falls over, some other node is elected to take its place.

   Nevertheless, as we have demonstrated in our efforts to stabilize the system (:tstn:`023`), this can have a huge impact in the system performance and adds considerable complexity in configuring the system.

-  The Quality of Service (QoS) dictates how messages are delivered under different network scenarios.

   DDS has an extremely rich QoS system with several configuration parameters.
   Nevertheless, while this might sound like a desirable feature at a first glance, it has some serious implications.
   To begin with, a large number of configuration parameters also means higher complexity, which makes it harder to predict the system behaviour under unexpected conditions.
   We have encoutered inumerous unexpected behaviour that were later linked to a certain unexpected bahaviour cause by a default setting.

.. _internal benchmarks: https://tstn-033.lsst.io/#performance

In addition to the features encounted in DDS, it is worth mentioing that it was also already in use by other projects under the NOAO/CTIO umbrella, as is the case of SOAR and the 4m Blanco telescopes on Cerro Pachon and Tololo respectively (see, for instance, the `4M TCSAPP Interfaces Quick Reference`_). 

.. _4M TCSAPP Interfaces Quick Reference: https://www.soartelescope.org/DocDB/0007/000711/001/4M%20TCSAPP%20Environment%20and%20Interfaces%20Quick%20Reference.pdf

The combined in-house expertise and powerfull set of features, made DDS a perfect middleware technology  candidate for the Vera Rubin Observatory at the time.
It is, therefore, no surprise that it was selected.

Nevertheless, it is worth mentioning that the software engineers at the time did anticipated the potential for future updates.
This led to the development of abstraction levels to isolate the middleware technology from the higher level system components, which is the idea behind SAL.

The initial version of SAL used the `RTI-Connext`_  implementation of DDS.
Nevertheless, the parent component (RTI) does not provide a public license for their software.
This alone adds some overhead due to the distributed (and mostly public) nature of the Rubin Observatory development efforts.

At the time, some preliminary benchmarks had shown that the ADLink-OpenSpliceDDS alternative implementation provided some considerable improvements over RTI-Connext.
Furthermore, ADLink-OpenSpliceDDS provides a public version of their library.
The public version is one major release behind the licensed edition and doesn't support some important professional features.
Even though the public version is not suitable for a production environment, it is certainly suitable for day-to-day development and testing, especially since inter-operability is guaranteed by the DDS standards.

Given the advantages of ADLink-OpenSpliceDDS over RTI-Connext implementation, we decided to switch early on in the project.
The transition required low-level of effort and had no impact on to the higher level software, which is expected for a well desiged API.

.. _RTI-Connext: https://www.rti.com/products

The Present
===========

TBD

The Future
==========

TBD

Summary
=======

TBD

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
