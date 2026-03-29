# Semantic & Similarity-Based Answer Evaluation System
### Final Year Project — NLP

---

## Project Structure

```
answer_eval/
├── app.py                   # Flask entry point & routing
├── requirements.txt
├── modules/
│   ├── extractor.py         # Text extraction (PDF, OCR, DOCX, TXT, Excel)
│   ├── parser.py            # Question segmentation & matching
│   ├── evaluator.py         # NLP scoring (TF-IDF + semantic)
│   └── storage.py           # Excel results storage
├── templates/
│   ├── index.html           # Upload & evaluation UI
│   └── results.html         # Results viewer
├── uploads/                 # Temporary upload folder (auto-created)
└── results/
    └── results.xlsx         # Generated results workbook
```

---

## Setup Instructions

### 1. Install Tesseract OCR (system package)
```bash
# Ubuntu / Debian
sudo apt-get install tesseract-ocr

# macOS
brew install tesseract

# Windows → download installer from:
# https://github.com/UB-Mannheim/tesseract/wiki
```

### 2. Create & activate virtual environment
```bash
python -m venv venv
source venv/bin/activate     # Windows: venv\Scripts\activate
```

### 3. Install Python dependencies
```bash
pip install -r requirements.txt
```

### 4. Download NLTK data (one-time)
```bash
python -c "import nltk; nltk.download('stopwords'); nltk.download('punkt')"
```

### 5. Run the app
```bash
python app.py
```

Open: http://localhost:5000

---

## How Each Module Works

### extractor.py
Routes uploaded files to the correct extractor:
- **PDF** → pdfplumber; falls back to Tesseract OCR for scanned pages
- **Images** → Tesseract OCR (grayscale pre-processing)
- **DOCX** → python-docx (paragraphs + tables)
- **TXT** → direct file read (UTF-8 / latin-1)
- **Excel** → pandas (concatenates all cell values)

### parser.py
Detects question boundaries using regex patterns:
- `Q1`, `Q.1`, `Q1:`, `Question 1`
- `Part A`, `Part B`, `Section A`
- `1.`, `2.`, `10.` (plain numbered)
- `1(a)`, `1(b)`, `(a)`, `(b)`, `(i)`, `(ii)`

Matching strategy (in priority order):
1. Exact label match (Q1 → Q1)
2. Fuzzy digit-based match (QUESTION_1 ~ Q1)
3. Positional fallback (by order)

### evaluator.py
For each matched question pair:
1. **Preprocess** → lowercase, remove punctuation/digits, remove stopwords, stem
2. **TF-IDF vectorise** → from scratch (no sklearn dependency)
3. **Cosine similarity** → dot product / (|A| × |B|)
4. **Optional semantic** → Sentence Transformers (`all-MiniLM-L6-v2`)
5. **Mark bands** → ≥85% → full marks, ≥70% → 90%, ≥55% → 75%, etc.
6. **Feedback** → identifies missing key concepts

### storage.py
- Creates/appends to `results/results.xlsx`
- One sheet per subject
- Header row: Student Name | Register No. | Q1 | Q2 | … | Total | Percentage
- Styled with openpyxl (navy headers, alternating rows)

---

## API Endpoints

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/` | Upload/evaluate page |
| POST | `/evaluate` | Run evaluation (multipart form) |
| GET | `/subjects` | List Excel sheet names |
| GET | `/results` | Results viewer page |
| GET | `/results/<subject>` | JSON data for a subject |
| GET | `/download` | Download results Excel |

---

## Supported File Formats

| Format | Extension | Method |
|--------|-----------|--------|
| PDF (text) | .pdf | pdfplumber |
| PDF (scanned) | .pdf | Tesseract OCR |
| Image | .png .jpg .jpeg | Tesseract OCR |
| Word Document | .docx | python-docx |
| Plain Text | .txt | direct read |
| Excel | .xlsx .xls | pandas |

---

## Mark Band Table

| Similarity | Marks Awarded |
|-----------|--------------|
| ≥ 85% | 100% of max |
| ≥ 70% | 90% of max |
| ≥ 55% | 75% of max |
| ≥ 40% | 55% of max |
| ≥ 25% | 35% of max |
| < 25% | 0 |
| Missing | 0 (penalized) |
