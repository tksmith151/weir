# WEIR Design Doc: A Portable Core IR

Oct 30, 2026 · @Trevor

This doc proposes WEIR, a minimal, general-purpose intermediate representation (IR) for any program, from applications and compute kernels to a whole operating system, that maps onto CPUs, GPUs, FPGAs and ASICs through a system description. Its core is a frozen compatibility contract: any system that supports the core runs any core program, and everything above the core is optimization.

The name is a working name. A weir is a low dam that lets water step down from one level to the next; programs in WEIR lower level by level until every op reaches the core or runs natively on some device.

## Background and goals

The project started as a minimal, architecture-independent language capable of building an OS. Three decisions shaped it. First, the IR comes before the language, so a minimal core is what we design first. Second, the IR specifies a program for a *system*, not a single chip. Third, the IR is generic: it must be able to describe an OS, which sets the bar for low-level capability, but it must also program anything else reasonably well.

The hard part is not the OS-language half, since C-level systems features are well understood. It is mapping one program onto CPUs, GPUs, FPGAs and ASICs, which have different execution models, not just different instruction sets.

**Goals**

- An extremely minimal core from which higher-level semantics are built.
- Every higher-level construct has at least one guaranteed path down to the core.
- Each backend chooses which semantic level it understands natively; the rest is lowered first.
- Any system that supports the core runs any core program with no target-specific code.
- Hardware can target the minimal core and stay compatible with past software; everything above is optimization.
- General-purpose: applications, libraries, compute kernels and hardware pipelines are all first-class; an OS is the most demanding program the IR must express, not its only purpose.
- An OS can be written portably on top of the core.

**Non-goals**

- A surface programming language. That comes later and emits this IR.
- Guaranteeing that every program *succeeds* on every system. Compatibility promises correct behavior or a defined resource trap, not unlimited resources.

## Requirements

The IR must meet 21 requirements in four groups. The OS group sets the floor of low-level capability; the generality group keeps the IR from becoming OS-specific.

| ID | Requirement | Why |
| --- | --- | --- |
| **OS capability (the floor)** |  |  |
| R1 | Raw memory access: addresses, pointer↔integer conversion, pointer arithmetic | Kernels manage memory directly |
| R2 | Explicit data layout: size, alignment, padding, endianness, bitfields | Page tables, descriptors and device registers have fixed binary formats |
| R3 | Address spaces as part of the type (`mmio`, `phys`, `shared`, `local`, …) | GPUs and FPGAs have many address spaces; C has none |
| R4 | Volatile, side-effecting access the compiler cannot elide or reorder | Device I/O |
| R5 | Atomics, fences and a formally specified memory model | Kernel concurrency is otherwise undefined |
| R6 | No mandatory runtime: no GC, implicit heap, required unwinding or hidden allocation | Must run before anything else exists, including a stack |
| R7 | Control of stack switching, entry points and calling conventions | Boot entry, interrupt handlers, context switch |
| R8 | Arbitrary-width integers, `bits<N>` for any N | Free on CPUs; essential for FPGA and ASIC area |
| **Layered IR** |  |  |
| R9 | Extremely minimal core | Small enough to freeze and to implement on any system |
| R10 | Every higher-level op has a guaranteed, terminating lowering to the core | Any program can reach the core |
| R11 | Backends choose which level they support natively; only illegal ops are lowered | Lowering loses information, so targets get the highest form they understand |
| R12 | The core must not assume sequential execution | Hardware backends must not have to rediscover parallelism |
| **System compatibility** |  |  |
| R13 | Any system that supports the core runs any core program, with no target-specific code | The core is a compatibility contract |
| R14 | Support is judged per system, not per chip; a device may implement less than the core | Systems fill gaps by lowering, emulation or placement |
| R15 | An OS can be written without target-specific escape hatches | Otherwise the OS is the layer that breaks compatibility |
| R16 | Resource exhaustion is defined behavior | Every real system is finite |
| R17 | Observable behavior is precisely defined; timing is excluded | Compatibility needs a definition of "same result" |
| R18 | Old programs run on new systems, and new programs run (slower) on old ones | Hardware evolves under a stable software base |
| **Generality** |  |  |
| R19 | Any program is expressible with reasonable efficiency: applications, libraries, compute kernels, hardware pipelines | The IR is generic; the OS is a capability bar, not the purpose |
| R20 | Nothing OS-specific is mandatory; programs that don't use the system model pay nothing for it | Keeps ordinary code lean |
| R21 | Higher-level language features (floating point, closures, exceptions, garbage collection) are expressible as extensions or libraries | Lets the IR host many source languages |

## Architecture overview

The design has four layers between a program and the hardware. Programs use core ops plus extensions; extensions are lowered only where a device lacks them; the system mapper routes every remaining op to some device.

&#91;embedded content: architecture · program to devices, 4 layers\]

The core (highlighted) is the only fixed layer. Extensions, system descriptions and devices can all change without breaking existing programs.

## Core IR

The core is a dataflow graph with structured regions and explicit state edges, close to an RVSDG (Regionalized Value State Dependence Graph). It contains only what cannot be defined by lowering, plus enough computation to express everything else (R9, R12).

**Types.** `bits<N>` only. Signedness lives in ops (`slt` vs `ult`), not types. Integers, floats, pointers, structs and vectors are extensions over bits (R8).

**Pure computation.** `and`, `or`, `xor`, `not`, `add`, left and right shifts, `eq`, `ult`, `concat`, `extract`, `select`. A smaller set (NAND plus registers) would suffice in principle, but expansions would blow up. Multiply, divide and floating point are extensions.

**Memory and effects** (none of these can be lowered):

- `load` and `store`, parameterized by width and address space (R1, R3).
- One atomic primitive, compare-and-swap, and `fence` (R5). Atomicity cannot be built from plain loads and stores.
- **State tokens** threaded through every effectful op. Ordering is ordinary dataflow, not implicit program order (R4, R12).

**Structured control.**

- **gamma:** choose one of N regions.
- **theta:** a tail-controlled loop region.
- **lambda / apply:** function definition and call.

**Parallelism.** A region whose iterations are declared independent. It lowers trivially to a sequential `theta`, since running independent iterations in order is a valid schedule, but it tells the system it may spread the work across devices. Blocking synchronization belongs to the system model, not here.

**Escape hatch.** `target_op "name"(…)` with declared effects, legal only where the system defines it. A program that uses it is **not portable**; portable programs, including the OS, use the abstract system model instead (R15).

## Abstract system model

The core includes an abstract model of what an OS manages, so an OS needs no `target_op` (R15). Each system maps this model onto its real mechanisms. This is the part that lets an OS outlive its hardware, and the hardest part to design.

Ordinary programs never need to touch this model. An application gets services from its OS, and a compute kernel uses only core ops and extensions. Only code that manages execution, isolation or devices uses it directly (R20).

- **Execution contexts:** create, suspend and resume a unit of execution, with its state as an opaque value. This replaces "save registers, switch stack pointer," which is machine-specific (R7).
- **Isolation:** one context restricts what another can access. A capability model like CHERI is preferred over page tables, which bake in a granularity and a translation structure.
- **Events:** interrupts and traps delivered as a defined transfer into a handler context, including faults the program causes (bad access, resource exhaustion).
- **Devices:** a discoverable description of the devices present, accessed through typed address spaces (R3, R4).
- **Time:** a monotonic clock and timer events, for scheduling.

## Extensions and definitional lowering

Every op above the core is a signature plus a **definitional lowering**: an expansion into ops strictly closer to the core. The expansion *is* the op's semantics; there is no separate specification (R10).

- **Termination.** The "lowers-to" relation between op sets must be acyclic, so lowering always reaches the core.
- **Reference semantics.** A backend that claims native support for an op is checked against its expansion, by differential testing, translation validation or proof.
- **Alternative lowerings.** An op may have several (multiply as shift-add, a loop or a table). Only the definitional one is required; the others must be equivalent to it.
- **Lazy, per-op lowering.** The compiler lowers only the ops a target declares illegal, to a fixpoint. A GPU keeps `vector.add`; a scalar CPU expands it (R11).
- **Type legalization is lowering too.** `bits<13>` on a 32-bit CPU widens to `bits<32>` plus masking; `bits<128>` splits into pairs.
- **Lowerings are data.** They are written as patterns in the IR itself, not compiler code, so semantics are inspectable and ops can be added without changing the compiler.

First extensions to build: `int<N>` arithmetic, `struct` with explicit layout (R2), and `vector<N,T>`.

**Language features as extensions.** Floating point, closures, exceptions and garbage collection are extensions or libraries, not core features. The IR can host many source languages without the core paying for any of them (R21).

## System description, legality and mapping

A system is a set of devices plus the interconnect and shared memory between them. Each device declares what it supports; the system as a whole must support the core (R13, R14).

**Device description**

| Section | Describes |
| --- | --- |
| Types | Native widths, floating-point formats, vector and lane widths |
| Operations | Supported ops, with latency, throughput, and area or energy cost |
| Storage | Registers, memory hierarchy, address spaces, coherence |
| Execution model | Sequential thread, SIMT warps, spatial dataflow or VLIW |
| Concurrency | Supported atomics, ordering guarantees, barriers |
| Control | How the abstract system model's contexts, events and isolation map onto the hardware |
| ABI | Calling convention, stack layout, how call and return work |
| Target ops | Named machine operations for non-portable code |

**Legality** is a predicate over ops *and* their parameters, so even core ops can be restricted on a device:

```
device fpga_small {
  legal: bits<N> for N <= 64
         add, xor, …, select
         load/store in addrspace(bram), addrspace(axi)
         theta  where trip_count is static or bounded
         apply  where callgraph is acyclic
  illegal: addrspace(generic), cas
}
```

**Mapping.** For an op a device can't run, the system picks one of four routes:

1. **Lower** it through its definitional expansion.
2. **Emulate** it in software on the same device, such as CAS via a lock in a memory controller that supports atomics.
3. **Place** that region on another device, such as handing it from an FPGA pipeline to a CPU.
4. **Trap and emulate** at run time.

"Supports the core" is checkable: every core op is reachable on the system through at least one route. Code with no route on any device is a compile error naming the op and the violated constraint.

## Semantics

These decisions are expensive to change later, so they are fixed now.

- **Memory model.** Formally specified, for example a simplified C11/RC11 model, or sequential consistency by default with explicit weaker orderings (R5).
- **No undefined behavior in the core.** `add` wraps, out-of-range shifts have defined results, and the core has no division. An extension may add an "assume no overflow" flag; its lowering just drops the flag.
- **Address spaces are first-class from day one,** with explicit casts between them (R3).
- **Finite resources.** Running out of stack, heap or any other resource is a trap delivered through the event model. A program is compatible with every core system, though not guaranteed to succeed on all of them (R16).
- **Observable behavior** is the values written to memory and devices, and the order in which effects become visible, within the memory model (R17).
- **Timing is outside the semantics,** so systems may optimize freely. Real-time needs are constraints or hints a system may accept or reject; they never change a program's meaning.

## Compatibility and versioning

The core is frozen, and every future system must implement it. An op belongs in the core only if both are true:

1. It **can't be defined** as a lowering of other core ops (atomicity, memory access, contexts, events).
2. **Every system can provide it** in some form: native support, emulation or placement.

Everything else is an extension, however fundamental it seems.

**Programs carry their extension definitions.** A module ships the definitional lowerings of the extensions it uses. Old programs run on new systems because new systems support the core; new programs run on old systems, more slowly, by expanding the bundled definitions (R18). Hardware vendors compete on which extensions they accelerate natively.

**Versioning.** The core carries a version number from the start, so a core revision can be added without breaking existing programs.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| Design a full language first | The IR is the portability layer; a language can come later and target it |
| Sequential CFG core with basic blocks (LLVM style) | Every hardware backend would have to rediscover parallelism. Dataflow serializes easily; serial code does not parallelize cheaply (R12) |
| Core that every *device* must fully support | Excludes FPGAs and accelerators that can't do recursion or atomics; support per system avoids this (R14) |
| OS with a per-system HAL of `target_op`s | How OSes work today, and it breaks compatibility at the most important layer (R15) |
| Page tables as the isolation model | Bakes in a page granularity and a translation structure; capabilities only describe rights over regions |
| Lowering rules written in compiler code | Semantics become opaque; ops can't be added without changing the compiler |
| Undefined behavior and poison in the core | A major source of complexity and miscompiles in LLVM; a total core is simpler to verify |

## Prior art

| Area | Look at |
| --- | --- |
| System compatibility contract | IBM AS/400 TIMI (now IBM i), NVIDIA PTX, WebAssembly |
| Dataflow core with state edges | RVSDG (the jlm compiler), Sea of Nodes |
| Dialects, legality, conversion | MLIR's dialect conversion framework |
| Bit-level hardware core | CIRCT's `comb`, `seq` and `hw` dialects |
| Declarative lowering rules | Cranelift's ISLE |
| Capability declarations | SPIR-V |
| Machine descriptions | LLVM TableGen, GCC `.md` files, Sail and ARM ASL ISA specs |
| Formal semantics, checking lowerings | Sail, the K framework, Alive2 |
| Capability-based isolation | CHERI |
| OS-language surface with no runtime | Zig, Rust `no_std`; Forth and BCPL for minimal bootstrapping |

AS/400 is the closest precedent: programs compiled to TIMI kept running unchanged when the hardware moved from a custom CISC design to POWER.

## Open questions and first milestones

**Open questions**

- What exact op set and types does the abstract system model need? Can contexts, events and isolation be mapped efficiently onto both a conventional MMU-based CPU and a CHERI-style one?
- Which memory model: simplified RC11, or sequential consistency with explicit weaker orderings?
- Is the parallel region enough for GPUs, or does the core need a notion of groups and barriers?
- How are real-time constraints expressed and checked, given timing is outside the semantics?
- What is the format for bundled extension definitions, and how are they trusted or verified on load?
- Should the IR be built as MLIR dialects to reuse existing CPU, GPU and HLS backends?

**First milestones**

- [ ] Specify the core: types, ops, effects, structured control and the parallel region.
- [ ] Draft the abstract system model as concrete core ops with types and semantics.
- [ ] Define three extensions with definitional lowerings: `int<N>`, `struct`, `vector<N,T>`.
- [ ] Build one CPU backend and one HLS backend from the same IR.
- [ ] Boot a minimal kernel (contexts, events, one device) on two different systems with no target-specific code.
