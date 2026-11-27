# SYNCHRONIX v3.0.0 — Unified Hosted & Bare-Metal Self-Hosted Native Language

**Synchronix v3.0.0** unifies **Synchronix v2 (Hosted Main)** and **Synchronix Embedded v2 (Bare-Metal MCU)** into a single, 100% pure `.synchronix` self-hosted native programming language (zero C, C++, Rust, Python, Go, PowerShell, or Makefile source files) with full 1-to-1 architectural and feature parity across both editions.

Influenced by **Swift** (ARC, safety, modern design) and **Kotlin** (tracing GC, M:N concurrency, functional ergonomics), with integrated **$\Phi$ (Phi) Formal Logic** verification, **$\Lambda$ (Lambda) Functional Logic** closures, **compile-time `noalloc` & ISR safety enforcement**, **volatile MMIO / hardware barriers / interrupts**, and **dual-mode Hosted + Deterministic Embedded runtimes**.

## Unified Architecture Mapping (V2 Main + V2 Embedded $\rightarrow$ V3 Pure `.synchronix`)

| Domain | Subsystem | Unified v3 Capabilities | Pure Synchronix Source |
|---|---|---|---|
| **Portable & HAL** | ABI, MCU Memory Map & OS Helpers | Hosted x86_64/ARM64 + Bare-Metal Cortex-M4/ARM64/x86_64-bare ABIs, UART0/GPIOA/SysTick/NVIC MMIO defs | `include/portable_asm.synchronix`, `include/sx_defs.synchronix` |
| **Build System** | Self-Hosted Build & 4-Target Cross-Verifier | Compiles all 22 units, archives `libsynchronix_v3_host.a`, cross-verifies Hosted ARM64, Cortex-M4, ARM64-bare & x86_64-bare + 3 bare-metal `.a` RT archives | `build.synchronix` |
| **Phase 1: Frontend** | Lexer & Context Utilities | All V2 keywords + embedded hardware keywords, hex literals (`0x4002_0000`) w/ `_` separators, Flash/SRAM budget ctx | `src/frontend/lexer.synchronix`, `src/frontend/utils.synchronix` |
| **Phase 1: Frontend** | Parser (AST) | Generics `<T>`, Phi contracts, Lambda `=>`/`|>`, ARC/GC/Borrow/Move, `async`/`spawn`/`await`, `irq`/`noalloc`/`bare`, `Reg32`, `mmio_read`/`mmio_write`/`irq_disable`/`irq_enable`/`wfi`/`dmb` | `src/frontend/parser.synchronix` |
| **Phase 1: Frontend** | Semantic Analyzer & Safety Verifier | Type inference, Generics, Phi constant prover (incl. `%`), Lambda capture analysis + `noalloc`/`irq` allocation ban, ISR `await` ban, Phi MMIO alignment & null-addr prover | `src/frontend/sema.synchronix` |
| **Phase 2: Middle-End** | IR Generator $\rightarrow$ Unified SUMIR | Full V2 SUMIR + volatile hardware opcodes (`IR_MMIO_READ/WRITE`, `IR_IRQ_*`, `IR_WFI`, `IR_DMB`) | `src/middle/ir_gen.synchronix` |
| **Phase 2: Middle-End** | Memory Safety & SRAM Budgeter | Ownership/borrow checker, ARC retain/release insertion, GC shadow roots, static SRAM budget estimator (`--sram-kb`) | `src/middle/mem_safety.synchronix` |
| **Phase 2: Middle-End** | $\Phi$ & $\Lambda$ Optimizer | Zero-cost elimination of statically proven Phi proofs + ARC peephole elision | `src/middle/phi_lambda_opt.synchronix` |
| **Phase 2: Middle-End** | Volatile-Safe Lowering | SSA-safe constant folding, ISR-aware leaf inlining, volatile-preserving fixed-point DCE | `src/middle/lowering.synchronix` |
| **Phase 3: Backend** | Target-Specific Optimization | Linear-scan regalloc (`rbx/r12-r15`, `x19-x23`, `r4-r8`) + volatile-safe instruction scheduling | `src/backend/target_opt.synchronix` |
| **Phase 3: Backend** | 6-Target Native & Bare-Metal Codegen | **x86_64 Host**, **ARM64 Host**, **ARM Cortex-M4 Thumb-2**, **ARM64-bare**, **x86_64-bare**, **Unified LLVM IR** (with full `borrow`/`move` codegen across all targets) | `src/backend/codegen.synchronix` |
| **Phase 3: Backend** | Assembler & Linker Driver | `clang`/`lld` host native linking, hosted ARM64 cross-assembly, bare-metal `ld.lld --gc-sections` `.elf` linking + `llvm-size` | `src/backend/emitter.synchronix` |
| **Foundation: Runtime** | Dual-Mode Unified Memory Engine | **Hosted mode** (dynamic ARC + Tracing GC + 64 KB Arena, 1024 roots) + **Embedded mode** (64-slot static ARC/GC slabs + 4096 B Arena, peak watermarks, OOM telemetry) | `src/runtime/memory_rt.synchronix` |
| **Foundation: Runtime** | Unified M:N & RTOS Scheduler | **Hosted mode** (256-task M:N fibers) + **Embedded mode** (32-task bounded priority-aware RTOS scheduler + tick counter) | `src/runtime/async_rt.synchronix` |
| **Foundation: Runtime** | Dual-Mode $\Phi$ & Closure Runtime | **Hosted dynamic closures** + **Embedded 32-slot static closure pool** & freestanding Phi fault trap | `src/runtime/phi_lambda_rt.synchronix` |
| **Foundation: Stdlib** | Unified Standard Library (`std`) | Core math/generics + **Q16.16 fixed-point DSP**, **bitfields/popcount**, **CRC-8**, **circular ring buffer FIFO**, **host HAL simulator** | `src/runtime/stdlib.synchronix` |
| **Ecosystem Tooling** | Unified Cargo Package Manager & `size` | `new <name>` (hosted) & `new <name> --embedded` (Cortex-M4 `memory.ld` + firmware), `build`, `run`, `test`, `size` | `src/tooling/cargo_pkg.synchronix` |
| **Ecosystem Tooling** | Formatter & Unified Safety Linter | Canonical `fmt` + `[PHI-001]`, `[EMB-001]`, `[EMB-002]` lint rules | `src/tooling/fmt_lint.synchronix` |
| **Ecosystem Tooling** | Synchronix v3 LSP | JSON-RPC 2.0 server with all hosted + embedded keyword & intrinsic completions | `src/tooling/lsp.synchronix` |
| **Ecosystem Tooling** | Debugger & 3-Phase Profiler | DWARF `-g` LLDB debugger + 3-phase compiler/runtime/SRAM budget profiler | `src/tooling/debug_prof.synchronix` |
| **CLI Entry** | Unified `synchronix` v3 CLI Driver | All hosted + embedded subcommands, all 6 targets, dual-mode `--self-test` | `src/main.synchronix` |

## Build & Quick Start

```sh
# 1. Self-host rebuild + 4-target cross-verification (Hosted ARM64, Cortex-M4, ARM64-bare, x86_64-bare)
./build/sx_build.exe
./build/sx_build.exe all            # Build compiler & unified host archive only
./build/sx_build.exe clean          # Clean generated build artifacts

# 2. Run unified Hosted + Embedded self-hosted runtime verification
./build/synchronix.exe --self-test
./build/synchronix.exe --version

# 3. Compile, profile & run Hosted, Embedded, and Unified v3 showcases
./build/synchronix.exe profile examples/showcase.synchronix
./build/synchronix.exe run examples/v3_unified_showcase.synchronix
./build/synchronix.exe run examples/embedded_firmware.synchronix

# 4. Cross-compile for Hosted ARM64 or emit Unified LLVM IR
./build/synchronix.exe compile examples/showcase.synchronix --target arm64 -o build/showcase_arm64
./build/synchronix.exe compile examples/v3_unified_showcase.synchronix --emit-llvm -o build/v3_unified_llvm

# 5. Compile & link standalone bare-metal firmware ELFs (Cortex-M4, ARM64-bare, x86_64-bare)
./build/synchronix.exe compile examples/bare_blink.synchronix --target cortex-m --bare --noalloc -o build/bare_blink_m4
./build/synchronix.exe compile examples/bare_blink.synchronix --target arm64-bare --bare --noalloc -o build/bare_blink_a64
./build/synchronix.exe compile examples/bare_blink.synchronix --target x86_64-bare --bare --noalloc -o build/bare_blink_x64
./build/synchronix.exe size build/bare_blink_m4.elf

# 6. Verify all 4 static safety rejections (Use-After-Move, False Phi Proof, noalloc ISR, Unaligned MMIO)
./build/synchronix.exe compile examples/move_fail.synchronix
./build/synchronix.exe compile examples/phi_fail.synchronix
./build/synchronix.exe compile examples/noalloc_fail.synchronix
./build/synchronix.exe compile examples/mmio_fail.synchronix

# 7. Ecosystem tooling (Hosted & Embedded project scaffolding, fmt, lint, lsp)
./build/synchronix.exe new my_app
./build/synchronix.exe new my_firmware --embedded
./build/synchronix.exe fmt examples/v3_unified_showcase.synchronix
./build/synchronix.exe lint examples/embedded_firmware.synchronix
./build/synchronix.exe lsp
```

## License

Synchronix is licensed under the [Apache License, Version 2.0](./LICENSE). End-user terms are provided in [EULA.md](./EULA.md).
