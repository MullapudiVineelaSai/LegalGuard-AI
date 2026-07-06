# LegalGuard AI — AI-Powered Legal Document Risk Analyzer

An intelligent web application that helps users understand legal documents
(employment contracts, rental agreements, loan agreements, insurance
policies, privacy policies, terms & conditions, etc.) **before** they sign
them. Users upload a PDF/DOCX document; the system extracts the text,
runs an NLP-based clause-detection and risk-scoring engine, and returns a
risk score, a plain-English summary, and clause-by-clause recommendations.

## Tech Stack
| Layer | Technology |
|---|---|
| Frontend | HTML, CSS, Bootstrap 5, Bootstrap Icons |
| Backend | Python, Django 5 |
| Database | MySQL (default: SQLite for zero-config local dev) |
| AI / NLP | Rule-based clause detection + extractive summarization (pure Python, no model downloads needed). Written to be a drop-in target for spaCy / NLTK / scikit-learn pipelines — see "Extending the AI Engine" below. |
| Document Parsing | `python-docx` (DOCX), `pdfplumber` (PDF) |

## Features
- Secure user authentication (register / login / logout)
- Upload legal documents (PDF or DOCX) through a clean web UI
- Automatic text extraction from PDF and DOCX files
- NLP-based detection of 10 key legal clause types:
  termination conditions, confidentiality, penalty / liquidated damages,
  liability limitation, payment obligations, automatic renewal,
  arbitration, non-compete, indemnification, and data privacy clauses
- Overall document **risk score (0–100)** classified as **Low / Medium / High**
- Per-clause severity rating with plain-English explanation and a
  recommendation for what to check before signing
- Auto-generated simplified summary of the document
- Personal dashboard with document/risk statistics
- Full analysis history with the ability to re-view or delete reports
- Administrator dashboard: total users, total documents, risk breakdown,
  most common risky clause types, and recent uploads across all users
- Django admin panel for full data management

## Project Structure
```
legalguard/
├── analyzer/                  # Main Django app
│   ├── models.py              # Document, Analysis, ClauseFinding
│   ├── forms.py                # Registration & upload forms
│   ├── views.py                # Auth, upload, dashboard, reports, admin views
│   ├── urls.py
│   ├── admin.py                # Django admin registrations
│   ├── nlp/
│   │   ├── text_extractor.py   # PDF / DOCX text extraction
│   │   └── clause_analyzer.py  # Clause detection + risk scoring engine
│   └── templates/analyzer/     # Bootstrap-based HTML templates
├── legalguard/                 # Project settings, urls, wsgi/asgi
├── static/css/style.css
├── media/                      # Uploaded documents (created at runtime)
├── requirements.txt
└── manage.py
```

## Setup Instructions

### 1. Create a virtual environment and install dependencies
```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```
> `mysqlclient` requires MySQL development headers to build. If you only
> want to run locally on SQLite (default), you can remove that line from
> `requirements.txt` before installing.

### 2. (Optional) Configure MySQL
By default the project uses **SQLite** — no setup required. To use MySQL
as described in the project abstract:
```bash
export USE_MYSQL=1
export DB_NAME=legalguard_db
export DB_USER=root
export DB_PASSWORD=yourpassword
export DB_HOST=localhost
export DB_PORT=3306
```
Then create the database in MySQL:
```sql
CREATE DATABASE legalguard_db CHARACTER SET utf8mb4;
```

### 3. Run migrations
```bash
python manage.py migrate
```

### 4. Create an administrator account
```bash
python manage.py createsuperuser
```
Log in with this account and visit `/manage/dashboard/` for the custom
admin dashboard (or `/django-admin/` for the full Django admin).

### 5. Run the development server
```bash
python manage.py runserver
```
Visit **http://127.0.0.1:8000/**

## How the Risk Analysis Works
1. **Text extraction** — `pdfplumber` / `python-docx` pull raw text out of
   the uploaded file (including table content in DOCX files).
2. **Sentence segmentation** — the text is split into sentences using a
   lightweight regex tokenizer (no external model downloads required).
3. **Clause detection** — each sentence is matched against a curated set
   of regex/keyword patterns for 10 legal clause categories.
4. **Severity scoring** — within a matched clause, "high-risk modifier"
   phrases (e.g. *"without prior notice"*, *"sole discretion"*,
   *"irrevocable"*, *"unlimited liability"*, *"waive any right"*) increase
   that clause's individual risk level (Low/Medium/High).
5. **Aggregate scoring** — a weighted sum across all detected clauses,
   normalized to 0–100, plus a small "breadth" bonus for documents that
   contain many distinct risky clause categories, produces the overall
   document risk score and Low/Medium/High classification.
6. **Summarization** — a frequency-based extractive summarizer (Luhn-style)
   selects the most information-dense sentences to build a plain-English
   summary.
7. **Recommendations** — each detected clause type carries a plain-English
   explanation and an actionable recommendation, aggregated into a report.

## Extending the AI Engine
The engine lives entirely in `analyzer/nlp/clause_analyzer.py` and exposes
one function, `analyze_text(text) -> dict`. To upgrade it with heavier
NLP/ML (as referenced in the abstract):
- Swap the regex trigger matching for **spaCy** `Matcher`/`PhraseMatcher`
  or a fine-tuned NER pipeline.
- Replace/augment severity scoring with a **scikit-learn** text
  classifier (e.g. TF-IDF + Logistic Regression/SVM) trained on labeled
  clause-risk examples.
- Use **NLTK**/spaCy sentence tokenizers instead of the built-in regex
  tokenizer for more robust segmentation on complex legal prose.

Because `analyze_text()` has a stable input/output contract (`{findings,
risk_score, risk_level, summary, recommendations, word_count}`), any of
these swaps are drop-in changes that don't require touching `views.py`.

## Disclaimer
This tool provides general, automated informational analysis and is
**not a substitute for professional legal advice**. Users should consult
a qualified lawyer before signing any legally binding document.
