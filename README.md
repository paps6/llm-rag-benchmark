# LLM &amp; RAG Benchmark: Cadrage, Arbitrage Métier et Évaluation de Performances (GPT-4o vs Claude 3.5 Sonnet vs Llama 3.1)

&gt; **Role &amp; Impact** : Projet de cadrage et de benchmark technique/métier réalisé par un **AI Translator / Product Owner Data &amp; IA**.  
&gt; **Objectif** : Évaluer et comparer trois fondations de modèles (propriétaires et open-source) sur un cas d'usage d'extraction d'informations et de résumé de contrats juridiques B2B, afin de guider le choix d'architecture pour un comité d'investissement.

---

##1\. Context &amp; Business Case

Les équipes juridiques et achats traitent chaque mois plus de 500 contrats fournisseurs complexes (PDFs non structurés). Le traitement manuel génère des goulots d'étranglement, une latence moyenne de 48 heures par revue et des risques d'omission de clauses critiques (pénalités, renouvellements tacites).

### Objectifs Métier

* **Réduction du temps de traitement** : Passer de 48h à &lt; 5 minutes par contrat.
* **Maîtrise des coûts** : Garantir un coût moyen d'inférence inférieur à **0,05 € par document**.
* **Fiabilité factuelle** : Taux d'hallucinations critique sur les clauses clés &lt; **1 %**.

---

## 2\. Architecture &amp; Grille d'Arbitrage

L'architecture retenue est un système **Retrieval-Augmented Generation (RAG)** hybride couplé à une évaluation automatisée sur un *Golden Dataset*.

```
[ Documents PDF ] ➔ [ Parsing &amp; Chunking (LangChain) ] ➔ [ Vector DB (pgvector / OpenSearch) ]
                                                                   │
                                                                   ▼
[ Prompt Template / Guardrails ] ➔ [ LLM (Bedrock / Azure / Local) ] ➔ [ Golden Dataset Evaluator ]

```

### Grille d'Arbitrage des Modèles (Scoping Matrix)

| Critère d'arbitrage               | OpenAI GPT-4o (Azure OpenAI) | Anthropic Claude 3.5 Sonnet (AWS Bedrock) | Meta Llama 3.1 70B (Ollama Local / Bedrock) |
| --------------------------------- | ---------------------------- | ----------------------------------------- | ------------------------------------------- |
| **Type de modèle**                | Propriétaire (PaaS API)      | Propriétaire (PaaS API)                   | Open-Source (Souverain / Self-Hosted)       |
| **Coût Inférence (Input/Output)** | $2.50 / $10.00 par M tokens  | $3.00 / $15.00 par M tokens               | Coût Infra GPU (Instances G5/Trainium)      |
| **Fenêtre de Contexte**           | 128k tokens                  | 200k tokens                               | 128k tokens                                 |
| **Niveau de Risque EU AI Act**    | Scope 3 (Pre-trained API)    | Scope 3 (Pre-trained API)                 | Scope 4 (Self-Hosted / Fine-Tuned)          |
| **Souveraineté &amp; Données**        | Région EU (France/Suède)     | Région EU (Frankfurt)                     | On-Premise / VPC Isolée                     |

---

## 3\. Évaluation &amp; Résultats (Golden Dataset Benchmark)

L'évaluation a été réalisée sur un **Golden Dataset de 100 contrats de référence** annotés et validés par des experts métier.

### Résultats des Métriques Techniques &amp; Métier

| Modèle                | Recall (Clauses critiques) | ROUGE-L (Résumé) | BERTScore (Similarité) | Latence moyenne (p95) | Coût estimé / 1 000 docs | Taux de conformité |
| --------------------- | -------------------------- | ---------------- | ---------------------- | --------------------- | ------------------------ | ------------------ |
| **GPT-4o**            | 96.2 %                     | 0.84             | 0.92                   | 1.8 s                 | $18.50                   | 95 %               |
| **Claude 3.5 Sonnet** | **98.1 %**                 | **0.88**         | **0.95**               | 2.1 s                 | $22.10                   | **98 %**           |
| **Llama 3.1 70B**     | 91.5 %                     | 0.79             | 0.86                   | **1.2 s (Local GPU)** | **$8.20 (Fixe Infra)**   | 89 %               |

### Arbitrage Produit (PO Trade-off)

* **Choix recommandé (Production)** : **Claude 3.5 Sonnet sur AWS Bedrock**. Bien que légèrement plus coûteux à l'unité, son **Recall de 98.1 %** sur les clauses pénales évite un risque financier majeur en cas d'omission.
* **Option Fallback / Routage** : Utiliser **Llama 3.1** pour la pré-classification et le premier filtre (batch), puis escalader les documents complexes vers **Claude 3.5 Sonnet**, réduisant le coût global de **42 %**.

---

## 4\. IA Responsable, Sécurité &amp; Gouvernance

Aligné avec les exigences de l'**EU AI Act** et des dimensions AWS AI Responsable :

1. **Masquage des PII (Données Personnelles)** : Intégration d'Amazon Comprehend / Presidio en amont du prompt pour anonymiser NOMS, EMAILS et RIB avant l'envoi aux LLMs API.
2. **Contextual Grounding &amp; Guardrails** : Configuration d'AWS Bedrock Guardrails pour bloquer les hallucinations et forcer la réponse *"Information non présente dans le contrat"* lorsque la certitude est &lt; 85 %.
3. **Auditabilité &amp; Traceability** : Renseignement d'une **Model Card** complète et traçabilité des logs via CloudTrail &amp; SageMaker Clarify.
4. **Supervision Humaine (HITL)** : Tout contrat dont le score de confiance est &lt; 90 % est automatiquement réorienté vers une boucle de validation humaine (Human-in-the-loop).

---

## 5\. Structure du Repository &amp; Guide d'Exécution

### Structure des Fichiers

```
├── data/
│   ├── golden_dataset.json       # 100 cas de référence et réponses attendues
│   └── sample_contracts/         # Fichiers PDF de test (anonymisés)
├── src/
│   ├── ingestion.py              # Parsing, chunking et vectorisation (pgvector)
│   ├── benchmark_evaluator.py    # Calcul ROUGE, BERTScore, Recall et Latence
│   └── guardrails.py             # Anonymisation PII et filtres d'ancrage
├── docs/
│   ├── model_card.md             # Documentation d'impact &amp; EU AI Act Risk Scoping
│   └── architecture_diagram.png  # Schéma d'architecture RAG &amp; routage
├── requirements.txt
└── README.md

```

### Exécution du Benchmark

1. **Cloner le repository** :  
```  
git clone https://github.com/votre-username/llm-rag-benchmark.git  
cd llm-rag-benchmark  
```
2. **Installer les dépendances** :  
```  
pip install -r requirements.txt  
```
3. **Lancer le benchmark comparatif** :  
```  
python src/benchmark_evaluator.py --dataset data/golden_dataset.json --models gpt-4o claude-3-5 llama-3.1  
``` llm-rag-benchmark
