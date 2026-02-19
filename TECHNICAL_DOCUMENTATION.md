# Mutational Signature Caller - Technical Documentation

This document explains the architecture and implementation of the Mutational Signature Caller, a browser-based tool for decomposing cancer mutation profiles into COSMIC signatures.

## Overview

The tool performs two parallel analyses:
- **SBS (Single Base Substitution)**: 96-channel trinucleotide context analysis
- **DBS (Doublet Base Substitution)**: 78-channel dinucleotide analysis

Both use Non-Negative Least Squares (NNLS) regression to decompose observed mutation profiles into weighted combinations of known COSMIC signatures.

Additionally, the tool supports **Targeted Panel (TSO 500) analysis** with a specialized pipeline that applies opportunity correction, uses a restricted signature set, and calculates TMB.

---

## Architecture

```
Upload Step:  [Sequencing Type Selector] + [Genome Selector] + [File Drop]
                         │
                    (shared) VCF parse → annotate → gnomAD/ClinVar
                         │
Filter Step:     (shared) FilterPanel with all existing features
                         │
                    ┌────┴────┐
               WGS/WES    Panel (TSO 500)
                  │            │
              SBS + DBS    Opportunity correction
              pipeline     Restricted signature set (14 sigs)
                  │        TMB calculation
                  │        Panel-specific confidence
                    └────┬────┘
                         │
Results Step:   (shared) tabs + charts
                Panel mode adds: TMB banner, detectability badges, caveats
```

### WGS/WES Pipeline (Default)

```
VCF File
    │
    ▼
┌─────────────────┐
│   VCF Parser    │ ──► Extract SNVs, annotations, VAF, quality metrics
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ Context Fetcher │ ──► Fetch trinucleotide context from UCSC API (if missing)
└─────────────────┘
    │
    ├──────────────────────────────┐
    ▼                              ▼
┌─────────────────┐        ┌─────────────────┐
│ SBS Classifier  │        │ DBS Extractor   │
│ (96 channels)   │        │ (consecutive    │
│                 │        │  SNV pairs)     │
└─────────────────┘        └─────────────────┘
    │                              │
    ▼                              ▼
┌─────────────────┐        ┌─────────────────┐
│ NNLS Fitter     │        │ DBS Classifier  │
│ (SBS sigs)      │        │ (78 channels)   │
└─────────────────┘        └─────────────────┘
    │                              │
    │                              ▼
    │                      ┌─────────────────┐
    │                      │ NNLS Fitter     │
    │                      │ (DBS sigs)      │
    │                      └─────────────────┘
    │                              │
    └──────────────┬───────────────┘
                   ▼
           ┌─────────────────┐
           │  Results Panel  │
           │  (SBS/DBS/Both) │
           └─────────────────┘
```

### Panel (TSO 500) Pipeline

```
VCF File
    │
    ▼
┌─────────────────┐
│   VCF Parser    │ ──► Same parsing as WGS/WES
└─────────────────┘
    │
    ▼
┌────────────────────────┐
│ Panel Corrected Profile│ ──► Build raw 96-channel profile
│                        │     Apply opportunity correction:
│                        │       corrected[i] = raw[i] × (genomeOpp[i] / panelOpp[i])
│                        │     Normalize to probability distribution
└────────────────────────┘
    │
    ▼
┌────────────────────────┐
│ Panel NNLS Fitter      │ ──► Fit against 14 restricted signatures only
│ (restricted set)       │     (SBS1, SBS2, SBS4, SBS5, SBS6, SBS7a/b,
│                        │      SBS10a/b, SBS13, SBS15, SBS17b, SBS18, SBS44)
└────────────────────────┘
    │
    ▼
┌────────────────────────┐
│ Panel Confidence       │ ──► Mutation-count-based confidence tiers
│ + TMB Calculation      │     TMB = variants / 1.33 Mb
│                        │     Per-signature detectability ratings
└────────────────────────┘
    │
    ▼
┌────────────────────────┐
│ Results Panel          │ ──► TMB banner, confidence tier, detectability badges
│ (Panel mode)           │     No DBS toggle (not applicable)
└────────────────────────┘
```

---

## Key Components

### 1. VCF Parser (`src/services/vcf/VcfParser.ts`)

**Purpose**: Parse VCF files and extract single nucleotide variants.

**Key functions**:
- `parseVcfFile(file)` - Main entry point
- `extractSnvs(variant)` - Filter for single-base substitutions only
- `extractDbsVariants(variants)` - Find consecutive SNV pairs for DBS analysis

**DBS Extraction Logic**:
```typescript
// Sort variants by chromosome and position
// For each pair where pos[i+1] = pos[i] + 1 on same chromosome:
//   Combine into DbsVariant { ref: "XY", alt: "AB" }
```

**Annotations extracted**:
- Gene name (from VEP CSQ, SnpEff ANN, or direct INFO fields)
- Protein change (p. notation)
- VAF (from INFO AF/VAF or calculated from AD)
- Population AF (gnomAD, ExAC, 1000G)
- ClinVar significance

### Variant Filtering (`src/services/filter/VariantFilter.ts`)

**`FilterResult` interface** includes:
- `filtered` / `removed` / `protectedFiltered` — variant arrays
- `stats` — aggregate counts per filter category
- `removedByReason` — `Record<RejectionReason, Variant[]>` mapping each rejection reason to its specific variants, enabling per-category drill-down in the UI

**`RejectionReason`** (exported type): `'synonymous' | 'populationAF' | 'clinvarBenign' | 'dbsnp' | 'filter' | 'minDepth' | 'minAltReads' | 'minVaf' | 'strandBias'`

### Filter Panel (`src/components/filter/FilterPanel.tsx`)

The FilterPanel renders interactive filter controls and a real-time results preview. The breakdown items in the preview are **clickable buttons** — clicking one expands an inline table showing only the variants removed by that specific filter. Users can search within the table and manually re-include individual variants. Counts update live as variants are included. The expanded panel auto-closes if the corresponding filter is toggled off.

### 2. Context Fetcher (`src/services/context/ContextFetcher.ts`)

**Purpose**: Retrieve trinucleotide context for variants missing this information.

**API Used**: UCSC DAS server
```
https://genome.ucsc.edu/cgi-bin/das/{genome}/dna?segment={chrom}:{pos-1},{pos+1}
```

**Batching**: Variants are batched (default 50) to avoid overwhelming the API.

### 3. SBS Classifier (`src/services/context/ContextClassifier.ts`)

**Purpose**: Classify variants into 96 SBS channels.

**Channel Structure**:
- 6 substitution types (pyrimidine reference): C>A, C>G, C>T, T>A, T>C, T>G
- 16 flanking combinations per substitution (4 x 4 = 16)
- Total: 6 x 16 = 96 channels

**Strand Normalization**:
```typescript
// If reference is purine (A or G), take reverse complement
// Example: G>T with context AGA becomes C>A with context TCT
function normalizeToPyrimidine(ref, alt, fivePrime, threePrime)
```

**Output**: `MutationProfile` with 96-element count and normalized arrays.

### 4. DBS Classifier (`src/services/context/DbsClassifier.ts`)

**Purpose**: Classify DBS variants into 78 channels.

**Channel Structure**:
- 10 reference dinucleotides: AC, AT, CC, CG, CT, GC, TA, TC, TG, TT
- Variable alternates per reference (6-9 each)
- Total: 78 channels

**Strand Normalization**:
```typescript
// Use reverse complement if dinucleotide not in standard set
// Example: GT>CA becomes AC>TG (reverse complement)
```

**COSMIC Channel Order** (78 total):
```
AC>CA, AC>CG, AC>CT, AC>GA, AC>GG, AC>GT, AC>TA, AC>TG, AC>TT  (9)
AT>CA, AT>CC, AT>CG, AT>GA, AT>GC, AT>TA                       (6)
CC>AA, CC>AG, CC>AT, CC>GA, CC>GG, CC>GT, CC>TA, CC>TG, CC>TT  (9)
... (continues for all 10 reference dinucleotides)
```

### 5. NNLS Fitter (`src/services/signatures/NnlsFitter.ts`, `DbsFitter.ts`)

**Purpose**: Decompose mutation profile into signature weights.

**Mathematical Formulation**:
```
Given:
  A = Signature matrix (96×N for SBS, 78×N for DBS)
  b = Observed normalized profile (96×1 or 78×1)

Solve:
  minimize ||Ax - b||²
  subject to x ≥ 0

Where x is the signature weights vector
```

**Library Used**: `ml-fcnnls` (Fast Combinatorial Non-Negative Least Squares)

**Implementation**:
```typescript
import { fcnnls } from 'ml-fcnnls';
import { Matrix } from 'ml-matrix';

const A = new Matrix(signatureMatrix);  // channels × signatures
const b = Matrix.columnVector(profile.normalized);
const result = fcnnls(A, b);
const weights = result.K.getColumn(0);  // Extract solution
```

**Post-processing**:
1. Normalize weights to sum to 1
2. Reconstruct profile: `reconstructed = A × weights`
3. Calculate cosine similarity between observed and reconstructed
4. Filter signatures below 1% contribution threshold

### 6. Signature Data (`src/data/cosmic-signatures.ts`, `dbs-signatures.ts`)

**SBS Signatures** (12 included):
- SBS1: Spontaneous deamination (clock-like)
- SBS2, SBS13: APOBEC activity
- SBS3: BRCA1/BRCA2 deficiency
- SBS4: Tobacco smoking
- SBS5, SBS40: Clock-like (unknown)
- SBS6: MMR deficiency
- SBS7a, SBS7b: UV light
- SBS18: Reactive oxygen species
- SBS22: Aristolochic acid

**DBS Signatures** (8 included):
- DBS1: UV light (CC>TT dominant, ~85%)
- DBS2: Tobacco smoking
- DBS5: Platinum chemotherapy
- DBS7: MMR deficiency
- DBS11: APOBEC activity

**Data Format**:
```typescript
interface CosmicSignature {
  name: string;           // "SBS1", "DBS1", etc.
  profile: number[];      // 96 or 78 probability values
  etiology?: string;      // Biological cause
}
```

---

## Metrics and Quality Assessment

### Cosine Similarity
```typescript
cosine(a, b) = (a · b) / (||a|| × ||b||)
```
Measures how well the reconstructed profile matches the observed profile.
- ≥0.95: Excellent
- ≥0.90: Good
- ≥0.80: Moderate
- <0.80: Poor

### RMSE (Root Mean Square Error)
```typescript
RMSE = sqrt(Σ(observed[i] - reconstructed[i])² / n)
```

### Reconstruction Error
```typescript
error = (Σ residuals²) / (Σ observed²) × 100%
```

---

## Reproducing This Implementation

### Dependencies
```json
{
  "ml-fcnnls": "^3.0.0",    // NNLS solver
  "ml-matrix": "^6.11.1",   // Matrix operations
  "@gmod/vcf": "^5.0.10",   // VCF parsing
  "recharts": "^2.12.7",    // Charts
  "react": "^18.3.1"
}
```

### Step-by-Step Implementation

1. **VCF Parsing**
   - Use a VCF library or parse manually (tab-delimited)
   - Extract: CHROM, POS, REF, ALT, INFO, FORMAT fields
   - Filter for SNVs only (ref.length === 1 && alt.length === 1)

2. **Context Fetching**
   - For each variant, fetch position-1 to position+1 from reference genome
   - UCSC DAS API or local reference FASTA
   - Cache results to avoid repeated fetches

3. **Channel Classification**
   - Normalize to pyrimidine reference (reverse complement if needed)
   - Map to channel index using lookup table
   - Count mutations per channel, normalize to sum=1

4. **Signature Fitting**
   - Obtain signature matrix from COSMIC (or use provided data)
   - Implement NNLS or use library (ml-fcnnls, scipy.optimize.nnls)
   - Normalize weights, calculate reconstruction metrics

5. **Visualization**
   - Bar chart with 96/78 bars, colored by substitution type
   - Pie/bar chart for signature contributions
   - Overlay reconstructed profile for comparison

### Getting COSMIC Signature Data

Official source: https://cancer.sanger.ac.uk/signatures/downloads/

Files needed:
- `COSMIC_v3.5_SBS_GRCh38.txt` - 96-channel SBS profiles
- `COSMIC_v3.5_DBS_GRCh38.txt` - 78-channel DBS profiles

Format: Tab-delimited, channels as rows, signatures as columns.

---

## File Structure

```
src/
├── types/
│   └── index.ts              # TypeScript interfaces (incl. SequencingType, PanelAnalysisMetadata)
├── data/
│   ├── cosmic-signatures.ts  # SBS signature profiles (with restrictTo support)
│   ├── dbs-signatures.ts     # DBS signature profiles
│   └── panel-opportunities.ts  # TSO 500 opportunity vectors, restricted sigs, detectability
├── services/
│   ├── vcf/
│   │   └── VcfParser.ts      # VCF parsing, DBS extraction
│   ├── context/
│   │   ├── ContextFetcher.ts # UCSC API for trinucleotide context
│   │   ├── ContextClassifier.ts  # SBS 96-channel classification
│   │   └── DbsClassifier.ts      # DBS 78-channel classification
│   └── signatures/
│       ├── NnlsFitter.ts     # SBS NNLS fitting (with restrictTo support)
│       ├── DbsFitter.ts      # DBS NNLS fitting
│       ├── ConfidenceCalculator.ts  # Quality metrics (WGS/WES)
│       ├── PanelAnalyzer.ts       # Panel opportunity correction + TMB
│       └── PanelConfidence.ts     # Panel-specific confidence tiers
├── components/
│   ├── upload/
│   │   ├── SequencingTypeSelector.tsx  # WGS/WES vs Panel toggle
│   │   ├── GenomeSelector.tsx          # Genome build selector
│   │   ├── VcfUploader.tsx             # Upload orchestration
│   │   └── FileDropzone.tsx            # File input
│   ├── charts/
│   │   ├── TrinucleotideBarChart.tsx  # 96-channel chart
│   │   ├── DbsBarChart.tsx            # 78-channel chart
│   │   └── ContributionChart.tsx      # Pie/bar for signatures
│   └── results/
│       ├── ResultsPanel.tsx           # Main results with SBS/DBS toggle
│       └── PanelResultsBanner.tsx     # TMB, confidence, detectability
├── utils/
│   ├── cosineSimilarity.ts   # Vector math utilities
│   └── reverseComplement.ts  # DNA complement operations
└── App.tsx                   # Main application logic (branches WGS vs Panel)
```

---

## Panel (TSO 500) Analysis Pipeline

### Opportunity Correction (`src/services/signatures/PanelAnalyzer.ts`)

Targeted panels have biased trinucleotide composition. Opportunity correction re-weights observed counts:

```typescript
corrected[i] = rawCounts[i] * (genomeOpportunity[i] / panelOpportunity[i])
```

Where `panelOpportunity` is the count of each trinucleotide in the TSO 500 target regions, and `genomeOpportunity` is the genome-wide count.

### Restricted Signature Set (`src/data/panel-opportunities.ts`)

14 signatures selected for spectral distinctness and clinical relevance:
SBS1, SBS2, SBS4, SBS5, SBS6, SBS7a, SBS7b, SBS10a, SBS10b, SBS13, SBS15, SBS17b, SBS18, SBS44

The `getSignatureMatrix(restrictTo?)` function now accepts an optional filter parameter, and `fitSignatures(profile, restrictTo?)` passes it through.

### TMB Calculation

```typescript
TMB = variantCount / TSO500_CODING_REGION_MB  // 1.33 Mb
```

TMB >= 10 mut/Mb is classified as TMB-High (clinically relevant threshold for immunotherapy).

### Panel Confidence Tiers (`src/services/signatures/PanelConfidence.ts`)

| Mutations | Tier | Reliability |
|-----------|------|-------------|
| < 10 | unreliable | Not usable |
| 10-25 | very_limited | Only dominant signatures |
| 25-50 | limited | Approximate detection |
| 50-100 | moderate | Reasonably reliable |
| > 100 | good | Comparable to low-depth WES |

### Signature Detectability

Each restricted signature is rated for detectability from panel data:
- **High**: SBS1, SBS2, SBS4, SBS7a, SBS7b, SBS10a, SBS10b, SBS13
- **Moderate**: SBS6, SBS15, SBS17b, SBS18, SBS44
- **Low**: SBS5

---

## Testing

### Test Cases

1. **UV-exposed sample**: Should show high SBS7a/SBS7b and DBS1 (CC>TT)
2. **Smoking sample**: Should show high SBS4 and DBS2
3. **BRCA-deficient sample**: Should show high SBS3
4. **MMR-deficient sample**: Should show high SBS6 and DBS7
5. **No DBS mutations**: DBS toggle should be disabled, show "No DBS mutations detected"

### Validation

Compare results against:
- SigProfilerExtractor (Python)
- deconstructSigs (R)
- Signal (web tool)

Expected: Cosine similarity >0.95 for same input data.
