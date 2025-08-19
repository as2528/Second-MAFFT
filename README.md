# second-mafft — hardened MAFFT re-run + crop/ungap (Python CLI)

> **Scope:** This repository contains **my Python implementation** of the “second MAFFT” step. It builds upon a collaborator's C code. In Key properties I have listed the capabilities I added to the code. 

---

## Overview

Run MAFFT a second time on an input FASTA, crop the alignment to the non‑gap span of a chosen reference sequence, emit both cropped‑gapped and cropped‑ungapped FASTA, and raise a length alarm when sequences deviate from the reference beyond a tolerance band. Filesystem and subprocess behavior is hardened for reliability and reproducibility.

**Key properties**

* Atomic writes (temp → `fsync` → atomic replace)
* Strict overwrite policy: `skip` | `fail` (default) | `replace`
* MAFFT stderr capture, timeout, and `.stderr.txt` emission
* Reference header configurable via `--reference_name`
* Extra MAFFT args via `--mafft_args "…"`
* Length alarm: only prints when *cropped & ungapped* lengths fall outside `[ (1−tol)·Lref, (1+tol)·Lref ]`

---

**Outputs** (default paths unless you override with CLI flags):

* `…/temp_aligned2.fasta` — raw MAFFT stdout (second pass)
* `…/NAMEalignedV2.fasta` — **cropped, still gapped**
* `…/NAMEcroppedV2.fasta` — **cropped, ungapped** (used for the length alarm)
* `…/*.stderr.txt` — MAFFT stderr (warnings/errors)

---

## CLI

```text
--input                 Input FASTA for the second MAFFT run (default: NAMEcroppedV1.fasta)
--temp_aligned          Path for MAFFT stdout (defaults into a temp dir)
--aligned_out           Output path for cropped, gapped FASTA
--cropped_ungapped_out  Output path for cropped, ungapped FASTA
--output_dir            Directory for outputs (default: new temp folder)

--overwrite             {skip, fail, replace}  (default: fail)
--mafft_timeout         Seconds before killing MAFFT (default: 3600)
--mafft_args            Extra MAFFT args appended after --auto (quoted string)

--reference_name        Header name of the reference sequence (default: target_sequence)
--length_tolerance      Allowed ±fraction around reference length (default: 0.30)
```

**Exit codes**

* `0` success
* `1` input/parse/fs errors, policy violations, MAFFT failures, or invalid crop window

---

## Overwrite policy

| Mode      | When file exists | Behavior                               |
| --------- | ---------------- | -------------------------------------- |
| `skip`    | yes              | Do nothing (keep existing file)        |
| `fail`    | yes              | Exit with error (safe default)         |
| `replace` | yes              | Atomic replace (temp → fsync → rename) |

---

## What’s materially different vs. earlier C implementation (for transparency)

*This section documents design deltas without bundling or claiming any third‑party code.*

1. **FS & process hardening**

   * Atomic writes for all outputs, consistent policy across files
   * Timeouts for MAFFT; stderr persisted to `<out>.stderr.txt`
2. **Configurable knobs**

   * `--reference_name`, `--length_tolerance`, `--mafft_args`
   * All outputs individually overrideable or grouped via `--output_dir`
3. **Alarm semantics**

   * Alarm evaluates **cropped & ungapped** lengths; prints only on violations with explicit allowed interval
4. **Defensive FASTA I/O**

   * Uppercases, strips spaces, hard caps (`MAX_SEQ_LENGTH`, `MAX_NUM_SEQUENCES`), and clear errors on empty input
5. **Deterministic crop window**

   * Compute `[start, end]` as first/last non‑gap in the **reference sequence**; validate `ref_len > 0`

---

## Attribution & Boundaries

* This Python CLI builds upon an earlier, separate, single‑file C tool developed by a collaborator for the “second MAFFT” workflow.
* I **do not claim authorship** of their work.

---

## Reproducibility & safety notes

* Requires `mafft` on PATH. Test with `mafft --version`.
* Large inputs are guarded by `MAX_SEQ_LENGTH=20000` and `MAX_NUM_SEQUENCES=15000` (adjust in code if needed).
* Cropping fails fast if the reference header isn’t present or has no non‑gap span.
* MAFFT failures/timeouts exit non‑zero and persist `stderr` alongside outputs.

---

## Licensing & credit

* Apache‑2.0 (patent grant)

---
