<img src="phenota-logo.svg" alt="Company Logo" height="150">


# Anonymizer Repository Documentation

## Project Overview

**Anonymizer** is a sophisticated de-identification toolkit designed specifically for clinical text in Hebrew. It identifies and anonymizes Protected Health Information (PHI) and Personally Identifiable Information (PII) in Hebrew medical documents while maintaining clinical utility.

### Key Features
- Hebrew-specific NLP using AlephBERT transformer model
- Multi-signal PHI detection with intelligent conflict resolution
- Context-aware anonymization preserving clinical relevance
- Available as Python library, REST API, and Docker container
- Licensed under MIT

---

## Repository Structure

```
hebsafeharbor/
├── manager.py                 # Main orchestrator (HebSafeHarbor class)
├── identifier/                # PHI identification pipeline
│   ├── phi_identifier.py      # Identification orchestrator
│   ├── heb_nlp_engine.py      # Custom spaCy NLP engine
│   ├── signals/               # 14+ entity recognizers
│   ├── consolidation/         # Conflict resolution & entity merging
│   └── entity_spliters/       # Entity type refinement
├── anonymizer/                # PHI anonymization pipeline
│   ├── phi_anonymizer.py      # Anonymization orchestrator
│   ├── custom_operators/      # 5 custom anonymization operators
│   └── setting.py             # Anonymization masks configuration
├── common/                    # Utilities
│   ├── document.py            # Document data model
│   ├── terms_recognizer.py    # Aho-Corasick lexicon matcher
│   ├── date_utils.py          # Date parsing utilities
│   ├── city_utils.py          # Israeli city lists (1000+)
│   └── country_utils.py       # Country-to-region mapping
└── lexicons/                  # Medical terminology (Hebrew)
    ├── medications.py
    ├── diseases.py
    ├── medical_tests.py
    └── [7 more medical lexicons]
```

---

## De-Identification Process

The system uses a **two-stage pipeline**:

### Stage 1: PHI Identification

#### A. Multi-Signal Detection Architecture

The system combines **14+ independent recognizers**:

**1. HebSpacy NER (Transformer-based)**
- Built on **AlephBERT** model trained on Hebrew medical texts
- Detects: PERS (Person), LOC (Location), ORG (Organization), TIME, DATE
- Provides confidence scores for each detection
- Location: `identifier/signals/spacy_recognizer.py`

**2. Israeli ID Number Recognizer**
- Pattern matching: 8-9 digits with optional dash
- Validates using Luhn checksum algorithm
- Location: `identifier/signals/israeli_id_recognizer.py`

**3. Date Recognizers (Multiple types)**
- **HebDateRecognizer**: Hebrew calendar dates (א׳ בתשרי תש״ח)
- **HebLatinDateRecognizer**: Mixed Hebrew-Latin dates
- **PrepositionDateRecognizer**: Dates with Hebrew prepositions (ב-, מ-)
- **NoisyDateRecognizer**: Fragmented date patterns
- Location: `identifier/signals/` (multiple files)

**4. Lexicon-Based Recognizers**
Uses **Aho-Corasick automaton** for efficient multi-string matching:
- **CountryRecognizer**: Maps countries to regions
- **IsraeliCityRecognizer**: 1000+ Israeli cities/towns
- **DiseaseRecognizer**: Medical conditions
- **MedicationRecognizer**: Pharmaceutical substances
- **MedicalTestRecognizer**: Lab tests and procedures
- Location: `common/terms_recognizer.py`, `identifier/signals/`

**5. Presidio Predefined Recognizers**
- CreditCard, Email, Phone, URL, IP address patterns

#### B. Intelligent Entity Consolidation

**NerConsolidator** (`identifier/consolidation/ner_consolidator.py`) resolves conflicts when multiple recognizers detect overlapping entities:

**Conflict Types:**
1. **EXACT_MATCH**: Same category and boundaries → Select highest confidence
2. **SAME_CATEGORY**: Same type, different boundaries → Prefer longest span
3. **SAME_BOUNDARIES**: Same span, different types → Context-based resolution
4. **MIXED**: Different categories and boundaries → Multi-strategy resolution

**Resolution Strategies:**
- **PreferLongestEntity**: Prioritizes longer text spans
- **ContextBasedResolver**: Examines surrounding text (5-char window) for category-specific keywords
- **CategoryMajorityResolver**: Votes among multiple signals
- **Post-Consolidation Rules**: City-Country and Medical-specific rules

**Filtering Rules:**
- Removes single-character entities
- Filters English ALL-CAPS only entities
- Removes symbols-only entities
- DATE-specific filters: excludes floats (3.5), days of week, seasons

#### C. Entity Splitting

**DateEntitySplitter** (`identifier/entity_spliters/date_entity_splitter.py`) classifies dates into two types:

- **BIRTH_DATE**: Detected if preceded by "נולד/נולדה" (born), "תאריך לידה" (birth date) within 10 characters
- **MEDICAL_DATE**: All other dates (hospitalization, treatment dates)

This classification enables different anonymization strategies.

---

### Stage 2: PHI Anonymization

Applies **5 custom operators** to identified entities:

#### 1. ReplaceInHebrew (Default)
- PERS → `<שם_>` (Name)
- LOC/GPE → `<מיקום_>` (Location)
- ORG/FAC → `<ארגון_>` (Organization)
- EMAIL/PHONE/URL/IP → `<קשר_>` (Contact)
- ID/CREDIT_CARD → `<מזהה_>` (Identifier)
- DATE → `<תאריך_>` (Date)

#### 2. BirthDateAnonymizerOperator
**Clinical relevance preservation:**
- **DAY**: Always masked → `<יום_>`
- **MONTH**: Masked unless age < 1 year → `<חודש_>`
- **YEAR**: Masked only if age ≥ 89 years → `<שנה_>`

**Rationale**: Protects elderly patients (rare birth years identify individuals), preserves age information for clinical decisions.

Location: `anonymizer/custom_operators/birth_date_anonymizer_operator.py:56`

#### 3. MedicalDateAnonymizerOperator
**Clinical timeline preservation:**
- **DAY**: Masked → `<יום_>`
- **MONTH & YEAR**: Preserved

**Rationale**: Month/year needed for seasonal patterns and disease progression tracking.

Location: `anonymizer/custom_operators/medical_date_anonymizer_operator.py:43`

#### 4. CountryAnonymizerOperator
- Maps countries to regions using `COUNTRY_DICT`
- Falls back to `<מדינה_>` if unmapped
- Example: "ישראל" → preserved, others → regional designation

Location: `anonymizer/custom_operators/country_anonymizer_operator.py:25`

#### 5. IsraeliCityAnonymizerOperator
**Population-based privacy protection:**
- **Population > 2000**: Returned as-is (public knowledge, doesn't identify individuals)
- **Population < 2000**: Masked to `<מיקום_>` (could identify individuals in small communities)
- **Special cases**: Ambiguous cities and abbreviations preserved

Location: `anonymizer/custom_operators/israeli_city_anonymizer_operator.py:29`

---

## Processing Pipeline

```
User Input Text
     ↓
HebSafeHarbor (manager.py:64)
     ↓
PhiIdentifier.identify()
     ├─→ 14+ Signal Recognizers (parallel detection)
     ├─→ EntitySmootherRuleExecutor (pre-consolidation)
     ├─→ NerConsolidator (conflict resolution)
     └─→ DateEntitySplitter (BIRTH_DATE vs MEDICAL_DATE)
     ↓
PhiAnonymizer.anonymize()
     ├─→ BirthDateAnonymizerOperator
     ├─→ MedicalDateAnonymizerOperator
     ├─→ CountryAnonymizerOperator
     ├─→ IsraeliCityAnonymizerOperator
     └─→ ReplaceInHebrew (fallback)
     ↓
Anonymized Output + Metadata
```

---

## Technology Stack

- **Python**: 3.8+
- **NLP Framework**: spaCy 3.x
- **ML Model**: AlephBERT (Hebrew BERT transformer)
- **Privacy Library**: Microsoft Presidio (Analyzer & Anonymizer)
- **Pattern Matching**: Aho-Corasick automaton (pyahocorasack==1.4.4)
- **REST API**: FastAPI + Uvicorn
- **UI Demo**: Streamlit
- **Deployment**: Docker & Docker Compose

---


```

### REST API
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"docs": [{"id": "1", "text": "גדעון לבנה נולד ב-16.1.1990"}]}'
```

Response:
```json
{
  "docs": [{
    "id": "1",
    "text": "גדעון לבנה נולד ב-<יום_>.1.1990",
    "items": [{
      "text": "גדעון לבנה",
      "textEntityType": "PERS",
      "mask": "<שם_>",
      "maskOperator": "replace_in_hebrew"
    }]
  }]
}
```

### Docker
```bash
docker-compose up -d
# Server: http://server.localhost/docs
# Demo: http://demo.localhost
```

---

## Key Innovations

1. **Context-Aware Date Handling**: Birth dates vs. medical dates use different masking strategies
2. **Age-Aware Privacy**: Elderly patients (≥89) receive additional year masking
3. **Population-Based Location Privacy**: Small cities (<2000) anonymized, large cities preserved
4. **Multi-Signal Consolidation**: 14+ independent recognizers with intelligent conflict resolution
5. **Clinical Utility Preservation**: Balances privacy with medical relevance (month/year often preserved)
6. **Hebrew-Specific NLP**: AlephBERT trained on Hebrew medical corpus
7. **Efficient Lexicon Matching**: Aho-Corasick automaton for fast medical term recognition
8. **Audit Trail**: Maintains character position mapping for compliance tracking

This architecture represents a production-grade approach to clinical NLP that balances HIPAA-style de-identification requirements with the practical need to maintain clinically useful information in Hebrew medical texts.
