# GzOS System Architecture

## System Overview

GzOS is a trusted operating system that executed in [Trusted Execution Environment(TEE)](https://en.wikipedia.org/wiki/Trusted_execution_environment). Shortly speaking, TEE is a standalone computing environment (so-called **Secure World**) that can be used to execute applications that have higher security requirement. For example, data encryption, Digital Right Management(DRM), face unlock...etc.

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
This layer provides user interface that is required by applications. Each trusted OS should define its own API specifications. For example, [Trusty API](https://source.android.com/security/trusty/trusty-ref), [GlobalPlatform(GP) client API](https://globalplatform.org/specs-library/tee-client-api-specification/), [QSEECOM API](https://android.googlesource.com/platform/hardware/qcom/keymaster/+/master/QSEEComAPI.h). We can implement API specifications to support applications developed for a specific trusted OS. To implement an API specification, we need to create a pair of **client service** and **agent**. Client service is used to provide the services required by client app. For example, create a logical channel (or session) to a trusted app. The agent should receive message sent from client service, and relay it to corresponding trusted app.

## System Architecture

After briefly describe the layering architecture of GzOS, then I will introduce software components in each layer.

![overview](images/system_architecture.svg)

In our system, we use Linux as our host OS kernel, and [Zircon](https://fuchsia.googlesource.com/zircon) (which is also the kernel of Google Fuchsia) as our TEE kernel. Currently, Trustzone is the technology we used to construct TEE, but we have planned to implement Hypervisor solution in the future.

The components with dotted line are not existed currently, but we have planned to implement it. We've implemented an experimental Trusty API specification. The reason to implement Trusty API is to verify our design and also let us get familiar with Fuchsia's programming model. The development of GzOS API specification is under discussion, and we also planned to implement GP API specification.

Since Zircon is microkernel-based operating system, almost all components of our software stack are implemented in userspace. Only small fraction of code that required to access privileged resource (e.g, smc instruction) is kept in kernel. 

Below is the detailed description of each component:

### [Secure Monitor](https://github.com/OpenTrustGroup/zircon/tree/gzos/third_party/lib/sm) and [SMC Kernel Object](https://github.com/OpenTrustGroup/zircon/blob/gzos/kernel/object/smc_dispatcher.cpp)

They are the only 2 components executed in Zircon kernel.

In older ARM Architecture, Trustzone requires a firmware (called Secure Monitor) that performs world switch between host and TEE. The burden is already offloaded to [ARM Trusted Firmware](https://github.com/OpenTrustGroup/arm-trusted-firmware/tree/gzos) in ARMv8. The Secure Monitor here is just for receiving requests from Host(via smc instruction) and dispatch it to corresponding SMC handler. Secure monitor is also responsible for IRQ routing mechanism; that is, if a host IRQ occurs, we should switch back to host immediately to minimize host IRQ latency. Finally, the module should pre-allocate a host/TEE shared memory and create an interface for host OS to retrieve information of the shared memory.

Zircon is [capability-based](https://en.wikipedia.org/wiki/Capability-based_security) microkernel, which means creating a new **Kernel Object** is the  only way to expose service. To let userspace process (e.g. smc_service) register a command listener for receiving commands from host OS, we should create a **SMC Kernel Object**.

The SMC Kernel Object provides 2 interfaces (via system call) to userspace:

* Register a SMC command listener for a group of SMC commands.
* Get the host/TEE shared memory for data exchanging.

### [Trusty SMC driver](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty.c)

A linux module that provides API for upper layer software to issue SMC commands to TEE. It also got shared memory address from Secure Monitor, mapped it into Linux kernel's address space. Upper layer software can use the shared buffer to transmit/receive data. The module is also responsible for [IRQ routing mechanism](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty-irq.c); when TEE IRQ occurs it should schedule a worker that switches to TEE for IRQ handling.

### [smc_service](https://github.com/OpenTrustGroup/garnet/tree/gzos/bin/gzos/smc_service)

Purpose of this module is receiving SMC commands from host OS and leverage the utilities provided by [libtrusty_virtio](https://github.com/OpenTrustGroup/garnet/tree/gzos/lib/gzos/trusty_virtio) to construct logical links between client services in host OS and agents in GzOS.

It will register SMC command listener to SMC Kernel Object for receiving **Trusty Virtio** commands. Thus, all SMC commands whose category is Trusty Virtio will be routed to smc_service. Following is Trusty Virtio commands:

###### GetDescriptor

SMC service should allocate Trusty Virtio devices at initialization time. Typically, each Trusty Virtio device owns a Virtio queue and provides a logical connection between client service and an agent. We should create Trusty Virtio device for each agent. Each Virtio device has turnable parameters, like Virtio queue element size and number of elements. Then, the list of Virtio devices will be encoded into a **resource table**. GetDescriptor command is used to retrieve the resource table. Host OS can use this command to know how many agents are in TEE, and their parameters and protocol type as well. Then, host OS can create corresponding client services.

###### Start

This command is used to activate all Virtio devices, and Virtio queue of all Virtio devices will become active. After this command, all client services can send messages to agents. Before issue this command, host OS should scan the resource table, and allocates Virtio ring buffers based on the parameters defined in resource table. Virtio standard requires the host OS preparing TX/RX rings at initialization time, and the buffer should be allocated from the shared memory provided by Secure Monitor. After this command, all Virtio devices will move to ACTIVE state.

###### Stop

Stop all Virtio devices. After the command, all agents cannot receive any messages from client services. And all Virtio devices will move to RESET state

###### Reset

Stop a specific Virtio device. After the command, the corresponding agent cannot receive messages from client service. And, the Virtio device will move to RESET state 

###### Kick VQ

This command is used to notify TEE a message is put into Virtio queue. 

###### Nop

This command is only used by ARM TrustZone (and passive hypervisor). If TrustZone technology is used, the physical CPU is shared between the host and TEE, and Secure Monitor is responsible for switching among them. The host OS uses this command to switch to TEE. It usually happened when an event is occurred in TEE. i.e. timer interrupt or I/O events. After switching, the CPU will execute TEE OS until it becomes idle (no pending runnable threads).

Host OS has a corresponding [Trusty Virtio driver](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty-virtio.c) that issues commands described above to construct logical channel between client services and agents.

### [Tipc Agent](https://github.com/OpenTrustGroup/garnet/blob/gzos/bin/gzos/ree_agent/tipc_agent.cc)

An instance of agent which interprets the Trusty IPC protocol. Currently, this is the only agent implemented in our system. The corresponding client service is [Trusty IPC driver](https://github.com/OpenTrustGroup/linux/blob/gzos/drivers/trusty/trusty-ipc.c), which provided interface for client apps connecting to a **TipcPort** published by trusted app, and creating logical channel on the port. The detailed behavior is described in the [Trusty API specification](https://source.android.com/security/trusty/trusty-ref).

## Trusty API Implementation

Before discussing the detailed implementation of Trusty API, we need to briefly introduce Fuchsia's application model. Most applications in Fuchsia are managed by [Application Manager (appmgr)](https://github.com/OpenTrustGroup/garnet/tree/master/bin/appmgr). Applications are placed in a container, called **Environment**. Environments are tree structured, and the environment where appmgr resident in is called root Environment. User can use the [API](https://github.com/OpenTrustGroup/garnet/tree/gzos/public/lib/app) provided by appmgr to create new Environment and launch new processes in it.

There are two methods to publish service in Fuchsia:

1. Parent Environment can provide services to its children. This is called **Environment services**. Processes run in an Environment can request its Environment services.

2. Process itself can provide services to its parent. The parent should create a pair of [Zircon channel](https://fuchsia.googlesource.com/zircon/+/master/docs/objects/channel.md), and pass one end to the process while launching it. Then, the parent can use the other end of the channel to request services published by the process. Please note only parent process who holds the other end of the channel can access services published by the process. Parent process can then send the channel to the other process, let it able to access services.

After briefly introduce the Fuchsia service mode, let's go through each software components.

![overview](images/tipc_agent.svg)

### [Sysmgr](https://github.com/OpenTrustGroup/garnet/tree/gzos/bin/gzos/sysmgr)

Appmgr will create a new child Environment called **app**, and provides loader and launcher service to the Environment. The launcher service is used to create new process, and the loader service is used to load binary. It also launches **sysmgr** which is responsible for managing all services in the system.

To publish a new service, developer can add an entry in **services.config** which contains service name and the url of the binary which provided the service.  Sysmgr will automatically launch the process when corresponding service is requested. The config file looks like below:

```
{
  "services": {
    "chromium.web.ContextProvider": "chromium",
    "fuchsia.bluetooth.control.Control": "bt-gap",
    "fuchsia.bluetooth.gatt.Server":  "bt-gap",
    "fuchsia.bluetooth.le.Central":  "bt-gap",
...
```

The services are statically configured, which means after the system boots, we cannot add new services dynamically. The static service model is not suitable for us, since Trusty API didn't define any restrictions on a trusted app to publish services. As long as the service name is not conflicted, the app can publish as many services as it wants (until kernel memory exhausted; this is not secure indeed). Thus, we created another [service model](https://github.com/OpenTrustGroup/garnet/blob/gzos/bin/gzos/sysmgr/dynamic_service_app.cc) for Trusty app. Please note we implement this model since we need to be faithful to Trusty API (and pass Trusty IPC unit-test). But, we realized it's not secure, too. The model may be dropped in the future.

Instead of using a global **services.config** for service configuration, we config services on a per-process basis. Each trusted app in GzOS should provide a file called **manifest.json**, here is an [example](https://github.com/OpenTrustGroup/garnet/blob/gzos/bin/gzos/trusty_app/ipc-unittest/srv/manifest.json). Each manifest file should define an array called **public_services**. Sysmgr will launch the app when one of it's public services is requested.

Sysmgr will scan all app's public services at initialization time and create proxy services for them. The proxy service will re-direct the service request to corresponding trusted apps (launch it if it's not launched yet). Then, appmgr creates a new Environment called **trusty**, and provides proxy services to it. Such that a trusted app created in trusty Environment can request public services defined by other trusted apps. Although Tipc agent is not managed by appmgr; but appmgr provides a mechanism for outside processes to access proxy services in sysmgr. Thus, client apps can also access (via Tipc agent) services defined by trusty apps.

In order to let trusted apps publish services that is not defined in public service array (This is allowed since Trusty API doesn't set restriction on it). We provide additional **[Service Registry interface](https://github.com/OpenTrustGroup/garnet/blob/gzos/public/lib/gzos/sysmgr/fidl/service_registry.fidl)** to each trusted app. With the Service Registry interface, trusted app can register its local service, and even wait on a specific service to be published by other trusted apps.

### [libtrusty_ipc](https://github.com/OpenTrustGroup/garnet/tree/gzos/public/lib/gzos/trusty_ipc)

This module provides basic building blocks that can be used to implement Trusty API. It provides IPC mechanism for trusted apps to talk with each other (Sounds weird, we built an IPC mechanism on top of Zircon's IPC mechanism. But, we didn't find other approaches to implement Trusty API yet). Both Tipc agent and Trusty apps should use this library to talk. The **Service Registry** provided by sysmgr is required to publish **TipcPort**. Please refer to [Trusty API specification]((https://source.android.com/security/trusty/trusty-ref)) for more information.

### [libtrusty_app](https://github.com/OpenTrustGroup/garnet/tree/gzos/public/lib/gzos/trusty_app)

This module implements Trusty API based the basic building blocks provided by libtrusty_ipc.