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

TBD

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
