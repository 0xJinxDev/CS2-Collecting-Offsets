# grok.md — N208 HANDOFF — FAIL PATH CLOSED; COHERENT SUCCESS STATE OPEN

**Last updated:** 2026-07-21, through N208  
**Current frontier:** enter the `0x2003c6 → 0x259b1ec` successor with coherent success-path state, then recover every validation node and both successors.  
**Project state:** original unmap/writer/return objective CLOSED; natural failure path to `DriverEntry RAX=0xC0000022` and unload CLOSED; `VAL_FAIL_SELECTOR_007` destinations CLOSED; full validation graph OPEN.  
**Critical N208 correction:** forcing only the alternate destination reaches the success frame but preserves fail-path state. The resulting dereference can be non-canonical, so this is edge proof, not proof of a valid success continuation.

---

# 0. PERMANENT MISSION CHARTER — NEVER DELETE

## Main objective

Recover the **complete validation graph executed by `vgk` DriverEntry**, not merely the first fail-path selector and not merely the path that happened to return `STATUS_ACCESS_DENIED` in the current environment.

For **every reachable validation/check** discovered:

1. Recover the surrounding basic block.
2. Disassemble the complete validation logic, including predicate construction.
3. Reconstruct the logic in C-like pseudocode.
4. Determine what property is being validated.
5. Identify every success and failure successor.
6. Follow both successors and continue recursively.
7. Continue until all reachable validation checks have been semantically recovered, or the unresolved input/decision origin is proved external to DriverEntry.

The known fail-path selector is only one node, not the final objective.

## Scope

### In scope
- Outer packed entry stub and its pre-dispatch checks.
- Live/decrypted REAL DriverEntry.
- All reachable internal dispatch stages, helpers, API-status checks, cookie/integrity checks, environment/object checks, fail-fast checks, and status-return selectors.
- Both normal/success continuations and all failure continuations reachable from DriverEntry.
- Cleanup checks executed inside DriverEntry when they affect or observe validation state.
- Indirect/computed control-flow edges needed to connect validation nodes.
- The terminal ntos DriverEntry-return test and unload path as the graph boundary.

### Boundary/context only
The generic Memory Manager leaf-PTE unmap machinery, service-manager exit-code translation, and pure status-packaging/MBA propagation are not DriverEntry validation nodes. Preserve them as terminal/context edges, but do not spend new research effort on them unless they hide a still-unresolved DriverEntry predicate.

## What counts as a validation/check node

Create a validation node when code does one or more of the following:

- Tests a value and selects different successors (`jcc`, conditional indirect transfer, `cmov`, `setcc`, table index, predicate-fed opaque dispatch).
- Compares an API return/status and changes control flow or final status.
- Validates a pointer, object, handle, registry/service state, symbolic link, code/data cookie, timing value, platform property, image state, or environment property.
- Selects fail-fast, failure-status materialization, cleanup, continuation, or success.
- Computes an opaque Boolean whose value controls a later branch.

Do **not** create a validation node for:

- Pure constant materialization.
- Reversible MBA that only encodes an already selected constant.
- Stack/register propagation of an already selected status.
- Generic dispatch with no recovered predicate yet; record it as a control-flow node/worklist item instead.
- Generic ntos/MM implementation details after DriverEntry has already returned failure.

## Definition of done

The mission is complete only when:

1. Every reachable conditional/semantic check from the outer entry and REAL DriverEntry has a closed validation-node record.
2. Each node has both successors identified, or an explicit proof that a successor is unreachable in this binary/environment.
3. Every reachable indirect transfer is resolved sufficiently to connect the validation graph, or its destination/origin is proved external.
4. Every leaf ends in one of:
   - successful DriverEntry return,
   - failed DriverEntry return with exact NTSTATUS,
   - fail-fast/bugcheck/exception terminal,
   - external routine whose remaining validation semantics are proven outside DriverEntry,
   - unreachable/dead code with evidence.
5. No unresolved worklist node remains merely because the known failure path was recovered.

---

# 1. PERMANENT DOCUMENTATION CONTRACT — NEVER DELETE

1. **`grok.md`** — clean handoff and active frontier. Rewrite after meaningful progress.
2. **`summary.md`** — self-contained current-session summary. Overwrite each session.
3. **`progress.md`** — cumulative evidence log. Append only.
4. **`validation_graph.json`** — authoritative semantic validation graph. Create if absent; update after every closed or newly discovered node.
5. **`validation_worklist.md`** — unresolved checks, indirect edges, missing successors, and required evidence. Create if absent; never silently drop entries.
6. **`validation_nodes/VAL_<id>.md`** — one durable record per validation node. Create if absent.

Treat undocumented knowledge as lost.

## Required validation-node schema

Every `VAL_<id>.md` and corresponding JSON node must contain the standard fields from `semantic_id` through `next_unresolved_dependency`. A node is not closed merely because its failing edge is known—the success edge must also be identified and placed on the worklist.

## Graph-edge types

```text
TRUE / FALSE | SUCCESS / FAILURE | STATUS_SUCCESS / NTSTATUS_FAILURE
COOKIE_OK / COOKIE_BAD | OBJECT_PRESENT / OBJECT_MISSING
DISPATCH_TARGET | CLEANUP | RETURN_STATUS | FAIL_FAST | EXTERNAL_CALL | UNREACHABLE
```

---

# 2. EVIDENCE STANDARD — NEVER DELETE

## Proof hierarchy

1. **PROVED_DYNAMIC**
2. **PROVED_STATIC**
3. **PARTIAL**
4. **PROVISIONAL**
5. **RETIRED**
6. **EXTERNAL**

## Critical operational invariants

- `GVA_EXEC_BREAK` GPRs are authoritative; stop-mode stack requires an observed pause and the correct vCPU.
- LBR/LER are empty on this host; **LBR enable-on-arm stays OFF** because it hangs QEMU.
- Live decrypted image may diverge from SOI. QEMU hit-vCPU `insn` bytes are authoritative for the executed site.
- Do not abandon a VALIDATED densify base merely because cloaked pages are unreadable.
- There are at most four shared hardware debug slots. Clear DR/BPs before exit and before switching mechanisms.
- `gva_dr_watch [-s] 0xGVA 8` only—never pass a log path to the HMP command.
- `gva_exec_break stop=1` is not reliably instruction-precise. Use the gdbstub for coherent register/state intervention.
- Host `/proc/<qemu>/mem` writes can bypass guest D-cache. A host-visible write is not proof the guest consumed the value.
- GDB hardware breakpoints conflict with `gva_exec_break`/DR because they share debug registers.
- Mid-frame densify in `0x1f177fc` must not require the fail cookie; R13 is clobbered inside the frame.
- Hot DEST/CFG nodes use nonstop logging plus semantic filters; sticky stop can flood or crash the guest.
- Never reclassify pure status/MBA packaging as a validation predicate without a controlling branch/property.
- Never claim a valid success path from a forced destination alone; coupled state must be coherent.

## Important retired claims / anti-regressions

- N21 exact invalidation-writer and `UNMAP_INV` overclaims remain retired.
- SC `WIN32_EXIT_CODE 31` is SCM-facing and is not the kernel producer BOOLEAN, DriverEntry C22, C365, or direct `RtlNtStatusToDosError` of the proved statuses.
- N28 provisional writer site is superseded by the N29 real zero writer.
- Predicted return-address breakpoints are retired; use RIP-matched function exits.
- N48b status-ret epilogue claim is retired; `0xb2d82b` is an MBA status stage.
- CR3 stack reads are not globally reliable. Trust only captures whose pause/vCPU/coherency conditions were verified.
- Do not return to parent-PDE overlays, N23 GDB deferred attribution, passive HOLD loops, or blind nearest-`jmp rax` rotation without new evidence.

---

# 3. CURRENT EXECUTIVE STATUS — N208

## A. Original narrow objective: CLOSED

The project has already proved the original protected-image teardown chain:

```text
vgk distributed leaf mapping disappears
→ exact leaf zero writer: ntos mov [rsi],rdi with RDI=0
→ producer function and RIP-matched exit
→ producer returns BOOLEAN 1
→ MmUnloadSystemImage / driver-load-failure chain
```

Key retained milestones:

- N28: distributed persistent target mapping removal.
- N29: true entrypoint leaf-PTE zero writer and real `UNMAP_INV_ID`.
- N39: target-bound function exit, RIP matched, RAX=1.
- N40/N40b: direct leaf-PTE unmap producer body and BOOLEAN semantics.
- N41/N42: upward chain through `MmUnloadSystemImage` and DriverEntry failure handling.

Do not spend additional work re-proving this chain unless a new validation predicate is hidden behind it.

## B. DriverEntry failure terminal: CLOSED

```text
REAL DriverEntry graph
→ DriverEntry returns 0xC0000022 STATUS_ACCESS_DENIED
→ ntos fail block promotes loader status to 0xC0000365 STATUS_FAILED_DRIVER_ENTRY
→ load-context path stores 0xC00000BB on the observed branch
→ MmUnloadSystemImage
```

SCM later reports Win32 31, but direct DOS mappings are:

```text
C22 → 5
C34 → 2
C00000BB → 50
C365 → 647
C0000001 → 31
```

Therefore Win32 31 is not a direct conversion of the proved kernel-path statuses.

## C. REAL DriverEntry and dispatch graph: PARTIAL but substantial

Closed live prefix:

```text
outer EP
→ REAL trampoline RVA 0x4c408
→ REAL body RVA 0x22722a
→ dispatcher 0x181e15
→ multi-target MBA graph
→ named cleanup/status nodes, including IoDeleteSymbolicLink("\\DosDevices\\vgk")
→ fail-selector control chain
```

The graph is update-resilient at the semantic layer, but the complete set of reachable validation predicates is not yet enumerated.

## D. VAL_FAIL_SELECTOR_007 destination encoding: CLOSED

### Natural fail edge

N127 proved the exact transfer:

```text
source RVA 0x1f1b999: jmp rax
RAX = module_base + 0x173b643
→ status materializer ENTRY 0x173b643
```

N129 proved the dual PE targets selected earlier in frame `0x1f177fc`:

```text
fail PE = 0x14173b643 → live RVA 0x173b643
alt  PE = 0x1402003c6 → live RVA 0x2003c6
```

Relocation lift:

```text
live_target = preferred_PE_target + (module_base - 0x140000000)
```

### Forced alternate edge

N201 used a gdbstub hardware breakpoint at the selector store and coherently set RDX to `0x1402003c6`. The subsequent jump reached `module_base+0x2003c6`. This proves the alternate encoded edge is executable.

### Node status

`VAL_FAIL_SELECTOR_007` remains **PARTIAL**, not CLOSED, because:

1. The exact environmental/semantic meaning of all selector inputs is not fully named.
2. The alternate edge has not been observed under naturally coherent success state.
3. The successor subtree has not been recursively recovered.

## E. Failure-packaging inputs: mechanically CLOSED, semantic labels partly OPEN

### Code6 / V2

```text
0x2ae09ed: rsi = 7 + (-1) = 6
0x2ae0a30: entry+0x1c8 = 6
0x1deef2e frame: propagates 6 to entry+0xf0 (V2)
VAL_007: test V2; only nonzero matters
```

The numeric constant is mechanically closed as opaque constant materialization. Its domain meaning (stage/check enum, if any) is still unknown.

### al slot

```text
0x176d3ab → entry+0x90, observed value around 0x70006e
VAL_007 later consumes the low-byte-derived Boolean
```

Writer is closed; semantic property is open.

### V1

```text
V1 = 0xaad9b040 | (mid16 << 16) | 0x8622
fail observation: bit62 clear
```

Packaging chain is closed:

```text
module_base
→ entry+0x110 = B & (A | (module_base + 0xC0F01))
→ 0x176c728 STASH/xform chain
→ dual-pack SRC
→ NOT
→ 0x176d584 writes V1 to entry+0x170
→ VAL_007 loads V1 and tests bit62
```

The variable mid16 is an ASLR/module-base fingerprint, not an environmental validation result. The fixed high bits controlling the branch still need a semantic explanation upstream.

## F. Success successor: CONTROL EDGE PROVED; VALID STATE OPEN

Forced path:

```text
0x2003c6
→ 0x259b1ec frame entry
→ r12 predicate preparation
→ pointer computation
→ 0x259b40c: mov r8, [rax]
```

N203/N204/N205 proved entry into the frame and progression to the dereference. N208 captured:

```text
RAX = 0xfffb8f7e9e67f438
```

This address is non-canonical and would fault before the later `bt rax,0x3e`. N205 observed a canonical pointer in another forced run, so the forced state is unstable rather than a valid success-path proof.

**Interpretation:** changing only the destination register does not update the coupled stack/register/cookie state that a natural success selection would carry. The success subtree therefore remains unvalidated.

---

# 4. AUTHORITATIVE PATH SPINES

## Failure destination/control spine

```text
R11 context 0x2c0590
→ dispatcher / opaque helper chain
→ DEST 0x1b9af6
→ CFG guard 0xaa850
→ sibling MBA/thunk chain
→ frame 0x1f177fc
→ source 0x1f1b999 jmp rax
→ fail materializer 0x173b643
→ packaging propagation
→ DriverEntry RAX=C22
→ ntos fail block
→ MmUnloadSystemImage
```

## VAL_007 input packaging spine

```text
2adc frame: publish code6=6 at entry+0x1c8
→ 1dee frame: publish V2=6 and entry+0x110 module-base-derived value
→ 176c frame:
   entry+0x90 al field
   entry+0x110 → STASH → E65F → SRC → NOT → V1 at entry+0x170
→ 1f177fc VAL frame:
   load/test V1
   test V2
   consume al-derived Boolean
   construct fail PE and alternate PE
   relocate selected PE to live GVA
→ indirect jump
```

## Alternate successor observed under force

```text
forced selected PE 0x1402003c6
→ live 0x2003c6
→ frame 0x259b1ec
→ pointer dereference at 0x259b40c
→ OPEN: coherent pointer state, subsequent predicates, both successors
```

---

# 5. ACTIVE WORKLIST — PRIORITY ORDER

## P0.D2 — obtain coherent success-path state

**Goal:** reach `0x2003c6 → 0x259b1ec` with state that corresponds to the alternate predicate, not merely a forced destination.

Required evidence:

1. Capture a complete register/critical-slot snapshot immediately before the dual-PE selection and at `0x259b1ec`.
2. Identify which upstream predicate inputs must change together: V1 bit62/fixed high bits, V2 nonzero state, al-derived Boolean, fail cookie, and pointer-packaging inputs.
3. Prefer changing the earliest coherent predicate/input with the gdbstub, not patching the final destination or host physical memory.
4. Re-enter the frame and prove the pointer at `0x259b40c` is canonical, mapped, and semantically identified.
5. Continue through every conditional node; record both successors in `validation_graph.json` and per-node files.

**Stop/abort conditions:**

- Non-canonical or unmapped dereference pointer.
- Fail cookie/state unchanged when the hypothesized success path requires a different package.
- Only the destination changes while downstream state remains identical to the fail run.
- A forced run faults before the first genuine successor predicate; document and do not call it success.

## P0.71g — entry+0x50 byte writer and meaning

Current evidence: byte `0x27` participates in the 1dee/module-base packaging calculations. Determine whether it is merely an opaque constant/field or a validation result. Do not promote it to a validation node unless it controls a branch or status.

## P0.74 — al-derived property

Writer `0x176d3ab` is closed, but the semantic source of the low-byte Boolean consumed by `VAL_007` remains open. Trace the value backward to its controlling property and both outcomes.

## P0.A — code6 meaning

Mechanics are closed (`7 + -1`, then nonzero test). Search for a genuine enum/table/semantic use only if it influences another predicate. If no such use exists, classify it permanently as opaque nonzero packaging and close the semantic work item as non-validation.

## P1 — success subtree recursion

Once coherent success entry is proved:

1. Close the pointer construction and dereference node in `0x259b1ec`.
2. Recover the later bit tests, dual target construction, and indirect exit.
3. Follow all success/failure successors recursively.
4. Identify successful DriverEntry return or the next validation frame.
5. Repeat until no reachable unresolved node remains.

---

# 6. CURRENT NODE TABLE

| Node / object | Status | Current evidence / missing work |
|---|---|---|
| Original leaf unmap writer + return | **PROVED** | N29/N39/N40 |
| DriverEntry C22 → unload | **PROVED** | N52 chain |
| REAL DriverEntry prefix | **PROVED/PARTIAL graph** | live body and dispatch spine recovered |
| `VAL_FAIL_SELECTOR_007` transfer source | **PROVED** | `0x1f1b999 jmp rax` |
| `VAL_FAIL_SELECTOR_007` fail destination | **PROVED** | natural `0x173b643` |
| `VAL_FAIL_SELECTOR_007` alternate destination | **PROVED edge** | forced `0x2003c6` via gdb |
| `VAL_FAIL_SELECTOR_007` semantics | **PARTIAL** | coupled predicate meaning and natural alternate state open |
| `SLOT_CODE6_ENTRY_1C8` | **PROVED** | `0x2ae0a30`, value 6 |
| `SLOT_AL_ENTRY_90` | **PROVED writer / semantic open** | `0x176d3ab` |
| `SLOT_V1_ENTRY_170` | **PROVED** | `0x176d584` |
| `SLOT_ENTRY_110_D4` | **PROVED** | function of module base |
| `FRAME_176C728_PACK` | **PROVED packaging** | al/V1 chains recovered |
| `SUCC_THUNK_2003C6` | **PROVED forced reachability** | not a coherent natural-success proof |
| `FRAME_259B1EC` | **PARTIAL** | entry/deref reached; pointer state and rest of graph open |
| Full success validation subtree | **OPEN** | not yet enumerated |
| Full mission definition of done | **OPEN** | unresolved reachable nodes/successors remain |

---

# 7. KEY ARTIFACTS

```text
n127_1f177fc_1784565913/      transfer source to fail materializer
n129_peva_1784578225/         dual PE targets and relocation lift
n135c_b7_1784587322/          code6 writer
n135e_a30_1784587490/         code6→V2 confirmation
n144_b0*/n144_b700*/          al and V1 writers
n177_b1_1784611132/           SRC materializer
n181*/n182*/n184*–n188*/      entry+0x110 and module-base-derived V1 chain
n201_gdb_1784619916/          forced jump to alternate RVA 0x2003c6
n203_frame_1784620815/        success-frame entry
n204_deep_1784621598/         deeper forced-frame predicates
n205_deref_1784622105/        dereference reached, canonical sample
n208_bt3e_1784622988/         non-canonical pointer under forced fail-state package
validation_nodes/VAL_FAIL_SELECTOR_007.md
validation_nodes/SUCC_THUNK_2003C6.md
validation_nodes/WRITER_AB_1DF1748.md
validation_graph.json
validation_worklist.md
```

---

# 8. HANDOFF VERDICT

## What is finished

- The original image-unmap investigation and internal producer return are finished.
- The observed DriverEntry failure and unload terminal are finished.
- The fail selector's exact transfer source and both encoded target addresses are finished.
- The code6/V2 and module-base-derived V1 packaging mechanics are finished.

## What is not finished

- A naturally coherent success selection has not been captured.
- The alternate successor frame has not been traversed safely beyond its first pointer dereference.
- The semantic property represented by the selector's fixed V1 bits/al-derived Boolean is not closed.
- The complete success subtree and potentially other reachable validation branches remain unenumerated.
- Therefore the permanent mission's definition of done is not met.

## Immediate next-session sentence

```text
Do not force only the final PE target again. Use gdb-coherent intervention or natural-state capture at the earliest coupled VAL_007 inputs, prove a canonical/mapped pointer at 0x259b40c, then recursively close every predicate and both successors in FRAME_259B1EC.
```
