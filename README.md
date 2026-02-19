# Mutational Signature Caller

A React + TypeScript web application that analyzes VCF files to identify cancer mutational signatures using the COSMIC SBS database and NNLS algorithm.

## Features

- **VCF File Upload**: Drag-and-drop or click to upload VCF files
- **Genome Support**: GRCh37 (hg19) and GRCh38 (hg38)
- **Sequencing Type Selection**: WGS/WES and Targeted Panel (TSO 500) analysis modes
- **96-Channel Classification**: Maps mutations to COSMIC-standard trinucleotide contexts
- **NNLS Signature Fitting**: Non-negative least squares decomposition against COSMIC SBS signatures
- **Targeted Panel (TSO 500) Analysis**:
  - Opportunity correction for biased trinucleotide composition
  - Restricted signature set (14 spectrally distinct signatures)
  - TMB calculation with TMB-High/TMB-Low classification
  - Panel-specific confidence tiers and per-signature detectability badges
- **Interactive Visualizations**:
  - 96-channel mutation profile bar chart
  - Signature contribution pie/bar charts
- **Quality Metrics**: Cosine similarity, RMSE, and reconstruction error
- **Clickable Filter Breakdown**: Click any filter category in the results preview to drill into the specific variants removed by that filter, with search and manual re-include
- **Export**: Download results as CSV or JSON

## Getting Started

### Prerequisites

- Node.js 18+
- npm or yarn

### Installation

```bash
cd mutational-signature-caller
npm install
```

### Development

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser.

### Build

```bash
npm run build
```

### Run Tests

```bash
npm test
```

## Usage

1. Select your sequencing type (WGS/WES or Targeted Panel TSO 500)
2. Select your genome reference (GRCh37 or GRCh38)
3. Upload a VCF file containing SNV variants
4. The app will:
   - Parse SNV variants from the VCF
   - Fetch trinucleotide context for variants (if not in VCF INFO field)
   - Classify mutations into 96 channels
   - Fit COSMIC signatures using NNLS
   - Display results with visualizations

### VCF Requirements

- Standard VCF format (v4.0+)
- SNV variants (single nucleotide variants)
- Optional: Trinucleotide context in INFO field (CONTEXT, TRI, or TRINUC)

If trinucleotide context is not provided, the app will fetch it from UCSC/Ensembl APIs.

## Project Structure

```
src/
├── App.tsx                    # Main application component
├── components/
│   ├── upload/               # File upload + sequencing type selector
│   ├── results/              # Results display + panel banner
│   └── charts/               # Visualization components
├── services/
│   ├── vcf/                  # VCF parsing
│   ├── context/              # Trinucleotide context handling
│   └── signatures/           # NNLS fitting, panel analysis, confidence
├── data/                     # COSMIC signatures + panel opportunity data
├── types/                    # TypeScript definitions
└── utils/                    # Utility functions
```

## Key Algorithms

### 96-Channel Classification

Mutations are classified into 96 categories based on:
- 6 substitution types (C>A, C>G, C>T, T>A, T>C, T>G) - normalized to pyrimidine reference
- 16 flanking base combinations (4 × 4 for 5' and 3' bases)

### NNLS Signature Fitting

Given:
- **A**: COSMIC signature matrix (96 rows × N signatures)
- **b**: Observed mutation profile (96-element vector)

Solve: minimize ||Ax - b||² subject to x ≥ 0

Where **x** is the signature weights vector.

### Cosine Similarity

```
cosine(a, b) = (a · b) / (||a|| × ||b||)
```

Used to measure how well the reconstructed profile matches the observed profile.

## Technologies

- **React 18** - UI framework
- **TypeScript** - Type safety
- **Vite** - Build tool
- **Recharts** - Charting library
- **@gmod/vcf** - VCF parsing
- **ml-fcnnls** - NNLS algorithm
- **ml-matrix** - Matrix operations
- **Vitest** - Testing

## COSMIC Signatures

This tool uses COSMIC SBS (Single Base Substitution) signatures v3.5. The included signatures cover the most common mutational processes:

- **SBS1**: Spontaneous deamination of 5-methylcytosine (aging)
- **SBS2/13**: APOBEC cytidine deaminase activity
- **SBS3**: Defective HR repair (BRCA1/2)
- **SBS4**: Tobacco smoking
- **SBS5/40**: Unknown (clock-like)
- **SBS6/15/21/26/44**: Mismatch repair deficiency
- **SBS7a/b**: UV light exposure
- **SBS18**: Reactive oxygen species damage
- **SBS22**: Aristolochic acid exposure
- And more...

For the complete signature database, visit [COSMIC Signatures](https://cancer.sanger.ac.uk/signatures/sbs/).

## License

MIT

## Acknowledgments

- [COSMIC Mutational Signatures](https://cancer.sanger.ac.uk/signatures/)
- [mljs/fcnnls](https://github.com/mljs/fcnnls)
- [@gmod/vcf](https://github.com/GMOD/vcf-js)
