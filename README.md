# IDB Metadata Audit Pipeline v4 — LangGraph (Schema 3 Robust)

**nodalab.ai — Technical Screening Test**

## Architecture

```
START → ┌─ FCI Rules (Python) ─┐
        ├─ FCI Semantic (Haiku)├──→ State Aggregator
        └─ Embeddings (local) ─┘          │
                                          ▼
                                ┌─ Accuracy Batch 1 ─┐
                                ├─ Accuracy Batch 2  ├─→ Quality Gate
                                └─ Accuracy Batch 3 ─┘       │
                                                        ┌────┴────┐
                                                     PASS      FAIL
                                                        │         │
                                                        │    ReJudge (Haiku ×5)
                                                        │         │
                                                        └────┬────┘
                                                             │
                                                         Enrichment
                                                      (Embeddings → Haiku)
                                                             │
                                                        ┌────┴────┐
                                                   No disputes  Disputes
                                                        │         │
                                                        │    Critic (Sonnet)
                                                        │    + Override
                                                        └────┬────┘
                                                             │
                                                         Cross-Check
                                                    (FCI ↔ Accuracy ↔ Enrichment)
                                                             │
                                                         Reporter
                                                         (Sonnet)
                                                             │
                                                        Merge + Charts
                                                             │
                                                            END
```

## Tiered Model Usage

| Node | Model | Why | Volume | Cost |
|------|-------|-----|--------|------|
| FCI Semantic | Haiku | Simple quality checks | 150 | ~$0.15 |
| Accuracy (×3 votes) | Haiku | High volume, focused | ~3,600 | ~$3.60 |
| ReJudge (×5 votes) | Haiku | Enhanced prompt for failures | ~75 | ~$0.08 |
| Enrichment | Haiku | Candidate evaluation | ~1,200 | ~$1.20 |
| Critic | **Sonnet** | Deep reasoning on disputes | ~15 | ~$0.15 |
| Reporter | **Sonnet** | Audit narrative quality | ~20 | ~$0.20 |
| **Total** | | | **~5,060** | **~$5.40** |

## Robustness Features

### Quality Gate
After accuracy judging, checks:
- **Judged rate ≥ 95%** — did enough API calls succeed?
- **Unanimous rate ≥ 80%** — are votes consistent?
- If FAIL → re-judges disputed terms with enhanced prompt and 5 votes

### Critic Override
Sonnet reviews disputed verdicts and can **change** them:
- If 3 Haiku votes were split 1-1-1, Sonnet makes final call
- Override is tracked in `critic_verdict`, `critic_reasoning` columns

### Cross-Check
Validates consistency between modules:
- Keywords missing in FCI but all terms "Correct" → suspicious
- Enrichment suggests adding a term that was marked "Incorrect" → flag
- Abstract scored low quality but terms are fine → investigate
- >50% terms incorrect → needs full re-indexing

## Quick Start

```bash
pip install -r requirements.txt

# Full pipeline with LangGraph (~10 min)
python main.py --api-key sk-ant-your-key

# Sequential fallback (no LangGraph needed)
python main.py --api-key sk-ant-your-key --sequential

# Custom settings
python main.py --api-key sk-ant-... --n-votes 5 --max-workers 25 --n-report 30
```

## Outputs

```
outputs/
├── comparative_metadata_*.csv    # Final merged dataset
├── accuracy_detail_*.csv         # Per-term LLM verdicts + justifications
├── enrichment_detail_*.csv       # Enrichment suggestions with LLM eval
├── fci_semantic_*.csv            # Semantic quality per paper
├── crosscheck_*.csv              # Cross-module validation flags
├── audit_reports_*.json          # Sonnet narrative reports
├── audit_dashboard_v4.png        # Combined 6-panel dashboard
├── charts/                       # 10 individual chart PNGs
│   ├── 01_fci_distribution.png
│   ├── 02_llm_verdicts.png
│   ├── 03_llm_scores.png
│   ├── 04_llm_agreement.png
│   ├── 05_top_issues.png
│   ├── 06_enrichment_actions.png
│   ├── 07_overall_status.png
│   ├── 08_worst_papers.png
│   ├── 09_fci_semantic.png
│   └── 10_top_incorrect_terms.png
└── chart_data/                   # CSV data behind each chart
```

## Project Structure

```
├── main.py                        # Entry point + LangGraph/sequential runner
├── config.py                      # All parameters, model configs, thresholds
├── requirements.txt
├── src/
│   ├── pipeline_state.py          # TypedDict shared state
│   ├── graph.py                   # LangGraph orchestrator (Schema 3)
│   ├── nodes.py                   # All 12 node functions
│   ├── llm_engine.py              # Core LLM: single call, N-vote, parallel
│   ├── embedding_scorer.py        # Title-only embeddings (enrichment)
│   ├── consistency_checker.py     # FCI rule-based checks
│   ├── data_loader.py             # JSON-LD parser
│   └── charts.py                  # Visualizations + data export
```
