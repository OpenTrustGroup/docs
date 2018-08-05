# GzOS System Architecture

## System Overview

GzOS is a secure operating system that executed in [Trusted Execution Environment(TEE)](https://en.wikipedia.org/wiki/Trusted_execution_environment). Shortly speaking, TEE is a standalone computing environment(so-called **Secure World**) that can be used to execute applications that have higher security requirement. For example, data encryption, Digital Right Management(DRM), face unlock...etc.

TEE can be constructed via the following technology:

* [ARM TrustZone technology](https://www.arm.com/products/security-on-arm/trustzone)
* Virtual machines on top of a [hypervisor](https://en.wikipedia.org/wiki/Hypervisor)
* Dedicated processor([Trusted Platform Module](https://en.wikipedia.org/wiki/Trusted_Platform_Module) and [Qualcomm Secure Processing Unit](https://www.qualcomm.com/snapdragon/security)).

No matter which technology is used, TEE will provide isolated and dedicated physical resources (like CPU time, memory, interrupt controller, I/O ports, timers...etc). Thus, we are able to execute programs, and even host a simple operating system. Host and TEE should be able to communicate with each other, and thus client applications on host can issue command to trusted applications on TEE.

In our platform, we divided communication protocol into 3 different layers, each layer will be described below

![overview](images/system_overview.png)

### Physical Layer

The layer is responsible for exchanging messages between the two execution environment. The implementation is depended on which underlying technology is used. For example, Trustzone requires a software component called **Secure Monitor** which is responsible for doing world switch, ARM also defined **Secure Monitor Call(SMC)** which is used to send command to Secure Monitor for world switching. Hypervisor may need to inject virtual interrupt to notify the virtual machine a new message is arrived. Some SoCs may have dedicated hardware for this purpose. No matter which technology is used, shared memory is usually used for exchanging message.

### Transport Layer

This layer provides bi-directional message queue for efficiently data transfer between **client service** and **agent**. **Virtio queue** is the technology we used, which is widely used in the field of I/O virtualization. We can construct more than one Virtio queue if we have multiple agents in TEE, and the host can use different Virtio queue to communicate with different agents. An agent is a software component which interprets the packets sent from Host client services, and relay the payload to trusted App if necessary. We should create agents for each protocols in Application Layer.

### Application Layer
This layer provides user interface that is required by applications. Each secure OS should define its own API specifications. For example, [Trusty API](https://source.android.com/security/trusty/trusty-ref), [GlobalPlatform(GP) client API](https://globalplatform.org/specs-library/tee-client-api-specification/), [QSEECOM API](https://android.googlesource.com/platform/hardware/qcom/keymaster/+/master/QSEEComAPI.h). We can implement API specifications to support applications developed for a specific secure OS. To implement an API specification, we need to create a pair of **client service** and **agent**. Client service is used to provide the services required by client app. For example, create a logical channel (or session) to a trusted app. The agent should receive message sent from client service, and relay it to corresponding trusted app.

