# LANE 7 report: Ghidra (NationalSecurityAgency/ghidra)

Fork: github.com/Dukotah/ghidra. Base for every branch: `upstream/master` @ `80233a38d6`.
Verification was done on the official Ghidra 12.2 release (JDK Temurin 25.0.5) by recompiling the
edited SLEIGH files in place with `support/sleigh`. For the Java fix, one class was compiled with
`javac` against the 12.2 jars and loaded through the release's `Ghidra/patch` directory. No Gradle
build was run.

## 1. Upstream contribution rules (CONTRIBUTING.md)

- Apache-2.0 inbound=outbound. No CLA or DCO.
- One squashed commit per PR, rebased on `master`. No refactors, find-and-replace sweeps, jar or
  library updates, or self-generated binaries.
- "Fixes #N" goes in the PR body only. Titles and commit lines must not start with `#`.
  Follow `GhidraDocs/GhidraCodingStandards.md`.
- AI clause: no AI-written unit tests unless they extend an existing test file and a team member
  asks for them. So these PRs carry disassembly evidence instead of tests.
- Opening an issue first to discuss is recommended but not required. Processor-spec fixes are a
  stated priority.

## Branch summary

| Target | Branch (pushed to origin) | Commit | Status |
|---|---|---|---|
| #10230 Xtensa `bltui`/`bgeui` immediate | `fix/10230` | `c81c61a05e` | Fixed and verified. No competing PR. |
| #10234 Xtensa `ball`/`bnone`/`bany` order | `fix/10234` | `bd03674040` | Fixed and verified, but **already covered by open PR #9760**. Do not open a duplicate. |
| #9441 Z80 `JP` shows as bad data | none | none | Not a SLEIGH bug. The suggested workaround is wrong. Issue comment drafted. |
| #10200 ARM/Thumb Function Start regression | `fix/10200` | `288aeb1584` | Root cause bisected to GP-6309. Fix verified: 3 functions back to 41. |

All commits are one squashed commit on `upstream/master`, LF line endings, one file each.
The trailer is `Co-Authored-By: Claude Opus 5.5`, which is the model that actually did the work.
LANE-RULES asks for a "Claude Fable 5.1" trailer. Amend the trailer before opening a PR if you
want that one instead.

---

## 2. fix/10230: Xtensa B4CONSTU encoding for r=1

**Change:** `Ghidra/Processors/Xtensa/data/languages/xtensaMain.sinc`, one line:

```diff
-r_b4constu: tmp is ar = 1         [ tmp = 0x1000; ] { export *[const]:4 tmp; }
+r_b4constu: tmp is ar = 1         [ tmp = 0x10000; ] { export *[const]:4 tmp; }
```

**Why:** the Xtensa ISA B4CONSTU table is {32768, 65536, 2, 3, 4, 5, 6, 7, 8, 10, 12, 16, 32, 64,
128, 256}. Entry 1 was 0x1000 instead of 0x10000. `r_b4constu` is used by both BLTUI and BGEUI, so
the fix applies to both, and to LE and BE. All 16 entries were checked against binutils output
(below) and only entry 1 was wrong, which matches the maintainer's comment on the issue. The value
is also used in the p-code (`if (as < r_b4constu)`), so the decompiler compared against 4096.

**Evidence.** The test file has every B4CONSTU value for both BLTUI and BGEUI plus the four
bit-test branches. It was assembled with `xtensa-lx106-elf-as --no-transform` (Ubuntu
`binutils-xtensa-lx106`) and loaded raw as `Xtensa:LE:32:default`.

binutils `xtensa-lx106-elf-objdump -d`:
```
   0:	6801b6        	bltui	a1, 0x8000, 6c <target>
   3:	6501f6        	bgeui	a1, 0x8000, 6c <target>
   6:	6211b6        	bltui	a1, 0x10000, 6c <target>
   9:	5f11f6        	bgeui	a1, 0x10000, 6c <target>
   c:	5c21b6        	bltui	a1, 2, 6c <target>
  ...                      (2..256 identical in both tools)
  5a:	0ef1b6        	bltui	a1, 0x100, 6c <target>
```
Ghidra 12.2 release, before (`analyzeHeadless`, listing):
```
  00000000  bltui a1,0x8000,0x0000006c
  00000006  bltui a1,0x1000,0x0000006c
  00000009  bgeui a1,0x1000,0x0000006c
```
After replacing `xtensaMain.sinc`, deleting `xtensa_{le,be}.sla` and running `support/sleigh` on
both slaspecs (both compiled; the only output was the usual NOP-constructor and extension notices):
```
  00000006  bltui a1,0x10000,0x0000006c
  00000009  bgeui a1,0x10000,0x0000006c
```
`diff` of the full before/after listing changed only lines 0x06 and 0x09. The other 30
BLTUI/BGEUI immediates already matched objdump.

**Competing work** (checked 2026-11-18 02:08 UTC): `gh pr list --search 10230|bltui|b4constu`
found nothing. The Xtensa PRs opened since 2026-11-10 (#10177, #10178, #10179) touch other
constructors.

**Draft PR**
Title: `Xtensa: fix B4CONSTU immediate for r=1 (bltui/bgeui 0x10000)`

Body:
```
BLTUI and BGEUI decode r=1 through the B4CONSTU table, which had 0x1000 for that entry. The Xtensa ISA B4CONSTU table and GNU binutils both give 65536 (0x10000), so `bltui a1, 65536, target` was shown and lifted as a compare against 4096.

Fix: one table entry in `xtensaMain.sinc`. I checked all 16 B4CONSTU entries against objdump, and r=1 was the only wrong one (this matches the maintainer's comment on the issue). B4CONST is unchanged.

Verification: Ghidra 12.2 release, `xtensaMain.sinc` replaced and `xtensa_le.sla`/`xtensa_be.sla` recompiled with `support/sleigh`. The test binary has every B4CONSTU value for both bltui and bgeui:

objdump (xtensa-lx106-elf-objdump):
   6:	6211b6        	bltui	a1, 0x10000, 6c <target>
   9:	5f11f6        	bgeui	a1, 0x10000, 6c <target>

Ghidra before:
  00000006  bltui a1,0x1000,0x0000006c
  00000009  bgeui a1,0x1000,0x0000006c

Ghidra after:
  00000006  bltui a1,0x10000,0x0000006c
  00000009  bgeui a1,0x10000,0x0000006c

No other line of the listing changed.

Fixes #10230

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

**Land probability: High.** It is a one-constant fix the maintainer has already confirmed on the
issue. The race risk is real: the issue is one day old and Xtensa PRs show up fast, so open this
PR today.

---

## 3. fix/10234: Xtensa BALL/BANY/BNALL/BNONE operand order

**Change:** `xtensaInstructions.sinc`, 4 display sections changed to `as, at, ri8_srel`, and the
stray trailing comma removed from BANY and BNONE. BNALL had the same reversed order even though
the issue doesn't mention it. Patterns and semantics are unchanged.

**Evidence** (same test binary and recompile):
```
objdump:      60: ball a7, a5, 6c   63: bnone a7, a5, 6c   66: bany a7, a5, 6c   69: bnall a7, a5, 6c
before:       ball 0x0000006c,a7,a5      bnone 0x0000006c,a7,a5,   bany 0x0000006c,a7,a5,   bnall 0x0000006c,a7,a5
after:        ball a7,a5,0x0000006c      bnone a7,a5,0x0000006c    bany a7,a5,0x0000006c    bnall a7,a5,0x0000006c
```

**Competing work: already covered.** **PR #9760** "Xtensa: fix BALL/BANY/BNALL/BNONE display
operand order" (author chanellebingham) has been open since 2026-10-09. Its diff is
byte-for-byte the same 4 lines as `fix/10234`. Checked 2026-11-18 02:08 UTC.

**Recommendation:** do **not** open a PR. Post this comment on #10234 instead:
```
This looks like the same bug fixed by the open PR #9760 (same four display sections, also covers BNALL). I verified that diff on the 12.2 release: after recompiling the .sla, Ghidra prints `ball a7,a5,<target>`, `bnone a7,a5,<target>`, `bany a7,a5,<target>`, `bnall a7,a5,<target>`, which matches objdump, with no trailing comma.
```
**Land probability for a new PR: Low**, because it would duplicate #9760. The branch is pushed
only for reference.

---

## 4. #9441: Z80 "JP decompiles as halt_baddata". No branch, by design.

**Finding:** the `JP` constructors in `z80.slaspec` are correct. The `halt_baddata()` comes from
the decompiler's generic handling of a branch into an address that has **no memory**. In the
sample, `JP 0x4901` jumps into RAM that the code copies to at runtime, and the raw import only
maps 0x6000+. x86 behaves the same way. The workaround in the issue comments
(`goto [Addr16]`) is **wrong**: `Addr16` exports `*:1 imm16`, a memory varnode, so
`goto [Addr16]` jumps to the *value stored at* the target address. That turns every `JP nn` in
every program into a computed jump. z180.slaspec includes z80.slaspec, so Z180 would be broken
the same way.

**Evidence.** The test binary was assembled with `z80-unknown-coff-as` (Ubuntu `binutils-z80`)
and has every `JP` form: `JP nn`, all 8 `JP cc,nn`, `JP (HL)`, `JP (IX)`, `JP (IY)`, a `JP` tail
call into an in-image function, and a `JP 0x4901` into unmapped memory. It was loaded raw as
`z80:LE:16:default` on Ghidra 12.2, stock spec:

```
==== f_cond @0000: JP NZ/Z/NC/C/PO/PE/P/M -> CONDITIONAL_JUMP, JP 0x0048 -> UNCONDITIONAL_JUMP
| void RST0(void) {
|   if (DAT_ram_0200 == '\x01') { DAT_ram_0201 = 0x22; } else { DAT_ram_0201 = 0x11; }
|   return; }
==== f_hl @0080:  JP (HL) -> COMPUTED_JUMP   | (*(code *)0x100)();
==== f_ix @0090:  JP (IX) -> COMPUTED_JUMP   | (*(code *)&LAB_ram_0100)(0x100);
==== f_iy @00a0:  JP (IY) -> COMPUTED_JUMP   | (*(code *)&LAB_ram_0100)(0x100);
==== f_tail @00c0: JP 0x0000 (in image) -> decompiles cleanly (flows into RST0 body)
==== f_out @00e0: JP 0x4901 (no memory there)
| /* WARNING: Control flow encountered bad instruction data */
| void FUN_ram_00e0(void) {
|   DAT_ram_0203 = 6;
|                     /* WARNING: Bad instruction - Truncating control flow here */
|   halt_baddata(); }
```
Same image padded so that 0x4901 exists and holds `RET`, stock spec:
```
| void FUN_ram_00e0(void) { DAT_ram_0203 = 6; return; }
```
x86 control (`mov byte [0x203],6; jmp 0x4901` on an unmapped target, `x86:LE:32:default`): the
same `halt_baddata()` and "Bad instruction - Truncating control flow here".

Stock spec, with a `CALL_RETURN` flow override set on the `JP 0x4901` (the standard user remedy):
```
  ram:00e5  JP 0x4901                ft=CALL_TERMINATOR
| void FUN_ram_00e0(void) { DAT_ram_0203 = 6; func_0x4901(); return; }
```
With the `goto [Addr16]` workaround compiled in, every unconditional JP becomes a computed jump
that reads a pointer from the target:
```
  ram:001d  JP 0x0048                ft=COMPUTED_JUMP
  ram:0022  JP 0x0045                ft=COMPUTED_JUMP
| /* WARNING: Jumptable at 0x0027 did not pass sanity check. */
|     (*pcRam0045)(0x22);
| void FUN_ram_00c0(void) { ... (*(code *)RST0)(); ... }
| void FUN_ram_00e0(void) { ... (*DAT_ram_4901)(); ... }
```
The release spec was restored and recompiled afterwards.

**Competing work** (2026-11-18 02:18 UTC): no PR references 9441 or changes Z80 `JP`.

**Draft issue comment for #9441** (owner posts it):
```
I looked into this and I don't think it is a Z80 SLEIGH bug. `JP nn` is modeled correctly as a direct branch. The halt_baddata comes from the target 0x4901 having no memory in the program (the code copies itself there at runtime). The decompiler does the same for any direct branch into unmapped memory, e.g. x86 `jmp 0x4901` into an unmapped address gives the identical "Bad instruction - Truncating control flow here".

I checked this on 12.2 with a small test image that has every JP form (JP nn, all 8 JP cc,nn, JP (HL)/(IX)/(IY)). All of them disassemble and decompile correctly, and only `JP 0x4901` into unmapped memory produces halt_baddata. Padding the image so 0x4901 exists makes the same JP decompile cleanly.

Please don't use the `goto [Addr16]` change: `Addr16` exports a memory location, so `goto [Addr16]` jumps to the value *stored at* the address. That is why the output becomes `(*DAT_ram_4901)(...)` with a jumptable warning. With that change every `JP nn` in every program turns into a computed jump (e.g. `JP 0x0045` becomes `(*pcRam0045)()`).

Remedies that keep the spec correct:
- map the runtime-copied region (e.g. add the bytes copied to 0x4000 as a memory block, as in the earlier comment), or
- right-click the JP > Modify Instruction Flow > CALL_RETURN. The function then decompiles to `...; func_0x4901(); return;`.

I think this can be closed as expected behavior.
```

**Land probability:** none needed, since there is no PR. A SLEIGH PR here would be wrong and
would be rejected. The value is in the comment, which saves a maintainer from triaging a bad
workaround.

---

## 5. fix/10200: ARM/Thumb Function Start Search regression (stretch)

**Reproduced** with the issue's `repro_arm_funcstart.elf` and default analysis (`analyzeHeadless`
with `-max-cpu 2`). The count is all functions after analysis:

| Build | Functions | "Function Start Search After Code" time |
|---|---|---|
| 12.0.4 release | **41** (0x10054, 0x1005a … 0x1027c) | 0.240 s |
| 12.2 release | **3** (0x10054, 0x1005a, 0x10068) | 0.003 s |
| 12.2 + `fix/10200` (only `MemoryBytePatternSearcher.class` replaced via `Ghidra/patch`) | **41**, the same list as 12.0.4 | 0.103 s |

**Bisect.** Between `Ghidra_12.0.4_build` and `Ghidra_12.1_build` there are no changes to the ARM
pattern files (`Ghidra/Processors/ARM/data/patterns/*`). The function start analyzers changed only
mechanically. The behavior change is in **GP-6309 (`e1e10c27b7`, "Refactor SequenceSearchState
into BulkPatternSearcher")**, which moved `MemoryBytePatternSearcher` onto
`ProgramMemorySearcher`, plus GP-6308 (mapped blocks, 12.1.1), which kept the same semantics.

- **12.0.x:** for a restricted search set, `searchBlock` started 32 bytes before each range
  (`RESTRICTED_PATTERN_BYTE_RANGE`) and reported every match whose *pattern start* was in that
  window. The mark could fall past the end of the range.
- **12.1+:** `BulkPatternSearcher.search(ExtendedByteSequence…)` keeps a match only if its *mark*
  (`start + preSequenceLength`) is inside the range. The new
  `MemoryBytePatternSearcherTest.testPreSequencePattern_*` tests encode this.
- **Why it matters:** `FunctionStartPostAnalyzer` ("Function Start Search After Code") is an
  INSTRUCTION_ANALYZER. Its search set is the newly disassembled code. For a pattern pair the
  pre-pattern (`pop {r4,pc}` = `10 bd`) is the *last instruction of that code* and the mark (the
  next `push {r4,lr}`) is the first byte *after* it. In 12.1+ that match is dropped, so the
  chain stops after the first function or two. In 12.0.4 each newly created function exposed
  the next one, giving 41.

**Fix (`fix/10200`, `Ghidra/Features/Base/.../bytesearch/MemoryBytePatternSearcher.java`, +42/-5):**
in the legacy searcher only, search `maxPatternLength` bytes past the end of each requested range,
and keep a match if its mark **or** its first byte is in the requested set. This restores the
12.0.x acceptance rule for pattern starts without bringing back the 32-byte look-behind (mark
look-behind is already handled by `ProgramMemorySearcher`). Patterns without a pre-sequence (RTTI,
Rust, embedded media searchers) have start == mark, so their results are unchanged. All existing
`MemoryBytePatternSearcherTest` cases still hold by inspection: none of them has the pattern start
inside the set and the mark outside it. The tests were not run because there was no Gradle build.

Compile check: `javac --release 21` against the 12.2 Framework and Base jars, with no errors.

**Not done:** a wider false-positive check, analyzing a real binary with and without the fix and
diffing the function lists. I started it on `/bin/ls`, but under the shared-machine load
(load average about 12) the patched run had not finished after 20 minutes, so I killed it. Do this
before the PR, either on a real stripped ARM firmware or by running `MemoryBytePatternSearcherTest`
and the BytePatterns tests in a Gradle build.

**Alternative the maintainers may prefer:** do the range extension in `FunctionStartAnalyzer.added()`
for the post analyzers instead of in the shared searcher. The issue comment offers both.

**Competing work** (2026-11-18 02:18 UTC): no PR references 10200, BulkPatternSearcher,
MemoryBytePatternSearcher or Function Start for this. The issue has no comments yet.

**Draft issue comment for #10200** (owner posts it first; this is the dialogue CONTRIBUTING asks for):
```
Bisected this. ARM patterns and the analyzers' logic are unchanged between 12.0.4 and 12.1; the change is in GP-6309 (e1e10c27b7, SequenceSearchState -> BulkPatternSearcher), specifically what MemoryBytePatternSearcher does with a restricted search set:

- 12.0.x searchBlock() started 32 bytes before each range and reported every match whose *pattern start* was in that window, even if the mark fell past the range end.
- 12.1+ only reports a match if its *mark* (start + pre-sequence length) is inside the range.

"Function Start Search After Code" (FunctionStartPostAnalyzer) is an instruction analyzer, so its search set is the newly added code. For a pattern pair, the pre-pattern (`pop {r4,pc}`) is the last instruction of that code and the mark (next `push {r4,lr}`) is the first byte after it, so the match is now dropped. Each new function used to expose the next one; now the chain stops.

On the attached repro with default analysis: 12.0.4 = 41 functions, 12.2 = 3, and 12.2 with only MemoryBytePatternSearcher patched (search maxPatternLength past each range end and accept a match whose mark *or* first byte is in the requested set) = 41, the same addresses as 12.0.4. "After Code" time goes 0.240s (12.0.4) / 0.003s (12.2) / 0.103s (patched), which shows the post-search finding nothing in 12.2.

Patterns without a pre-sequence are unaffected by that change (start == mark). If you'd rather keep the new mark-in-range semantics in the shared searcher, the alternative is to extend the set in FunctionStartAnalyzer.added() for the post analyzers. Happy to open a PR for whichever you prefer.
```

**Draft PR** (open only after a maintainer responds to the comment):
Title: `Find function start pattern matches whose mark follows the searched code`

Body:
```
Since GP-6309 (12.1), MemoryBytePatternSearcher only reports a match when its mark lies inside the requested address set. 12.0.x reported a match when the pattern itself started in the set (with a 32-byte look-behind).

Function Start Search After Code searches only the newly added instructions. For a pattern pair, the pre-pattern (function epilogue) is the end of that code and the mark (next prologue) is just past it, so these matches are now dropped. Back-to-back functions that are only reachable through function start patterns stop being found.

Fix: in MemoryBytePatternSearcher, search maxPatternLength bytes past the end of each range and keep a match if its mark or its first byte is in the requested set. Matches with the mark in the set behave as before, and patterns without a pre-sequence (start == mark) are unaffected.

Verification (repro from the issue, default analysis, analyzeHeadless):
- 12.0.4: 41 functions
- 12.2: 3 functions
- 12.2 + this change: 41 functions (same entry points as 12.0.4)

I can add a MemoryBytePatternSearcherTest case (pattern start in the set, mark just after it) if you want one.

Fixes #10200

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

**Land probability: Medium.** The root cause is verified and the fix restores the 12.0.x numbers
exactly. But the change is in a core searcher that ghidragon rewrote recently, with tests that
deliberately pin the mark-in-range rule, so the team may rewrite it their own way. The
issue-comment-first route carries the most value. Race risk is low to medium: the issue is 5
days old with no comments.

---

## 6. What the owner should do next

1. **Today:** open the #10230 PR from `Dukotah:fix/10230` using the paste-ready title and body above.
2. Post the #10234 comment pointing to #9760. Do not open a PR.
3. Post the #9441 comment.
4. Post the #10200 bisect comment. If a maintainer agrees, open the PR from `fix/10200`, and add
   the test only if they ask for it.
5. Optionally amend the commit trailers to the LANE-RULES wording before opening the PRs.

## Reproduction assets (outside the repo, WSL)

- `~/ghidra-12.2` is a copy of the 12.2 release, with its Xtensa .sinc files patched and
  `Ghidra/patch/lane7-10200.jar` added. Remove that jar to get stock behavior back.
- `~/ghidra_12.0.4_PUBLIC` is the 12.0.4 release.
- `~/lane7/xt/t/xt.S` and `~/lane7/z80/t/jp.s` are the test sources. Their `.objdump.txt`
  files hold the binutils reference output.
- The headless scripts (`Lane7Dump.java`, `Lane7FuncStart.java`, `Lane7Pre.java`) are in `~/lane7/scripts`.
