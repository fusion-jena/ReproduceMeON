# Beyond Similarity: LLM-Based Generation of Typed Semantic Links for Ontology Networks

Code, data, and evaluation materials for the paper submitted to SEMANTiCS 2026.

> *Author information withheld for double-blind review.*

---

## Repository layout

```
anonymous_submission/
├── data/
│   ├── ontologies/                              OWL files (4 domains, 33 ontologies)
│   │   ├── Computational/                       CSO, dockeronto, SWO, WICUS, ...
│   │   ├── Experiments/                         EXPO, SmartProtocols, ISA, ...
│   │   ├── Microscopy/                          OME, CMPO, omeroriken, ...
│   │   └── ML/                                  DMOP, MLSchema, OntoDM, OntoDT, ...
│   ├── extracted_triples/                       Pre-extracted RDF triple CSVs (one per domain)
│   ├── filtered_combined_sentences.csv          Context sentences for DistilBERT fine-tuning
│   ├── filtered_similarity_scores_clusters.csv  94,895 pre-filtered concept pairs
│   └── ReproduceMeON_Sample_annotated.xlsx      Expert-annotated relationships (429 pairs)
│                                                Annotator 1: all 4 domains (429 pairs)
│                                                Annotator 2: 3 domains (274 pairs)
├── figures/                                     Evaluation figures (PNG + PDF)
├── output/                                      Pipeline writes intermediate files here
├── 01_Pipeline.ipynb                            Full pipeline (ontology parsing → GPT-4o)
├── 02_Evaluation.ipynb                          Baselines, ablation, IAA, figures
├── requirements.txt                             Python dependencies
├── LICENSE                                      MIT
└── README.md
```

---

## Quick start

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

Python ≥ 3.9 recommended.

### 2. Reproduce all evaluation results and figures (no GPU, no API key required)

Open **`02_Evaluation.ipynb`** and run all cells.  
All pre-computed data is provided in `data/`. Figures are saved to `figures/`.

The notebook covers:

- Quantitative baseline comparison (Jaccard, TF-IDF, DistilBERT, Sentence-BERT vs. LLM pipeline)
- Ablation study (AUC-ROC on pre-filtered high-similarity pairs)
- Human annotation precision breakdown (by domain and by relationship type)
- Inter-Annotator Agreement (Cohen's κ per domain and overall)
- Error analysis by relationship type

### 3. Run the full pipeline (optional)

Open **`01_Pipeline.ipynb`**. Steps and requirements:

| Step | Description | Requirements |
|------|-------------|--------------|
| 1 — RDF Extraction | Parses OWL files → CSV triples | CPU |
| 2 — Sentence Generation | Builds context sentences from triples | CPU |
| 3 — DistilBERT Fine-tuning | MLM fine-tuning, 25 epochs, batch=16, lr=5e-5 | GPU recommended (~30 min on T4) |
| 4 — Embedding Generation | 768-dim mean-pooled embeddings per concept | GPU recommended |
| 5 — Clustering + Similarity | KMeans + 90th-percentile cosine threshold (800k → 95k pairs) | CPU |
| 6 — GPT-4o Generation | Typed relationship labelling with natural-language justification | OpenAI API key |

Set your OpenAI API key in the config cell of `01_Pipeline.ipynb`:

```python
OPENAI_API_KEY = "sk-..."   # or: export OPENAI_API_KEY=... before launching Jupyter
```

Steps 1–5 write intermediate files to `output/`. The evaluation notebook reads from `data/` directly if those files are already present.

---

## Data description

| File | Description |
|------|-------------|
| `data/ontologies/` | Source OWL files for all four domains (33 ontologies total) |
| `data/extracted_triples/rdf_triples_*.csv` | RDF (subject, predicate, object) triples with rdfs:label and rdfs:comment |
| `data/filtered_combined_sentences.csv` | 4,070 context sentences (class-level triples) used for DistilBERT fine-tuning |
| `data/filtered_similarity_scores_clusters.csv` | 94,895 concept pairs after 90th-percentile cosine similarity filtering |
| `data/ReproduceMeON_Sample_annotated.xlsx` | Expert-annotated sample of 429 GPT-4o-generated relationships |

### Annotation file — `ReproduceMeON_Sample_annotated.xlsx`

Four sheets, one per domain:

| Sheet | Pairs | Annotator 1 columns | Annotator 2 columns |
|-------|------:|---------------------|---------------------|
| Machine Learning | 137 | `Validation`, `Certainty` | `Validation_2`, `Certainty_2` |
| Computational | 94 | `Validation`, `Certainty` | `Validation_2`, `Certainty_2` |
| Experimental Workflow | 43 | `Validation`, `Certainty` | `Anotator_2`, `Certa` |
| Microscopy | 155 | `Validation`, `Certainty` | — (single annotator) |

`Validation` / `Validation_2` / `Anotator_2`: 1 = valid relationship, 0 = invalid.  
`Certainty` / `Certainty_2` / `Certa`: 1 = high certainty, 0 = uncertain.

Inter-annotator agreement computed in `02_Evaluation.ipynb` (Section 9):

| Domain | n | Cohen's κ | Agreement |
|--------|--:|----------:|----------:|
| Machine Learning | 137 | 0.671 | 87.6% |
| Computational | 94 | 0.817 | 94.7% |
| Experimental Workflow | 43 | 0.720 | 90.7% |
| **Overall** | **274** | **0.723** | **90.5%** |

κ = 0.723 falls in the *substantial agreement* range (Landis & Koch 1977: 0.61–0.80).

---

## Figures

All figures are generated by `02_Evaluation.ipynb` and written to `figures/`:

| File | Description |
|------|-------------|
| `roc_pr_curves.png/pdf` | ROC and Precision-Recall curves for all baselines and LLM pipeline |
| `baseline_comparison.png/pdf` | Precision / Recall / F1 bar chart per method |
| `ablation_similarity.png/pdf` | Cosine score distributions and AUC-ROC on pre-filtered pairs |
| `precision_breakdown.png/pdf` | Precision by domain and by relationship type |
| `iaa_agreement.png/pdf` | Cohen's κ and agreement percentage by domain |
| `error_analysis_rel_type.png/pdf` | Error rate and error count by relationship type |

---

## Reproducibility notes

- DistilBERT fine-tuning uses `seed=42`; minor numerical variation across hardware is expected.
- GPT-4o outputs are stochastic (`temperature=0.2`). The annotated XLSX reflects the original run; re-running Step 6 may produce different relationship labels or justifications for the same concept pair.
- The fine-tuned DistilBERT model weights are not included due to file size; run Steps 3–5 of `01_Pipeline.ipynb` to regenerate them.
- Pre-computed similarity scores (`filtered_similarity_scores_clusters.csv`) were produced with the fine-tuned model. To regenerate exactly, run Steps 3–5 first.
- All evaluation results and figures in `02_Evaluation.ipynb` are fully reproducible from the pre-provided files in `data/` without running the pipeline or requiring a GPU.
