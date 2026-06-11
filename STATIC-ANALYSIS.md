# HashForge/Nockhash Miner Static Analysis Report

This document records static reverse-engineering work performed against a suspicious HashForge/Nockhash miner release.

The target binary was never executed, sandboxed, emulated, or loaded through its own runtime path. All observations below came from file reads, hashes, static ELF metadata, string extraction, disassembly, and offline analysis of loader behavior.

This report intentionally does **not** include executable samples, decrypted payloads, encrypted internal payload dumps, decryption keys, IVs, exact extraction offsets, extraction scripts, decryption scripts, full disassembly listings of decryptors, or instructions sufficient to reconstruct hidden payloads.

The analysis below is provided without warranty or guarantee of any kind.

## Publication note

This report is intended to document security-relevant observations about a publicly distributed binary release.

It does not redistribute:

* the original release artifact;
* the packed executable;
* recovered inner payloads;
* decrypted GPU code;
* encrypted internal blob dumps;
* extraction scripts;
* decryption scripts;
* keys;
* IVs;
* turnkey offsets or byte ranges;
* complete disassembly listings of decryption routines.

Where packing, cryptography, or payload-hiding behavior is described, the description is limited to the minimum needed to support the security analysis. The goal is to warn users and document observable risk without enabling third parties to bypass the binary’s protections or redistribute hidden copyrighted components.

## Scope and methodology

The investigation treated the HashForge/Nockhash binaries as unsafe executable samples.

The release artifact was obtained from the project’s official public GitHub release page while it was publicly accessible. The analysis did not use leaked source code, private repositories, credentials, insider information, private communications, or unauthorized access.

The binaries were not executed.

The extracted shell wrappers were not executed.

No VM, emulator, debugger, sandbox, tracer, or dynamic runtime path was used to observe behavior.

Static tools used included:

* `file`;
* `rabin2`;
* `objdump`;
* `binwalk`;
* `r2` disassembly;
* Python byte parsing;
* hashing utilities;
* string extraction.

## Public artifact identification

The local working set included:

* a downloaded release archive;
* a packed top-level x86-64 binary;
* an extracted directory containing the executable plus wrapper/config scripts such as `h-run.sh`, `h-config.sh`, `h-manifest.conf`, and `h-stats.sh`.

The release archive and x86-64 binary were obtained from the public GitHub release: https://github.com/hashforge-pool/hashforge-miner/releases/tag/v1

The release archive had SHA-256:

```text
e71b88a29039620e24c629398902658926456ddb264e01f01c84a5149568bd08
```

The packed top-level binary had SHA-256:

```text
6c73b89031690505d74b38d6850dd4d049dfde3e6ef7274e4af61bea784d64ff
```

These hashes identify the public release artifacts analyzed for this report.

## High-level findings

The analyzed release is a real miner/prover binary wrapped in a custom anti-analysis packer.

Confirmed characteristics:

* the public release distributes a packed opaque Linux x86-64 executable;
* the top-level binary is a custom loader rather than the directly inspectable miner body;
* the loader recovers a hidden inner ELF payload;
* the loader performs anti-debugging and anti-instrumentation checks;
* the loader appears to manually map the recovered inner ELF rather than launching it through ordinary file execution;
* the recovered host application is Rust-based;
* the host application uses CUDA driver APIs;
* the host application contains a hidden CUDA/PTX module that is recovered in memory and loaded dynamically;
* miner/prover strings and CUDA kernel-name indicators match real Nockchain mining/proving functionality;
* Stratum protocol strings and pool-like configuration fragments are present;
* dev-fee logic is present;
* process-execution and networking imports are present.

No direct static-string proof was found for:

* SSH key theft;
* seed phrase theft;
* wallet-file theft;
* cloud metadata theft;
* cron/systemd persistence;
* generic downloader behavior.

The absence of those strings is not proof of absence. Behavior may be dynamically constructed, encrypted, absent from string tables, or not visible through this static methodology.

The appropriate conclusion from the current evidence is:

> This is a high-risk opaque packed miner binary with hidden mining/proving code and dev-fee logic. It should not be treated as trustworthy. No direct static-string proof of wallet theft or persistence was found in the recovered host application.

## Outer binary triage

Static ELF metadata identified the top-level binary as an x86-64 Linux ELF executable. It is statically linked, stripped, and lacks ordinary section headers.

The layout is not typical of a normal compiler-produced application binary. Program-header analysis showed a small executable loader region followed by a much larger high-entropy payload region.

This layout is consistent with a custom packed binary.

The large payload region did not appear to be a normal archive, conventional compressed object, recognizable embedded ELF, or known UPX-style wrapper. Entropy across the payload region was essentially flat/high, which is consistent with encryption or strong packing.

## Section-header removal and stripped metadata

The outer binary is stripped and lacks normal section headers.

This frustrates ordinary static-analysis workflows that expect named sections such as:

* `.text`;
* `.rodata`;
* `.data`;
* `.symtab`;
* `.strtab`.

Program headers remain sufficient for Linux to map the top-level executable, but the absence of section metadata makes disassembly, symbol recovery, and data/code boundary identification less straightforward.

This is not inherently malicious by itself. In combination with encryption, anti-debugging, hidden payload loading, and binary-only distribution, it contributes to the opacity and risk profile of the release.

## Custom outer ELF packer

The top-level binary is a small statically linked ELF loader wrapped around a much larger high-entropy payload region.

The loader’s role is to recover a hidden inner ELF payload and transfer execution to it. This means the public release artifact is not the true miner body in a directly inspectable form. Instead, the real host application is embedded as a hidden payload inside a custom loader.

This is functionally similar to common executable packers, but it does not appear to be a stock UPX-style wrapper. The observed layout and loader behavior are consistent with a bespoke Linux ELF packer.

## Loader behavior from static disassembly

Static disassembly showed that the outer binary’s entry path calls a loader routine and transfers control to a recovered entry point. This indicates that the outer binary is not the final miner body.

The loader contains anti-analysis behavior. Static analysis identified behavior consistent with:

* reading Linux process status;
* checking whether the process is being traced;
* exiting if tracing is detected;
* removing dynamic-loader instrumentation variables;
* disabling core dumps.

The dynamic-loader environment-variable scrubbing is especially relevant because variables such as `LD_PRELOAD`, `LD_AUDIT`, and `LD_DEBUG` are commonly used by analysts to intercept library calls, observe runtime behavior, or alter dynamic-linker behavior.

These checks are not required for ordinary mining functionality. Their presence supports the conclusion that the binary was designed to resist inspection.

## Hidden inner payload

The outer binary contains a large high-entropy region that static tools did not recognize as a normal archive, compressed object, UPX payload, or embedded ELF.

Offline static analysis showed that this region recovers to a valid inner ELF after applying the loader’s embedded transformation. The recovered inner payload began with ELF magic and was written only to a private local analysis directory.

This report does not include:

* the recovered inner ELF;
* the full unpacking recipe;
* exact key material;
* exact byte ranges;
* scripts for reproducing the unpacking;
* complete offsets/lengths sufficient to automate extraction.

The security-relevant conclusion is that the public binary contains a hidden inner ELF payload that is recovered and executed through custom loader logic.

## Stream-cipher-style outer transformation

The outer loader uses a stream-cipher-style transformation to recover the hidden inner ELF.

Static analysis identified behavior consistent with:

* deriving key material from bytes embedded in the loader;
* applying a key-scheduling-like step;
* generating a byte stream;
* XORing the stream over the hidden payload region.

The security-relevant point is not the cryptographic strength of the transformation. The relevant point is that the real host application is hidden behind a custom recover-and-map stage rather than distributed as a normal inspectable ELF.

This report intentionally does not publish the exact algorithm details, key material, offsets, or extraction script.

## Manual ELF mapping

After recovering the inner ELF, the loader does not appear to simply drop an unpacked executable and execute it normally.

Static disassembly indicates behavior consistent with:

* manually mapping the recovered ELF into memory;
* setting memory protections;
* adjusting process startup metadata;
* transferring control to the recovered entry path.

This matters because it avoids leaving an obvious unpacked executable on disk and complicates analysis workflows that expect the operating system’s normal ELF loader to map the final program.

Manual ELF loading is common in packers, protectors, and malware loaders. It does not prove malicious payload behavior by itself, but it is a deliberate anti-analysis design choice.

## Inner ELF metadata

The recovered inner payload is an x86-64 Linux PIE ELF dynamically linked with interpreter:

```text
/lib64/ld-linux-x86-64.so.2
```

It imports CUDA driver APIs from `libcuda.so.1`, normal libc/libpthread/libm/libdl symbols, and process/networking functions.

Notable CUDA imports include:

```text
cuInit
cuCtxCreate_v2
cuCtxDestroy_v2
cuDeviceGet
cuDeviceGetCount
cuMemAlloc_v2
cuMemcpyHtoD_v2
cuMemcpyDtoH_v2
cuModuleLoadDataEx
cuModuleGetFunction
cuLaunchKernel
```

This establishes that the host binary uses CUDA driver APIs and dynamically loads a CUDA module from memory.

Notable process and system imports include:

```text
fork
execvp
posix_spawnp
setuid
setgid
setsid
chroot
kill
```

Networking-related imports include:

```text
socket
connect
recv
send
getaddrinfo
```

These imports do not prove malicious behavior by themselves, but they expand the risk profile of an opaque packed miner binary.

## Rust host application evidence

ASCII strings were extracted from the recovered inner ELF for static analysis. The full strings file is not included in this report or repository.

The extracted strings exposed Rust panic strings, Cargo registry paths, project paths, HashForge-specific names, CUDA/NVML symbols, Stratum protocol strings, and text around CUDA module-loading behavior.

Rust evidence included strings such as:

```text
/rustc/.../library/...
/root/.cargo/registry/src/...
called Result::unwrap() on an Err value
thread '...' panicked at
```

Project/path evidence included strings such as:

```text
oxzd/src/commands.rs
oxzd/src/spawn_workers.rs
oxzd/src/idle.rs
oxzd-lib/src/algo.rs
oxzd-lib/src/algo/worker.rs
hashforge::spawn_workers
/host/algos/nock/nock-gpu/src/worker_module/mod.rs
/host/algos/nock/nock-gpu/src/nocklib/...
```

This establishes that the non-GPU host/control code is written in Rust and includes project paths associated with OXZD, HashForge, and Nock GPU mining/proving code.

Observed Rust/Cargo ecosystem indicators included references consistent with:

```text
Tokio
Rustls
Clap
Serde
Rayon
cust
aes-soft
chacha20
rand_chacha
NVML/CUDA bindings
```

The GPU code is separate from the Rust host code and is loaded dynamically through the CUDA driver API.

## Miner and pool evidence

The recovered host application contains real miner/prover material.

Strings include:

```text
Nockchain hash algorithm
oxzd
oxzd-lib
nock-gpu
```

CUDA kernel-name indicators reference Nock-related phases and Fiat-Shamir operations.

Stratum-related strings include:

```text
mining.subscribe
mining.authorize
stratum+tcp
stratum+ssl
```

A pool-like hardcoded fragment was observed:

```text
.poolhub.me:5077
```

Fee-related strings include:

```text
dev_fee
Algo Fee:
Switching to fee section, notifying API handler
Switching to non-fee section, notifying API handler
Received fee pause message, pausing workers
```

One address-like string appeared adjacent to fee-related logic:

```text
BiDq8LZy67RvvFaJc6u6K7Tiky7TfheRWiXsZdrJryQuLX8tSxDfY1A
```

I treat this as a likely dev-fee or miner-address indicator. I do not treat it as proof of credential theft or proof that the binary targets user wallets.

## Suspicious behavior search

I searched the recovered host application strings for credential-theft, persistence, cloud-metadata, and generic downloader indicators, including terms such as:

```text
.ssh
id_rsa
authorized_keys
/etc/passwd
/etc/shadow
169.254
curl
wget
cron
systemd
LD_PRELOAD
```

I did not find clear static-string evidence of SSH key theft, cloud metadata theft, cron/systemd persistence, or a generic downloader.

Absence of these strings is not proof of absence. Behavior may be hidden, dynamically constructed, encrypted, absent from string tables, or otherwise missed by this static methodology.

After unpacking the main host application, I did not find obvious plaintext evidence of a generic credential stealer or persistence subsystem.

## Hidden CUDA/PTX module

Static analysis of the CUDA driver path identified code that prepares data for `cuModuleLoadDataEx`.

The recovered host application contains an embedded high-entropy CUDA-related blob. The host allocates a buffer, copies this embedded blob, recovers it in memory, validates it as text, and loads it through the CUDA driver API via Rust CUDA bindings.

This report intentionally omits:

* the exact extraction offset;
* the exact extraction length;
* the full disassembly sequence needed to locate the blob automatically;
* the recovered key material;
* recovered IV material;
* extraction scripts;
* decryption scripts;
* the encrypted internal blob;
* the recovered PTX payload;
* hashes of internal hidden payloads.

The security-relevant conclusion is that the host binary contains a hidden CUDA module that is recovered in memory and loaded dynamically.

## CUDA/PTX obfuscation

Before recovery, the embedded CUDA blob does not expose normal PTX or CUDA markers. Static searches did not find ordinary plaintext indicators such as:

* PTX directives;
* CUDA target declarations;
* fatbin magic;
* common NVIDIA section names;
* obvious kernel entry declarations.

After local recovery, the module was confirmed to be NVIDIA PTX generated by the NVVM/CUDA toolchain. The recovered PTX is not published in this report.

The hidden CUDA payload appears to be PTX rather than a plain SASS cubin/fatbin.

The recovered PTX contains 41 `.entry` directives and 41 `.visible` directives.

Kernel-name indicators include references to:

```text
gpu_phase1_kernel
gpu_phase4d_ntt_kernel
gpu_fiat_shamir_squeeze_batch_kernel
gpu_final_hash_batch_kernel
```

This supports the conclusion that the binary includes real GPU proof/mining code rather than only fake miner UI strings.

## Runtime construction of CUDA recovery material

The CUDA module recovery material is not stored as one obvious contiguous plaintext array. Static analysis indicates that it is assembled at runtime from obfuscated constants inside the CUDA module-loading path.

This appears intended to defeat simple string scanning, binary grep, and naive extraction attempts.

This is best described as constant obfuscation rather than durable cryptographic protection. The binary must contain enough information to recover its own payload, so the protection is ultimately recoverable by determined static analysis.

This report does not publish the key material, IV material, exact extraction location, recovery script, or enough detail to reproduce the recovery automatically.

## Layered obfuscation model

The overall structure is layered:

```text
Public release archive
  -> packed outer Linux ELF
    -> custom loader
      -> hidden inner ELF recovery
        -> manually mapped Rust host application
          -> hidden embedded CUDA/PTX module
            -> runtime CUDA driver loading of recovered PTX
```

This layered design conceals both the CPU-side host/control logic and the GPU-side mining/proving implementation.

The layers serve different purposes:

* the outer layer hides the actual Rust host ELF;
* the loader layer frustrates tracing and instrumentation;
* the manual mapping layer avoids normal unpacked-file execution;
* the inner CUDA layer hides the PTX module;
* runtime construction of recovery material hides the module recovery path from simple static scanning.

None of these techniques alone proves wallet theft, credential theft, persistence, or remote compromise.

Taken together, they are unusual for a trustworthy open miner release. A miner distributed from a mostly empty repository as a packed binary with anti-debugging, manual ELF loading, hidden payloads, and hidden GPU kernels should be treated as high-risk.

## Programming language conclusion

The non-GPU parts were written in Rust.

The evidence is strong:

* Rust standard library source paths;
* Cargo registry paths;
* Rust panic strings;
* Rust async/runtime crate indicators;
* OXZD Rust project paths;
* Rust CUDA wrapper path indicators around CUDA module loading.

The GPU module is PTX generated by NVIDIA NVVM. The original GPU source language cannot be proven solely from the PTX header, but the module was compiled through the CUDA/NVVM toolchain.

The host-side code that recovers and loads the CUDA module is Rust.

## Security interpretation

The binary uses layered custom packing and payload hiding to conceal both its Rust host application and its CUDA/PTX mining code from ordinary static inspection.

The protections are recoverable by static analysis, but their presence materially increases user risk and reduces the trustworthiness of the release.

The strongest confirmed risks are:

* custom anti-analysis packer;
* hidden inner ELF;
* hidden embedded CUDA module;
* real mining/proving capability;
* dev-fee logic;
* networking capability;
* process-execution capability;
* binary-only distribution from a mostly empty public repository.

I did not find direct static-string proof of:

* SSH key theft;
* seed phrase theft;
* wallet-file theft;
* cloud metadata theft;
* cron/systemd persistence;
* generic downloader behavior.

The conservative interpretation is that the binary is not proven to be a wallet stealer or persistence implant by the static evidence collected so far. It is, however, an opaque packed miner with anti-analysis behavior, hidden miner/prover code, and dev-fee logic.

## User guidance

Users should not run this binary on a trusted workstation, server, GPU rig, wallet host, or machine containing credentials.

If analysis or execution is necessary, it should be done only in an isolated environment with:

* no wallet files;
* no SSH keys;
* no cloud credentials;
* no private repository credentials;
* no browser profiles;
* no production network access;
* no mounted host directories containing secrets;
* outbound network access disabled or tightly controlled unless dynamic network behavior is specifically being studied.

## Material intentionally not published

This report and repository intentionally do not include:

* original executable samples;
* the packed executable;
* recovered inner payloads;
* decrypted PTX;
* encrypted internal blob dumps;
* full strings dumps;
* extraction scripts;
* decryption scripts;
* decryption keys;
* IVs;
* key schedules;
* exact byte ranges for hidden payloads;
* exact virtual addresses of recovery functions;
* turnkey offsets and lengths;
* complete disassembly listings of recovery routines;
* pseudocode for decryptors or unpackers;
* instructions sufficient to automatically reconstruct hidden payloads.

## Verification material included

This report includes hashes of the public release artifacts analyzed.

### Release archive SHA-256

```text
e71b88a29039620e24c629398902658926456ddb264e01f01c84a5149568bd08
```

### Packed top-level binary SHA-256

```text
6c73b89031690505d74b38d6850dd4d049dfde3e6ef7274e4af61bea784d64ff
```

Internal artifact hashes are not published in this report. They are retained privately for chain-of-custody, reproducibility by the original analyst, and potential disclosure to trusted researchers, platform abuse teams, security vendors, or counsel.

## Conclusion

The HashForge/Nockhash release analyzed here is a real miner/prover binary with a custom anti-analysis packer and hidden CUDA/PTX module. The binary contains dev-fee logic, Stratum mining protocol support, CUDA GPU execution paths, hidden payload recovery, and anti-debugging behavior.

I did not find direct static-string proof that the binary steals wallets, exfiltrates SSH keys, establishes persistence, or downloads second-stage malware.

However, the packing, anti-analysis behavior, hidden inner ELF, hidden CUDA module, and opaque release structure make it inappropriate to run on trusted systems.

The most defensible conclusion from the current static evidence is:

> This is a high-risk opaque packed miner binary with hidden mining/proving code and dev-fee logic. It should not be treated as trustworthy. No direct static-string proof of wallet theft or persistence was found in the recovered host application.
