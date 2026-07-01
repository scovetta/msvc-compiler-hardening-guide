# Compiler Options Hardening Guide for C and C++ — MSVC

*A Microsoft contribution to the [OpenSSF Best Practices Working Group](https://best.openssf.org).*

*2026-07-01 (draft)*

This guide describes hardening options for the **Microsoft Visual C/C++
compiler (MSVC)** — that is, `cl.exe` and `link.exe` as shipped with
Visual Studio. It is the MSVC companion to the OpenSSF
[Compiler Options Hardening Guide for C and C++][openssf-gcc-clang],
which covers GCC and Clang. Both guides are driven by the same security
considerations and share the same structure.

[openssf-gcc-clang]: https://best.openssf.org/Compiler-Hardening-Guides/Compiler-Options-Hardening-Guide-for-C-and-C++

The Microsoft Learn article [*Build reliable and secure C++
programs*][ms-learn-reliable-secure] gives a broad
[NISTIR 8397][nistir-8397]-aligned view of the development lifecycle —
threat modeling, automated testing, dependency hygiene, and the SDL
practices that surround the compiler. The present guide is the
compiler-options companion to that lifecycle picture.

[ms-learn-reliable-secure]: https://learn.microsoft.com/cpp/code-quality/build-reliable-secure-programs
[nistir-8397]: https://nvlpubs.nist.gov/nistpubs/ir/2021/NIST.IR.8397.pdf

This guidance targets the MSVC toolset that ships with Visual Studio
2026 (version 18.0). If you encounter a `D9002` (compiler) or
`LNK4044` (linker) "unrecognized option" warning, the option is not
present in your toolset:

```text
cl : Command line warning D9002 : ignoring unknown option '<opt>'
LINK : warning LNK4044: unrecognized option '<opt>'; ignored
```

Consult [Appendix C](#appendix-c) for the earliest Visual Studio
version that shipped each option.

## Who should use this document?

This document is written for **C and C++ developers** who build production
software for Windows or for Microsoft-supported cross-platform targets
(Xbox, Android, Linux) with MSVC, and who want their binaries to
benefit from the compiler's and the operating system's defense-in-depth
mitigations. It assumes basic familiarity with the MSVC command line
(`cl.exe`, `link.exe`), with MSBuild or CMake-based projects, and with
C/C++ as a language. It does not assume any prior security expertise.

## How to use this guide

This guide concentrates on the **flags, warnings, and link-time options
most closely associated with compiler settings**. The recommendations
group into five progressive stages. Adopt them in order; each one is
independently valuable, and later stages assume the earlier ones are in
place.

1. **Harden your compile and link settings.** Establish a baseline of
   compiler and linker flags that turn on the platform's defense-in-depth
   mitigations and reject undefined or ambiguous constructs at build time.
   Enable a meaningful warning level, opt in to specific off-by-default
   warnings that catch real defects, and put discipline around the
   exceptions (suppressions, third-party headers, legacy code). → §7
   (per-option reference), §8 (warnings management).
2. **Produce and publish symbols.** Generate and publish debug
   information for every shipping binary so post-ship incidents
   can be triaged efficiently. → §9.
3. **Expand correctness with static analysis.** Add MSVC's built-in
   code analyzer (`/analyze`) to your continuous integration or
   scheduled builds. It catches a class of defects — uninitialized
   memory, lifetime errors, SAL contract violations — that ordinary
   compiler warnings don't surface. → §10.
4. **Verify with dynamic analysis.** Build separate test binaries with
   AddressSanitizer (and related sanitizers when applicable) and run
   your test corpus against them to surface runtime issues — heap and
   stack overflows, use-after-free, double-free — within the domain
   that the compiler and static analyzer address but cannot prove.
   → §11.
5. **Verify the shipping binary.** Run a post-build check such as
   [BinSkim](https://github.com/microsoft/binskim) against your shipped
   binaries to confirm the flags from stage 1 landed. Flags can be
   silently dropped by linker order, by a static library that defaulted
   differently, or by a project-page toggle no one noticed. See
   [*Enable compiler errors*](#enable-compiler-errors) in §5 for a
   recommended workflow.

Stages 1–2 belong in every build. Stage 3 belongs in CI. Stage 4
belongs in test pipelines. Stage 5 belongs in the release pipeline.

## TL;DR: what compiler options should I use?

In a typical MSVC build that uses `cl.exe` as the compiler driver and
`link.exe` as the linker, **add the options below to your existing build
command line.** These are the hardening settings MSVC does *not* enable
by default. The mitigations that are *already* on by default — `/GS`,
`/DYNAMICBASE`, `/NXCOMPAT`, and (on 64-bit) `/HIGHENTROPYVA` — are
listed separately at the end of this section; you verify those rather
than add them. None of the options below changes program behavior in a
way that breaks a well-formed C or C++ program.

For all C and C++ code, **compile with**:

```text
/W4 /WX /sdl
/w14388  (elevate the off-by-default signed/unsigned comparison warning; see §8)
/guard:cf /guard:ehcont /Qspectre
/Zi /ZH:SHA_256  (/Zi emits the PDB; /ZH:SHA_256 is the default source hash since VS2022 — pass it explicitly on older toolsets)
/external:I path/to/third_party_headers /external:W0 /external:templates-
/std:c++20   (or /std:c++latest in greenfield code; /std:c++20+ implies /permissive-)

```

For all C and C++ code, **link with**:

```text
/guard:cf /guard:ehcont
/CETCOMPAT
/SOURCELINK:sourcelink.json    (see §9 for how to author sourcelink.json)

```

For **32-bit (x86) executables** also link with:

```text
/LARGEADDRESSAWARE /SAFESEH

```

(`/LARGEADDRESSAWARE` is what gives x86 binaries access to >2 GB of
virtual address space, which is what makes `/DYNAMICBASE` ASLR entropy
meaningful on 32-bit. `/SAFESEH` is the x86-only safe-SEH handler-table
mechanism.)

**Already on by default — verify, don't add.** `/GS`, `/DYNAMICBASE`,
`/NXCOMPAT`, and (on 64-bit) `/HIGHENTROPYVA` are enabled by default in
current MSVC and link.exe. Confirm they reached the shipped binary with
[BinSkim](https://github.com/microsoft/binskim) rather than adding them
to your build, and pass them explicitly only when you target older
toolsets whose defaults were weaker. See [Background](#background) and
[Appendix C](#appendix-c).

The remainder of this matrix splits along two axes: the
**role** of the code (whether the binary ships to customers or
supports development), and the **build flavor** (a release/optimized
build versus a debug or sanitizer-instrumented build of the same
sources). For consistency, the rest of this guide uses *shipping
code* and *release build* in the strict senses defined here.

For **shipping code in a release/optimized build** (any binary
delivered to customers — a product, a shared library, an installer's
support component) compile with the optimizer enabled and link the
appropriate C/C++ runtime, and emit a source-link map so post-ship
incidents can be debugged (see
[§9 Maintaining debug information](#maintaining-debug-information)):

```text
/O2                              (compile — enable optimizations)
/MD                              (compile — link the dynamic CRT; use /MT instead for a self-contained binary)
/SOURCELINK:sourcelink.json      (link — see §9 to author the JSON)

```

For **shipping code in a sanitizer-instrumented build** — typically
a separate CI configuration that compiles the *same product sources*
with extra instrumentation and runs them against the team's existing
tests — also build with:

```text
/fsanitize=address /Zi

```

so that AddressSanitizer can catch memory-safety bugs in the product
code as the tests drive it.

Each of these options is described in more detail in
[§7 Recommended compiler options](#per-option-detail) below.

## Background

### Why do we need binary hardening?

C and C++ continue to be the languages of choice for performance-sensitive
software, operating-system kernels, embedded systems, and large parts of
the application stack on every operating system in production today. The
two languages also remain the dominant source of memory-safety
vulnerabilities in shipped software, accounting for many
critical CVEs in widely-used components year after year.

The C++ language and its standard library have introduced many features
designed to reduce the incidence of these defects — `std::string_view`,
`std::span`, `std::optional`, smart pointers, ranges, `constexpr`,
`[[nodiscard]]`, `std::cmp_less` and the rest of the `<utility>` integer
comparison helpers — and modern code that uses these features
consistently is meaningfully safer than the C-style code that preceded
it. Compilers and operating systems have likewise added a deep stack of
*defense-in-depth* mitigations: stack-buffer-overflow checks, control-flow
integrity, data-execution prevention, address-space layout randomization,
shadow stacks, speculation barriers. Many of these — `/GS`, `/DYNAMICBASE`,
`/NXCOMPAT`, `/HIGHENTROPYVA`, `/LARGEADDRESSAWARE`, `/ZH:SHA_256` — are
*on by default* in modern MSVC and link.exe. Others — `/guard:cf`,
`/guard:ehcont`, `/CETCOMPAT`, `/Qspectre*`, `/sdl`, `/SOURCELINK` —
require explicit opt-in because they carry
project-specific trade-offs (code-size impact, performance impact,
toolchain version floors, build-pipeline integration). They are enabled
when the developer asks for them.

### How does compiler-options hardening work?

Compiler-options hardening covers three related kinds of protection:

1. **Compile-time checks.** These options instruct the compiler to look
   harder at the source code and warn about constructs that are known to
   be associated with defects — uninitialized variables, integer
   truncation, signed/unsigned comparison, non-virtual destruction of
   polymorphic objects, switch fall-through, deprecated APIs, and so on.
   They do not change the generated code; they change what the build will
   accept. The options in [Table 1](#table-1) are of this kind.
2. **Run-time mitigations.** These options instruct the compiler to emit
   additional code, or to mark the resulting binary in a way that causes
   the operating system to enforce additional checks at load time or run
   time. Stack-cookie probes, Control Flow Guard call-target checks,
   Data Execution Prevention, ASLR, shadow stacks, and Spectre
   serialization fences all fall into this category. They change the
   generated code and the binary so that an exploit that *would* have
   succeeded against an unprotected binary will instead crash the
   process. The options in [Table 2](#table-2) are of this kind.
3. **Runtime instrumentation for testing.** Sanitizer-instrumented
   builds add bookkeeping at every load, store, allocation, and free
   so that errors the compile-time checks cannot see — out-of-bounds
   access into the heap, use-after-free, double-free, container
   overflow — fault deterministically when exercised by tests or
   fuzzing. MSVC ships AddressSanitizer for x86 and x64 Windows (GA),
   with ARM64 support in preview from MSVC Build Tools version 14.50.
   Sanitizer builds are *not* a shipping-build mitigation; they are a
   defect-finding tool intended for test and fuzzing pipelines. See
   [§11 Sanitizers](#sanitizers).

### What does compiler-options hardening *not* do?

Compiler-options hardening reduces the impact of bugs that are already
in the code. It does **not**:

- find authentication or authorization defects,
- find injection vulnerabilities (SQL, command, XSS, etc.),
- find concurrency or time-of-check/time-of-use defects,
- substitute for fuzzing, code review, secure-design practices, static
  analysis beyond `/analyze`, dependency auditing, or threat modelling,
- prevent attacks against logic that the developer wrote intentionally.

Compiler hardening is one layer of defense-in-depth, not a substitute
for the secure-development practices that surround it. Those practices
are **out of scope** for this compiler-focused guide; adopt them
alongside it and consult their own references rather than this one:

- **Secure-development process** — threat modeling, design and code
  review: the
  [Microsoft SDL](https://www.microsoft.com/securityengineering/sdl/),
  the [OpenSSF Best Practices Badge](https://www.bestpractices.dev/),
  and [OpenSSF Scorecard](https://github.com/ossf/scorecard).
- **Dependency hygiene** — keep third-party libraries current and watch
  them for advisories with
  [Dependabot](https://github.com/dependabot),
  [vcpkg's vulnerability tracker](https://learn.microsoft.com/vcpkg/concepts/vulnerabilities),
  and the [OSV](https://osv.dev/) database.

Two adjacent activities *do* have a compiler dimension and are therefore
covered here: source-level static analysis via [`/analyze`](#code-analysis),
and dynamic analysis via the sanitizers and fuzzing in
[§11](#sanitizers). Binary-level policy verification with
[BinSkim](https://github.com/microsoft/binskim) is covered under
[Enable compiler errors](#enable-compiler-errors). The remainder of this
guide does not revisit the out-of-scope topics above.

## Best practices for compiler-options hardening

The following practices apply regardless of which specific options you
adopt from the tables below.

### Stay current

Use a supported compiler and a supported standard library. New language
versions add features that make hardening easier, and new compiler
versions add warnings, mitigations, and sanitizers that catch defects
that older versions could not.

### Generate and publish debug symbols

Generate full debug symbols for every release build, and publish them so
that crash dumps, sanitizer reports, and security researchers can resolve
addresses back to the exact source revision that built the binary. With
MSVC this means compiling with `/Zi` (or `/Z7`) and `/ZH:SHA_256`, and
linking with `/SOURCELINK`. See [§9 Maintaining debug
information](#maintaining-debug-information) for the full set of options,
the `/Zi`-versus-`/Z7` trade-off, and the recommended symbol-server
workflow, including how to author the `sourcelink.json` manifest the
linker embeds.

<span id="enable-compiler-errors"></span>

### Enable compiler errors

Develop with warnings as errors. Set `/W4 /WX` so that any unaddressed
warning fails the build, and add `/sdl` to elevate the SDL-required
warning set from level-3/4 warnings into errors. Pair these with
`/external:I`, `/external:W0`, and `/external:templates-` so that the
warning-as-error policy applies to *your* code without being drowned out
by diagnostics from third-party headers you cannot fix.

Pair the build-time controls above with **post-link policy
verification**: [BinSkim](https://github.com/microsoft/binskim) is a
Microsoft open-source static analyzer that reads the linked PE binary
and its PDB and asserts that hardening options (CFG, ASLR/DEP, `/sdl`,
critical warnings, Source Link, `/Qspectre`, `/CETCOMPAT`, and many
others) were in fact applied. Because BinSkim runs against the shipped
artifact, it catches regressions that would otherwise slip past `/WX`
— for example a project that quietly lost `/Qspectre` because someone
toggled a property page. Configuring CI to fail on BinSkim violations
is the recommended way to make hardening regressions visible.

### Prevent sensitive-information disclosure

Spectre-class transient-execution attacks let an attacker observe data
that the program would otherwise have protected by a permission check.
Enable `/Qspectre` (and, for higher-assurance code paths, `/Qspectre-load`
or `/Qspectre-load-cf`) to have the compiler emit the necessary
serialization barriers at function call sites that load secret data
under speculative control. See [§7 `/Qspectre`](#qspectre) for the
trade-offs and for the relationship to OS-level and silicon-level
Spectre mitigations.

## Recommended compiler options

For the rest of this guide,

- **Table 1** lists compile-time checks that find defects in source code.
  None of them affects the binary; they only affect what the compiler
  accepts.
- **Table 2** lists run-time mitigations and binary-metadata flags. These
  change the generated code, the linker output, or both, and the OS
  loader inspects them when starting a process.
- Each row links to a detailed per-option section in
  [§7](#per-option-detail).

<span id="table-1"></span>
**Table 1.** Recommended MSVC compiler options that enable strictly
compile-time checks.

| Compiler flag | Description |
|:--- |:--- |
| [`/W4`](#w4) | Enable warning level 4 — the broadest set short of `/Wall`. |
| [`/WX`](#wx) | Treat warnings as errors. Use during development and CI. |
| [`/sdl`](#sdl) | Enable additional security warnings and elevate a curated set to errors. Adds the run-time checks listed in [Table 2](#table-2). |
| [`/permissive-`](#permissive) | Strict ISO C++ conformance. Disables the legacy "Microsoft extensions" that historically silenced standards-violating code. On by default when `/std:c++20` (or higher) is in effect. |
| [`/analyze`](#analyze) | Run the MSVC source-level static analyzer (PREfast). CI / scheduled, not every local build — see [§10](#code-analysis). |
| [`/analyze:plugin EspXEngine.dll`](#analyze-plugin) | Load the EspX extension engine, which hosts the C++ Core Guidelines checker (`CppCoreCheck`) and other PREfast plug-ins, all off by default. CI / scheduled, not every local build — see [§10](#code-analysis). |
| [`/utf-8`](#utf-8) | Treat source and execution character sets as UTF-8 unless overridden. Avoids C4828 and the silent locale-dependent mis-encoding of literals. |
| [`/external:I`](#external-i)<br/>[`/external:W0`](#external-w)<br/>[`/external:templates-`](#external-templates) | Treat third-party / system headers as "external" so `/W4 /WX` policy applies to your code only, while warnings inside templates instantiated from your code remain visible. |
| [`/wd<n>` / `/we<n>` / `/wo<n>`](#warning-control) | Per-warning-number suppression, elevation to error, or report-once. Use sparingly; see [§8 Suppressing warnings](#suppressing-warnings). |
| `/w14388` | Surface off-by-default **C4388** (`signed/unsigned mismatch`) at `/W4`, catching a signed index compared against an unsigned container `size()` on 64-bit builds. See the [C4018 entry](#c4018). |

> **Note on `/Wall`.** MSVC also exposes `/Wall`, which enables *every*
> warning, including those the compiler team deliberately keeps off
> because they are too noisy in practice. `/Wall` is useful for an
> opportunistic audit, but is **not** a baseline recommendation; see
> [`/Wall`](#wall) and [Appendix B](#appendix-b) for the off-by-default
> inventory.

<span id="table-2"></span>
**Table 2.** Recommended MSVC compiler and linker options that enable
run-time protection mechanisms or attach security-relevant metadata to
the produced binary.

| Compiler / linker flag | Description |
|:--- |:--- |
| [`/GS`](#gs) (compiler, default on) | Insert stack-cookie probes around vulnerable stack frames to detect stack-buffer overflows. |
| [`/sdl`](#sdl) (compiler) | In addition to the compile-time effect above, adds run-time checks: nulls pointers after `delete` / `free`, zero-initializes class members not initialized by the user, forces `__declspec(safebuffers)` to be ignored, and (with `/RTC`) terminates the process on certain integer overflows. |
| [`/guard:cf`](#guard-cf) (compiler **and** linker) | Emit Control Flow Guard call-site checks. Requires both compile-time emission and linker opt-in for the binary to be marked CFG-compatible. |
| [`/guard:ehcont`](#guard-ehcont) (compiler) + `/CETCOMPAT` (linker) | Emit EH-continuation metadata describing valid exception-resumption targets. Use together with `/CETCOMPAT` to enable hardware-enforced shadow-stack return-address checks on CET-capable CPUs (Tiger Lake+, Zen 3+). |
| [`/Qspectre`](#qspectre)<br/>[`/Qspectre-load`](#qspectre-load)<br/>[`/Qspectre-load-cf`](#qspectre-load-cf) (compiler) | Insert speculation barriers to mitigate CVE-2017-5753 (Spectre v1) and its load/load-control-flow variants. |
| [`/CETCOMPAT`](#cetcompat) (linker) | Mark the binary as compatible with Intel CET / AMD Shadow Stack. The OS then enforces a hardware shadow stack on return addresses. |
| [`/DYNAMICBASE`](#dynamicbase) (linker, default on) | Mark the binary as relocatable, allowing the loader to randomize its base address (ASLR). |
| [`/HIGHENTROPYVA`](#highentropyva) (linker, x64) | Use the full 64-bit address space for ASLR entropy on x64 / ARM64. |
| [`/NXCOMPAT`](#nxcompat) (linker, default on) | Mark the binary as compatible with Data Execution Prevention (DEP / NX). |
| [`/SAFESEH`](#safeseh) (linker, x86 only) | Generate a safe SEH handler table on x86. Has no effect on x64 or ARM64 (which use the platform's typed-table SEH). |
| [`/LARGEADDRESSAWARE`](#largeaddressaware) (linker) | Mark a 32-bit executable as able to use addresses > 2 GB; default for x64 and ARM64. Required for ASLR entropy to be useful on 32-bit. |
| [`/fsanitize=address`](#fsanitize-address) (compiler, **not** for shipping builds) | AddressSanitizer instrumentation: precise detection of heap, stack, and global-variable overflows, use-after-free, use-after-scope, double-free, and related defects. Build test and fuzz binaries with this. |
| [`/Zi`](#zi) + [`/ZH:SHA_256`](#zh-sha256) (compiler) | Generate PDB debugging information with SHA-256 file content hashes. Required for source-link, symbol-server indexing, and post-incident diagnostics. |
| [`/SOURCELINK`](#sourcelink) (linker) | Embed a source-link mapping into the PDB so debuggers can fetch sources directly from your VCS. |

Sections [§7.1](#per-option-detail) onward describe each option in
detail.

<span id="per-option-detail"></span>

## 7. Detailed description of each compiler option

Per-option sections follow the same shape as the OpenSSF GCC/Clang
guide: each begins with a one-line **Synopsis** that names the option
family and what it does, followed by sub-sections on performance, when
not to use the option, and any additional considerations.

Where MSVC's terminology differs from GCC/Clang's, the section calls out
the GCC/Clang analog so that developers who maintain code on both
toolchains can map the two sets of recommendations onto each other.

### 7.1 Compile-time check options

<span id="w4"></span>

#### `/W<n>` and `/W4` — warning levels

##### Synopsis

`/W<n>` selects the *baseline* warning level for the compiler, where
`n` is `0`, `1`, `2`, `3`, `4`, or `all`. Each numeric level is a
superset of the previous one. MSVC's warning levels are organized by
the strength of the evidence the diagnostic produces, not by how
optional any of them is for hardening: every numbered level matters
to a secure build. `/W1` is the smallest set of warnings — diagnostics
the compiler will emit even without any explicit `/W` flag — and is
reserved for code that is almost certainly wrong (for example, a
function declared with mismatched signatures, or a missing return
statement on a non-`void` path). `/W2` and `/W3` widen the set with
diagnostics for code that is very likely wrong but where the compiler
sees a residual possibility that the construct is intentional. `/W4`
is the broadest level whose signal-to-noise ratio is high enough for
use as a build-time gate; some legitimate constructs at `/W4` need an
explicit annotation or a localized warning suppression, but the
diagnostic-by-diagnostic noise is bounded. `/Wall` enables everything,
including warnings that fire on standard-library and OS-SDK headers
and that are therefore impractical as a baseline.

Microsoft's recommendation for hardening builds is `/W4`, paired with
[`/WX`](#wx) and [`/sdl`](#sdl). `/W4` is the MSVC analog of GCC and
Clang's `-Wall -Wextra`.

##### Performance

Warning levels do not affect generated code or compile time in any
meaningful way. There is no run-time cost.

##### When not to use this option

Lower warning levels (`/W3`, `/W2`, `/W1`) are appropriate only for
exploratory builds, for builds whose entire purpose is to suppress a
single specific defect class, or for legacy code where the team has
made a deliberate, time-bounded decision to defer a `/W4` migration.
Never ship code that was built at a level below `/W3` to a customer.

##### Additional considerations

A specific warning can be moved between levels with `/w<l><n>` —
e.g. `/w14996` makes warning C4996 a level-1 warning so it surfaces
even at `/W1`. See [§8 Suppressing warnings](#suppressing-warnings) for the
full set of per-warning controls.

<span id="wx"></span>

#### `/WX` — treat warnings as errors

##### Synopsis

`/WX` treats every emitted warning as an error and fails the build.
On its own `/WX` does not change *which* warnings the compiler emits;
it changes how the build *reacts* to them.

`/WX` is the MSVC equivalent of GCC and Clang's `-Werror`. The same
operational guidance applies on all three compilers: enable it in CI,
enable it in pre-submit, enable it in release builds, and *do not*
make a habit of disabling it on a per-developer basis.

The same OpenSSF caveat that applies to GCC/Clang `-Werror` also
applies here: `/WX` is appropriate for *binaries you produce*. If you
ship a source distribution that downstream consumers will build with
arbitrary future MSVC versions, do not force `/WX` on their build —
a future toolset's new warning could break their build for reasons
unrelated to their code. Enforce `/WX` in your own CI, not in your
public build files.

##### Performance

None. `/WX` only changes the compiler's exit code policy.

##### When not to use this option

There are two legitimate exceptions:

1. **Snapshot tracking of an upstream toolchain.** When you are
   evaluating a new MSVC pre-release for compatibility with a large
   codebase, you may want to run `/W4` without `/WX` first so that all
   newly emitted diagnostics surface in one CI run instead of stopping
   at the first one. Re-enable `/WX` before the toolchain ships.
2. **Vendored third-party code you do not own.** Use
   [`/external:I`](#external-i) and [`/external:W0`](#external-w) to
   exclude those headers from `/WX` rather than disabling `/WX`
   project-wide.

##### Additional considerations

A specific warning can be made an error without enabling `/WX` for
the whole build by using `/we<n>` (e.g. `/we4996`). This is useful
during a staged rollout: pick the warnings the team wants to enforce
first, then expand the set until the whole build can run under `/WX`.

<span id="sdl"></span>

#### `/sdl` — additional security checks

##### Synopsis

`/sdl` is the MSVC switch that opts a build in to the
[Microsoft Security Development Lifecycle](https://www.microsoft.com/securityengineering/sdl/)-recommended
set of additional compile-time and run-time security checks. It does
two distinct things:

1. **Compile time.** Elevates a fixed set of nine warnings to errors,
   regardless of the baseline `/W<n>` setting: C4146 (unary minus on
   unsigned), C4308 (negative constant converted to unsigned), C4532
   (`continue`/`break`/`goto` in `__finally`), C4533 (jump past
   initialization), C4700 / C4703 (uninitialized local / uninitialized
   local pointer), C4789 (destination buffer too small), and C4995 /
   C4996 (deprecated function / function marked unsafe). The exact
   list is the `/sdl` mandatory security-warning set and has been
   stable since Visual Studio 2013. All
   nine are discussed individually in
   [§8](#discussion-of-selected-warnings).
2. **Run time.** Adds `/GS` (the buffer-overrun check) and prevents it
   from being disabled by `/GS-` for the rest of the command line;
   forces `__declspec(safebuffers)` to be ignored so that every
   function gets the `/GS` cookie; nulls pointers after `delete` /
   `free` so that use-after-free dereferences fault at a known
   address; zero-initializes class members that the user has not
   explicitly initialized; and (with `/RTC`) terminates the process
   on certain integer-overflow conditions instead of producing a
   defined-but-wrong result.

`/sdl` is the most cost-effective single option a developer can add
to an MSVC build to improve its security posture. Microsoft's
recommended baseline is `/W4 /WX /sdl`, paired with the
off-by-default elevation `/w14388` listed in the headline
compile-with block. That elevation surfaces the signed/unsigned
comparison defect class, which is off by default in the compiler
(see [§8](#discussion-of-selected-warnings)). `/sdl` itself does **not**
turn on any off-by-default warnings; its compile-time effect is
strictly to escalate severity of warnings the compiler is already
emitting. Under `/WX` the nine `/sdl` `/we` promotions are
mechanically redundant — `/sdl`'s distinctive contribution at
`/W4 /WX` is therefore the run-time and codegen behaviors above, not
the warning escalations.

BinSkim's [`BA2026 EnableMicrosoftCompilerSdlSwitch`](https://github.com/microsoft/binskim/blob/main/src/BinSkim.Rules/PERules/BA2026.EnableMicrosoftCompilerSdlSwitch.cs)
post-link rule verifies that a shipped binary was actually compiled
with `/sdl` by inspecting the `IMAGE_DEBUG_TYPE_VC_FEATURE` debug
directory entry. Include `BA2026` in your post-build BinSkim policy to
confirm that `/sdl` reached the final artifact.

##### Performance

The compile-time portion is free; it only changes diagnostics. The
run-time portion (forced pointer initialization, additional `/GS`
coverage) has measurable but small cost — typically <1% on
representative workloads. Microsoft has documented the cost as
"negligible for most code" in the [`/sdl` reference page](https://learn.microsoft.com/cpp/build/reference/sdl-enable-additional-security-checks).

##### When not to use this option

`/sdl` is appropriate for almost every project. The two scenarios
in which it may need adjustment are:

1. **Hard real-time or kernel-mode code** that cannot tolerate the
   additional initialization writes on every entry to a function with
   pointer-typed locals. Measure first; `/sdl`'s cost is small in
   practice, but a hot real-time path may justify a narrowly scoped
   exception.
2. **Codebases that cannot yet pass the elevated-warning set.** In
   this case, enable `/sdl` *without* `/WX` first, fix the warnings
   the elevation surfaces, then add `/WX` back.

##### Additional considerations

A useful complement to `/sdl` is [`/analyze`](#code-analysis) (the
MSVC source-level static analyzer, PREfast), which finds many of the
same defect classes but along control-flow paths that the front-end
diagnostics alone cannot see. The two are designed to coexist; use
both. See [§10 Code analysis](#code-analysis) for the recommended
cadence and CI integration.

<span id="permissive"></span>

#### `/permissive-` — strict ISO C++ conformance

##### Synopsis

`/permissive-` disables MSVC's longstanding "Microsoft extensions"
which historically allowed several categories of standards-violating
code to compile. These extensions include implicit `int`, missing
`typename` in dependent contexts, two-phase name lookup loopholes,
non-const lvalue references binding to temporaries in argument
passing, and a number of legacy preprocessor behaviors.

In practical terms, `/permissive-` causes MSVC to behave much more
like GCC and Clang. Code that compiles cleanly under `/permissive-`
is substantially more portable, and a number of warning classes that
were merely diagnostic without `/permissive-` become hard errors with
it — which is what you want for hardening.

`/permissive-` is **on by default** when `/std:c++20` or higher is in
effect. New projects targeting C++20 inherit it automatically. Older
projects targeting C++17 or earlier should add it explicitly.

##### Performance

None. The option only changes the front-end's acceptance rules; it
does not affect generated code.

##### When not to use this option

A small set of vendored third-party headers — most notoriously the
Microsoft Foundation Classes (MFC) and parts of the older ATL — were
written before `/permissive-` existed and may not compile under it.
For those headers, use `/external:I` to mark them external, or
selectively re-disable `/permissive-` for the affected translation
units with `#pragma push_macro` techniques. Do not disable
`/permissive-` for *your own* code.

##### Additional considerations

`/permissive-` is the foundation for several other modernization
flags such as `/Zc:__cplusplus` (have `__cplusplus` report the
correct value), `/Zc:preprocessor` (use the C++11-conformant
preprocessor), and `/Zc:inline` (enforce one-definition-rule for
inline functions). Each of these is recommended for modern code; see
the Microsoft Learn page on [conformance options](https://learn.microsoft.com/cpp/build/reference/zc-conformance).

<span id="analyze"></span>

#### `/analyze` — MSVC static analyzer (PREfast)

##### Synopsis

`/analyze` runs MSVC's source-level static analyzer over each
translation unit and emits warnings for defects it finds along
path-sensitive control-flow paths within each function (with
SAL-annotated function summaries used to reason across call sites
in the same translation unit) — buffer overruns,
use-after-free, null-pointer dereference, lock-order violations,
SAL-annotated contract violations, and a long list of others. It is
the MSVC analog of `clang --analyze`, and it is shipped
in every edition of Visual Studio.

`/analyze` warnings have numbers in the C6xxx and C28xxx ranges to
distinguish them from front-end warnings (Cxxxx) and from Core
Guidelines checker warnings (C26xxx).

##### Performance

`/analyze` substantially increases compile time — typically 2x–4x for
the analyzed translation units, because the analyzer must perform
abstract interpretation along multiple paths. In practice it runs in
CI rather than on every developer build; see
[§10 Code analysis](#code-analysis) for the recommended cadence,
CI integration, and SARIF/log output.

##### When not to use this option

`/analyze` has no run-time effect and no impact on the produced
binary. The only reason to leave it off is compile-time cost, and
that is best handled by running it in CI rather than on every
developer machine, not by disabling it altogether.

##### Additional considerations

Pair `/analyze` with `/analyze:plugin EspXEngine.dll` (see below) to
also enable the EspX C++ Core Guidelines checker, and with [BinSkim's
BA2007 rule](https://github.com/microsoft/binskim) for post-link
validation that the resulting binary was built with the critical
compiler warnings enabled.

<span id="analyze-plugin"></span>

#### `/analyze:plugin EspXEngine.dll` — EspX extension engine (Core Guidelines + other checkers)

##### Synopsis

`EspXEngine.dll` is MSVC's extension engine for PREfast. It hosts
checker plug-ins that emit diagnostics in the C26xxx range, **all of
which are off by default**. The one relevant here is `CppCoreCheck.dll`,
the C++ Core Guidelines checker (described below); EspXEngine hosts
other PREfast plug-ins as well, documented on the [Use the C++ Core
Guidelines checkers](https://learn.microsoft.com/cpp/code-quality/using-the-cpp-core-guidelines-checkers)
page on Microsoft Learn.

Selection happens via the `Esp.Extensions` environment variable
when invoking the compiler from the command line — for example,
`set Esp.Extensions=CppCoreCheck.dll` — or via
MSBuild project properties (`EnableCppCoreCheck`, etc.). For
documentation see the
[Use the C++ Core Guidelines checkers](https://learn.microsoft.com/cpp/code-quality/using-the-cpp-core-guidelines-checkers)
and [`/analyze` reference](https://learn.microsoft.com/cpp/build/reference/analyze-code-analysis)
pages on Microsoft Learn.

The [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
themselves are a set of rules for safer modern C++, curated by Bjarne
Stroustrup, Herb Sutter, and the editors of the C++ Foundation. The
`CppCoreCheck.dll` plug-in emits the corresponding C26xxx diagnostics.
These checks are **off by default** and are organized into rule sets
that map to the Core Guidelines profiles and categories — Arithmetic,
Bounds, Type-safety, Lifetime, Const, Class, Enum, the
GSL / owner / unique / raw / shared-pointer resource-management sets,
STL, and others. Many of them presuppose adoption of Core Guidelines
coding patterns and the [Guidelines Support Library
(GSL)](https://github.com/microsoft/GSL) — for example, diagnostics
that direct you to replace a raw pointer with `gsl::span` or a
`static_cast` with `gsl::narrow`. See [Use the C++ Core Guidelines
checkers](https://learn.microsoft.com/cpp/code-quality/using-the-cpp-core-guidelines-checkers)
for the rule-set catalog and how to select a subset.

##### Performance

Same compile-time cost as `/analyze` itself; the plugin runs as part
of the analyzer pass.

##### When not to use this option

The C26xxx checks are off by default. Because many of them enforce
Core Guidelines / GSL patterns, enabling them on a codebase that has
not adopted those patterns can produce a high volume of diagnostics
that reflect coding-style conformance rather than discrete defects.
The [rule-set documentation](https://learn.microsoft.com/cpp/code-quality/using-the-cpp-core-guidelines-checkers)
catalogs the available sets and how to enable a subset.

<span id="utf-8"></span>

#### `/utf-8` — source and execution character set is UTF-8

##### Synopsis

`/utf-8` is a convenience that sets both `/source-charset:utf-8`
(MSVC reads source files as UTF-8) and `/execution-charset:utf-8`
(narrow string literals are encoded as UTF-8 in the resulting
binary). Without `/utf-8`, MSVC infers the source encoding from the
file's byte-order mark or, in its absence, from the active Windows
code page — which has caused a long tail of "compiles on my machine,
breaks in CI" issues.

`/utf-8` also resolves diagnostic C4828 ("file contains a character
starting at offset N that is illegal in the current source character
set"). Adopting `/utf-8` consistently across a project is the
recommended modern baseline.

##### Performance

None.

##### When not to use this option

If your codebase deliberately mixes source files in multiple legacy
encodings (Shift-JIS, Big5, Windows-1252, etc.), `/utf-8` will break
the build. In that case the right answer is to *convert* the source
files to UTF-8 once, then enable `/utf-8`.

##### Additional considerations

Wide string literals (`L"..."`) are unaffected by `/utf-8`; they are
always encoded as UTF-16 on Windows. For UTF-8 string literals, use
the C++20 `u8"..."` form, which produces `const char8_t[]`.

<span id="external-i"></span>
<span id="external-w"></span>
<span id="external-templates"></span>

#### `/external:I`, `/external:W0`, `/external:templates-` — external-header diagnostic isolation

##### Synopsis

The most common reason teams disable warnings or back off from
`/W4 /WX` is that third-party headers — Boost, Windows SDK, vendor
SDKs, generated protobuf headers — emit warnings the team cannot fix
because they do not own the source. The `/external:*` family solves
this problem cleanly:

- `/external:I <path>` marks a directory as containing *external*
  headers.
- `/external:anglebrackets` additionally marks every header included
  with angle brackets (`#include <...>`) as external.
- `/external:W<n>` sets the warning level *for code inside external
  headers* to `n`. The standard recommendation is `/external:W0` —
  no warnings inside third-party headers.
- `/external:templates-` makes warnings *propagate out of* an
  external template when the template was instantiated by your own
  code. This is what you want: a `std::vector` instantiation in your
  translation unit should still warn about your problems even though
  `<vector>` itself is external.

The combination `/external:I path /external:W0 /external:templates-`
is the modern, supported answer to the "we can't use `/W4 /WX`
because of Boost" problem. It also obsoletes most of the older
ad-hoc workarounds — wrapper headers that bracket third-party
`#include` directives with `#pragma warning(push, 0)` /
`#pragma warning(pop)`.

The feature shipped first as `/experimental:external` in VS 2017
15.6 and was promoted to a stable, non-experimental option in VS
2019 16.10.

##### Performance

None.

##### When not to use this option

Use `/external:*` for genuinely third-party headers. The team must
agree on *what counts as external* — typically anything not under
`src/` or `include/` of your own project.

##### Additional considerations

For build systems that consume environment variables — CMake with
generator expressions, MSBuild with property sheets — `/external:env`
reads the list of external paths from an environment variable, which
is the most maintainable form. See the [MSVC `/external` docs](https://learn.microsoft.com/cpp/build/reference/external-external-headers-diagnostics).

The legacy `#pragma warning(push)` / `#pragma warning(pop)` /
`#pragma warning(disable: N)` pattern remains available and is the
right tool for *targeted* per-line suppressions inside your own code
(see [§8 Suppressing warnings](#suppressing-warnings)); `/external:*` is
the right tool for *whole-header* suppressions of code you do not own.

<span id="warning-control"></span>

#### `/wd<n>`, `/we<n>`, `/wo<n>` — per-warning controls

##### Synopsis

Per-warning controls modify the policy for a single Cxxxx warning
without affecting the baseline `/W<n>` level:

- `/wd<n>` — *disable* warning n.
- `/we<n>` — *elevate* warning n to an error (regardless of `/WX`).
- `/wo<n>` — *emit warning n only once* per build.
- `/w<l><n>` — *move* warning n to warning level *l*. For example,
  `/w14996` makes C4996 visible even at `/W1`.

Use these to harden a build incrementally: pick the specific
diagnostics the team wants to enforce first, elevate them with `/we`,
and add the rest later.

##### Performance

None.

##### When not to use this option

`/wd<n>` is the right tool only when (a) the diagnostic is a known
false positive in a specific place and (b) you cannot use
`/external:*` or `#pragma warning` to scope the suppression more
narrowly. In particular, do not use `/wd<n>` to project-globally
silence warnings that are flagging real defects. See
[§8 Suppressing warnings](#suppressing-warnings) for the recommended
order of preference.

##### Additional considerations

Document each `/wd<n>` in a comment in the build script that names
*why* the warning is being suppressed and what would have to change
for it to be re-enabled. A bare list of disabled warnings tends to
accrete without bound; a documented list is reviewable.

<span id="wall"></span>

#### `/Wall` — all warnings

##### Synopsis

`/Wall` enables every warning the compiler knows about, including
the [warnings that are off by default](#appendix-b). The set is much
larger than `/W4` — it includes warnings that fire on the C++
standard-library headers, on the Windows SDK headers, and on many
patterns that are perfectly idiomatic in modern code (e.g. C4514,
"unreferenced inline function has been removed", which fires on
nearly every template-heavy translation unit).

##### Performance

None.

##### When not to use this option

`/Wall` is impractical as a build-time baseline because of the
noise. Use it as an *audit* tool: build periodically with `/Wall`
to inventory the diagnostics that the off-by-default set would
emit, then promote individual warnings into your `/W4` build with
`/w14<n>` (or `/we<n>`) if they fire on real defects. See
[Appendix B](#appendix-b) for the full off-by-default list.

##### Additional considerations

`/Wall` is the MSVC analog of Clang's `-Weverything`: both options
enable every diagnostic the toolchain knows how to emit, including
many that fire on standard-library and OS-SDK headers. That makes
them useful as periodic audit tools — running them once a release
cycle to inventory new diagnostics worth promoting — but impractical
as CI gates.

### 7.2 Run-time mitigation and binary-metadata options

The options in this section change the *binary* that the build
produces. Some of them require both a compiler flag and a linker flag;
the per-option text calls out the pairing in each case. Several of them
mark the binary as compatible with a Windows or hardware mitigation
that the OS loader will then enforce; if the linker flag is missing,
the marker is absent and the loader does not apply the mitigation, even
if the compiler emitted the supporting code.

<span id="gs"></span>

#### `/GS` — stack-buffer-overrun detection

##### Synopsis

`/GS` causes the compiler to emit a **security cookie** in the prologue
of every function whose stack frame contains a buffer the compiler
considers vulnerable (most importantly, a fixed-size character array or
a struct containing one). The function's epilogue compares the cookie
against a known good value before returning; a mismatch causes
`__report_gsfailure` to fast-fail the process. `/GS` is the MSVC analog
of GCC and Clang's `-fstack-protector-strong`.

`/GS` is on by default in MSVC and has been since VS 2002. The reason
to list it here is that some build environments — particularly
hand-written makefiles for kernel-mode or embedded code — explicitly
disable it with `/GS-`. Verify it is on.

Adding [`/sdl`](#sdl) extends `/GS`'s coverage to additional vulnerable
patterns the heuristics would otherwise miss, and applies the cookie
check to functions marked `__declspec(safebuffers)` regardless of the
attribute. Microsoft recommends the combination `/GS` (on by default)
plus `/sdl` (explicit) for hardening builds.

##### Performance

The per-function cost is a single XOR-with-RSP at entry and a single
compare-and-branch at exit, both in the function prologue/epilogue.
Microsoft documents the aggregate cost as <1% on representative
workloads. The cost is *zero* for functions whose frames contain no
vulnerable buffer — the compiler does not insert the cookie at all.

##### When not to use this option

Keep `/GS` on for shipping code. The narrow exception is functions
where the cookie itself would interfere with correctness — most
often, hand-written stack-walking primitives in a debugger or
profiler. Use `__declspec(safebuffers)` on those
specific functions if you are not using `/sdl`.

##### Additional considerations

`/GS` is unrelated to `/Gs<n>`, which controls the threshold at which
the compiler inserts a stack-probe call (`__chkstk`). Both have their
place; do not confuse them.

<span id="guard-cf"></span>

#### `/guard:cf` — Control Flow Guard

##### Synopsis

[Control Flow Guard (CFG)](https://learn.microsoft.com/windows/win32/secbp/control-flow-guard)
is a Windows mitigation that prevents an attacker who has gained the
ability to overwrite an indirect-call target from redirecting control
to an arbitrary address. The compiler instruments every indirect call
site with a check against a kernel-maintained bitmap of valid call
targets; an indirect call to an address that is not in the bitmap
faults to a fast-fail.

CFG is **a paired flag** — `/guard:cf` must be passed to *both* the
compiler and the linker. The compiler flag instructs MSVC to emit the
call-site check sequences; the linker flag emits the metadata that
populates the per-image bitmap and marks the PE header so the Windows
loader recognizes the binary as CFG-compatible. If either flag is
missing, CFG is silently not applied.

Microsoft's recommendation is to pass `/guard:cf` to both compile
**and** link.

##### Performance

CFG instrumentation adds roughly 1–2% to binary size and a small
percentage to indirect-call latency (a single bit-test against a
process-global bitmap, well-predicted by modern branch predictors).
For most workloads the aggregate cost is sub-1% throughput. CPU- and
workload-specific data is in [Microsoft's CFG performance
documentation](https://learn.microsoft.com/cpp/build/reference/guard-enable-control-flow-guard).

##### When not to use this option

CFG depends on the Windows loader, so it has no effect on non-Windows
targets, on Windows XP / Vista / 7 RTM (the loader support was added
in Windows 8.1 Update 3 / Windows 10), or on binaries that are loaded
via a custom loader that does not honor the CFG bit.

##### Additional considerations

For maximum coverage, combine `/guard:cf` with [`/guard:ehcont`](#guard-ehcont)
and `/CETCOMPAT` so that the hardware shadow stack covers return-address
flow in addition to CFG's coverage of forward-call flow. Together
these three options give the binary protection against both forward-
and backward-edge control-flow hijacks.

<span id="guard-ehcont"></span>

#### `/guard:ehcont` — EH continuation metadata for CET

##### Synopsis

`/guard:ehcont` causes the compiler to emit metadata describing the
valid *exception-resumption* targets in every function. When a binary
linked with `/CETCOMPAT` runs on a CPU that supports Intel CET / AMD
Shadow Stack, the Windows loader uses this metadata in combination
with the hardware shadow stack to verify that an exception unwind
resumes at a programmer-intended `__except` filter or `__finally`
block — not at an attacker-chosen address.

`/guard:ehcont` is a no-op on x86 (which uses table-based SEH that the
compiler validates differently) and on CPUs without CET support; it is
safe to enable unconditionally on x64.

##### Performance

The instrumentation is metadata-only — it does not change the
generated code. The only run-time cost is a marginal increase in image
size (a few bytes per `__except`/`__finally`).

##### When not to use this option

`/guard:ehcont` has no effect on CPUs that cannot enforce it and
improves the security posture on CPUs that can.

##### Additional considerations

`/guard:ehcont` must be paired with `/CETCOMPAT` at link time for the
hardware to enforce the metadata. See [`/CETCOMPAT`](#cetcompat).

<span id="qspectre"></span>

#### `/Qspectre` — Spectre v1 (CVE-2017-5753) mitigation

##### Synopsis

`/Qspectre` instructs the compiler to recognize the canonical
*bounds-check-bypass* pattern (CVE-2017-5753, "Spectre variant 1") and
to emit an `LFENCE` instruction (or equivalent serialization on
ARM64) between the bounds check and the subsequent guarded load. The
serialization barrier prevents the CPU from speculatively executing
the load with attacker-chosen out-of-bounds indices, which would
otherwise have left observable cache-state side effects.

There is no exact GCC equivalent at the compiler level; Clang ships
`-mspeculative-load-hardening` (SLH) as a broader speculation blocker.
GCC's `-mindirect-branch=thunk` and `-mfunction-return=thunk` target a
*different* Spectre variant (v2, branch-target injection), which on
Windows is mitigated through OS-level *retpoline* and microcode rather
than a compiler flag.

Microsoft's recommendation is to enable `/Qspectre` for any binary
that processes untrusted inputs. [BinSkim's `BA2024
EnableSpectreMitigations`](https://github.com/microsoft/binskim/blob/main/src/BinSkim.Rules/PERules/BA2024.EnableSpectreMitigations.cs)
rule fires when a binary compiled with a `/Qspectre`-capable toolset
(VS 2017 15.7 and later) ships without the mitigation, so this
recommendation is also enforceable in CI without a code review.

> **Scope.** `/Qspectre`'s heuristic recognizes the canonical
> bounds-check-bypass shape; speculation patterns that depart from it
> are not covered. The default Microsoft CRT (`msvcrt*`, `ucrt*`,
> `vcruntime*`) is not built with `/Qspectre` — link the
> Spectre-mitigated runtime (`*_spectre.lib`) to close that gap. On
> hot, load-bound loops the inserted `LFENCE` instructions carry a
> measurable cost.

##### Performance

The cost of `/Qspectre` is the cost of the inserted `LFENCE`
instructions. On most workloads the aggregate impact is well under 1%
because the heuristic that selects bounds-check-bypass sites is
conservative — the compiler does not insert the barrier on every
indexed load. Microsoft documents `/Qspectre`'s performance impact in
the [`/Qspectre` reference page](https://learn.microsoft.com/cpp/build/reference/qspectre).

##### When not to use this option

`/Qspectre` mitigates Spectre v1 only. The other Spectre variants
(v2 / branch-target injection, v4 / speculative store bypass, and so
on) are addressed by separate compiler and OS mitigations — for
example, `/Qspectre-load` and `/Qspectre-load-cf` (below), the OS-level
Indirect Branch Restricted Speculation (IBRS), and microcode updates.
`/Qspectre` is not a complete Spectre defense; it is one layer.

##### Additional considerations

The Spectre family of attacks is an active area of research; new
variants emerge with new silicon. Treat `/Qspectre` as a *floor* of
compiler-side defense and follow Microsoft's [transient-execution
guidance](https://learn.microsoft.com/security-updates/securityadvisories/2018/180002)
for the current full set.

<span id="qspectre-load"></span>

#### `/Qspectre-load` — Spectre mitigations for all memory loads

##### Synopsis

`/Qspectre-load` extends `/Qspectre`'s coverage from the
bounds-check-bypass pattern to *every* load instruction. The compiler
emits an `LFENCE` before every memory load that could otherwise
participate in a speculative side-channel.

This is a much stronger mitigation than `/Qspectre` and a much
more expensive one. It is appropriate for high-assurance code paths —
cryptographic primitives, secret-handling code, sandbox boundaries —
where the value of the assets being protected justifies the
performance cost.

##### Performance

The cost is substantial — typically a single-digit percentage
slowdown on memory-bound workloads, and noticeably higher (10%+) on
the very tightest loops. Measure on representative workloads before
adopting project-wide.

##### When not to use this option

`/Qspectre-load` is *not* a default-on recommendation for ordinary
code. Use it selectively: identify the translation units that handle
secrets and compile *those* with `/Qspectre-load`; compile the rest
with `/Qspectre`.

##### Additional considerations

For control-flow-only Spectre mitigation (less coverage, lower cost),
see [`/Qspectre-load-cf`](#qspectre-load-cf) below.

<span id="qspectre-load-cf"></span>

#### `/Qspectre-load-cf` — Spectre mitigations for control-flow load instructions

##### Synopsis

`/Qspectre-load-cf` is the middle option between `/Qspectre` and
`/Qspectre-load`. It instruments only those load instructions that
participate in *control-flow* decisions (loads of branch targets,
function pointers, virtual-table pointers, jump-table indices). This
gives most of `/Qspectre-load`'s coverage for the speculation paths
that matter most to control-flow hijack — at a fraction of the cost.

##### Performance

Substantially cheaper than `/Qspectre-load`; typically 1–3% slower
than a `/Qspectre`-only build.

##### When not to use this option

The same selectivity guidance applies as for `/Qspectre-load`: enable
it for translation units where the additional coverage matters.

##### Additional considerations

The `/Qspectre`, `/Qspectre-load-cf`, and `/Qspectre-load` flags select
among **three alternative coverage modes**, not additive switches. If
more than one is passed on the command line, the rightmost wins. From
narrowest (cheapest) to broadest (most expensive):

- `/Qspectre` — instrument the canonical Spectre v1 bounds-check-bypass
  pattern only.
- `/Qspectre-load-cf` — also instrument loads that feed control-flow
  decisions (branch targets, function pointers, vtable pointers,
  jump-table indices).
- `/Qspectre-load` — instrument *every* memory load instruction.

`/Qspectre-jmp` is an orthogonal flag that mitigates unconditional
jump instructions; it can be combined with whichever mode above is
selected. Pick the mode that matches each translation unit's threat
exposure.

<span id="cetcompat"></span>

#### `/CETCOMPAT` (linker) — Intel CET / AMD Shadow Stack compatibility

##### Synopsis

`/CETCOMPAT` is a *linker* flag that marks the resulting PE image as
compatible with Intel Control-flow Enforcement Technology (CET) and
its AMD equivalent. On CPUs that implement the hardware shadow stack
(Intel Tiger Lake and later, AMD Zen 3 and later), the Windows loader
enables the shadow stack for the process and the CPU enforces that
every `RET` instruction returns to the address recorded on the shadow
stack. Stack-buffer-overflow exploits that overwrite the return
address — even ones that bypass `/GS` — fault deterministically.

`/CETCOMPAT` should be paired with [`/guard:ehcont`](#guard-ehcont)
(compiler) so the loader has the metadata it needs to validate
exception-unwind targets.

##### Performance

The hardware shadow stack runs in parallel with the regular stack and
imposes no measurable per-call overhead on CET-capable CPUs. On
non-CET CPUs the `/CETCOMPAT` marker is ignored entirely; the binary
behaves as if it had not been built with `/CETCOMPAT`.

##### When not to use this option

`/CETCOMPAT` applies to x64 binaries that target Windows 10 21H1 or
later. Older Windows versions ignore the marker.

##### Additional considerations

For details on how `/CETCOMPAT` interacts with kernel-mode
hibernation, fast-fail handling, and third-party JITs that emit code
into RWX pages, see the [Microsoft CET deployment
documentation](https://learn.microsoft.com/cpp/build/reference/cetcompat).

<span id="dynamicbase"></span>

#### `/DYNAMICBASE` (linker) — Address Space Layout Randomization

##### Synopsis

`/DYNAMICBASE` marks the binary as relocatable, which causes the
Windows loader to load it at a randomized base address (Address Space
Layout Randomization, ASLR). Combined with [`/HIGHENTROPYVA`](#highentropyva)
on 64-bit binaries, this gives the loader the full 64-bit address
space worth of entropy to randomize across.

`/DYNAMICBASE` has been on by default since Visual Studio 2010. You do
not need to add it; verify it reached the shipped binary with
[BinSkim](https://github.com/microsoft/binskim) and do not pass
`/DYNAMICBASE:NO`.

##### Performance

The cost of `/DYNAMICBASE` is the cost of base-relocation processing
at load time (typically tens of microseconds) and the loss of one
optimization: an EXE built with `/DYNAMICBASE` cannot have its imports
patched with absolute addresses at link time, so cross-DLL calls go
through the import address table. The throughput cost is negligible.

##### When not to use this option

Do not disable `/DYNAMICBASE`. The only exceptions are special-purpose
binaries that must load at a fixed address (some kernel-mode drivers,
some embedded firmware), and even there the exception should be
narrowly scoped.

##### Additional considerations

For ASLR to be meaningfully effective on 64-bit Windows, pair
`/DYNAMICBASE` with `/HIGHENTROPYVA` (below). On 32-bit Windows, also
set `/LARGEADDRESSAWARE` so that the loader has more than 2 GB of
address space across which to randomize.

<span id="highentropyva"></span>

#### `/HIGHENTROPYVA` (linker) — 64-bit ASLR entropy

##### Synopsis

`/HIGHENTROPYVA` tells the Windows loader that the binary's pointers
are full 64-bit values, so the loader can randomize the base address
across the full 64-bit address space rather than truncating to 32-bit
entropy for compatibility with legacy 32-bit-only code paths.

`/HIGHENTROPYVA` is on by default for x64 binaries built with Visual
Studio 2012 and later. You do not need to add it; verify it is set.

##### Performance

None.

##### When not to use this option

The only reason to disable `/HIGHENTROPYVA` is interoperation with a
specific legacy component that cannot handle pointer values above the
4 GB boundary. Such components are increasingly rare.

##### Additional considerations

`/HIGHENTROPYVA` has no effect on x86 (32-bit) binaries; it is
implicitly disabled there. For ASLR to be effective on 64-bit Windows,
pair it with [`/DYNAMICBASE`](#dynamicbase).

<span id="nxcompat"></span>

#### `/NXCOMPAT` (linker) — Data Execution Prevention

##### Synopsis

`/NXCOMPAT` marks the binary as compatible with Data Execution
Prevention (DEP) — the hardware-enforced no-execute bit on data
pages. With `/NXCOMPAT` set, the Windows loader maps the binary's
data, heap, and stack pages as non-executable; an attempt to execute
an instruction fetched from those pages faults with an access
violation.

`/NXCOMPAT` is on by default and has been since Visual Studio 2005
SP1. You do not need to add it; verify it is set.

##### Performance

None.

##### When not to use this option

The only legitimate reason to disable `/NXCOMPAT` is a binary that
genuinely runs interpreter-generated machine code from a data page
without first remapping the page as executable — and modern JIT
runtimes (V8, .NET, Java, the MSVC C++ runtime's `std::function`
inline thunks) all do the right thing and remap.

##### Additional considerations

`/NXCOMPAT` is the loader marker; the actual enforcement is performed
by the CPU's no-execute bit and by the Windows memory manager. The
combination has been the floor of Windows process-level memory safety
for two decades.

<span id="safeseh"></span>

#### `/SAFESEH` (linker, x86) — Safe Structured Exception Handling

##### Synopsis

`/SAFESEH` emits a table of valid Structured Exception Handler (SEH)
entry points into the x86 binary. When an exception is dispatched, the
Windows OS exception dispatcher consults this table; an SEH frame
whose handler pointer is not listed in the table is rejected. This
defeats the classic "SEH overwrite" exploit pattern on x86.

`/SAFESEH` applies only to x86 binaries. On x64 and ARM64, SEH is
implemented via the platform's typed unwind tables, which are
inherently structured and do not need a separate "safe" mode.

##### Performance

None. The table is built at link time; the loader compares against it
once per exception dispatch.

##### When not to use this option

`/SAFESEH` has no effect on x64 or ARM64 binaries. For x86, there is
no good reason to disable it; the only obstacle is that *every*
object file linked into the image must itself be compiled with
SEH-aware metadata (which MSVC emits by default; the issue arises
only when linking object files from a non-MSVC toolchain).

##### Additional considerations

If `/SAFESEH` fails at link time because an object file lacks the
required metadata, the root cause is almost always a third-party
static library built without `/safeseh`. Rebuild it with MSVC or
update to a current version.

<span id="largeaddressaware"></span>

#### `/LARGEADDRESSAWARE` (linker) — full 32-bit address space

##### Synopsis

`/LARGEADDRESSAWARE` marks a 32-bit binary as able to use the full
4 GB of user-mode address space (otherwise the loader limits it to
the low 2 GB for compatibility with 1990s-era pointer-arithmetic
bugs). Without `/LARGEADDRESSAWARE`, the randomization *range* for an
ASLR'd image on 32-bit Windows is constrained to roughly the low 1 GB
of address space (after reserving the kernel half and architectural
mappings), which yields only a few bits of effective ASLR entropy —
not enough to be meaningfully exploit-resistant.

`/LARGEADDRESSAWARE` is the default for x64 and ARM64 binaries.

##### Performance

None.

##### When not to use this option

Only if the 32-bit binary has documented dependence on pointer values
being below the 2 GB boundary — e.g., it stuffs flag bits into the
high bit of a pointer. Such code is a hardening anti-pattern in its
own right.

##### Additional considerations

If `/LARGEADDRESSAWARE` exposes a latent pointer-sign-extension bug
in legacy code, fix the bug rather than disabling the flag.

<span id="fsanitize-address"></span>

#### `/fsanitize=address` — AddressSanitizer

See [§11 Sanitizers](#sanitizers) for the full discussion. Briefly:

`/fsanitize=address` enables AddressSanitizer (ASAN) — a precise,
shadow-memory-based detector for the entire class of memory-safety
defects that C and C++ are uniquely prone to: heap-buffer-overflow,
heap-use-after-free, stack-buffer-overflow, stack-use-after-scope,
global-buffer-overflow, double-free, use-after-poison, and others.
Apply it primarily to **the same shipping product sources, built in a
sanitizer-instrumented (non-release) configuration**, and exercised by
the team's existing tests, fuzzers, and CI suites — the bugs that
matter are bugs in the product code, and ASAN catches them in exactly
the call patterns the tests already drive. The test scaffolding itself
rarely needs to be rebuilt with ASAN; the slowdown does not usually
pay back on unit-test harness code.

ASAN is **not** appropriate for the **release build** of shipping
code — the 2x–3x runtime overhead and 3x memory overhead make it
impractical, and a shipping binary should not be relying on dynamic
memory-safety detection in the first place. Ship the release build
*without* ASAN after the sanitizer-instrumented build of the same
sources passed every test.

##### Compatibility constraints

- VS 2019 16.9 or later (MSVC toolset 14.28).
- x86 or x64 only (no ARM64 in MSVC ASAN yet).
- Incompatible with `/ZI` (edit-and-continue), `/INCREMENTAL` linking,
  and `/RTC` runtime checks. Remove or replace those before enabling
  `/fsanitize=address`.
- The MSVC linker automatically links the correct AddressSanitizer
  runtime when it sees `/fsanitize=address`-built objects (this is the
  `/INFERASANLIBS` linker default; do not disable it unless you are
  linking a custom ASAN runtime).

<span id="zi"></span>
<span id="zh-sha256"></span>

#### `/Zi` and `/ZH:SHA_256` — debug information with SHA-256 file hashes

See [§9 Maintaining debug information](#maintaining-debug-information)
for the full discussion. Briefly:

`/Zi` produces a `.pdb` file containing debugging
information — line numbers, type information, public symbol names,
local variable names, source file paths. A single `cl.exe` invocation
shares one compiler PDB across every object file it compiles (the
default is `vc<n>.pdb`; control the name with `/Fd`). The linker then
consumes those object files and the compiler PDB to emit a single
image-level PDB per binary. `/Zi` is required for source-link, for
symbol-server publishing, for post-incident dump analysis, and for
AddressSanitizer's stack-trace symbolication.

`/ZH:SHA_256` selects SHA-256 as the algorithm MSVC uses to hash the
contents of every source file consumed by the build. The hash is
recorded in the PDB and used by debuggers and source-link tooling to
verify that the source the developer is looking at is the source that
built the binary. The `/ZH` option itself is available starting in
Visual Studio 2019 version 16.4 with three choices: `/ZH:MD5`,
`/ZH:SHA1`, and `/ZH:SHA_256`. MD5 is the historical (and Visual Studio
2019) default; both MD5 and SHA-1 are cryptographically broken for
collision resistance and should not be relied on for source-verification
provenance.

`/ZH:SHA_256` is the default in **Visual Studio 2022 version 17.0** and
later; in Visual Studio 2019 and other earlier toolsets, pass it
explicitly. Builds on toolsets too old to offer `/ZH` at all (before
Visual Studio 2019 16.4) should prioritize a toolchain upgrade.
(Visual Studio 2026
version 18.6.0 / MSVC 14.51 add `/ZH:SHA384` and `/ZH:SHA512` options,
but these exceed the current IFC module-format hash size limit; stick
with `/ZH:SHA_256` for compatibility with C++ modules.)

<span id="z7"></span>

> **Alternative: `/Z7` (embedded debug info).** `/Z7` produces the same
> CodeView debug records as `/Zi` but embeds them directly into the
> `.obj` file instead of writing a separate compiler PDB. The linker
> still produces an image-level PDB the same way. `/Z7` is useful in
> two scenarios: build-cache workflows (sccache / ccache) where a
> shared compiler PDB is awkward to cache; and single-artifact builds
> that ship one self-contained `.lib` or `.obj`. The trade-off is
> larger object files (since every `.obj` carries its own copy of the
> debug records) and somewhat slower link time. `/Zi` remains the
> recommendation here because it composes naturally with the
> symbol-server publishing workflow described in
> [§9](#maintaining-debug-information); teams whose build pipeline
> already pushes them toward `/Z7` can keep doing so — the
> downstream symbol-server steps in §9 do not change.

<span id="sourcelink"></span>

#### `/SOURCELINK` (linker) — embed source-link map in the PDB

See [§9 Maintaining debug information](#maintaining-debug-information).
Briefly:

`/SOURCELINK` embeds a JSON [source-link
manifest](https://github.com/dotnet/sourcelink) into the PDB. When a
debugger encounters a stack frame from this binary, the manifest
maps each source-file path to a versioned URL in the source-control
system the binary was built from. The debugger fetches the source on
demand; the developer reading the dump sees exactly the source that
built the binary. No more "I don't have that revision checked out."

Source-link is the standard provenance mechanism for shipping
binaries. Adopt it. BinSkim's [`BA2027 EnableSourceLink`](https://github.com/microsoft/binskim/blob/main/src/BinSkim.Rules/PERules/BA2027.EnableSourceLink.cs)
verifies post-link that the shipped PDB embeds a SourceLink manifest.

<span id="discussion-of-selected-warnings"></span>
<span id="warnings-management"></span>

## 8. Warnings management

This section is the warnings home of the guide. It first discusses
the individual MSVC warnings that the recommended hardening baseline
elevates from "warning" to "error" — either directly via
[`/sdl`](#sdl), or implicitly via [`/W4`](#w4) + [`/WX`](#wx), or
indirectly by enabling a feature (such as `/permissive-`) that
re-categorizes the warning. The baseline also explicitly enables one
off-by-default warning that the others miss — C4388, via
[`/w14388`](#c4018) — without elevating it to an error. The closing
subsection, [Suppressing warnings](#suppressing-warnings), covers the
recommended discipline for the exceptional cases — when a diagnosed
construct is, in fact, intended.

Each subsection follows the same shape: the canonical compiler text,
the defect class the warning detects, an example that fires the
warning, and the recommended modern fix. Code examples are illustrative
only; do not assume the surrounding context is production-ready code.

The full set of warnings elevated by `/sdl` is the nine listed in
[§7.1 `/sdl`](#sdl) above, stable since VS 2013.

<span id="c4018"></span>

### C4018 — signed/unsigned mismatch

**Compiler text:** `'%$K': signed/unsigned mismatch` (level 3)

**Defect class:** A comparison or arithmetic expression mixes a signed
operand with an unsigned operand. The C and C++ usual-arithmetic-
conversions rules silently convert the signed value to unsigned,
which flips negative values to large positive ones and causes the
expression to evaluate to a "true" or "false" result the developer
did not intend. This is one of the most common sources of bounds-check
failures and off-by-one defects in C and C++ code.

**Example that fires the warning:**

```cpp
void f(int a, unsigned int b) {
    if (a < b) { }        // C4018: '<': signed/unsigned mismatch
}

```

**Note on `int` vs `std::size_t`:** the most common form of this
defect in real code is comparing a signed loop index with the return
value of `.size()` on a standard-library container, which is
`std::size_t`. That comparison does **not** emit C4018 on a 64-bit
build — the implicit promotion through `long long` routes it to a
different warning, **C4388 ("`signed/unsigned mismatch`")**, which is
[off by
default](https://learn.microsoft.com/cpp/preprocessor/compiler-warnings-that-are-off-by-default).
To surface it under `/W4`, enable it explicitly with
`/w14388`. The defect class and the recommended fixes are identical.

```cpp
#include <vector>
int find_first_negative(const std::vector<int>& v) {
    for (int i = 0; i < v.size(); ++i) {        // C4388 with /w14388 (not C4018)
        if (v[i] < 0) return i;
    }
    return -1;
}

```

If `v.size()` exceeds `INT_MAX`, `i` overflows into the negative
range, the comparison silently misbehaves, and the loop runs forever
or terminates early.

**Recommended fix (simplest, works in any C++ standard):**

```cpp
#include <optional>
#include <vector>
std::optional<std::size_t> find_first_negative(const std::vector<int>& v) {
    for (std::size_t i = 0; i < v.size(); ++i) {
        if (v[i] < 0) return i;
    }
    return std::nullopt;
}

```

Match the loop variable's type to the container's `size_type`
(`std::size_t`). This eliminates the implicit signed→unsigned
conversion entirely, and the return type expresses "not found"
without a negative-sentinel convention that would have re-introduced
the signed/unsigned ambiguity. Use `std::optional<std::size_t>`
(C++17) for the sentinel; pre-C++17 codebases can return
`static_cast<std::size_t>(-1)` with a documented convention.

**Alternative fix (when a signed loop variable is genuinely required):**

```cpp
#include <utility>     // std::cmp_less
#include <vector>
int find_first_negative(const std::vector<int>& v) {
    for (int i = 0; std::cmp_less(i, v.size()); ++i) {
        if (v[i] < 0) return i;
    }
    return -1;
}

```

If the loop variable must remain signed — for example because it
participates in arithmetic that can legitimately go negative —
`std::cmp_less` (added in C++20, in header `<utility>`) compares
integers correctly regardless of signedness, with no implicit
conversion. The full family — `std::cmp_equal`, `std::cmp_not_equal`,
`std::cmp_less`, `std::cmp_less_equal`, `std::cmp_greater`,
`std::cmp_greater_equal` — covers every comparison form. Note this
fix does *not* protect against `i` itself overflowing when the
container grows beyond `INT_MAX` — for that you still need the
unsigned-loop form above.

**Pre-C++20 fallback:**

```cpp
#include <SafeInt.hpp>
for (SafeInt<int> i = 0; i < v.size(); ++i) { … }

```

`SafeInt` ([github.com/dcleblanc/SafeInt](https://github.com/dcleblanc/SafeInt))
performs the comparison correctly and throws `SafeIntException` if any
arithmetic operation would overflow. It is the recommended pre-C++20
defense.

**Don't:**

```cpp
for (int i = 0; (unsigned)i < v.size(); ++i) { … }     // hides the bug
for (int i = 0; i < (int)v.size(); ++i) { … }          // hides the bug

```

Casting one operand to the other's type silences the warning without
fixing the underlying issue. If `v.size()` overflows `int`, the cast
makes the comparison wrong in a worse way.

### C4146 — unary minus on unsigned type

**Compiler text:** `unary minus operator applied to unsigned type,
result still unsigned` (level 2, elevated to error under `/sdl`)

**Defect class:** `-x` where `x` is an unsigned integer evaluates to
the two's-complement negation in the unsigned type, which is a very
large positive number. The most common bug is taking the absolute
value of what the programmer believed was a signed quantity but had
already been silently widened to unsigned.

**Example:**

```cpp
size_t distance = -offset;     // C4146: offset is size_t

```

**Fix:** Decide what type the value should have. If it can legitimately
be negative, store it in a signed type. If it cannot, the `-` operator
is meaningless and the expression is a logic bug.

```cpp
std::ptrdiff_t distance = -static_cast<std::ptrdiff_t>(offset);

```

<span id="c4244"></span>
<span id="c4267"></span>

### C4244 / C4267 — possible loss of data in conversion

**Compiler text:**

- `'%$L': conversion from '%$T' to '%$T', possible loss of data` (C4244 levels 2/3/4)
- `'%$L': conversion from 'size_t' to '%$T', possible loss of data` (C4267 level 3)

**Defect class:** An implicit conversion narrows a value to a type
that cannot represent all of its possible values. This is the
narrowing form of the integer-truncation defect class
([CWE-197](https://cwe.mitre.org/data/definitions/197.html)). The
specific warning number depends on the source and destination types,
but the defect is the same.

**Example:**

```cpp
#include <vector>
void process(const std::vector<long long>& src, std::vector<int>& dst) {
    int count = src.size();                  // C4267: size_t -> int (narrowing)
    for (int i = 0; i < count; ++i) {
        dst[i] = src[i];                     // C4244: long long -> int (narrowing)
    }
}

```

Two narrowing sites fire two different warning numbers: C4267 for
the `size_t → int` initialization, C4244 for the `long long → int`
assignment. A C-style cast at either site (e.g., `int count =
(int)src.size();`) silences the warning *without fixing the defect*
— never use a cast as the fix.

**Recommended fix:** Use `std::span<T>` to eliminate the pointer-and-
length idiom entirely:

```cpp
#include <span>
void copy(std::span<const char> src, std::span<char> dst) {
    std::ranges::copy(src.begin(), src.begin() + std::min(src.size(), dst.size()), dst.begin());
}

```

`std::span` (C++20, header `<span>`) carries the length with the
pointer, eliminates the narrowing site, and gives the algorithm a
chance to use the correct integer types throughout.

**For an explicit narrowing where you have verified the value fits:**

```cpp
#include <gsl/narrow>     // microsoft/GSL
auto i_int = gsl::narrow<int>(n);     // throws gsl::narrowing_error if value doesn't fit
auto i_int_unchecked = gsl::narrow_cast<int>(n);     // no check; documents intent

```

`gsl::narrow<T>` is checked and throws on truncation. `gsl::narrow_cast<T>`
is unchecked but expresses the intent more clearly than a plain
`static_cast`. The Core Guidelines checker (`/analyze:plugin
EspXEngine.dll`) emits C26472 to flag `static_cast`s that should be
`gsl::narrow*`.

### C4302 / C4308 — truncation and negative-to-unsigned conversion

**Compiler text:**

- `'%$L': truncation from '%$T' to '%$T'` (C4302 level 2)
- `negative integral constant converted to unsigned type` (C4308 level 2; elevated to error under `/sdl`)

**Defect class:** A more specific form of C4244. Worth calling out
because the diagnostic identifies a specific narrowing pattern (often
a pointer-to-integer cast that drops bits) or a specific sign-conversion
of a negative constant that flips it to a large positive value.

**Example:**

```cpp
void* p = some_handle();
int handle_low = (int)p;     // C4302 on x64 — drops the high 32 bits

```

**Fix:** Use the correct integer width for the cast:

```cpp
#include <cstdint>
auto handle_full = reinterpret_cast<std::uintptr_t>(p);

```

If you genuinely need only the low 32 bits, use `gsl::narrow_cast`
to document that intent.

<span id="c4510"></span>
<span id="c4610"></span>
<span id="c4611"></span>

### C4510 / C4610 / C4611 — unconstructable classes and SEH-vs-C++ interaction

**Compiler text:**

- `'%$pS': default constructor was implicitly defined as deleted` (C4510 level 4)
- `%$B '%$S' can never be instantiated - user defined constructor required` (C4610 level 4)
- `interaction between '%$S' and C++ object destruction is non-portable` (C4611 level 4)

**Defect class:** C4510 / C4610 are paired warnings that fire when a
class has a reference or `const` member but no user-provided
constructor — meaning the compiler-generated default constructor would
be `= delete`, and the class is therefore unusable. The defect is a
silent ABI failure: the class compiles but cannot be constructed by
any value-initializing code path.

C4611 fires when a C++ object's destruction interacts with the
Microsoft-specific structured-exception-handling `__try` / `__except`
mechanism — the SEH unwinder does not run C++ destructors, so an
exception transferred through a `__except` block leaks every C++
object on the path.

**Fix for C4510 / C4610:** Either give the class a
user-provided constructor that initializes the reference / `const`
member, or remove the member if it does not belong in the class. The
modern idiom is to mark deliberately non-copyable types explicitly
with `= delete`:

```cpp
class Connection {
    const std::string url;      // implicit default ctor would be = delete
public:
    explicit Connection(std::string url) : url(std::move(url)) {}

    Connection(const Connection&)            = delete;     // explicit, not implicit
    Connection& operator=(const Connection&) = delete;
};

```

**Fix for C4611:** Do not mix C++ exception handling and SEH in the
same function. Wrap the SEH-using code in a separate C function and
use C++ try/catch in the calling C++ code.

### C4532 / C4533 — control flow that skips destructors

**Compiler text:**

- `'%$*': jump out of __finally / finally block has undefined behavior during termination` (C4532 level 1)
- `initialization of '%1$S' is skipped by 'goto %2$pS'` (C4533 level 1)

**Defect class:** A `goto`, `break`, or `continue` jumps over the
initialization of a stack object with non-trivial destructor — or
out of a `__finally` block during stack unwind. Either case leaves
the object in a destroyed-but-not-constructed state. C4533 in
particular is a UB-on-execution defect that the standard requires
diagnosing.

**Note on modern MSVC behavior:** in current MSVC, this defect is
routed through three different diagnostics depending on context:

- **`/std:c++20` and later:** always **error C2362** ("initialization
  of '%1$S' is skipped by 'goto %2$pS'"). The C++20 standard
  ([N4659 §9.7p3](https://timsong-cpp.github.io/cppwp/n4659/stmt.dcl#3))
  makes a program that jumps from a point where a variable with
  automatic storage duration is not in scope to a point where it is
  in scope **ill-formed**, so the compiler raises an error rather
  than a warning.
- **Pre-C++20 with `/Za` (strict conformance) or any type with a
  destructor:** error C2362 as well, for the same reason.
- **Pre-C++20, type without a destructor, MS extensions enabled
  (the default):** warning C4533 at `/W1`.

In other words, in any modern C++20+ build, you will see C2362 as a
compile error, not C4533 as a warning. Treat this as a hard failure
to fix at the source.

**Fix:** Restructure the control flow to construct the object before
any path that would jump past it, or scope the object to a block that
does not contain the jump:

```cpp
// before (C2362 in C++20, or C4533 pre-C++20)
if (failed) goto cleanup;
std::lock_guard lock(m);
…
cleanup:

// after — scope the lock so the goto is outside its lifetime
if (failed) goto cleanup;
{
    std::lock_guard lock(m);
    …
}
cleanup:

```

In modern code, prefer RAII over `goto cleanup` entirely.

### C4700 / C4701 / C4703 — uninitialized local variable

**Compiler text:**

- `uninitialized local variable '%s' used` (C4700, error by default — no level; **elevated to error under `/sdl`**)
- `potentially uninitialized local variable '%s' used` (C4701, level 4)
- `potentially uninitialized local pointer variable '%s' used` (C4703, level 4; **elevated to error under `/sdl`**)

**Defect class:** A local variable is read before it has been
written. The value read is whatever bytes happen to be on the stack —
a textbook information-disclosure surface and (when the value is a
pointer or a count) a memory-safety surface.

C4700 fires when the compiler proves the variable is unconditionally
uninitialized at the read. C4701 fires when a path exists that would
read an uninitialized non-pointer value. C4703 is the same as C4701
specialized for pointer-typed locals — split out because of the
elevated memory-safety risk of dereferencing an uninitialized pointer,
and promoted to error by `/sdl` for that reason.

**Fix:** Always brace-initialize local variables at declaration:

```cpp
int count{};               // zero-initialized; both C4700 and C4701 silenced
std::string buffer{};      // default-constructed
SomeStruct s{};            // value-initialized (all members zero/default)

```

Brace-init has the additional benefit of being a narrowing-detection
site: `int x{narrowing_source}` is a compile error, where
`int x = narrowing_source` is a silent narrowing conversion (warning
at best).

`/sdl` adds run-time forcing of pointer-typed locals to a known
sentinel value, so even if the compiler cannot diagnose the read
statically, the resulting NULL-pointer fault is deterministic rather
than a wild read.

### C4789 — buffer will be overrun

**Compiler text:** `in function '%s' buffer '%s' of size %d bytes
will be overrun; %d bytes will be written starting at offset %d`
(level 1; elevated to error under `/sdl`).

**Defect class:** The optimizer's bounds analysis has *proven
statically* that a memory write will exceed the destination buffer's
size. This is the direct compile-time signal for the classic stack
buffer-overrun defect class
([CWE-121](https://cwe.mitre.org/data/definitions/121.html),
[CWE-787](https://cwe.mitre.org/data/definitions/787.html)).

This warning fires from the **back-end optimizer**, so it requires
optimizations to be enabled (`/O1`, `/O2`, or `/Ox`). Debug builds
(`/Od`) will not emit it. In practice this means C4789 is one of the
warnings that distinguishes a release build's diagnostic value from a
debug build's — running `/O2 /W4 /WX` on shipping configurations
catches buffer overruns the debug build silently let through.

**Example pattern:** any constant-offset write past the end of a
fixed-size stack array, where the offset and the buffer size are
both known to the optimizer (e.g., `memcpy` with a constant `n`
larger than `sizeof(dst)`, or `*(p + k) = v` with constant `k`
exceeding the bounds).

**Fix:** Use a bounds-aware API (`memcpy_s`, `strcpy_s`,
`std::copy_n` with a properly computed length, `std::span` member
functions) and propagate the destination's actual capacity through
the call chain rather than relying on a literal that has to be kept
in sync. For C++ code, `std::span<T>` (C++20) eliminates the
fixed-buffer-with-separate-length idiom that gives rise to most C4789
sites.

This warning is on Microsoft BinSkim's `BA2007 EnableCriticalCompilerWarnings`
required list — if your team's deployment pipeline runs BinSkim,
suppressing or disabling C4789 will fail the gate.

```cpp
// before (C4789, /O2 required)
void f() {
    char buf[5];
    std::memcpy(buf, "ABCDEFGHIJ", 10);   // C4789: buffer 'buf' of size 5 bytes will be overrun
    std::printf("%s\n", buf);              // observable use; without it, DCE removes the memcpy
}

// after — destination sized correctly, copy bounded by source size
void f() {
    std::array<char, 16> buf{};
    constexpr char src[] = "ABCDEFGHIJ";
    std::copy_n(src, sizeof(src), buf.begin());   // sizeof(src) <= buf.size()
    std::printf("%s\n", buf.data());
}
```

> **Reproducer note.** C4789 is back-end-emitted, so the optimizer must
> actually generate stores to the offending buffer. If the buffer is
> unused, dead-code elimination will silently remove the entire site
> *before* C4789 has a chance to fire — this is why the "before" example
> above passes the result to `printf`. In real code an unused stack
> buffer is itself a defect; the printf is a stand-in for the actual
> downstream use that exists in any non-trivial caller.

### C4995 / C4996 — deprecated API used

**Compiler text:**

- `'%$I': name was marked as #pragma deprecated` (C4995 level 3)
- `'%$S': %$*` (C4996 level 3, message text comes from the deprecation attribute)

**Defect class:** The code calls an API the platform vendor has
explicitly deprecated — usually because the API has a documented
security defect or a documented better-API replacement. The list
includes `strcpy`, `strcat`, `sprintf`, `gets`, `_alloca`, `tmpnam`,
many of the locale-sensitive `<ctype.h>` functions, and the
Windows-specific `_CRT_SECURE_NO_WARNINGS`-gated set.

**Fix:** Use the replacement the deprecation message names. For the
C runtime functions, the replacements are typically the `_s`-suffixed
secure variants (`strcpy_s`, `sprintf_s`, etc.) or the standard C11
`Annex K` bounds-checked variants. For C++ code, prefer
`std::string` / `std::format` / `std::span`-based equivalents.

```cpp
// before (C4996)
char buf[256];
strcpy(buf, src);

// after — C++20 std::format
auto s = std::format("{}", src);

// or — C11 bounds-checked
strcpy_s(buf, sizeof buf, src);

```

**Avoid:** `#define _CRT_SECURE_NO_WARNINGS` to silence the entire
class. The deprecation list represents accumulated industry knowledge
about which APIs do not compose with secure-coding practices; the
right answer is to use the safer API, not to hide the warning.

### C5045 — Spectre mitigation insertion notification

**Compiler text:** `Compiler will insert Spectre mitigation for memory
load if /Qspectre switch specified%s%s` (off by default, no level)

**Defect class:** This is not a defect warning — it is an
*informational* warning that fires at every site where, if the build
had `/Qspectre` enabled, the compiler would have inserted a Spectre
v1 mitigation barrier. It is intended as an audit tool: build with
`/Wall` (or with `/w15045` to specifically enable it) and inspect the
list of sites to understand the compiler's coverage of your code.

**Fix:** None required. Enable [`/Qspectre`](#qspectre) to actually
insert the mitigations.

<span id="appendix-a"></span>
<span id="suppressing-warnings"></span>

### Suppressing warnings

`/W4 /WX` is a productive baseline only when the team has a way to
*selectively* relax the policy at the smallest possible scope.
The recommended order of preference for relaxing a warning, from most
preferred (smallest scope) to least preferred (largest scope):

#### A.1 Use `/external:*` for third-party headers

If the warning fires inside a header you do not own — Boost, the
Windows SDK, a vendor library — make the header *external* with
[`/external:I`](#external-i), [`/external:W0`](#external-w), and
[`/external:templates-`](#external-templates). The warning policy
then applies to your code only.

```text
/external:I path\to\boost\include /external:W0 /external:templates- /W4 /WX

```

This is the modern, supported answer. It scales: as soon as a
project adopts `/external:*`, dozens of ad-hoc per-warning suppressions
in build scripts and wrapper headers can be removed.

#### A.2 Use `#pragma warning(push)` / `(disable)` / `(pop)` for a tight scope

For a single instance of a known-safe construct in your own code,
bracket the offending construct with a tightly-scoped `#pragma
warning` push/pop:

```cpp
#pragma warning(push)
#pragma warning(disable: 4996)     // we know strncpy is risky; here we have proven the bound
strncpy(dst, src, dst_size - 1);
#pragma warning(pop)

```

Always include a *comment* explaining why the suppression is correct.
A `#pragma warning(disable)` without justification is technical debt;
with a justification it is documentation.

#### A.3 Use `__pragma` for a single statement inside a macro

`__pragma` is the function-like form of `#pragma` that can be used
inside a macro expansion or other context where a `#`-prefixed
directive would be syntactically invalid:

```cpp
#define UNUSED(x) __pragma(warning(suppress: 4189)) (void)(x)

```

`warning(suppress: N)` is equivalent to `disable: N` followed by
re-enable, scoped to the next statement only.

#### A.4 Use `/wd<n>` for a project-wide suppression of last resort

If a warning fires throughout the project and the team has determined
it is not actionable in the project's coding style — e.g., C4127
"conditional expression is constant" in template-heavy code where
the condition is a `constexpr` whose value depends on the
instantiation — use `/wd<n>` on the project-wide compiler command
line:

```text
/wd4127     # constant conditional: intentional in our template metaprogramming

```

Document each `/wd<n>` in a comment beside the build-system entry
that adds it. A bare list of disabled warnings tends to accrete
without bound; a documented list is auditable.

#### A.5 Do not disable warnings as a workaround for `/WX`

If the team is tempted to add `/wd<n>` because a warning is firing
*correctly* on real code and the code cannot easily be fixed, the
right answer is to fix the code (even if the fix is a `gsl::narrow`
or a `static_cast` with a comment), not to disable the warning. A
suppressed warning will fire again when the same pattern is reintroduced
elsewhere; a fixed call site stays fixed.

<span id="maintaining-debug-information"></span>

## 9. Maintaining debug information

Hardening defects in shipping binaries are usually discovered through
post-incident analysis — crash dumps, security-research disclosures,
fuzzer findings against released binaries. Every one of those workflows
depends on having the debug information for the *exact* revision that
built the binary, available at the time the binary is analyzed,
indexed against a hash the analyzer can verify.

The MSVC tools for this — `/Zi`, `/ZH:SHA_256`, `/SOURCELINK`,
`/PDBALTPATH`, and a symbol server — are individually simple. The
discipline is to use them *together* on every release build, and to
keep the PDBs and the sources both reachable from a shipped binary's
metadata.

### Recommended workflow

1. **Compile with `/Zi`.** `/Zi` writes debug information
   into a single compiler PDB shared by every object file in that
   `cl.exe` invocation (default `vc<n>.pdb`; rename via `/Fd`). The
   linker then consumes those object files plus the compiler PDB to
   emit a single image-level PDB per binary. SHA-256 source-file
   hashing — recorded in the PDB so downstream tools can verify source
   provenance — is the default in current toolsets; see
   [`/ZH:SHA_256`](#zh-sha256) in §7.2 for older-toolset handling. To
   embed the CodeView records in each `.obj` instead of
   a shared compiler PDB — useful for build-cache (sccache/ccache) and
   single-artifact workflows — substitute [`/Z7`](#z7) for `/Zi`; the
   linker still emits the image-level PDB and the remaining steps are
   unchanged.
2. **Link with `/DEBUG:FULL /SOURCELINK:sourcelink.json /PDBALTPATH:%_PDB%`.**
   `/DEBUG:FULL` embeds the full debug stream into the PDB.
   `/SOURCELINK` embeds the source-link mapping. `/PDBALTPATH:%_PDB%`
   rewrites the recorded PDB path in the EXE/DLL to the basename of
   the PDB (rather than the absolute path on the build machine), so a
   debugger searches the symbol server for it rather than a specific
   local path.
3. **Do not ship private PDBs with the product image.** Archive the
   PDB alongside the binary in your release pipeline (MSVC PDBs are
   already separate files; the PE image only carries a debug-directory
   reference to the PDB, not the PDB content itself).
4. **Publish the PDB to a symbol server** indexed by the PE image's
   `DebugDirectory` GUID and age. Use [Microsoft's symstore
   format](https://learn.microsoft.com/windows-hardware/drivers/debugger/using-symstore)
   or any compatible server (Azure Artifacts symbol server,
   private DevOps symbol server, etc.).
5. **Tag the source-control revision the build was made from.**
   Source-link's manifest references this tag; without it, the
   debugger cannot fetch the source.

### Source-link manifest

`/SOURCELINK` embeds a `sourcelink.json` manifest — a mapping from
local source paths to versioned URLs in your source-control system —
into the PDB. The linker embeds the file verbatim, so your build must
write the manifest with the build's actual commit revision *before*
the link step. The JSON format, and the per-host mappings for GitHub,
Azure DevOps Git, GitLab, and self-hosted repos, are documented in
[`/SOURCELINK` on Microsoft Learn](https://learn.microsoft.com/cpp/build/reference/sourcelink)
and the [Source Link JSON
schema](https://github.com/dotnet/sourcelink/blob/main/json-schema.md).

<span id="code-analysis"></span>

## 10. Code analysis (`/analyze`)

[`/analyze`](https://learn.microsoft.com/cpp/build/reference/analyze-code-analysis)
is MSVC's source-level static analyzer (PREfast). It is shipped in
every edition of Visual Studio, runs as a compiler-driven analyzer
pass, has no run-time effect, and makes no change to the produced
binary — it only emits diagnostics. See
[`/analyze`](#analyze) and [`/analyze:plugin`](#analyze-plugin) in
§7.1 for the flag-level reference; this chapter covers when and how
to run it.

`/analyze` substantially increases compile time — typically 2x–4x for
analyzed translation units, because the analyzer must perform abstract
interpretation along multiple paths through each function. The
recommended cadence is therefore CI-driven rather than developer-build
driven:

- Run `/analyze` in nightly CI across every translation unit.
- Run `/analyze:only` in pre-submit CI for the translation units a PR
  touches.
- Treat new `/analyze` warnings as build-breaking once a clean
  baseline has been established.

`/analyze` is a per-translation-unit analyzer, not a whole-program
analyzer. SAL annotations let it reason across call sites within a
translation unit and across explicit annotations on external
functions, but it cannot in general prove invariants that cross
translation-unit boundaries. The
dynamic-analysis stage ([§11](#sanitizers)) covers what static
analysis cannot.

### What `/analyze` detects

`/analyze` warnings sit in the C6xxx and C28xxx ranges (distinct from
front-end Cxxxx warnings and from the Core Guidelines C26xxx warnings
emitted by the C++ Core Guidelines checker). They cover buffer overruns,
use-after-free, null-pointer dereference, lock-order violations,
SAL-annotated contract violations, integer-overflow-induced buffer
math errors, and a long list of others. `/analyze` is the MSVC analog
of `clang --analyze`.

The C++ Core Guidelines checker ([`/analyze:plugin
EspXEngine.dll`](#analyze-plugin), detailed in §7.1) adds the C26xxx
diagnostics. For a hardening audience its **Bounds** and **Lifetime**
profiles are the relevant part — they target out-of-bounds access and
dangling references directly; the broader Core Guidelines / GSL
coding-pattern profiles are off by default and high-volume on code
that has not adopted them.

### CI integration

`/analyze` writes diagnostics to stderr by default;
`/analyze:log:format:sarif` emits [SARIF
2.1](https://sarifweb.azurewebsites.net/) instead, consumable by
GitHub code-scanning, Azure DevOps, and most static-analysis
dashboards. [`.ruleset`
files](https://learn.microsoft.com/cpp/code-quality/using-rule-sets-to-specify-the-cpp-rules-to-run)
enable, disable, or re-classify individual rules per project — useful
for staging adoption across a large codebase. Note that the front-end
[`/external:*`](#external-i) flags do **not** silence `/analyze`
diagnostics in third-party headers; use
[`/analyze:external-`](https://learn.microsoft.com/cpp/build/reference/analyze-external)
or `CAExcludePath` for that.

<span id="sanitizers"></span>

## 11. Sanitizers

[AddressSanitizer](https://learn.microsoft.com/cpp/sanitizers/asan)
(ASAN) is a dynamic memory-safety analyzer integrated into MSVC since
Visual Studio 2019 16.9. Apply it to **the shipping product sources, built in a
sanitizer-instrumented (non-release) configuration**, and exercise
that binary with the team's existing test suite, fuzz corpus, and CI
workloads. ASAN-instrumented binaries fail deterministically at
runtime when instrumented code accesses memory that the ASAN runtime
has marked as invalid; the resulting report names the exact defect
class and the line of source that caused it.

### What ASAN detects

MSVC ASAN detects the major memory-safety defect classes — see the
[MSVC ASAN error
catalog](https://learn.microsoft.com/cpp/sanitizers/asan-error-examples)
for the exhaustive list with a worked example of each:

- **Out-of-bounds access** on the heap, stack, globals, and `alloca`
  buffers (overflow and underflow).
- **Use after lifetime end** — use-after-free, use-after-return, and
  use-after-scope.
- **Bad frees and allocator mismatches** — double-free, `new`/`delete`
  type mismatch, and allocator/deallocator mismatch.
- **Allocation-size faults** and **CRT intercept overlaps** (e.g.
  `memcpy` / `strncat` parameter overlap).
- **Manually poisoned memory** marked via the ASAN runtime API.

### What MSVC ASAN does *not* yet support

- Other sanitizers: `/fsanitize=thread` (ThreadSanitizer),
  `/fsanitize=leak` (LeakSanitizer), `/fsanitize=memory`
  (MemorySanitizer), `/fsanitize=undefined` (UndefinedBehaviorSanitizer),
  `/fsanitize=hwaddress` (HWAddressSanitizer). On other platforms,
  Clang implements all of these; on Windows, only ASAN (plus
  `/fsanitize=fuzzer` for LibFuzzer integration and
  `/fsanitize=kernel-address` for KASan in 17.11+) ships today.
- `initialization-order-fiasco` detection for cross-translation-unit
  static-initializer order dependencies.
- `intra-object-overflow` detection for inter-field overruns inside
  a single struct.
- Profile-guided optimization (PGO).

(`container-overflow` detection — for example, catching reads past
`vector::size()` that are still inside `vector::capacity()` — **is**
supported through STL annotations. `std::vector` and `std::string`
carry these annotations; see [Microsoft Learn's
`container-overflow` page][asan-container-overflow] and
[Appendix C](#appendix-c) for the toolsets that introduced them.
Annotations for additional types (such as `std::optional`) are being
added in newer toolsets; consult the Learn page for the current list.
Note
that the `detect_container_overflow` *runtime* option is not honored,
but the STL annotations themselves work. All translation units linked
into the image must be built with annotations enabled (the default) to
avoid One Definition Rule (ODR) violations.)

[asan-container-overflow]: https://learn.microsoft.com/cpp/sanitizers/error-container-overflow?view=msvc-170

These are well-documented gaps. For projects with strong
cross-platform coverage, consider running the test corpus under both
MSVC-ASAN on Windows and Clang-ASAN-plus-UBSan on Linux to get the
union of coverage.

### Build constraints

ASAN's instrumentation is incompatible with:

- `/ZI` (edit-and-continue debug info). Use plain `/Zi` instead.
- `/INCREMENTAL` linking. Pass `/INCREMENTAL:NO` to the linker.
- `/RTC` (run-time checks — these are MSVC's older debug-build
  checks. ASAN is generally preferred for memory-safety testing,
  though `/RTC` includes some uninitialized-local and stack-frame
  checks that ASAN does not exactly cover).
- `/fsanitize=fuzzer` and `/fsanitize=address` *can* be combined; the
  combination is the recommended fuzzing build.

`stack-use-after-scope` is on by default in MSVC ASAN and cannot be
disabled. `stack-use-after-return` requires an additional compiler
flag (`/fsanitize-address-use-after-return`) and a runtime opt-in
(`ASAN_OPTIONS=detect_stack_use_after_return=1`); it is off by default
because it requires extra stack memory per thread.

### Fuzzing

`/fsanitize=fuzzer` enables [LibFuzzer](https://llvm.org/docs/LibFuzzer.html)
integration: the build links the LibFuzzer runtime, which provides
`int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size)` as the
entry point. The fuzz harness drives a guided coverage-based search
for inputs that crash the target. Combine with `/fsanitize=address`
so that ASAN catches the memory-safety defects the fuzzer discovers.

<span id="appendix-b"></span>

## Appendix B — Warnings off by default

MSVC ships several hundred warnings that are not enabled at any of
`/W1` through `/W4`. These are the warnings the compiler team has
determined to be too noisy, too narrow, or too dependent on coding
style to enable in a default build, but useful as an opt-in for teams
that want stricter checking. **Selected security-relevant entries**
are reproduced below for reference; Microsoft documents the complete
set in [Compiler warnings that are off by default](https://learn.microsoft.com/en-us/cpp/preprocessor/compiler-warnings-that-are-off-by-default?view=msvc-170).

We do not recommend enabling the whole list project-wide; instead,
inspect the list periodically and opt in to the specific warnings
that catch defect classes the project cares about. Use `/w14<n>` to
make a specific warning visible at `/W1`, or `/we<n>` to make it a
build-breaking error.

| Cnnnn | Highest level | Compiler text |
|:---|:---:|:---|
| C4061 | 4 | enumerator '%$I' in switch of enum '%$T' is not explicitly handled by a case label |
| C4062 | 3 | enumerator '%$I' in switch of enum '%$T' is not handled |
| C4242 | 3 | '%$L': conversion from '%$T' to '%$T', possible loss of data |
| C4254 | 4 | '%$L': conversion from '%$T':%d to '%$T':%d, possible loss of data |
| C4263 | 4 | '%$pS': member function does not override any base class virtual member function |
| C4264 | 4 | '%$pS': no override available for virtual member function from base '%$pS'; function is hidden |
| C4265 | 4 | '%$S': class has virtual functions, but its non-trivial destructor is not virtual |
| C4296 | 4 | '%$L': expression is always %s |
| C4365 | 4 | '%$L': conversion from '%$T' to '%$T', signed/unsigned mismatch |
| C4388 | 4 | '%$L': signed/unsigned mismatch |
| C4456 | 4 | declaration of '%$I' hides previous local declaration |
| C4457 | 4 | declaration of '%$I' hides function parameter |
| C4458 | 4 | declaration of '%$I' hides class member |
| C4459 | 4 | declaration of '%$I' hides global declaration |
| C4555 | 1 | expression has no effect; expected expression with side-effect |
| C4582 | 4 | '%$pS': constructor is not implicitly called |
| C4583 | 4 | '%$pS': destructor is not implicitly called |
| C4623 | 4 | '%$S': default constructor was implicitly defined as deleted |
| C4625 | 4 | '%$S': copy constructor was implicitly defined as deleted |
| C4626 | 4 | '%$S': assignment operator was implicitly defined as deleted |
| C4640 | 3 | construction of local static object is not thread-safe |
| C4710 | 4 | '%$pS': function not inlined |
| C4711 | — | function '%$pS' selected for automatic inline expansion |
| C4774 | 4 | '%$pS' : format string expected in argument %d is not a string literal |
| C4820 | 4 | '%$S': '%d' bytes padding added after data member '%$I' |
| C4826 | 2 | conversion from '%$T' to '%$T' is sign-extended. This may cause unexpected runtime behavior |
| C4928 | 1 | illegal copy-initialization; more than one user-defined conversion has been implicitly applied |
| C5026 | 4 | '%$S': move constructor was implicitly defined as deleted |
| C5027 | 4 | '%$S': move assignment operator was implicitly defined as deleted |
| C5031 | 4 | #pragma warning(pop): likely mismatch, popping warning state pushed in different file |
| C5032 | 4 | detected #pragma warning(push) with no corresponding #pragma warning(pop) |
| C5038 | 3 | data member '%$I' will be initialized after data member '%$I' |
| C5039 | 4 | '%$I': pointer or reference to potentially throwing function passed to '%$I' under -EHc. Undefined behavior may occur if this function throws an exception. |
| C5045 | — | Compiler will insert Spectre mitigation for memory load if /Qspectre switch specified |
| C5204 | 4 | '%$S': class has virtual functions, but its trivial destructor is not virtual; instances of objects derived from this class may not be destructed correctly |
| C5219 | 4 | implicit conversion from '%$T' to '%$T', possible loss of data |
| C5240 | 4 | '%$S': attribute '%$I' is ignored in this syntactic position |
| C5249 | 1 | '%$pS' of type '%$T' has named enumerators with values that cannot be represented in the given bit field width of %d |
| C5258 | 4 | explicit capture of '%$I' is not required for this use |
| C5259 | 4 | '%$S': explicit specialization requires 'template <>' |
| C5262 | 4 | implicit fall-through occurs here; are you missing a break statement? |

*(The table above shows selected security-relevant entries; see the
linked catalog above for the complete set.)*

<span id="appendix-c"></span>

## Appendix C — First-available MSVC version per option

The tables in [§6](#recommended-compiler-options) omit "first available"
information to keep the descriptions readable. This appendix lists the
earliest Visual Studio release that shipped each recommended option,
sorted from oldest to most recent. If you encounter a `D9002`
(compiler) or `LNK4044` (linker) "unrecognized option" warning, the
option is not present in your toolset; locate it below to determine the
minimum toolchain required.

| Option | First available |
|:--- |:--- |
| [`/W4`](#w4) | VS 6.0 |
| [`/WX`](#wx) | VS 6.0 |
| [`/Wall`](#wall) | VS 6.0 |
| [`/wd<n>` / `/we<n>` / `/wo<n>`](#warning-control) | VS 6.0 |
| `/w14NNN` (warning-level elevation flag form) | VS 6.0 |
| [`/LARGEADDRESSAWARE`](#largeaddressaware) | VS 6.0 |
| [`/GS`](#gs) | VS 2002 (13.0) |
| [`/Zi`](#zi) | VS 2002 (13.0) |
| [`/Z7`](#z7) | VS 6.0 |
| [`/SAFESEH`](#safeseh) | VS 2003 (13.10) |
| [`/analyze`](#analyze) | VS 2005 (14.0) |
| [`/DYNAMICBASE`](#dynamicbase) | VS 2005 (14.0) |
| [`/NXCOMPAT`](#nxcompat) | VS 2005 SP1 (14.0) |
| [`/sdl`](#sdl) | VS 2012 (17.0) |
| [`/HIGHENTROPYVA`](#highentropyva) | VS 2012 (17.0) |
| [`/guard:cf`](#guard-cf) | VS 2015 (14.0) |
| [`/analyze:plugin EspXEngine.dll`](#analyze-plugin) | VS 2015 (14.0) |
| [`/utf-8`](#utf-8) | VS 2015 Update 2 (14.0) |
| [`/permissive-`](#permissive) | VS 2017 15.5 (default in new VS 2017 15.5+ projects) |
| [`/Qspectre`](#qspectre) | VS 2017 15.5 |
| [`/external:I` / `/external:W0` / `/external:templates-`](#external-i) | VS 2017 15.6 with `/experimental:external`; VS 2019 16.10 without |
| [`/SOURCELINK`](#sourcelink) | VS 2017 15.8 |
| [`/ZH:SHA_256`](#zh-sha256) | VS 2019 16.4 |
| [`/Qspectre-load`](#qspectre-load) | VS 2019 16.5 |
| [`/Qspectre-load-cf`](#qspectre-load-cf) | VS 2019 16.5 |
| [`/guard:ehcont`](#guard-ehcont) | VS 2019 16.7 |
| [`/CETCOMPAT`](#cetcompat) | VS 2019 16.7 |
| [`/fsanitize=address`](#fsanitize-address) | VS 2019 16.9 |
| `/std:c++20` (language version; recommended in [§5.4](#stay-current)) | VS 2019 16.11 |

The `container-overflow` sanitizer check (see [§11](#sanitizers))
depends on STL annotations added per standard-library type rather than
by a compiler option. The earliest toolset that shipped each:

| Standard library type | First available |
|:--- |:--- |
| `std::vector` | VS 2022 17.2 |
| `std::string` | VS 2022 17.6 |

The `std::vector` and `std::string` versions are documented on the
[Microsoft Learn `container-overflow` page][asan-container-overflow].
Annotations for additional standard types (such as `std::optional`)
are being added in newer toolsets; consult that page for the current
list.

## Contributors

Thanks to the following contributors to this guide (in alphabetical
order, by surname). All affiliations are at the time of contribution.

- Gabriel Dos Reis, Microsoft
- Michael C. Fanning, Microsoft
- Gabor Horvath, Microsoft
- Chris McKinsey, Microsoft
- Billy O'Neal, Microsoft
- Mahmoud Saleh, Microsoft
- Jay White, Microsoft

This MSVC companion guide is contributed by Microsoft to the OpenSSF
Best Practices Working Group as a sibling to the
[Compiler Options Hardening Guide for C and C++][openssf-gcc-clang],
authored by Thomas Nyman (Ericsson) and David A. Wheeler (Linux
Foundation), with contributions from George-Andrei Iosif, Christopher
"CRob" Robinson, and others listed in that document.

> **Copyright and license:** intentionally omitted at this draft stage.
> The final ship vehicle (OpenSSF Best Practices WG publication, a
> Microsoft-hosted repository, or both) will determine the appropriate
> license terms.
