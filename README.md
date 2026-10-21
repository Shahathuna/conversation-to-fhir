# Conversation → Structured Clinical Record (SNOMED CT · FHIR)

**Turning doctor–patient conversations into coded, structured clinical history with large language models, PostgreSQL + pgvector, and HL7 FHIR.**

![Status](https://img.shields.io/badge/status-pilot%20study-orange) ![Python](https://img.shields.io/badge/python-3.10%2B-blue) ![FHIR](https://img.shields.io/badge/FHIR-R4-red) ![License](https://img.shields.io/badge/license-MIT-green)

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Shahathuna/conversation-to-fhir/blob/main/notebooks/Conversation_to_FHIR_Colab.ipynb)

> **Pilot results, not final.** This repository reports a first test on 24 public conversations. The aim is to show that the pipeline works end to end and to find what needs improving. See [Limitations](#limitations) and [Next steps](#next-steps).

---

## Why

During a consultation, much of the clinical history (presenting complaint, past illnesses, medications, allergies, family and social history) is spoken but only partly written down. Information missed from the record is lost to clinical care and to research. This project tests whether a language-model pipeline can **capture that history from the conversation itself** and turn it into **coded, interoperable records**.

## Pipeline

![Pipeline](figures/pipeline.png)

| Step | What happens | Tools |
|---|---|---|
| 1. Conversation | A transcript (MTS-Dialog, typed) or an audio recording transcribed by Whisper | Python, faster-whisper |
| 2. Entity extraction | The LLM lists every clinical statement with its history section, a clinical term, status (present / absent / past / uncertain), subject (patient or relative), details and a confidence score | Gemini |
| 3. SNOMED CT coding | Hybrid search (PostgreSQL full-text + pgvector semantic search, merged by reciprocal rank fusion, filtered by SNOMED hierarchy per section). The LLM then chooses one candidate or "none", and **codes outside the candidate list are rejected** | PostgreSQL, pgvector, Gemini embeddings |
| 4. FHIR R4 | Each finding becomes a FHIR resource in one Bundle per conversation, validated against the FHIR schema | `fhir.resources` |
| 5. Evaluation | Fact-level comparison with the expert-written note for each conversation | LLM judge + clinician review sheet |

FHIR mapping: Condition (complaints, history, review of systems; "absent" → `refuted`, "past" → `resolved`), MedicationStatement, AllergyIntolerance, FamilyMemberHistory (with relationship), Observation (smoking: LOINC 72166-2; alcohol, social), Procedure (surgical history). Every resource keeps the patient's own words in `note`, so each record can be traced back to the conversation.

## Pilot results

**Data:** 24 conversations from the official MTS-Dialog test set 1 (3 per history section, random seed 42), each scored against its expert-written note section.
**Model:** `gemini-3.5-flash-lite` for extraction, coding and judging. **Terminology:** a 78-concept demo list (not yet SNOMED CT).

| Measure | Result |
|---|---|
| Findings extracted | 94 |
| Facts in the expert notes | 88 |
| **Precision** (findings supported by the conversation) | **98.9%** (93 / 94) |
| **Recall** (expert-note facts captured) | **88.6%** (78 / 88) |
| **F1** | **93.5%** |
| Hallucination rate | 1.1% (1 finding) |
| Correct history section detected | 95.8% (23 / 24 conversations) |
| FHIR resources structurally valid | 100% |
| Findings given a code | 35.1%, limited by the 78-concept demo terminology |

![Accuracy by section](figures/accuracy_by_section.png)

### What the pilot shows

1. **Extraction is precise.** Only 1 of 94 findings was not supported by the conversation. Negations ("no sweating" → *Sweating, absent*), family members ("my dad had a heart attack" → *Family history of myocardial infarction, father*) and uncertain diagnoses ("could be angina" → *uncertain*) were handled correctly.
2. **Long narratives are harder.** Recall was lowest for the history of present illness (80%) and family/social history (80%), which mix many details and topics. The single hallucination was in family/social history.
3. **Self-reported confidence does not catch errors.** All 94 findings were given confidence ≥ 0.9, including the wrong one, so a confidence threshold removes nothing. Better uncertainty methods are needed (see Next steps).
4. **Coding works, but coverage needs the full terminology.** The codes inspected so far were correct, including synonyms ("short of breath" → *Dyspnoea*). Most uncoded terms (e.g. CT thorax, oxygen saturation) are simply not in the demo list.

Full tables: [`results/summary.csv`](results/summary.csv) · [`results/by_section.csv`](results/by_section.csv) · [`results/confidence_bands.csv`](results/confidence_bands.csv)

## Example

Input: [`examples/sample_consultation.txt`](examples/sample_consultation.txt), a synthetic chest-pain consultation. Part of the output:

| Section | Said | Clinical term | Status | Subject | Code |
|---|---|---|---|---|---|
| Chief complaint | "tight feeling in my chest" | Chest tightness | present | patient | DEMO-002 |
| Review of systems | "no sweating" | Diaphoresis | absent | patient | DEMO-007 *Sweating* |
| Past medical history | "type 2 diabetes" | Type 2 diabetes mellitus | present | patient | DEMO-020 |
| Medications | "Metformin 500 milligrams twice a day" | Metformin | present | patient | DEMO-044 |
| Allergies | "penicillin… a bad rash" | Allergy to penicillin | present | patient | DEMO-056 |
| Family history | "dad had a heart attack when he was 55" | Family history of myocardial infarction | present | father | DEMO-062 |
| Smoking | "20 a day for 20 years… stopped five years ago" | Ex-smoker | present | patient | DEMO-070 |
| Assessment / plan | "could be angina" | Angina pectoris | uncertain | patient | DEMO-028 |

One resulting FHIR resource: [`examples/sample_fhir_condition.json`](examples/sample_fhir_condition.json).

## How to run

Everything runs in the browser, with nothing to install locally.

1. **Database:** create a free PostgreSQL project at [neon.tech](https://neon.tech) and copy its connection string. Use the direct connection, not the pooler.
2. **Model:** create a Gemini API key at [Google AI Studio](https://aistudio.google.com).
3. **Notebook:** open the notebook with the **Open in Colab** button above. In Colab's 🔑 *Secrets* panel, add `GEMINI_API_KEY` and `NEON_DATABASE_URL` and enable notebook access.
4. **Run:** use *Runtime → Run all*. The notebook creates the tables and the pgvector index, loads the terminology, runs the pipeline on MTS-Dialog and the evaluation, then downloads the results.

For real SNOMED CT codes, register with [NHS TRUD](https://isd.digital.nhs.uk/trud), set `TERMINOLOGY = "snomed"` and add your TRUD API key. The notebook downloads and loads the release; embeddings are created on demand.

## Repository contents

```
notebooks/Conversation_to_FHIR_Colab.ipynb   the full pipeline and evaluation (Google Colab)
results/                                      pilot results as CSV
figures/                                      pipeline diagram and results chart
examples/                                     sample consultation and FHIR output
```

## Limitations

- **Small pilot:** 24 conversations, 3 per section, so per-section numbers are indicative only.
- **Short, single-topic conversations:** MTS-Dialog snippets are simpler than real consultations.
- **LLM judge:** the evaluator is the same model family as the extractor and may be lenient. A clinician review of a sample is planned, and the notebook exports a review sheet for this.
- **Demo terminology:** coding results are not yet SNOMED CT.
- **No audio evaluation yet:** speech-to-text is implemented but not evaluated, because MTS-Dialog has no audio.

## Next steps

- [ ] Load SNOMED CT (UK edition, NHS TRUD) with ICD-10 maps and re-evaluate coding
- [ ] Run all 169 history conversations, then the full test sets
- [ ] Test better uncertainty methods: self-consistency across repeated runs, cross-model agreement, retrieval distance for codes
- [ ] Clinician review of extraction and coding on a sample
- [ ] Compare models and prompt versions; keep measurements (severity, duration) as details of the main finding
- [ ] Evaluate full consultations and simulated audio with clinicians

## Data and citation

MTS-Dialog is used under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The notebook downloads it from the [official repository](https://github.com/abachaa/MTS-Dialog); no conversation data is redistributed here.

> Ben Abacha A, Yim W, Fan Y, Lin T. *An Empirical Study of Clinical Note Generation from Doctor-Patient Encounters.* EACL 2023, pp. 2291–2302. https://aclanthology.org/2023.eacl-main.168

SNOMED CT is not included. Use requires a licence (free in the UK via NHS TRUD).

## Author

**Shahathuna Mohamed** · MSc Health Data Science, Swansea University · shadumicro@gmail.com

Research prototype, developed with AI coding assistance. Not a medical device and not for clinical use.

## License

Code: [MIT](LICENSE).
