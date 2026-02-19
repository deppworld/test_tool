# Why TMZ.vcf is Not Being Called as SBS11

## Executive Summary

The mutational signature caller is failing to identify SBS11 (Temozolomide signature) in the TMZ.vcf file due to an **incorrect SBS11 reference profile** in the signature database. The synthetic profile previously used did not match the actual COSMIC v3.5 SBS11 signature pattern.

---

## Problem Identification

### Expected Result
TMZ.vcf should show **SBS11** as the dominant signature, as this sample is from a temozolomide-treated patient.

### Observed Result
The caller was not detecting SBS11 because the reference profile pattern was incorrect.

---

## Root Cause Analysis

### 1. Profile Pattern Mismatch

**Actual COSMIC SBS11 Pattern:**
SBS11 is characterized by C>T mutations where the **3' flanking base (position after the mutated C)** is C or T:

| Context | Probability | Percentage |
|---------|-------------|------------|
| T[C>T]C | 0.1582 | 15.8% |
| A[C>T]C | 0.1472 | 14.7% |
| C[C>T]C | 0.1262 | 12.6% |
| T[C>T]T | 0.1192 | 11.9% |
| G[C>T]C | 0.1152 | 11.5% |
| A[C>T]T | 0.1092 | 10.9% |
| G[C>T]T | 0.0697 | 7.0% |
| C[C>T]T | 0.0652 | 6.5% |

**Key Insight:** The dominant pattern is **N[C>T]C** and **N[C>T]T** - ALL four 5' bases show elevated C>T when followed by C or T.

**Previous Synthetic Profile (Incorrect):**
The synthetic profile incorrectly emphasized **C[C>T]N** contexts (5' base = C), which only captures a fraction of the true pattern.

### 2. Temozolomide Mechanism

Temozolomide (TMZ) is an alkylating chemotherapy agent that:
1. Methylates DNA at the O6 position of guanine
2. This O6-methylguanine mispairs with thymine during replication
3. Results in G:C → A:T transitions
4. When viewed from the pyrimidine strand: **C>T mutations**
5. The mutation occurs preferentially at **CpC** and **CpT** dinucleotides

This explains why **N[C>T]C** and **N[C>T]T** contexts (where the C is followed by C or T) are dominant - the methylation targets these specific dinucleotide contexts.

---

## TMZ.vcf Mutation Pattern

Analysis of the TMZ.vcf file shows predominant:
- **C>T** mutations (e.g., chr1:9777648 C>T, chr1:11174421 C>T)
- **G>A** mutations (complement of C>T, e.g., chr1:9775917 G>A)

These mutations occur across many trinucleotide contexts with the 3' base being C or T, which is consistent with SBS11.

---

## Resolution

### Fix Applied
The SBS11 profile has been replaced with the **actual COSMIC v3.5 signature data** containing the correct 96-channel probability values.

### Key Changes to SBS11 Profile:
1. **Before:** Synthetic values with incorrect context emphasis
2. **After:** Actual COSMIC v3.5 values with correct N[C>T]C and N[C>T]T dominance

### Dominant C>T Contexts (Corrected):
```
A[C>T]C: 0.147209  (was incorrectly low)
A[C>T]T: 0.109155  (was incorrectly low)
C[C>T]C: 0.126179  (was partially correct)
C[C>T]T: 0.065193  (was partially correct)
G[C>T]C: 0.115164  (was incorrectly low)
G[C>T]T: 0.069699  (was incorrectly low)
T[C>T]C: 0.158225  (was incorrectly low - THIS IS THE HIGHEST!)
T[C>T]T: 0.119169  (was incorrectly low)
```

---

## COSMIC Version Update

The documentation has been updated to reference **COSMIC v3.5** (previously stated v3.4).

---

## Verification Steps

After applying the fix, TMZ.vcf analysis should show:
1. **SBS11** as the dominant or highly contributing signature
2. High cosine similarity (>0.90) between observed profile and SBS11
3. Signature contribution percentage reflecting TMZ exposure

---

## References

- COSMIC SBS11: https://cancer.sanger.ac.uk/signatures/sbs/sbs11/
- COSMIC Signatures v3.5: https://cancer.sanger.ac.uk/signatures/
- Temozolomide mechanism: Alkylation of O6-guanine leading to G:C→A:T transitions

---

*Document generated: Analysis of SBS11 calling failure in TMZ.vcf*
