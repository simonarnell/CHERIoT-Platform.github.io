---
layout: post
title:  "Safe and Secure Configuration Management on CHERIoT"
date:   2024-07-30
categories: security philosophy
author: Phil Day
---
All systems rely on some form of configuration data.
Fully immutable systems, where configuration is baked in at build time, may work for container based environments where re-deployment is relatively easy, but in other systems the configuration interface forms a significant part of the attack surface, and misconfiguration, whether accidental or malicious, is a major sources of security vulnerabilities.
For example the recent CrowdStrike outage was caused by a bug in an in-kernel parser that crashed when given an new configuration value.  

The solution in many cases is to create an often complex trust model around and within the configuration interface to limit and control configuration changes, where the complexity itself adds to the threat profile.
The threat profile is further increased when we consider the need to process serialised data which may itself be compromised.

A new contribution to [cheriot-demos](https://github.com/CHERIoT-Platform/cheriot-demos) from [Configured Things](https://www.configuredthings.com/) shows how the CHERIoT features can be used to create a system where serialised configuration data can be received from an external source and securely and safely distributed to a number compartments with a simple and auditable trust model.

Firstly, [static sealed capabilities](https://cheriot.org/book/top-concepts-top.html#sealing_intro) are used to provide unforgeable tokens of authority for operations on configuration data.
Separate sealed capabilities for each configuration item define which compartment is allowed to provide new values and which compartments are allowed to consume them.
Sitting between the provider and consumers is a configuration broker, the only compartment that can unseal the tokens and therefore verify the operations.
This allows the integrity and confidentiality of configuration data flows through the system to be both audited at build time and enforced at run time.

The second major aspect is that sitting alongside the broker are a collection of parser [compartments](https://cheriot.org/book/top-concepts-top.html#_isolating_components_with_threads_and_compartments) whose role is to translate the untrusted serialised data from the providers into trusted values for the consumers.
In a traditional system such parsers are vulnerable to a range of attacks such as injection and buffer overflow.
By isolating each parser into its own sandbox compartment and removing the ability to consume heap space we can contain the scope of such attacks to just the parsing of that value.
The compartment ensures memory violations have no impact on the broker or any other part of the system, and the lack of heap space prevents any temporal impact.
Each parser has its own static sealed capability granting it the right to register with the broker as the parser for a specific item type.
Running as stateless isolated sidecars to the broker ensures the integrity of the parsers; the only interface to a parser is via a sentry guarded callback that it registers with the broker, and registration is based on the content of tokens that only the broker can unseal.
Build time [auditing](https://cheriot.org/book/top-concepts-top.html#_auditing_firmware_images) can ensure that parsers do not call any other compartments.
In the context of the CrowdStrike issue the impact of the parser bug would have been fully contained within the parer compartment; the system wouldn't have been updatable, but it would have remained available.   

This pattern of implementing strongly defined trust boundaries, treating all data as untrusted regardless of its origin, and recognising that the data may be harmful even to try and parse, is aligned with the [principles-based guidance](https://www.ncsc.gov.uk/collection/cross-domain-solutions) published by the National Cyber Security Centre (NCSC) for cross domain operations.
In this case each CHERIoT compartment can be considered to be operating as a separate trust domain.

The demo also makes extensive use of the [permissions](https://cheriot.org/book/top-concepts-top.html#permissions) available on capabilities;  For example the broker removes all permissions except Load(Read) from the pointer to the serialised data it passes to the parser.
Not only does this prevent the parser from unintentionally modifying its input, but it also prevents it from capturing it (no Global permission) or dereferencing it (no LoadStoreCapability).
It also removes all permissions except Store(Write) from the pointer passed to hold the parsed value.
With no persistent heap space allocation available to the parser we can therefore be sure that the parsed value is only derived from the values in the serialised input and constants in the parser’s code.

To ensure that configuration values remain immutable the broker removes the Store(Write) permission from the pointer once the parser has populated the data.
As there is no mechanism in the architecture to add permission back into a capability, there is simply no subsequent access from within the broker or anywhere that can then modify the value.
The broker also removes the LoadStoreCapability permission, so that it is not possible to dereference from the configuration value even if a parser was somehow fooled into including a pointer.

The combination of compartments, sealed capabilities, and fine grained control of  permissions to protect and isolate the parser and its output is something that’s simply not possible on a conventional architecture.