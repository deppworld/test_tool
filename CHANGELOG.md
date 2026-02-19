# Changelog

## [Unreleased]

### Added
- **Targeted Panel (TSO 500) Analysis Pathway**: New sequencing type selector that branches the analysis pipeline for targeted panel data.
  - Sequencing type toggle (WGS/WES vs Targeted Panel TSO 500) in the upload step
  - Opportunity correction to account for biased trinucleotide composition in panel data
  - Restricted signature set (14 spectrally distinct signatures) to avoid overfitting with few mutations
  - TMB (Tumor Mutational Burden) calculation with TMB-High/TMB-Low classification (10 mut/Mb threshold)
  - Panel-specific confidence tiers (unreliable / very limited / limited / moderate / good) calibrated for low mutation counts
  - Per-signature detectability badges (high / moderate / low) in results
  - Panel results banner with TMB display, confidence tier, detectability badges, and limitation caveats
  - DBS toggle hidden in panel mode (not applicable for targeted panels)
  - Panel metadata included in JSON export
  - WGS/WES pipeline remains completely unchanged
- **Clickable Filter Category Breakdown**: In the Filter Results Preview, each filter category (e.g. "Synonymous: 5", "Low VAF: 10") is now a clickable button that expands an inline table showing only the variants removed by that specific filter.
  - Search within each category by gene, location, or protein change
  - "Include" button per row to manually re-include variants in analysis
  - Counts update live as variants are manually included
  - Panel auto-closes when a filter category becomes empty (e.g. filter toggled off)
- Exported `RejectionReason` type from `VariantFilter.ts` for use across the application
- Added `removedByReason` map to `FilterResult` interface, tracking which variants were removed by each specific filter
