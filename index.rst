:tocdepth: 1

Abstract
========

After researching alternatives to ADLink-OpenSpliceDDS (and DDS) we conclude that `Kafka`_ provides the best alternative for the Vera Rubin Observatory Control System (Rubin-OCS) Middleware.
The middleware is the backbone of the Rubin-OCS, and is fundamental for stable operation of the observatory.
The highly distributed nature of the Rubin-OCS places tight constraints in terms of latency, availability and reliability for the middleware.
Here we gather information to answer common questions regarding technology choices, describe the in-house work done to obtain a stable system, highlight our concerns with the current Data Distribution Service (DDS) technology, and its potential impact for near-future/commissioning and future/operations.

Introduction
============

"Middleware" is the term used to describe software applications used for communication in software systems.
These technologies became popular with the advent of *distributed systems*, which come as a solution to the problem of parallel computation.
As software and hardware systems became more complex, it becomes impractical to develop and execute them in a single process and, eventually, a single node.
With that, software evolved from *monolithic* applications, where a single program executes in a single process or node, to *distributed* applications, where the system is divided into a number of smaller applications; each running on their own process or node.
For these applications to work together coherently they must be able to communicate with each other, thus giving origin to middleware technologies.

How a *distributed system* is broken down into smaller pieces is heavily dependent upon the problem.
Some systems are only broken down into a small number of components each still in charge of large contexts, others are broken down into many small applications that are in charge only of small simple tasks.
The latter has gained substantial popularity recently and is commonly referred to as *microservices*.
These systems are behind many of popular large services in use today like Google and Amazon. 

The architecture of *distributed systems* can take many shapes and forms.
For instance, some systems are designed to emulate *monolithic* applications.
The application is composed of a number of smaller applications but there is a hierarchical organization, with components at the top, in charge of communicating and operating components at the bottom.
The advantage of these systems is that they are easier to understand and to maintain.
Since each component is isolated from the rest of the system, and only communicates with a components on the top and bottom of the hierarchical chain, adding new components is relatively easy and have minimal impact in the system.
The disadvantage is that the system is more vulnerable to outages, if a component on the top of the hierarchical chain becomes unavailable those components at the bottom also become unavailable.

More modern *distributed systems* have been favoring less hierarchical approaches, following the principles of *`reactive systems`_*.
In these systems each component is designed as an independent entity that reacts to input data, be it from other components or external data services.
Since these systems are designed with separation of concerns in mind (e.g. each component must be able to act independently), *reactive systems* are usually extremely resilient.
If one component becomes unavailable, the others are expected to continue to operate, taking precautious to deal with the missing agent.
At the same time, it can become quite burdensome to update and grow these systems as their complexity increases exponentially with the number of different components.

.. _reactive systems: https://www.reactivemanifesto.org

In any case, the middleware plays a crucial part in any kind of *distributed system*, acting as the glue that binds the system together.

The Vera Rubin Observatory Control System (Rubin-OCS) is designed following the principles of a *distributed*, *`reactive <reactive systems>`_* architecture.
The system is composed of a number of independent components that work together to execute cohesive operations.

The middleware is encapsulated with a layer of abstraction known as the Service Abstraction Layer (SAL), which uses the ADLink-OpenSpliceDDS implementation of the Data Distribution Service (DDS) message passing system.

With this high-level overview of the system in mind, we will now focus on SAL and the different aspects of the current middleware technology (ADLink-OpenSpliceDDS).

The Past
========

Probably the most important thing for us to consider when speaking about the past, is trying to understand why DDS was selection in the first place and then why the ADLink-OpenSpliceDDS implementation was adopted.

To put it in perspective, the first commit to the current `SAL code repository`_ dates back to August 2014, some 8 years from the time of this writing and almost a whole year before the construction `first stone`_.
For comparison, the Apache Kafka message system first stable release dates back to January 2011, whereas the `DDS version 1.0 standard` dates back to December 2004 (it is hard to pinpoint the initial stable release for any of the DDS implementations since the older libraries are mostly closed-source proprietary code).

.. _SAL code repository: https://github.com/lsst-ts/ts_sal
.. _first stone: https://www.nsf.gov/news/news_summ.jsp?cntn_id=134805&org=NSF&from=news
.. _DDS version 1.0 standard: https://www.omg.org/spec/DDS/1.0

At the time the middleware technology was selected, DDS was a mature standard.
The technology defines a powerful real-time message system protocol with high inter-operability between platforms and programming languages.
In fact, DDS has most of the important features we recognize as crucial for the Rubin-OCS, including:

-  Real-time message transfer capabilities.

   DDS is a broker-less messaging system with small overhead.
   As such, it is capable of high-throughput and low-latency data transfer.
   For example, in our `internal benchmarks`_, DDS reached transfer rates of the order of >16kHz with millisecond latency.
   This is certainly way beyond the requirements of our system, which are mostly constrained by the throughput of M1M3 (REQ?) and the throughput/latency required for tracking.

   It is worth noting that the Rubin-OCS requirements are not clear with respect to latency constrains.
   Requirements LTS-TCS-PTG-0008 and LTS-TCS-PTG-0001 in :lts:`583` only specify lead-times between 50-70ms with standard deviation of 3ms for tracking demands.
   At best, these requirements can only be used to determine upper limits on latency, e.g. demands must be delivered with sufficient time to account for the lead-times.

   If is safe to say that delivering the demands 3/4 ahead of the lead-time should be sufficient time for the mount to process them, which means latency around 10-20ms.

-  Durability service.

   In messaging systems, "durability" refer to the capability of a system to store published data and serve it to components that join the system afterwards.
   This service is crucial for a distributed system like Rubin-OCS as it guarantees that components coming online at any time are able to determine the state of the system by accessing previously published information.

   A surprising number of systems do not provide any type of durability service, especially those that are deemed "real-time".

   This mostly boils down from the fact that most real-time capable systems are broker-less (like DDS).
   Nevertheless, in order to provide a durability service, a system must have some kind of broker, that can store published messages and distribute them when needed.

   DDS provides a rather elegant solution to this problem.
   Basically, each independent node can be configure to act as a broker for durability service.
   One of those systems is elected as the "master" node, which will be in charge of actually distributing the data.
   If the master node falls over, some other node is elected to take its place.

   As we have demonstrated in our efforts to stabilize the system (:tstn:`023`), this can have a huge impact in the system performance and adds considerable complexity in configuring the system.

-  The Quality of Service (QoS) dictates how messages are delivered under different network scenarios.

   DDS has an extremely rich QoS system with many configuration parameters.
   While this might sound like a desirable feature at a first glance, it has some serious implications.
   To begin with, a large number of configuration parameters also means higher complexity, which makes it harder to predict the system behavior under unexpected conditions.
   We have encountered many problems that were traced to unexpected behavior caused by QoS settings.

.. _internal benchmarks: https://tstn-033.lsst.io/#performance

In addition to the features in DDS, it is worth mentioning that it was also already in use by other projects under the NOAO/CTIO umbrella, including the SOAR and the 4m Blanco telescopes on Cerro Pachon and Tololo respectively (see, for instance, the `4M TCSAPP Interfaces Quick Reference`_). 

.. _4M TCSAPP Interfaces Quick Reference: https://www.soartelescope.org/DocDB/0007/000711/001/4M%20TCSAPP%20Environment%20and%20Interfaces%20Quick%20Reference.pdf

The combined in-house expertise and powerful set of features, made DDS a perfect middleware technology candidate for the Vera Rubin Observatory at the time.
It is, therefore, no surprise that it was selected.

It is worth mentioning that the software engineers at the time did anticipate the potential for future updates.
This led to the development of abstraction levels to isolate the middleware technology from the higher level system components, which is the idea behind SAL.

The initial version of SAL used the `RTI-Connext`_  implementation of DDS.
Unfortunately, the parent component (RTI) does not provide a public license for their software.
This alone adds substantial overhead to the development and deployment cycle, especially given the distributed (and mostly public) nature of the Rubin Observatory efforts.
In addition to the cost of purchasing licenses, we are also required to distribute the licensed code to team member and external collaborators/vendors.
Furthermore, we must also make sure collaborators are not publicising the software/license, which could have potential legal repercussions to the project. 

Alternatively, the ADLink-OpenSpliceDDS implementation shows comparable benchmarks to that of RTI-Connext, with the benefit of providing a public version of their library.
The public version is (usually) one major release behind the professional edition and excludes some important features we end up requiring for the production environment.
Even though the public version is not suitable for a production environment, it is certainly suitable for day-to-day development and testing, especially since inter-operability is guaranteed by the DDS standards.

Given the advantages of ADLink-OpenSpliceDDS over RTI-Connext implementation, we decided to switch early on in the project.
The transition required low-level of effort and had no impact on to the higher level software, which is expected for a well designed API.

.. _RTI-Connext: https://www.rti.com/products

The Present
===========

At the present state of the project, we have been routinely deploying and testing a stable system comprised of the majority of the components that are part of the Rubin-OCS at the summit (e.g. production environment), the NCSA Test Stand (decommissioned in February 2022) and the Tucson Test Stand.

Achieving this stage of the project was not without its challenges related to DDS and, more specifically, with the ADLink-OpenSpliceDDS implementation.
In fact, it took our team a good part of a year to be able to obtain a stable system.
Most of our findings are summarized in :tstn:`023`.

However, even after all these efforts we still encounter DDS-related issues.
As we mentioned above, some of them are a result of the choice of configuration settings, which are quite extensive in DDS.
Others are related to network outages (momentarily or not), and/or fluctuations in the network traffic and how they are handled by the ADLink-OpenSpliceDDS library.

A more serious and worrisome category of issues are related to errors encountered in the ADLink-OpenSpliceDDS software stack, in particular:

-  It is common to encounter segmentation faults, one of the most serious types of software errors that are hard to investigate.
-  It is very expensive and time consuming to evaluate new releases and track down the problems far enough to provide reasonable bug reports.
   Then, it usually takes them a long time to reproduce and fix the problem, and if the fix appears in the next release, there are often new bugs.
-  Most or all of the stable versions we have used are a result of applying patches provided by ADLink to older releases, rather than using a new unpatched release.
-  In at least one case a patch we still use was withdrawn by the company, with no reasonable alternative.
-  We have encountered crashes on the daemon used to handle the DDS traffic, which requires restarting all components running on that particular node.
-  There are issues with the daemon that prevent us from using a more robust configuration, that would be more resilient to network outages.

In general, we believe the project is not receiving an appropriate return of investment with ADLink-OpenSpliceDDS.

Furthermore, ADLink has recently `announced`_ that the public version of OpenSpliceDDS is no longer going to be supported.
Their previous policy was to keep the community/public library one major version behind the licensed edition.
Nevertheless, since the announcement, it is now two major versions behind.
If ADLink continues to maintain the commercial version, the public version will continue to lag farther behind, until it likely becomes impossible to use a mix of the two (the free version for development, the commercial version for deployment).
However, we suspect ADLink will **not** continue to update/support the commercial version for long.
In their announcement, they made it clear that users of their *commercial* library should migrate to the new and upcoming `Cyclone DDS`_ library, whereas users of the community/public edition are left with no recourse.

.. _announced: https://github.com/ADLINK-IST/opensplice#announcement
.. _Cyclone DDS: https://projects.eclipse.org/projects/iot.cyclonedds

Altogether this situation is extremely worrisome, especially as it suggest ADLink-OpenSpliceDDS might be heading towards its end-of-life support, risking our ability to maintain the software over life of the survey.
It is worth noting that this would violate a couple of our systems requirements, more specifically, requirements OCS-REQ-0006 and OCS-REQ-0022 :cite:`LSE-62`, which concerns support for the expected lifetime of the project (e.g. the 10 years survey operations).

The Future
==========

Anticipating the need to replace OpenSpliceDDS by some other middleware technology in the future, our team has been studying possible alternatives.
We focused most of our efforts in protocols that support the so-called publish-subscribe model, which is the one used by DDS, but we also explored other alternatives as well.
The details of our study are outside the scope of this document, however, we have categorized our findings as follows:

-  Alternative DDS implementations.

   ADLink-OpenSpliceDDS is one of many implementations of the DDS standard.
   Notably, RTI-Connext, which was initially used in SAL is still a viable option worth exploring.
   We scheduled a meeting with an engineer and a commercial representative from RTI to discuss the several questions we had with their system, both technical and licensing.
   Unfortunately, not much have changed since we replaced RTI-Connext with ADLink-OpenSpliceDDS, and the issues we had in the past were still relevant.
   It is also worth noting that their Python support is still a concern (see furthermore).

-  Lack of durability service.

   As we mentioned previously, a good fraction of message passing systems lacks support for durability service, especially those that are deemed "real-time" systems which, in general, opt to a broker-less architecture.
   Some examples of message systems that falls in this category are ZeroMQ and nanomessage.
   Both these solutions are advertised as broker-less with "real-time" capabilities.
   ZeroMQ is known by its simplicity and easy to use whereas nanomessage was adopted as the message system for GMT.

-  Python libraries and support for asyncio.

   With Python being a popular language, one would expect to find broad support for the majority of the message passing systems.
   The reality though, is that most systems provide Python support only through non-native C bindings.
   This is, for instance, the case with the ADLink-OpenSpliceDDS we currently use.
   It is also extremely rare to find message systems with native support for Python asyncio, which is heavily used in salobj.

-  Real-time capabilities.

   Although the definition of what a real-time message passing system is not well defined, it is generally accepted that they must have latency on the range of 6-20 milliseconds or better :cite:`DBLP:books/daglib/0007303`.
   The vast majority of message passing systems claim to be capable of real-time data transport.
   However, because the definition of real-time is somewhat loose, it is not straightforward to verify or challenge those claims.
   Ultimately, these need to be put into context for a particular system and verified.
   For our particular case, we should be able to meet the tracking requirements with latency around 10-20ms.

   Any system we choose must first be capable of achieving these levels of latency under the conditions imposed by our system, regardless of their claims.

-  Alternative architectures.

   There are some existing frameworks both in industry and adopted by different observatories that, in principle, could provide a viable alternative to DDS as a middleware though they implement different architectures.
   Probably the best example of frameworks on this category is `TANGO`_ which, in turn, is designed on top of the `CORBA`_ middleware.

   Contrary to DDS, which defines a data-driven (publish-subscribe) architecture, CORBA implements an object-oriented model which is more suitable for a hierarchical system architecture.
   Although it would be, in principle, possible to use CORBA in a data-driven scenario, it is not what it was designed for, which makes it hard to anticipate pitfalls we could encounter in the adoption process.
   Therefore, even though we explored some of these alternative architectures systems, and some of them shows some promise, it seems like a larger risk than to find a suitable publish-subscribe alternative to DDS.

   .. note::

      It is worth noting that both CORBA and DDS standards are managed by the same organization, the Object Management Group (`OMG`_) and both rely on the Interface Description Language (`IDL`_).


.. _TANGO: https://www.esrf.fr/computing/cs/tango/tango_doc/icaleps99/WA2I01.html
.. _CORBA: https://www.corba.org
.. _OMG: https://www.omg.org
.. _IDL: https://www.omg.org/spec/IDL/4.2/About-IDL/

After extensively researching alternatives to ADLink-OpenSpliceDDS (and DDS) we believe that our best alternative is `Kafka`_.

`Kafka`_ is an open source event streaming platform that is broadly used in industry.
In fact, it is already an integral part of the Rubin-OCS, as it is used in the EFD to transport the data from DDS to influxDB (:sqr:`034` :cite:`SQR-034`).
It is also used in the LSST Alert Distribution service :cite:`LDM-612`.
Overall we already have extensive in-house expertise.

.. _Kafka: https://kafka.apache.org

The fact that we are already using Kafka in the system reliably to ingest data into the EFD gives us confidence that it is, at the very least, able to handle the overall data throughput.
Our main concern is than to verify that Kafka can handle the latency requirements of our system.
In principle, Kafka is advertised as a "real-time" system and numerous benchmarks exists online showing it can reach latencies at the millisecond regime.
Nevertheless, it is unclear those benchmarks would be applied to our systems constrains, giving the typical message size, network architecture and other relevant factors.

We then proceeded to perform benchmarks with the intention to evaluate Kafka's performance considering our system architecture.
The results, which are detailed in :tstn:`033`, are encouraging.
In summary, we obtain similar latency levels for both Kafka and DDS.
In terms of throughput, DDS is considerably better than Kafka for smaller messages, though we obtain similar values for larger messages.
It is also worth mentioning again that the overall throughput we achieve with Kafka, for small and large messages, is above our systems requirements.

Overall, our detailed study shows that Kafka would be a viable option for replacing DDS as the middleware technology in our system.
For the full technical report see :tstn:`033`.

Summary
=======

After considerable effort fine tuning the DDS middleware configuration, we were finally able to obtain a stable system, that is capable of operating at large scale with low middleware-related failure rate.
At the current advanced state of the project, which is approaching its final construction stages, one might be tempted to accept this part of the project as concluded.

As we demonstrated, there are a number of issues hiding underneath that may pose significant problems in the future, or even be seen as violating system requirements.

Overall our experience with DDS has been frustrating and disappointing.
Even though the technology is capable of achieving impressive throughput and latency, in reality, it proved to be extremely cumbersome and hard to manage and debug on large scale systems.
On top of if all we also face a potentially end-of-life cycle of the adopted library, which makes the problem considerably worse.

After exploring different solutions to the problem of long-term maintenance of our middleware, we propose to replace DDS by the already in-use Kafka.
Our benchmarks shows that Kafka is able to fulfill our system throughput and latency requirements.
We also shown that transitioning to Kafka would require minimum effort and minimum code refactoring.

We also note that there are major advantages of transitioning to Kafka before the end of construction.
For instance, developers are actively engaged with the system and motivated.
Furthermore, it also gives us the opportunity to perform the transition while system uptime pressure is not as large as it will become once commissioning of the main telescope commences.

Given our development cycle and the current state of the system we expect to be able to fully transition to Kafka in a 1 to 2 deployment cycles (1-3 months approximately), with no impact to the summit and minimum to no downtime on the Tucson Test Stand.
This estimate is based on the assumption that we have finished porting all our code-base to support Kafka, including the remaining salobj-based services that were not ported as part of :tstn:`033` efforts as well as providing a Kafka-based version of SAL to drive the C++, LabView and Java applications.
We do not anticipate spending too much time tunning Kafka, since these efforts have already been done by SQuaRe to support EFD ingestion.
Overall, we expect the total efforts to take between 6 months to a year.


.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
