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

How a *distributed system* is broken down into smaller pieces is heavily dependent uppon the problem.
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

At the present state of the project, we have been routinely deploying and testing a stable system comprised of the majority of the components that are part of the Rubin-OCS at the summit (e.g. production environment) and the Tucson Test Stand.

Achieving this stage of the project was not without its challenges related to DDS and, more specifically, with the ADLink-OpenSpliceDDS implementation.
In fact, it took our team a good part of a year to be able to obtain a stable system.
Most of our findings are summarized in :tstn:`023`.

Nevertheless, even after all these efforts we still encounter DDS-related issues.
As we mentioned above, some of them are a result of the choice of configuration settings, which are quite extensive in DDS.
Others are related to network outages (momentarily or not), and/or fluctuations in the network traffic and how they are handled by the ADLink-OpenSpliceDDS library.

A more serious and worrisome cathegory of issues are related to errors encountered in the ADLink-OpenSpliceDDS software stack.
For example, we have encountered crashes on the daemon used to handle the DDS traffic, which requires restarting the components running on that particular node.
We have also encountered issues with the daemon that prevent us from using a more robust configuration, that would be more resilient to network outages.
For those issues that we are able to track down, we routinely open cases with ADLink, however, their responses to some of the issues we encounter are akin to what we expect given the premium required for a professional edition.

Furthermore, ADLink has recently `announced`_ that the public version of OpenSpliceDDS is no longer going to be supported.
Their previous policy was to keep the public library one major version behind the licensed edition.
They had also granted us permission to publicly use the licensed Python bindings with the public library, which was required due to otherwise unfixable issues with the public edition.
Furthermore, in their announcement, they also make it clear that users of the library should migrate to the new and upcomming `Cyclone DDS`_.

.. _announced: https://github.com/ADLINK-IST/opensplice#announcement
.. _Cyclone DDS: https://projects.eclipse.org/projects/iot.cyclonedds

Altogether this situation is extremely worrisome, especially as it suggest ADLink-OpenSpliceDDS might be heading towars its end-of-life support, exposing potential issues fullfiling a couple requirements.
More specifically, requirements OCS-REQ-0006 and OCS-REQ-0022 :cite:`LSE-62`, which concerns the expected lifetime of the project (e.g. the 10 years survey operations).

The Future
==========

Anticipating the need to replace OpenSpliceDDS by some other middleware technology in the future, our team has been studying possible alternatives.
We focused most of our efforts in protocols that support the so-called publish-subscribe model, which is the one used by DDS, but we also explored other alternatives as well.
The details of our study are outside the scope of this document, however, we have cathegorized our findings as follows:

-  Alternative DDS implementations.

   ADLink-OpenSpliceDDS is one of many implementations of the DDS standard.
   There are, in fact, a breadth of alternative solutions available, both public and private.

   Notably, RTI-Connext, which was initially used in SAL is still viable option worth exploring.
   We scheduled a meeting with an engineer and a commercial representative from RTI to discuss the several questions we had with their system, both technical and licensing.
   Unfortunately, not much have changed since we replaced RTI-Connext with ADLink-OpenSpliceDDS, and the issues we had in the past were still relevant.
   It is also worth noting that their Python support is still a concern (see furthermore).

-  Lack of durability service.

   As we mentioned previously, a good fraction of message passing systems lacks support for durability service, especially those that are deemed "real-time" systems which, in general, opt to a brokerless architecture.

   Some examples of message systems that falls in this cathegory are ZeroMQ and nanomessage.
   Both these solutions are advertised as brokerless with "real-time" capabilities.
   ZeroMQ is known by its simplicity and easy to use whereas nanomessage was adopted as the message system for GMT.

-  Python libraries and support for asyncio.

   With Python being a popular language, one would expect to find broad support for the majority of the message passing systems.
   Nevertheless, the reality of it is that most systems provide Python support through non-native C bindings.
   This is, for instance, the case with the ADLink-OpenSpliceDDS we currently use.

   It is also extremely rare to find message systems with native support for Python asyncio, which is heavily used in salobj.

-  Real-time capabilities.

   Although the definition of what a real-time message passing system is not well defined, it is generally accepted that they must have latency on the range of 6-20 milliseconds or better :cite:`DBLP:books/daglib/0007303`.

   The vast majority of message passing systems claim to be capable of real-time data transport.
   
   Nevertheless, because the definition of real-time is somewhat loose, those claims can be challenged and most importantly, need to be put into context for a particular system and verified.

   As mentioned previously, the tracking requirements on Rubin-OCS demands latency of around 3ms.
   Any system we choose must first be capable of achieving these levels of latency under the conditions imposed by our system, regardless of their claims.

-  Alternative architectures.

   There are some existing frameworks both in industry and adopted by different observatories that, in principle, could provide a viable alternative to DDS as a middleware though they implement different architectures.

   Probably the best example of frameworks on this cathegory is `TANGO`_ which, in turn, is desiged on top of the `CORBA`_ middleware.

   .. note:

      It is worth noting that both CORBA and DDS standards are managed by the same organization, the Object Management Group (`OMG`_`) and both rely on the Interface Description Language (`IDL`_).

   Contrary to DDS, which defines a data-driven (publish-subscribe) architecture, CORBA implements an object-oriented model which is more suitable for a hierarchical system architecture.

   Although it would be, in principle, possible to use CORBA in a data-driven scenario, it is not what it was desiged for, which makes it hard to anticipate pitfalls we could encounter in the adoption process.

   Therefore, even though we explored some of these alternative architectures systems, and some of them shows some promise, it seems like a larger risk than to find a suitable publish-subscribe alternative to DDS.

.. _TANGO: https://www.esrf.fr/computing/cs/tango/tango_doc/icaleps99/WA2I01.html
.. _CORBA: https://www.corba.org
.. _OMG: https://www.omg.org
.. _IDL: https://www.omg.org/spec/IDL/4.2/About-IDL/

After extensively researching alternatives to ADLink-OpenSpliceDDS (and DDS) we have finally converged into a best alternaltive; `Kafka`_.

`Kafka`_ is an open source event streaming platform that is broadly used in industry.
In fact, it is already an intergral part of the Rubin-OCS, as it is used in the EFD to transport the data from DDS to influxDB (:sqr:`034` :cite:`SQR-034`).

.. _Kafka: https://kafka.apache.org

The fact that we are already using Kafka in the system reliably to ingest data into the EFD gives us confidence that it is, at the very least, able to handle the overall data throughput.
Our main concern is than to verify that Kafka can handle the latency requirements of our system.
In principle, Kafka is advertised as a "real-time" system and inumerous benchmarks exists online showing it can reach latencies at the millisecond regime.
Nevertheless, it is unclear those benchmarks would be applied to our systems constrains, giving the tipical message size, network architecture and other relevant factors.

We then proceded to perform benchmarks with the intention to evaluate Kafka's performance considering our system architecture.
The results, which are detailed in :tstn:`033`, are encouraging.
In summary, we obtain similar latency levels for both Kafka and DDS.
In terms of throughput, DDS is considerably better than Kafka for smaller messages, though we obtain similar values for larger messages.

Overall, our detailed study shows that Kafka would be a viable option for replacing DDS as the middleware technology in our system.
For the full technical report see :tstn:`033`.

Summary
=======

After considerable effort fine tunning the DDS middleware configuration, we were finally able to obtain a stable system, that is capable of operating at large scale with low middleware-related failure rate.
At the current advanced state of the project, which is approaching its final construction stages, one might be tempted to accept this part of the project as concluded.

Nevertheless, as we demonstrated, there are a number of issues hidding underneath that may pose significant problems in the future, or even be seen as violating system requirements.

Overall our experience with DDS has been, to say the least, underwhelming.
Even thought the technology is capable of achieving impressive throughputs and latency, in reality, it proved to be extremely cumbersome and hard to manage and debug on large scale systems.
On top of if all we also face a potentially end-of-life cycle of the adopted library, which makes the problem considerably worse.

After extensive exploring different alternatives to the problem we propose a potential long term solution to the problem, namely, replace DDS by the already in-use Kafka.
Our benchmarks shows that Kafka is able to fullfill our system throughput and latency requirements.
We also shown that transitioning to Kafka would require minimum effort and minimum code refactor.

We also note that there are major advantages of transitioning to Kafka before the end of construction.
To begin with, we take advantage of a "standing army", as developers are actively engaged with the system and motivated.
Furthermore, it also gives us the opportunity to perform the transition in a time when uptime pressure is not as large as it will become once commissioning of the main telescope commences.
Given our development cycle and the current state of the system we expect to be able to fully transition to Kafka in a 1 to 2 deployment cycles (1-3 months approximately), with no impact to the summit and minimum to no downtime on the Tucson Test Stand.


.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
