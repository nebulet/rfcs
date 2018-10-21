- Feature Name: Terminology and Boundary Definition
- Start Date: 2018-10-11
- RFC PR: (leave this empty)
- Nebulet Issue: (leave this empty)

# Summary
[summary]: #summary

The Nebulet operating system, being so different from popular operating systems
of today, will require new terminology to refer to abstraction boundaries.
Terms such as "userspace" and "kernelspace" will carry too much baggage from
existing OSes, which is likely to confuse potential contributors. This RFC
proposes we define the following terms to refer to parts of our OS, adapted
from [EROS](https://web.archive.org/web/20160412201504/http://www.coyotos.org/docs/misc/eros-structure.pdf):
the _kernel_, which implements threading and inter-proces communication (IPC)
as a wasm module, the _nucleus_, which implements a small wasm JIT natively,
and the _firmament_, which specifies IPC interfaces for drivers to implement.
Applications will be implemented atop these abstractions.

# Motivation
[motivation]: #motivation

Today's common operating systems generally use hardware-based isolation
approaches, such as protection rings on x86 or processor modes on ARM, to
protect the kernel's memory from tampering. In colloquial operating systems
discussion, the phrases "kernelspace" and "ring 0" have become synonymous,
as have the phrases "userspace" and "ring 3". As Nebulet uses software-based
isolation and not protection rings, these terms will be unfortunately
overloaded, confusing potential contributors. The world of microkernel-based
research OSes has provided some alternative terminology, some of which is
potentially useful here.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Nebulet operating system architecture is defined in terms of four
components:

- The _kernel_, which implements the core functionality of a microkernel OS:
  thread/process scheduling and isolation, and inter-process communication
  (IPC). These two pieces of functionality will be further specified in future
  RFCs. The kernel is a WebAssembly module that runs atop the nucleus.
- The _nucleus_, which implements a small WebAssembly runtime in native code. A
  nucleus will implement the minimal set of hardware interfaces required by the
  kernel and will likely resemble an exokernel on its surface. A nucleus need not
  be implemented in "ring 0", it could be implemented as a Linux userspace
  program.
- The _firmament_, which specifies interfaces for hardware drivers. Drivers in
  Nebulet, like in any microkernel OS, are just processes, which means that a
  sufficiently flexible operating system will require standardized driver
  interfaces so as to make applications hardware-independent.
- _Applications_ are software compiled to WebAssembly that the user runs.

**Example 1:** a [UTM](https://en.wikipedia.org/wiki/Unified_threat_management)
appliance running on an Atheros SoC based on Nebulet might use:

- A nucleus compiled for ARM, possibly making use of the SoC's secure boot chip
  to implement a tamper-resistant boot.
- The official Nebulet kernel.
- Ethernet drivers that implement the Data Plane firmament specification.
- A cryptographic accelerator driver that uses a custom cryptography chip on
  the SoC, that implements the Cryptography firmament specification.
- A firewall implemented as a Nebulet application that uses the Data Plane
  firmament IPC interfaces.
- A VPN implemented as a Nebulet application that uses both the Cryptography and
  Data Plane firmament IPC interfaces.

**Example 2:** an x86-64 laptop based on Nebulet might use:

- A nucleus compiled for x86-64, with some integration with UEFI Secure Boot.
- The official Nebulet kernel.
- Graphics drivers that implement the GPU firmament specification.
- Wi-Fi drivers that implement the Data Plane firmament specification.
- An implementation of Layer 3+ networking, standardized as the Sockets
  firmament specification.
- A Vulkan library that uses the GPU firmament specification.
- A desktop environment implemented as a Nebulet application that statically
  links the Vulkan library into its wasm module. The desktop environment
  provides an IPC protocol for opening windows and drawing widgets. This
  particular protocol is not part of the firmament, as it is not relevant to a
  hardware driver.
- Firefox, which uses the desktop environment's IPC interface to create a window
  onscreen, statically links to the Vulkan library for rendering, and uses the
  Sockets firmament specification.

**Example 3:** a cluster of [TALOS II 2U servers](https://www.raptorcs.com/content/TL2SV1/intro.html)
being used as a machine learning laboratory might use:

- A nucleus compiled for POWER9, that integrates with OpenBMC.
- The official Nebulet kernel.
- NVIDIA graphics drivers that implement the GPGPU firmament specification.
- Ethernet drivers that implement the Data Plane firmament specification.
- An implementation of the Sockets firmament.
- TensorFlow for clusters ported to a Nebulet application that uses the GPGPU
  and Socket firmaments.
- Jupyter and the entire SciPy stack (lol) ported to Nebulet, using the Sockets
  firmament and many ad-hoc application-layer IPC interfaces.

Note that in all examples, we can re-use the Nebulet kernel and firmament
interfaces.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The nucleus must provide a thin set of abstractions over the hardware, in
particular the CPU. These abstractions must include:
- Access and control over CPU power states, if applicable.
- Access and control over CPU interrupts, if applicable.
- Access and control of the MMU, if applicable.
- Access and control over CPU registers if necessary for common drivers on
  the target hardware.

The nucleus can also implement part of a verified boot chain if such a feature
is desirable. In this case, the nucleus will be verified by the bootloader and
it will verify the Nebulet kernel and other startup data, such as initial
ramdisk or commandline.

The Nebulet kernel will use these functions to implement the core functionality
of threading, scheduling, process isolation, and inter-process communication.

A firmament specification will describe the IPC interfaces and datatypes that
an application using that piece of hardware will interact with. The Data Plane
firmament specification used in the above examples might be defined as follows
(using Rust-like syntax):

```rust
// The supporting structs and enums are assumed to have
// functions that can be called easily from the receiver side.
// We'll need to flesh out the IPC system for this to be in any
// way realistic.
struct MacAddr([u8; 6]);
enum Ethertype {
    Arp,
    Ipv4,
    Ipv6,
}

struct EthernetPacket {
    data: Bytes,
}

impl EthernetPacket {
    pub fn dst(&self) -> &MacAddr;
    pub fn src(&self) -> &MacAddr;
    pub fn ethertype(&self) -> &Ethertype;
    pub fn next_layer(&self) -> &L3Packet;
}

interface EthernetNic {
    fn send(&mut self, pkt: EthernetPacket) -> Result<()>;
    fn send_inplace(&mut self, func: impl Fn(&mut EthernetPacket) -> Result<()>) -> Result<()>;
    fn recv(&mut self) -> Future<Result<EthernetPacket>>;
}
```

An interface is implemented by the driver side and used by the application.
Firmament interfaces could be provided to the application on initialization,
like in [CloudABI](https://cloudabi.org), or they could be dynamically queried,
depending on the security model, which is yet to be specified.

# Drawbacks
[drawbacks]: #drawbacks

Implementing all drivers as WebAssembly applications may not be able to provide
maximum performance along the critical path of an application, in particular,
networking- and GPU-intensive applications may find such performance limits to
be problematic.

Specifying all classes of hardware as firmament drivers will be a tall order,
depending on how wide interest in Nebulet is. The RFC process should allow for
upstreaming the most useful ones in the community, though, as Nebulet becomes
used in domains the original designers did not anticipate.

The terminology chosen is rather alien, and not typically used in the context of
operating systems research and development. This may confuse seasoned operating
systems developers who could be valuable assets to this project.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- **Why is this design the best in the space of possible designs?** This design
  introduces architectural boundaries that are sensible for a unique operating
  system such as Nebulet.
- **What other designs have been considered and what is the rationale for not
  choosing them?**
  - A _monolithic kernel_ design was not chosen because this niche is already
    filled extremely well by existing OSes such as Linux. Running WebAssembly
    in kernel space on Linux is actively being pursued through the
    [WasmJIT](https://github.com/rianhunter/wasmjit) project.
- **What is the impact of not doing this?** Architectural boundaries would be
  allowed to emerge organically. This will likely result in an overspecialized
  operating system based on its initial applications, or a poorly-documented
  set of semi-specified, semi-organic interfaces, both limiting code re-use and
  diluting the value proposition of Nebulet.

# Prior art
[prior-art]: #prior-art

This terminology and boundary specification was mostly inspired by
[EROS](https://web.archive.org/web/20160412201504/http://www.coyotos.org/docs/misc/eros-structure.pdf).
Some other microkernel operating systems, such as [Robigalia](https://robigalia.org),
have chosen to adopt similar terminology.

The concept of an operating system kernel making use of a JIT-compiled environment
dates back to the mainframe days, with IBM mainframes including
hardware-assisted JIT compilation, which enabled fast, architecture-independent
applications. More recently, Microsoft Research has experimented with
Singularity, an operating system using the .NET virtual machine. Likewise, Sun
Microsystems developed a Java-based operating system in the late 90s.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How do we handle cases where pure WebAssembly drivers aren't fast enough?
  - Should we allow native drivers to be loaded into the nucleus?
  - Should we allow native drivers to be statically compiled into the nucleus?
    If so, does the nucleus also implement the firmament interfaces?
  - Can we offer (possibly privileged) nucleus interfaces that mitigate this in
    common cases?
- What should the process of standardizing firmament interfaces look like? Can
  we re-use the RFC process? Can we make templates for firmament RFCs?
- How does this interact with our IPC system and security model? Both must be
  specified independently of this RFC.
- How do applications acquire handles to firmament functions?
