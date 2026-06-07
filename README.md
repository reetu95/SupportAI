### SupportAI

**SupportAI** is an end-to-end applied AI prototype for duplicate question detection and support-ticket deflection.

The project explores how classical NLP, transformer fine-tuning, hybrid retrieval, and confidence-based routing can be combined to simulate the intelligence layer of a customer support automation system.

This is **not yet a production-ready application**. It is complete as a modeling, retrieval, and evaluation prototype.

---

## Business Objective

Customer support teams often receive repeated questions written in different ways.

Examples:

* I forgot my password.
* I cannot log in.
* How do I recover my account?
* Can someone help me reset my password?

Different wording can represent the same intent.

The goal of SupportAI is to detect semantically similar support questions, retrieve relevant support answers, and decide whether to:

* answer automatically
* suggest a possible response
* escalate to a human agent

The main business goal is not only high accuracy. The system must also avoid **false deflection**, where a customer receives the wrong automated answer instead of being escalated.

---

## Project Background

This project started from an older NLP idea around question-pair similarity using the Quora Question Pairs dataset.

I revisited the problem from a modern AI engineering perspective by combining:

* classical NLP baselines
* transformer fine-tuning
* model comparison
* retrieval-augmented generation concepts
* hybrid retrieval
* confidence-based decision routing
* custom validation
* failure analysis

---

## Datasets Used

### 1. Quora Question Pairs

Used for duplicate-question classification.

The task was to predict whether two questions are semantically duplicates.

Columns used:

* `question1`
* `question2`
* `is_duplicate`

This dataset acted as a proxy dataset for learning semantic duplicate detection.

### 2. Bitext Customer Support Dataset

Used as the support knowledge base for the retrieval layer.

Columns used:

* `instruction`
* `category`
* `intent`
* `response`

This dataset helped simulate a customer support environment with support queries, intents, categories, and responses.

---

## System Pipeline

The final prototype pipeline is:

```text
User question
   -> Hybrid retrieval
   -> ELECTRA duplicate classifier / reranker
   -> Confidence-based decision routing
   -> Answer, suggest, or escalate
```

---

## What I Built

### 1. Classical NLP Baseline

I first built a baseline using:

* TF-IDF vectorization
* Logistic Regression
* question-pair text combination
* classification metrics
* confusion matrix analysis

Baseline results:

| Metric    | Score |
| --------- | ----: |
| Accuracy  | 77.7% |
| Precision | 76.1% |
| Recall    | 57.7% |
| F1-score  | 65.6% |
| ROC-AUC   | 83.9% |

The baseline had decent accuracy, but recall was low. It missed many duplicate questions, which is important in support-ticket deflection because missed duplicates mean missed automation opportunities.

---

### 2. Transformer Fine-Tuning

I fine-tuned and compared multiple transformer models:

* DistilBERT
* BERT
* RoBERTa
* ALBERT
* ELECTRA
* MiniLM
* XLM-RoBERTa
* DeBERTa variants

Best model:

| Model   | Accuracy | Precision | Recall | F1-score | ROC-AUC |
| ------- | -------: | --------: | -----: | -------: | ------: |
| ELECTRA |    89.0% |     84.4% |  85.8% |    85.1% |   95.3% |

ELECTRA was selected because it gave the best overall balance across F1-score and ROC-AUC.

RoBERTa was also strong and was kept as a comparison model.

---

### 3. RAG / Retrieval Layer

I built a support knowledge base using the Bitext dataset and compared three retrieval methods:

* FAISS vector search
* BM25 keyword search
* Hybrid retrieval

Retrieval results:

| Retrieval Method    | Recall@1 | Recall@5 | MRR@5 |
| ------------------- | -------: | -------: | ----: |
| FAISS Vector Search |    96.3% |    99.0% | 97.4% |
| BM25 Keyword Search |    93.7% |    99.1% | 96.0% |
| Hybrid Retrieval    |    97.5% |    99.2% | 98.2% |

Hybrid retrieval performed best because it combined semantic similarity with keyword matching.

---

### 4. End-to-End Pipeline

After retrieval, I used the fine-tuned ELECTRA classifier to rerank retrieved results and estimate duplicate probability.

The system then made a decision based on confidence:

| Confidence Level  | Decision                  |
| ----------------- | ------------------------- |
| High confidence   | Answer automatically      |
| Medium confidence | Suggest possible answer   |
| Low confidence    | Escalate to human support |

---

## Evaluation

### Clean Evaluation

On the cleaner support dataset evaluation, the end-to-end pipeline performed strongly:

| Metric            |  Score |
| ----------------- | -----: |
| Intent Accuracy   |  97.2% |
| Category Accuracy | 100.0% |
| Answer Rate       |  88.8% |
| Suggest Rate      |   6.0% |
| Escalation Rate   |   5.2% |

However, these results were very high because the Bitext dataset is clean and contains many paraphrases per intent.

---

### Custom Validation

To test the system more realistically, I created a small custom validation set with messier support queries.

Custom validation results:

| Metric                  | Score |
| ----------------------- | ----: |
| Known Category Accuracy | 77.8% |
| Known Intent Accuracy   | 66.7% |
| Decision Accuracy       | 60.0% |

This exposed an important limitation: the classifier was fine-tuned on Quora-style question pairs, not real support-domain question pairs.

---

## What Went Wrong and How I Handled It

### 1. Baseline recall was low

The TF-IDF + Logistic Regression baseline had a recall of 57.7%.

This meant the model missed many duplicate questions.

How I handled it:

* Used the baseline as a benchmark.
* Moved to transformer fine-tuning for better semantic understanding.
* Compared multiple transformer models instead of relying on one.

---

### 2. Some transformer models were unstable

DeBERTa variants produced unstable or poor results in this setup.

One model produced invalid scores, and another collapsed toward majority-class behavior.

How I handled it:

* Added safer metric calculation.
* Used `zero_division` handling for precision and recall.
* Compared models using F1-score, recall, precision, and ROC-AUC.
* Did not select unstable models even if they were theoretically strong.

---

### 3. Accuracy alone was not enough

A model can have good accuracy but still miss many duplicates.

How I handled it:

* Evaluated precision, recall, F1-score, and ROC-AUC.
* Used F1-score as a balanced model-selection metric.
* Looked at recall carefully because support deflection depends on catching duplicate queries.
* Looked at precision carefully because wrong automated answers create business risk.

---

### 4. Retrieval metrics looked too high

The Bitext retrieval results were very high.

This happened because the dataset is clean and contains many similar paraphrases per intent.

How I handled it:

* Created a more realistic custom validation set.
* Compared clean benchmark performance with custom query performance.
* Reported the gap honestly instead of hiding it.

---

### 5. The classifier had a domain gap

The duplicate classifier was trained on Quora question pairs, but the final use case was customer support.

This caused errors on messy support queries.

How I handled it:

* Identified the issue through custom validation.
* Used confidence thresholds to reduce unsafe automatic answers.
* Marked support-domain fine-tuning as the next major improvement.

---

### 6. Threshold tuning changed system behavior

Lower thresholds answered more questions but increased business risk.

Higher thresholds were safer but escalated more queries.

How I handled it:

* Tested different answer and suggest thresholds.
* Prioritized avoiding false deflection.
* Designed the system to answer, suggest, or escalate instead of forcing every query into an answer.

---

## Key Learnings

This project showed that AI engineering is not only about training the best model.

Important tradeoffs included:

* precision vs recall
* automation vs escalation
* clean benchmark performance vs real-world behavior
* model accuracy vs business risk
* retrieval quality vs final answer safety
* latency vs model complexity

The most valuable learning was that strong benchmark metrics do not guarantee strong real-world performance.

---

## Limitations

Current limitations:

* The classifier is fine-tuned on Quora, not real support-domain duplicate pairs.
* The support dataset is clean and may not represent production support data.
* The custom validation set is small.
* The project is not yet deployed as an API.
* No dashboard has been built yet.
* No MLflow experiment tracking yet.
* No Docker containerization yet.
* No real company support-ticket data was used.

---

## Future Improvements

Planned next steps:

* Fine-tune ELECTRA on support-domain pairs created from the Bitext dataset.
* Add hard negative pairs between similar but different intents.
* Modularize the codebase into reusable Python scripts.
* Add FastAPI for inference.
* Build a Streamlit dashboard.
* Track experiments with MLflow.
* Add Docker support.
* Evaluate on larger and messier support datasets.
* Add monitoring for false deflection and escalation rate.

---

## Current Status

SupportAI is complete as an applied AI modeling and evaluation prototype.

It demonstrates the full AI workflow from business framing to baseline modeling, transformer fine-tuning, retrieval evaluation, classifier reranking, confidence routing, and custom validation.

It is not yet productionized.
