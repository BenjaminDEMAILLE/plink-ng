# b.1 stabilization: handoff notes (2026-09-30)

Goal: help Chang reach `v2.0.0-b.1.0`. Per #338 (his 2026-09-21 comment), master is
`v2.0.0-b.1-dev`, multipass merge landed (#493), and he tags b.1.0 once the first wave
of bug reports settles. Multiallelic dosage may stay mostly absent in b.1.

Plan (user approved all five points):
1. Provoke the bug wave ourselves: differential oracles vs 1.9, sanitizers, fuzzing on
   the ~45 commands added in September.
2. Clear our own PR backlog ("in or out for b.1?").
3. Interface stabilization: short list of proposals to Chang.
4. User-visible "under development" errors, ranked by impact.
5. Multiallelic dosage: ask Chang for the on-disk design before any code.

## State at end of 2026-09-30

- Point 2: DONE, recap posted on #344:
  https://github.com/chrchang/plink-ng/issues/344#issuecomment-5918309761
  Next: wait for his reply; rebase #476 and #450 if he agrees; close group 1
  (#401 #427 #425 #358 #357 #370) around 2026-10-07 if no reply.
- Point 1: four agents were still running at handoff (oracle stats/assoc, oracle
  LD/sets/regions, oracle import/export/merge, sanitizers + fuzz). Their local branches
  (`fix-b1-adjust-null`, `fix-b1-pmerge-mat-lock`, `fix-b1-set-name-dedup`, `san-b1`) had
  no commits yet. Check `git branch -v --list 'fix-b1-*'` and the agent worktrees under
  `.claude/worktrees/agent-*`. Nothing pushed, no PRs opened. Verify each bug on master
  yourself before opening a PR; add them to #510 or a new "b.1 stabilization" tracker.
- Points 3 to 5: report below, not yet posted. Show the draft to the user before posting.
- Shared binaries used by the agents (scratchpad, may be gone): plink2 master 17a8368f,
  plink19 1.9.1-dev. Rebuild: `cd 2.0/build_dynamic && SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX26.5.sdk make -j12`.

## Point 3: interface proposals (read from master 17a8368f)

Verified by hand:
- `--epistasis`/`--epistasis-boost` 'log10' is parsed (plink2.cc:3872, 6805) but
  `kfEpiLog10` is never read; header is always `P`, help promises -log10(p). Fix or reject.
- `--epistasis` help default "chrom,maybea1,orbeta,se,tz,p,err" (plink2_help.cc:1291)
  names keywords the parser rejects (`tz`, `err`).
- `--grm-maf` help says "MAF < 0.25 * sqrt(sample size)" (help:3030); code uses
  0.25 / sqrt(n) (plink2.cc:2998). Also new default: --pca/--make-rel error on rare
  variants; ask Chang to confirm explicitly.
- `--flip-scan 'ref-allele-based'` (plink2.cc:7719) vs 'ref-based' used by --r2/--epistasis.

From the review agent, not yet re-verified:
- `--meta-analysis`: 1.9 flag names (`-snp-field`, `-bp-field`) vs newer `-id-field`/
  `-pos-field`; defaults cannot read plink2 --glm output (looks for `SE`, `A2`; logistic
  writes `LOG(OR)_SE`, `OMITTED`/`AX`). plink2_adjust.cc:746.
- `--epistasis` QT summary `BEST_CHISQ` holds t^2 (plink2_epistasis.cc:2303); A1 columns
  named `ALLELE1/ALLELE2` though cols= keyword is `a1`.
- `--ld-score` writes `.ldscore[.zst]`, `.ldscore.M`, header `#CHROM POS ID L2`
  (plink2_ld.cc:13568, 13753); upstream ldsc expects `.l2.ldscore.gz`, `CHR SNP BP`.
- Founder / multiallelic defaults inconsistent: --ld-score needs `--ld-score-founders`
  (plink2.cc:9751) while others use founders-only + --nonfounders; --show-tags/--blocks/
  --test-mishap silently major-vs-rest, --condition errors without 'multiallelic'.
- `--blocks .blocks.det` header `BP1 BP2 NSNPS` vs --homozyg `POS1 POS2 NSNP`
  (plink2_ld.cc:14871); no zs, no cols=.
- `--test-mishap .missing.hap` fixed 1.9 header, no CHROM/POS (plink2_ld.cc:15910).
- `--set-table` header `SNP CHR BP` without '#' (plink2_set.cc:1406).
- `--show-tags` prints `NONE` for empty TAGS vs '.' elsewhere (plink2_ld.cc:15352);
  `--list-all`, `--tag-mode2` are top-level flags, could be modifiers.
- `--homozyg zs` only compresses .hom.summary (plink2_misc.cc:12032, 12171, 12250).
- `--tucc` puts T/U in IID suffixes, drops SID (plink2_family.cc:2368).
- `--twolocus` fixed extension, no zs/cols= (plink2_ld.cc:15661).
- `--epistasis-boost --covar` errors unless 'no-firth' (plink2_epistasis.cc:618) though
  help says firth-fallback is the default (help:1314).

## Point 4: "under development" errors, by user impact

1. VCF/BCF multiallelic dosage import (plink2_import.cc:3347, 8492). Needs MD writer.
2. `--make-bed multiallelics=` (plink2_data.cc:6401). Medium; PlanMultiallelicSplit exists.
3. `--pmerge` multiallelic + dosage (plink2_merge.cc:5709). Blocked on MD.
4. VCF/BCF export multiallelic rotation (plink2_export.cc:5284, 9014). Medium.
5. `--make-pgen multiallelics=` + `--sort-vars` (plink2_data.cc:10713). Small/medium.
6. `--epistasis` on case/control (plink2_epistasis.cc:1784).
7. `--flip-scan-ref-{p,b}file` (plink2.cc:7812): backend exists (plink2_ld.cc:16996),
   remaining work is founder filtering + multiallelic check. Easiest win.
8. multiallelic join `multiallelics=+` (plink2_data.cc:7505, 8542). Large.
9. BGEN multiallelic export/import (plink2_export.cc:3162, plink2_import.cc:14237).
10. `--merge-sids` (plink2_merge.cc:358). 11. `--pgen-diff` MD. 12. `--homozyg` 1.9 extras.

## Point 5: multiallelic dosage, questions for Chang

Defined today: track 4 = uint16 sum of ALT dosages; tracks 5-6 = delta-encoded
<sample x rarealt> list + 16-bit values, max 255 entries per sample
(pgenlib_misc.h:1024-1037); tracks 9-10 "analogous" for dphase (pgenlib_misc.h:1075);
spec says tracks 5-6/9-10 "not finalized" (pgen_spec.tex:635, 681). In-memory
PgenVariant fields exist (pgenlib_misc.h:748-765); no writer API.

Questions:
1. rarealt = ALT2+, ALT1 implicit as track4 minus sum: intended, or ALT1 explicit allowed?
2. Low-bit width of the rarealt index (ceil(log2(allele_ct-2)), zero for triallelic)?
   List header a varint count like difflists?
3. vrtype has no spare bits: must an empty track 5 always be written, and how?
4. Mode 10 (unconditional dosage) with >2 alleles: track 5 follows? Fixed-width modes
   0x03/0x04 counterpart in reserved 0x05-0x0f?
5. Tracks 9-10: subset bitarray over track-5 entries (like track 7 over 4), or separate list?
6. Header global-flag bit for "multiallelic dosage present" so b.1 readers refuse early?
7. Manhattan-distance hardcall consistency rule ("TBD", spec:588): required? --hard-call-threshold?
8. In-memory layout must match on-disk sample-major order? 255 cap kept?
9. Import: VCF DS Number=A directly? Per-allele HDS? GP ignored for multiallelic?

## State at end of 2026-09-30 (cloud session, second pass)

Point 1 done for this round: 3 agents (oracle vs 1.9, structural sweep, parser fuzzing)
plus manual checks found ~25 bugs; all verified by hand, fixed, with regression tests.
Full suite 72/72 normal + 72/72 ASan/UBSan (Linux x86-64, gcc, OpenBLAS).

8 branches pushed to the fork, each based on upstream master 17a8368, each built and
tested standalone, and merging cleanly together (simulated):
- fix-2-epistasis-blocks-alignment  (--blocks/--epistasis crashes at 400/448/800 samples;
  .epi.qt empty column; log10 no-op; ref-based rejected)
- fix-2-adjust-sidak-zero-p        (Sidak = 0 for p in ~1e-16..1e-27; abort on p = 0)
- fix-19-adjust-sidak-expm1        (same Sidak bug in 1.9)
- fix-2-test-mishap-labels         (all haplotype labels named the wrong allele)
- fix-2-strbox-dedup               (CopyAndDedupSortedStrptrsToStrbox never deduped;
  --make-set duplicate sets)
- fix-2-distance-nonautosomes      (chrX/Y/MT counted diploid; --parallel square)
- fix-2-malformed-inputs           (9 fuzz crashes, printf types, --adjust-file p>1 and
  long-allele buffer overflow, --meta-analysis buffer, --ld-score-annot parsing)
- fix-2-grm-maf-threshold          (error message and help formula)

Upstream PRs: this cloud session cannot reach chrchang/plink-ng through the GitHub App
(only the fork is installed), so the PRs are opened by the user from prefilled
compare links (title + body = commit messages).  Next: check that the 8 PRs exist,
follow their CI (sdist macos test_multithread is a known timing flake, pgenlib only).

Not fixed, for Chang to decide:
- --meta-analysis counts a within-file duplicate ID as extra studies (1.9 keeps first).
- --twolocus ignores ploidy (chrY females, haploid hets).
- --epistasis-boost STAT can print -3e-15; BEST_CHISQ likewise.
- --glm linear/logistic write "nan" A1_FREQ/MACH_R2 for all-missing variants
  (multinomial writes NA).
- --flip-scan chrX males coded 0/2 vs 1.9's 0/1 (undocumented difference).
- 1.9: --fast-epistasis boost prints P=0 for tiny negative STAT; --set-table with 0 sets
  writes no newlines.
Idea to propose: __attribute__((format(printf,1,2))) on log*printf* helpers would have
caught 12 format bugs at compile time.
