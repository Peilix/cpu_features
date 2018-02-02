# cpu_features [![Build Status](https://travis-ci.org/google/cpu_features.svg?branch=master)](https://travis-ci.org/google/cpu_features) [![Build status](https://ci.appveyor.com/api/projects/status/46d1owsj7n8dsylq/branch/master?svg=true)](https://ci.appveyor.com/project/gchatelet/cpu-features/branch/master)

A cross-platform C library to retrieve CPU features (such as available
instructions) at runtime.

## Design Rationale

-   **Simple to use.** See the snippets below for examples.
-   **Extensible.** Easy to add missing features or architectures.
-   **Compatible with old compilers** and available on many architectures so it
    can be used widely. To ensure that cpu_features works on as many platforms
    as possible, we implemented it in a highly portable version of C: gnu89.
-   **Sandbox-compatible.** The library uses a variety of strategies to cope
    with sandboxed environments or when `cpuid` is unavailable. This is useful
    when running integration tests in hermetic environments.
-   **Thread safe, no memory allocation, and raises no exceptions.**
    cpu_features is suitable for implementing fundamental libc functions like
    `malloc`, `memcpy`, and `memcmp`.
-   **Unit tested.**

### Checking features at runtime

Here's a simple example that executes a codepath if the CPU supports both the
AES and the SSE4.2 instruction sets:

```c
#include "cpuinfo_x86.h"

static const X86Features features = GetX86Info().features;

void Compute(void) {
  if(features.aes && features.sse4_2) {
    // Run optimized code.
  } else {
    // Run standard code.
  }
}
```

### Caching for faster evaluation of complex checks

If you wish, you can read all the features at once into a global variable, and
then query for the specific features you care about. Below, we store all the ARM
features and then check whether AES and NEON are supported.

```c
#include "cpuinfo_arm.h"

static const ArmFeatures features = GetArmInfo().features;
static const bool has_aes_and_neon = features.aes && features.neon;

// use has_aes_and_neon.
```

This is a good approach to take if you're checking for combinations of features
when using a compiler that is slow to extract individual bits from bit-packed
structures.

### Checking compile time flags

The following code determines whether the compiler was told to use the AVX
instruction set (e.g., `g++ -mavx`) and sets `has_avx` accordingly.

```c
#include "cpuinfo_x86.h"

static const X86Features features = GetX86Info().features;
static const bool has_avx = CPU_FEATURES_COMPILED_X86_AVX || features.avx;

// use has_avx.
```

`CPU_FEATURES_COMPILED_X86_AVX` is set to 1 if the compiler was instructed to
use AVX and 0 otherwise, combining compile time and runtime knowledge.

### Rejecting poor hardware implementations based on microarchitecture

On x86, the first incarnation of a feature in a microarchitecture might not be
the most efficient (e.g., AVX on Sandy Bridge). We provide a function to
retrieve the underlying microarchitecture so you can decide whether to use it.

Below, `has_fast_avx` is set to 1 if the CPU supports the AVX instruction
set&mdash;but only if it's not Sandy Bridge.

```c
#include "cpuinfo_x86.h"

static const X86Info info = GetX86Info();
static const X86Microarchitecture uarch = GetX86Microarchitecture(&info);
static const bool has_fast_avx = info.features.avx && uarch != INTEL_SNB;

// use has_fast_avx.
```

This feature is currently available only for x86 microarchitectures.

## What's supported

|                             | x86 | ARM | AArch64 |  MIPS   |  POWER  |
|---------------------------- | :-: | :-: | :-----: | :----:  | :-----: |
|Features revealed from CPU   | yes | no* | no*     | not yet | not yet |
|Features revealed from Linux | no  | yes | yes     | yes     | not yet |
|Microarchitecture detection  | yes | no  | no      | no      | not yet |
|Windows support              | yes | no  | no      | no      | not yet |

-   **Features revealed from CPU.** features are retrieved by using the `cpuid`
    instruction. *Unfortunately this instruction is privileged for some
    architectures, in which case we fall back to Linux.
-   **Features revealed from Linux.** We gather data from several sources
    depending on availability:
    +   from glibc's
        [getauxval](https://www.gnu.org/software/libc/manual/html_node/Auxiliary-Vector.html)
    +   by parsing `/proc/self/auxv`
    +   by parsing `/proc/cpuinfo`
-   **Microarchitecture detection.** On x86 some features are not always
    implemented efficiently in hardware (e.g. AVX on Sandybridge). Exposing the
    microarchitecture allows the client to reject particular microarchitectures.

