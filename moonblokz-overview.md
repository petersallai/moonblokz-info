# MoonBlokz Overview

## Purpose

MoonBlokz is a hyper-local DePIN blockchain designed to run on small, constrained devices that communicate over radio instead of relying on high-speed internet. Its purpose is to provide a local economic coordination layer — starting with cryptocurrency-style payments between nodes — in environments where centralized infrastructure is unavailable, unreliable, too expensive, or intentionally absent.

The project is not trying to build another global blockchain network. Instead, it explores how a blockchain can support local contracting, accounting, and payments in places where only local devices and local radio connectivity are available.

## Problem Statement

Most existing blockchain systems remove dependence on traditional financial intermediaries, but they still depend on a powerful global communications substrate: the internet. MoonBlokz starts from a different assumption: there are scenarios where internet access cannot be treated as a guaranteed foundation.

The project therefore asks a more constrained question:

> How can a blockchain-based economic system work if participating nodes are small, low-power devices that communicate only through weak, low-bandwidth radio links?

This makes MoonBlokz relevant for edge environments, temporary deployments, isolated communities, and other situations where a local economy may need to function without external network infrastructure.

## Vision

The long-term vision of MoonBlokz is to make it possible to create a functioning economy “where nothing is available” from an infrastructure perspective. In practical terms, this means enabling a set of local devices or operators to:

- exchange value,
- record transactions,
- coordinate work,
- define contracts,
- and operate with minimal dependence on centralized services.

This vision combines blockchain ideas with highly constrained embedded hardware and local radio networking.

## Representative Use Cases

Part I presents three illustrative use cases that define the direction of the project.

### 1. Autonomous device economies in space

MoonBlokz is imagined as a possible economic substrate for autonomous and semi-autonomous devices working far from Earth-based infrastructure, for example in future asteroid-mining or other space-resource operations. In such environments, lightweight local contracting and payment capabilities may be necessary for coordination between agents.

This is presented as a long-term, visionary scenario rather than an immediate commercial target, but it helps clarify the ambition behind the project.

### 2. Local payment network for a town or city

A second use case focuses on resilience in a disrupted environment. If a town or city could no longer rely on the normal centralized payment stack, a local blockchain operated by many households or local participants could provide an alternative transaction infrastructure.

This use case highlights MoonBlokz as a tool for community-scale financial continuity under degraded conditions.

### 3. Payments and contracting on rural work sites

The most grounded example is a large rural construction or industrial site where multiple companies, workers, and possibly automated devices collaborate with limited infrastructure. In this setting, MoonBlokz could support local contracts and payments while reducing administrative overhead.

This scenario is important because it demonstrates that the system is not only speculative; it also targets real operational settings where localized coordination has practical value.

## Additional Representative Use Cases Shown on the Current Public Website

In addition to the three illustrative Part I scenarios above, the current public MoonBlokz website presents a broader set of representative use-case areas. These should be treated as current public-facing positioning material rather than as Part I source content.

### 1. Industrial Worksites

Large industrial or isolated worksites may need local accounting, coordination, and service tracking across a wide physical area with incomplete connectivity and many known participants.

### 2. Agricultural Operations

Large farms or cooperating agricultural operations may benefit from local coordination between machines, operators, storage points, and shared resources in environments where connectivity is uneven.

### 3. Logistics and Ports

Logistics yards, warehouse parks, and port zones may use a local transaction and accounting layer to track repeated service interactions, equipment access, capacity use, and internal operational events.

### 4. Community Backup Payments

A town, district, or local community may need a fallback local transaction layer when centralized payment infrastructure is degraded, unreachable, or unavailable.

### 5. Disaster Response

Temporary emergency zones may need local coordination and operational accounting in environments where infrastructure is damaged, overloaded, or fragmented.

### 6. Off-Grid Communities

Communities operating with limited infrastructure over long periods may need local exchange and coordination tools that do not depend on always-on centralized services.

### 7. Machine-to-Machine Coordination

Some environments require structured local interaction not only between people, but also between active devices that register service events, access rights, handoffs, or internal transactions.

### 8. Research Expeditions

Remote research expeditions and field science bases may need local coordination, shared-resource tracking, and operational recordkeeping under intermittent connectivity.

### 9. Humanitarian Relief Operations

Relief camps and field operations may need internal accounting, controlled access to resources, work tracking, and local coordination in degraded environments.

### 10. Space Device Economies

The public website also retains a long-range MoonBlokz scenario focused on autonomous or semi-autonomous devices operating in extremely isolated environments where local economic coordination may be necessary.

## Core Design Framing

MoonBlokz is framed around a few foundational ideas already visible in Part I:

- **Hyper-local first:** the system is intended for local or regional clusters, not for a single worldwide network.
- **Infrastructure independence:** the network should continue to function without requiring high-speed internet.
- **Embedded-device viability:** the prototype must run on a standard, readily available microcontroller.
- **Radio-native communication:** nodes communicate through radio, with LoRa selected as the initial prototype target.
- **Mesh-oriented scale-out:** the physical size of the network should not be limited only by single-hop radio range; nodes are expected to relay traffic through a mesh.
- **Operational simplicity:** nodes should be easy to configure and require minimal setup.
- **Future extensibility:** the MVP starts with currency transfer, but the chain should later support broader payloads and higher-level logic.

In the current MoonBlokz direction, this simplicity goal does not conflict with compile-time backend selection in lower-level crates. Build-time feature selection is primarily a product-build concern: it defines which hardware backend, memory profile, and similar platform choices are compiled into a given blockchain application image. Once those deployment-level choices are fixed for a given build, node-level configuration can remain intentionally small. In the simplest case, if radio parameters are also fixed by that build or deployment profile, a node may only need values such as `node_id` and `private_key` to become operational.

## Project Goals

The article states the following goals and requirements for MoonBlokz:

1. The blockchain must securely and reliably handle cryptocurrency transactions between nodes.
2. The network should scale from as few as two nodes to tens of thousands of nodes.

   This scaling goal should be read at network level, not as a requirement that each node maintain a complete view of all other nodes. In the current architecture, per-node radio state is intentionally local and bounded. The connection matrix is used to track direct or otherwise immediately relevant local neighbors, while larger network size is achieved through relaying across multiple hops. If a node can directly observe more peers than fit into its bounded local matrix, the radio logic still aims to retain the more useful or stronger entries rather than treating matrix size as a global network-size cap.

3. The consensus algorithm must support the operation of the network.
4. The prototype must run on a standard microcontroller available on the market.
5. The network should be able to grow beyond direct radio range by using relaying in a mesh-like topology.
6. The implementation must handle weak, unreliable radio communication and temporarily disconnected parts of the network.
7. The solution should be portable across different microcontrollers or computers and different radio technologies.
8. Nodes should be easy to configure.
9. The chain should support chain-level configuration for both operational and macroeconomic parameters.
10. The MVP should initially focus on currency transactions, while keeping the design open to future payload types such as smart-contract-like capabilities or other extensions.

## Architectural Implications from Part I

Although Part I is primarily an introduction rather than a full architecture specification, it already implies several important architectural directions.

### Local-first network model

MoonBlokz assumes that economic coordination can emerge inside a local communication domain. That means the system architecture should optimize for local participation, intermittent reachability, and radio-driven propagation rather than continuous globally synchronized connectivity.

### Constrained-node architecture

Because the target platform is a microcontroller, the architecture must be lightweight in terms of CPU, memory, power usage, and storage. This constraint affects every subsystem, including networking, persistence, consensus, and cryptographic operations.

### Mesh communication model

The article explicitly states that radio strength should not define the total physical size of the network. This implies a relay-based mesh communication strategy in which nodes forward information across multiple hops. The architecture therefore has to tolerate partial connectivity and unreliable propagation.

### Portability by design

MoonBlokz is not intended to be tied permanently to one hardware platform or one radio technology. Even though LoRa is the initial prototype target, the project should preserve abstraction boundaries that allow later portability to other devices and communication layers.

### Configurable economic and operational behavior

Chain-level configuration is treated as a first-class requirement. This implies that the blockchain should not hard-code all operational assumptions; instead, it should allow deployments to tailor some behavior to specific environments and use cases.

## Assumptions About Hardware and Networking

Part I also lists core assumptions about the capabilities and limits of participating nodes.

### Time

- Nodes can measure elapsed time locally.
- Local timekeeping may be inaccurate.
- Global time is not assumed to exist.

This is a significant architectural constraint because many distributed systems depend on reliable clocks or globally synchronized timestamps.

### Storage

- Each node has some form of persistent storage.
- Storage may be very limited, potentially only a few megabytes.

This means blockchain storage, indexing, and synchronization mechanisms must be efficient and conservative.

### Communication

- Nodes communicate only through radio.
- The design must support low-energy and low-bandwidth radio communication.
- The prototype should work with LoRa.

This assumption strongly differentiates MoonBlokz from internet-native blockchain systems and drives many of the project’s design choices.

### Randomness

- Nodes may or may not have a true random number generator.
- The system should still be able to function without one, although some features — especially onboard key generation — may become limited.

This is a practical embedded-systems constraint and suggests that cryptographic workflows must be designed with degraded hardware capabilities in mind.

## MVP Scope and Future Evolution

The initial product scope is deliberately narrow. The MVP is expected to support currency transactions only. This keeps the first implementation focused while still establishing the core capabilities needed for a broader local blockchain platform.

At the same time, the article makes clear that the architecture should not end there. The blockchain should eventually support additional payload types and more advanced coordination mechanisms, potentially including smart-contract-like behavior or other domain-specific logic.

In other words, MoonBlokz is introduced as a minimal but extensible economic substrate for local embedded networks.

## What Part I Establishes

Part I does not yet define the detailed consensus algorithm, message propagation rules, data structures, or implementation internals. Instead, it establishes the project’s conceptual foundation:

- why MoonBlokz exists,
- what kind of environments it targets,
- what constraints shape the system,
- and what the first implementation is expected to achieve.

This makes the article best understood as the strategic and requirements-level introduction to the MoonBlokz series.

## Source Basis

This document is based on:

- **MoonBlokz series part I. — Building a Hyper-Local DePIN Blockchain** by Peter Sallai, published on Medium on February 18, 2025.
