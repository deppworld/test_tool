# How the Mutational Signature Caller Works

A plain-language explanation of what this tool does and why it matters.

---

## What Are Mutational Signatures?

Every cancer develops through DNA mutations - changes to the genetic code. But here's the fascinating part: **different causes of cancer leave different "fingerprints"** in the pattern of mutations they create.

Think of it like handwriting. Just as different people have distinctive handwriting styles, different cancer-causing agents (like UV light, tobacco smoke, or faulty DNA repair) create distinctive patterns of mutations.

Scientists have catalogued these patterns into what they call **mutational signatures**. The COSMIC database (Catalogue Of Somatic Mutations In Cancer) has identified dozens of these signatures, each linked to a specific cause:

| Signature | What Causes It |
|-----------|---------------|
| SBS7 | Ultraviolet (UV) light from sun exposure |
| SBS4 | Tobacco smoking |
| SBS1 | Normal aging (the "clock" signature) |
| SBS3 | Broken BRCA genes (breast/ovarian cancer risk) |
| SBS6 | Faulty DNA mismatch repair |
| DBS1 | UV light (seen as CC→TT changes) |

---

## What Does This Tool Do?

This tool takes a list of mutations found in a cancer sample and figures out **which signatures are present and in what proportions**.

It's like being a detective: given the evidence (mutations), determine which suspects (cancer-causing processes) were involved.

### Input
A VCF file - a standard format that lists all the mutations found in a tumor sample. This typically comes from DNA sequencing.

### Output
- A breakdown of which signatures contributed to the cancer
- Confidence metrics (how reliable is our analysis?)
- Visual charts showing the mutation patterns

---

## How Does It Work? (The Simple Version)

### Step 1: Read the Mutations

The tool reads your VCF file and extracts all the single-letter DNA changes (like A→T or C→G). These are called **SNVs** (Single Nucleotide Variants).

### Step 2: Look at the Neighborhood

For each mutation, we look at the letters immediately before and after it. Why? Because the surrounding DNA context matters!

For example, a C→T mutation might be caused by:
- UV light (if it's in a CC or TC context)
- Aging (if it's in a CG context)

This gives us 96 possible "types" of mutations (6 base changes × 16 surrounding contexts).

### Step 3: Count the Pattern

We count how many mutations fall into each of the 96 categories. This creates a "mutation profile" - essentially a fingerprint of this particular cancer.

### Step 4: Match to Known Signatures

Here's where the math happens. We have:
- The cancer's mutation profile (what we observed)
- Known signature profiles (what UV damage looks like, what smoking looks like, etc.)

The tool finds the **best combination** of known signatures that, when mixed together, recreates the observed pattern. It's like mixing paint colors - if your canvas is orange, the tool figures out you probably used red and yellow.

This uses a technique called **NNLS** (Non-Negative Least Squares) - a mathematical method that finds the best fit while ensuring no signature has a negative contribution (you can't have "negative UV exposure").

### Step 5: Report Results

The tool shows you:
- Which signatures were detected
- What percentage each contributes
- How well our reconstruction matches your data (quality score)

---

## Two Types of Analysis

### SBS (Single Base Substitutions)
The main analysis. Looks at single-letter changes with their surrounding context.
- 96 mutation categories
- Detects most signature types

### DBS (Doublet Base Substitutions)
A newer, complementary analysis. Looks for cases where **two adjacent letters** both mutated.
- 78 mutation categories
- Especially good for detecting UV damage (the classic CC→TT "sunburn" mutation)

The tool runs both analyses and lets you view results separately or combined.

---

## Understanding Your Results

### Cosine Similarity (Quality Score)
This tells you how well our signature combination explains your mutations.
- **≥95%**: Excellent - we can explain almost all mutations
- **90-95%**: Good - solid results
- **80-90%**: Moderate - some mutations aren't well explained
- **<80%**: Poor - the sample may have unusual signatures or too few mutations

### Why Might Results Be Poor?
- **Too few mutations**: Need at least 50-100 for reliable results
- **Novel signatures**: The cancer may have a signature not in the database
- **Mixed samples**: Contamination with normal tissue

### Mutation Count Matters
More mutations = more reliable results. Think of it like a survey - asking 1000 people gives better results than asking 10.

---

## What Can You Learn?

### Drilling Into Filtered Variants

The filter results preview shows a breakdown of how many variants were removed by each filter category (e.g. "Synonymous: 5", "Low VAF: 10"). You can **click any category** to expand an inline table showing exactly which variants were removed by that filter. From there you can search by gene or location, and selectively re-include variants you want to keep in the analysis.

### For Researchers
- Understand what biological processes drove a cancer's development
- Compare mutation patterns across patients
- Identify potential therapeutic targets

### For Clinicians
- Some signatures indicate specific treatment responses
- BRCA-deficiency (SBS3) suggests PARP inhibitor sensitivity
- MMR-deficiency (SBS6) suggests immunotherapy response

### For Patients (with clinician guidance)
- Understanding what caused your cancer
- UV signatures in melanoma confirm sun exposure role
- Smoking signatures confirm tobacco's contribution

---

## Targeted Panel (TSO 500) Analysis

### The Problem with Panels

Whole-genome sequencing (WGS) sequences all 3 billion letters of DNA. Targeted panels like TSO 500 only sequence about 1.33 million letters — roughly 0.04% of the genome. This creates two challenges for signature analysis:

1. **Very few mutations**: A panel might find only 5-50 mutations, compared to thousands from WGS
2. **Biased composition**: The panel only covers certain genes, so the trinucleotide composition is skewed

### How the Tool Handles Panels

When you select "Targeted Panel (TSO 500)", the tool applies three corrections:

1. **Opportunity correction**: Adjusts mutation counts to account for the biased trinucleotide content of the panel. Without this, the mutation profile would reflect the panel's gene selection rather than the true mutation process.

2. **Restricted signatures**: Instead of fitting against all 77+ signatures (which would massively overfit with so few mutations), it uses only 14 signatures that are spectrally distinct enough to be detected in panel data.

3. **TMB calculation**: Computes Tumor Mutational Burden (mutations per megabase), which is clinically relevant for immunotherapy decisions. TMB >= 10 mut/Mb is classified as "TMB-High".

### Confidence Tiers for Panels

The tool is honest about the limitations of panel-based signature analysis:

| Mutations | Confidence | What It Means |
|-----------|-----------|---------------|
| < 10 | Unreliable | Not enough data for any conclusions |
| 10-25 | Very Limited | Only very dominant signatures might be detected |
| 25-50 | Limited | Approximate signature detection possible |
| 50-100 | Moderate | Reasonably reliable for detectable signatures |
| > 100 | Good | Comparable to low-depth WES |

### Signature Detectability

Not all signatures can be reliably detected from panel data. Each signature gets a detectability rating:
- **High**: Strong spectral features, detectable even with few mutations (e.g., SBS4 tobacco, SBS7 UV)
- **Moderate**: Requires more mutations for reliable detection (e.g., SBS6 MMR, SBS18 ROS)
- **Low**: Hard to distinguish from noise with panel data (e.g., SBS5 clock-like)

---

## Limitations

1. **Signatures overlap**: Some signatures look similar, making them hard to distinguish
2. **Reference bias**: Only detects signatures in the COSMIC database
3. **Quantity needed**: Requires sufficient mutations for statistical power
4. **Not diagnostic**: This is a research tool, not a clinical diagnostic
5. **Panel limitations**: Targeted panel analysis is inherently less reliable than WGS/WES — treat panel signature results as exploratory, not definitive

---

## The Science Behind It

### Why 96 Categories?

DNA has 4 letters: A, C, G, T. A mutation changes one letter to another.

But we simplify by always looking from the perspective of C or T (the "pyrimidines"). This gives us 6 types of changes:
- C→A, C→G, C→T
- T→A, T→C, T→G

For each change, we consider what letter comes before and after (4 × 4 = 16 combinations).

6 changes × 16 contexts = **96 categories**

### Why Does Context Matter?

Different mutational processes have preferences. For example:
- **UV light** loves to mutate C when there's another C next to it (CC→TT)
- **APOBEC enzymes** prefer to mutate C when there's a T before it (TC→TT)

By looking at context, we can distinguish these different processes.

### The Fitting Process

Imagine you have a smoothie and want to know what fruits went into it. You know what pure apple, banana, and strawberry taste like. By carefully analyzing the flavor, you estimate: "This is probably 40% apple, 35% banana, 25% strawberry."

That's essentially what NNLS does with mutation signatures - it finds the mix of known signatures that best recreates the observed pattern.

---

## Glossary

- **SNV**: Single Nucleotide Variant - a single DNA letter change
- **DBS**: Doublet Base Substitution - two adjacent letters changing together
- **VCF**: Variant Call Format - standard file format for listing mutations
- **COSMIC**: Catalogue Of Somatic Mutations In Cancer - the signature database
- **Trinucleotide context**: The mutated letter plus its two neighbors
- **NNLS**: Non-Negative Least Squares - the math method for signature fitting
- **Cosine similarity**: A measure of how similar two patterns are (0-100%)

---

## Further Reading

- [COSMIC Signatures Database](https://cancer.sanger.ac.uk/signatures/) - Official source for signature data
- [Alexandrov et al., Nature 2013](https://www.nature.com/articles/nature12477) - The landmark paper that established mutational signatures
- [PCAWG Consortium, Nature 2020](https://www.nature.com/articles/s41586-020-1943-3) - Comprehensive analysis across cancer types
