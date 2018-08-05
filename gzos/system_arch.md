# GzOS System Architecture

## System Overview

GzOS is a secure operating system that executed in [Trusted Execution Environment(TEE)](https://en.wikipedia.org/wiki/Trusted_execution_environment). Shortly speaking, TEE is a standalone computing environment (so-called **Secure World**) that can be used to execute applications that have higher security requirement. For example, data encryption, Digital Right Management(DRM), face unlock...etc.

TEE can be constructed via the following technology:

* [ARM TrustZone technology](https://www.arm.com/products/security-on-arm/trustzone)
* Virtual machines on top of a [hypervisor](https://en.wikipedia.org/wiki/Hypervisor)
* Passive hypervisor (TODO: describe this in another document)
* Dedicated processor(e.g. [Trusted Platform Module](https://en.wikipedia.org/wiki/Trusted_Platform_Module) and [Qualcomm Secure Processing Unit](https://www.qualcomm.com/snapdragon/security)).

No matter which technology is used, TEE will provide isolated and dedicated physical resources (like CPU time, memory, interrupt controller, I/O ports, timers...etc). Thus, we are able to execute programs, and even host a simple operating system. Host and TEE should be able to communicate with each other, and thus client applications on host can issue command to trusted applications on TEE.

In our platform, we divided communication protocol into 3 different layers, each layer will be described below

![overview](images/system_overview.svg)

### Physical Layer

The layer is responsible for exchanging messages between the two execution environment. The implementation is depended on which underlying technology is used. For example, Trustzone requires a software component called **Secure Monitor** which is responsible for doing world switch, ARM also defined **Secure Monitor Call(SMC)** which is used to send commands to TEE. Hypervisor may need to inject virtual interrupt to notify the virtual machine a new message is arrived. Some SoCs may have dedicated hardware for this purpose. No matter which technology is used, shared memory is usually used for exchanging message.

### Transport Layer

This layer provides bi-directional message queue for efficiently data transfer between **client service** and **agent**. **Virtio queue** is the technology we used, which is widely used in the field of I/O virtualization. We can construct more than one Virtio queue if we have multiple agents in TEE, and the host can use different Virtio queue to communicate with different agents. An agent is a software component which interprets the packets sent from Host client services, and relay the payload to trusted App if necessary. We should create agents for each protocols in Application Layer.

### Application Layer
This layer provides user interface that is required by applications. Each secure OS should define its own API specifications. For example, [Trusty API](https://source.android.com/security/trusty/trusty-ref), [GlobalPlatform(GP) client API](https://globalplatform.org/specs-library/tee-client-api-specification/), [QSEECOM API](https://android.googlesource.com/platform/hardware/qcom/keymaster/+/master/QSEEComAPI.h). We can implement API specifications to support applications developed for a specific secure OS. To implement an API specification, we need to create a pair of **client service** and **agent**. Client service is used to provide the services required by client app. For example, create a logical channel (or session) to a trusted app. The agent should receive message sent from client service, and relay it to corresponding trusted app.

## System Architecture

After briefly describe the layering architecture of GzOS, then I will introduce software components in each layer.

![overview](images/system_architecture.svg)

In our system, we use Linux as our host OS kernel, and [Zircon](https://fuchsia.googlesource.com/zircon) (which is also the kernel of Google Fuchsia) as our TEE kernel. Currently, Trustzone is technology we used to construct TEE, but we have planned to implement Hypervisor solution in the future.

The components with dotted line are not existed currently, but we have planned to implement it. We've implemented an experimental Trusty API specification. The reason to implement Trusty API is to verify our design and also let us get familiar with Fuchsia's programming model. The development of GzOS API specification is under discussion, and we also planned to implement GP API specification.

Since Zircon is microkernel-based operating system, almost all components of our software stack are implemented in userspace. Only small fraction of code that required to access privileged resource (e.g, smc instruction) is kept in kernel. 

Below is the detailed description of each component:

### [Secure Monitor](https://github.com/OpenTrustGroup/zircon/tree/gzos/third_party/lib/sm) and [SMC Kernel Object](https://github.com/OpenTrustGroup/zircon/blob/gzos/kernel/object/smc_dispatcher.cpp)

They are the only 2 components executed in Zircon kernel.

In order ARM Architecture, Trustzone requires a firmware (called Secure Monitor) that performs the world switch between host and TEE. The burden is already offloaded to [ARM Trusted Firmware](https://github.com/OpenTrustGroup/arm-trusted-firmware/tree/gzos) in ARMv8. The Secure Monitor here is just for receiving requests from Host(via smc instruction) and dispatch it to corresponding handler. Secure monitor is also responsible for IRQ routing mechanism; that is, if a host IRQ occurs, we should switch back to host immediately to minimize host IRQ latency. Finally, the module should pre-allocate a host/TEE shared memory and create an interface for host OS to retrieve information of the shared memory.

Zircon is [capability-based](https://en.wikipedia.org/wiki/Capability-based_security) microkernel, which means services provided by the kernel should be presented by a **Kernel Object**. To let userspace process (e.g. smc_service) register a command listener for receiving commands from host OS, we should create a **SMC Kernel Object**.

The SMC Kernel Object provides 2 interfaces (via system call) to userspace:

* Request a command listener for group of SMC command ids
* Get the host/TEE shared memory for data exchanging.

### [Trusty SMC driver](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty.c)

This is a linux module that provides APIs for upper layer software to issue SMC commands to TEE. It also asking Secure Monitor where the shared memory located, mapped it into Linux kernel address space, thus upper layer software can use this buffer to transmit data. The module is also responsible for [IRQ routing mechanism](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty-irq.c); when TEE IRQ occurs it should schedule a worker that switches to TEE for IRQ handling.

### [SMC Service](https://github.com/OpenTrustGroup/garnet/tree/gzos/bin/gzos/smc_service)

The purpose of this module is to receive SMC commands from host OS and leverage the utilities provided by [libtrusty_virtio](https://github.com/OpenTrustGroup/garnet/tree/gzos/lib/gzos/trusty_virtio) to construct logical links between client service in host OS service and agents in GzOS.

It will register SMC command listener to SMC Kernel Object for receiving **trusty virtio** commands. Thus, all SMC commands whose category is **trusty virtio** will be routed to SMC service. There are 5 commands defined for trusty virtio:

#### GetDescriptor

SMC service should allocate list of Trusty Virtio devices at initialization time. Typically, each Trusty Virtio device owns a Virtio queue and provides a logical connection between client service and a agent. we should create Trusty Virtio device for each agent. Each Virtio device has turnable parameters, like queue element size and number of elements. Then, the list of Virtio devices will be encoded into a **resource table**. GetDescriptor command is used to retrieve the resource table. Host OS can use this command to know how many agents are in TEE, and their parameters and protocol type. Then, host OS can allocate corresponding client services.

#### Start

This command is used to activate all Virtio devices, and the Virtio queue of the all Virtio devices will become active. After this command, all client services can send messages to agents. Before issue this command, host OS should scan the resource table, and allocates Virtio ring buffers based on the parameters defined in resource table. Virtio standard requires the host OS preparing TX/RX ring buffers at initialization time, and the buffer should be allocated from the shared memory provided by Secure Monitor. After this command, all Virtio devices will move to ACTIVE state.

#### Stop

Stop all Virtio devices. After the command, all agents cannot receive any messages from client services. And all Virtio devices will move to RESET state

#### Reset

Stop a specific Virtio device. After the command, the corresponding agent cannot receive messages from client service. And, the Virtio device will move to RESET state 

#### Kick VQ

This command is used to notify TEE a message is put into Virtio queue. 

#### Nop

This command is only used by ARM TrustZone (and passive hypervisor). If TrustZone technology is used, the physical CPU will be shared between the host and TEE, and Secure Monitor is responsible for switch among them. The host OS uses this command to switch to TEE. It usually happened when an event is occurred in TEE. i.e. timer interrupt or I/O events. After switching, the CPU will execute TEE OS until it becomes idle (no pending runnable threads).

Host OS should have corresponding [Trusty Virtio driver](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty-virtio.c) that uses commands described above (and the shared memory) to construct logical channel between client services and agents.

### [Tipc Agent](https://github.com/OpenTrustGroup/garnet/blob/gzos/bin/gzos/ree_agent/tipc_agent.cc)

An instance of agent which interprets the Trusty IPC protocol. Currently, this is the only agent implemented in our system for supporting Trusty API. The corresponding client service is [Trusty IPC driver](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty-ipc.c), which provided the mechanisms for client apps connecting to a port published by trusted app, and creating logical channel on the port. The detailed behavior is described in the Trusty API specification.
