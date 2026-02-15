# GeneRx-AI: System Design Document

## System Architecture

### High-Level Overview

```
┌─────────────────┐
│   Streamlit     │  ← Clinician Interface (EHR Simulation)
│    Frontend     │
└────────┬────────┘
         │ HTTP
         ▼
┌─────────────────┐
│    FastAPI      │  ← REST API (/analyze_prescription)
│    Backend      │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌─────────┐ ┌──────────────┐
│Patient  │ │  LangChain   │  ← RAG Orchestration
│Database │ │  + LLM       │
└─────────┘ └──────┬───────┘
                   ▼
            ┌──────────────┐
            │  ChromaDB    │  ← CPIC Guidelines (Vector Store)
            └──────────────┘
```

### Component Descriptions

#### Frontend: Streamlit
- **Purpose**: Simulates EHR prescription interface for hackathon demo
- **Technology**: Streamlit (Python)
- **Features**:
  - Drug name autocomplete
  - Dosage input fields
  - Patient ID selector
  - Real-time alert display with color-coded risk levels
  - Expandable guideline source citations
  - Response time indicator

#### Backend: FastAPI
- **Purpose**: Serves prescription analysis API
- **Technology**: FastAPI (Python 3.10+)
- **Endpoints**:
  - `POST /analyze_prescription`: Main analysis endpoint
  - `GET /health`: System health check
  - `GET /patients`: List available synthetic patients
  - `GET /guidelines`: List loaded CPIC guidelines
- **Features**:
  - Async request handling
  - Input validation with Pydantic models
  - Error handling and logging
  - CORS configuration for frontend

#### Orchestration: LangChain
- **Purpose**: Manages RAG pipeline flow
- **Technology**: LangChain (Python)
- **Components**:
  - Retrieval chain for vector search
  - Prompt templates for LLM queries
  - Output parsers for structured responses
  - Memory management for conversation context (future)

#### Vector Store: ChromaDB
- **Purpose**: Stores and retrieves CPIC guideline embeddings
- **Technology**: ChromaDB (local persistent storage)
- **Contents**:
  - Chunked CPIC guideline text (500-token chunks with 50-token overlap)
  - Metadata: drug name, gene, guideline version, source PDF, page number
  - Embeddings: Generated using sentence-transformers (all-MiniLM-L6-v2)
- **Collections**:
  - `cpic_guidelines`: Main guideline collection
  - Future: `drug_labels`, `clinical_annotations`

#### Patient Database
- **Purpose**: Stores synthetic patient pharmacogenomic profiles
- **Technology**: SQLite (V1) or JSON files
- **Schema**:
```json
{
  "patient_id": "P001",
  "demographics": {
    "age": 45,
    "sex": "F",
    "ancestry": "European"
  },
  "genotypes": [
    {
      "gene": "CYP2D6",
      "diplotype": "*4/*4",
      "phenotype": "Poor Metabolizer",
      "activity_score": 0.0
    },
    {
      "gene": "CYP2C19",
      "diplotype": "*1/*2",
      "phenotype": "Intermediate Metabolizer",
      "activity_score": 1.0
    }
  ]
}
```

## Data Flow

### End-to-End Request Flow

1. **User Input** (Streamlit Frontend)
   - Clinician enters: Drug = "Warfarin", Patient = "P042", Dosage = "5mg daily"
   - Frontend sends POST request to `/analyze_prescription`

2. **Request Validation** (FastAPI Backend)
   - Validate JSON payload structure
   - Check patient_id exists in database
   - Normalize drug name (handle aliases)

3. **Genotype Retrieval** (Patient Database)
   - Query: `SELECT genotypes FROM patients WHERE patient_id = 'P042'`
   - Result: `CYP2C9: *3/*3 (Poor Metabolizer)`

4. **Semantic Query Construction** (LangChain)
   - Build query: "Warfarin dosing guidelines for CYP2C9 *3/*3 poor metabolizer"
   - Generate query embedding using same model as guideline embeddings

5. **Vector Search** (ChromaDB)
   - Similarity search for top 3 relevant chunks
   - Filter by drug="Warfarin" and gene="CYP2C9" metadata
   - Return chunks with similarity scores > 0.7

6. **LLM Analysis** (LangChain + OpenAI/Local LLM)
   - Prompt template:
```
You are a clinical pharmacogenomics expert. Analyze the following:

Patient Genotype: CYP2C9 *3/*3 (Poor Metabolizer)
Prescribed Drug: Warfarin 5mg daily

Relevant CPIC Guidelines:
{retrieved_chunks}

Task: Classify this prescription as Safe, Caution, or Danger.
Provide:
1. Risk Level (Green/Yellow/Red)
2. Brief Explanation (2-3 sentences)
3. Specific Recommendation (dosage adjustment or alternative)
4. Citation (quote relevant guideline text)

Output as JSON only. Do not add information not in the guidelines.
```

7. **Response Parsing** (LangChain Output Parser)
   - Parse LLM JSON output
   - Validate structure
   - Add source metadata (PDF links, page numbers)

8. **Alert Display** (Streamlit Frontend)
   - Render color-coded alert card
   - Display recommendation text
   - Show "View Source Guideline" button
   - Log interaction timestamp

### Example API Request/Response

#### Request
```json
POST /analyze_prescription
Content-Type: application/json

{
  "patient_id": "P042",
  "drug_name": "Warfarin",
  "dosage": "5mg daily"
}
```

#### Response
```json
{
  "status": "success",
  "analysis": {
    "risk_level": "red",
    "risk_label": "DANGER - Alternative Recommended",
    "summary": "Patient has CYP2C9 *3/*3 genotype (Poor Metabolizer) which significantly reduces warfarin metabolism. Standard 5mg dosing poses high bleeding risk.",
    "recommendation": "Reduce initial dose by 50-75% (start at 1-2mg daily) with frequent INR monitoring, OR consider alternative anticoagulant (e.g., apixaban) that does not require CYP2C9 metabolism.",
    "patient_genotype": {
      "gene": "CYP2C9",
      "diplotype": "*3/*3",
      "phenotype": "Poor Metabolizer"
    },
    "evidence": {
      "guideline": "CPIC Guideline for Warfarin and CYP2C9/VKORC1",
      "citation": "Individuals with CYP2C9 *3/*3 genotype require 30-40% lower warfarin doses compared to *1/*1 individuals to achieve therapeutic INR.",
      "source_url": "https://cpicpgx.org/guidelines/guideline-for-warfarin-and-cyp2c9-and-vkorc1/",
      "confidence": "high"
    }
  },
  "metadata": {
    "response_time_ms": 1247,
    "timestamp": "2026-02-15T10:32:15Z",
    "model_version": "gpt-4-turbo",
    "guideline_version": "CPIC_v2.1_2023"
  }
}
```

## API Schema

### Pydantic Models

```python
from pydantic import BaseModel, Field
from enum import Enum
from typing import Optional, List

class RiskLevel(str, Enum):
    SAFE = "green"
    CAUTION = "yellow"
    DANGER = "red"

class PrescriptionRequest(BaseModel):
    patient_id: str = Field(..., description="Synthetic patient identifier")
    drug_name: str = Field(..., description="Medication name")
    dosage: str = Field(..., description="Proposed dosage and frequency")

class GenotypeInfo(BaseModel):
    gene: str
    diplotype: str
    phenotype: str
    activity_score: Optional[float] = None

class Evidence(BaseModel):
    guideline: str
    citation: str
    source_url: str
    confidence: str

class Analysis(BaseModel):
    risk_level: RiskLevel
    risk_label: str
    summary: str
    recommendation: str
    patient_genotype: GenotypeInfo
    evidence: Evidence

class PrescriptionResponse(BaseModel):
    status: str
    analysis: Analysis
    metadata: dict
```

## Synthetic Data Strategy

### Patient Generation (`generate_patients.py`)

**Purpose**: Create realistic synthetic patient population with diverse pharmacogenomic profiles for testing and demo purposes.

**Approach**:
1. **Allele Frequency Sampling**
   - Use published population allele frequencies from PharmGKB
   - Sample diplotypes based on Hardy-Weinberg equilibrium
   - Stratify by ancestry groups (European, African, East Asian, etc.)

2. **Gene Coverage**
   - Priority genes: CYP2D6, CYP2C19, CYP2C9, SLCO1B1, TPMT, DPYD
   - Generate 2-5 pharmacogenes per patient
   - Include both common and rare variants

3. **Phenotype Assignment**
   - Map diplotypes to phenotypes using CPIC/PharmGKB tables
   - Calculate activity scores for metabolizer status
   - Handle star allele nomenclature (*1, *2, *3, etc.)

4. **Diversity Targets**
   - Generate 100-500 synthetic patients
   - Ensure representation of all metabolizer phenotypes:
     - Poor Metabolizers (5-10%)
     - Intermediate Metabolizers (20-30%)
     - Normal Metabolizers (50-60%)
     - Rapid/Ultra-rapid Metabolizers (10-15%)

5. **Output Format**
   - JSON file: `synthetic_patients.json`
   - SQLite database: `patients.db`
   - Include metadata: generation date, allele frequency source, version

**Example Generation Code**:
```python
import random
from typing import Dict, List

# CYP2D6 allele frequencies (European ancestry)
CYP2D6_ALLELES = {
    "*1": 0.35,   # Normal function
    "*2": 0.28,   # Normal function
    "*4": 0.20,   # No function
    "*10": 0.05,  # Decreased function
    "*41": 0.08,  # Decreased function
    "*5": 0.04    # No function (deletion)
}

def generate_diplotype(allele_freqs: Dict[str, float]) -> tuple:
    """Sample two alleles based on population frequencies"""
    alleles = list(allele_freqs.keys())
    weights = list(allele_freqs.values())
    return tuple(random.choices(alleles, weights=weights, k=2))

def assign_phenotype(diplotype: tuple, gene: str) -> str:
    """Map diplotype to metabolizer phenotype"""
    # Simplified logic - real implementation uses activity score tables
    no_function = ["*4", "*5", "*6"]
    decreased = ["*10", "*17", "*41"]
    
    allele1, allele2 = diplotype
    
    if allele1 in no_function and allele2 in no_function:
        return "Poor Metabolizer"
    elif allele1 in no_function or allele2 in no_function:
        return "Intermediate Metabolizer"
    elif allele1 in decreased or allele2 in decreased:
        return "Intermediate Metabolizer"
    else:
        return "Normal Metabolizer"
```

## Technology Stack Summary

### Core Technologies
- **Language**: Python 3.10+
- **Frontend**: Streamlit 1.28+
- **Backend**: FastAPI 0.104+
- **RAG Framework**: LangChain 0.1+
- **Vector Database**: ChromaDB 0.4+
- **LLM**: OpenAI GPT-4-Turbo (or local Llama 2)
- **Embeddings**: sentence-transformers (all-MiniLM-L6-v2)

### Supporting Libraries
- **Data Processing**: pandas, numpy
- **Database**: SQLite3, sqlalchemy
- **HTTP Client**: httpx, requests
- **Validation**: pydantic
- **Testing**: pytest, pytest-asyncio
- **Logging**: structlog
- **PDF Processing**: PyPDF2, pdfplumber (for guideline ingestion)

### Development Tools
- **Package Manager**: pip, poetry
- **Code Quality**: black, ruff, mypy
- **Version Control**: Git
- **Environment**: python-dotenv for config

## Deployment Architecture (Future)

### Local Development
- Single machine running all components
- ChromaDB in-memory or persistent local storage
- SQLite for patient database
- Streamlit dev server on localhost:8501
- FastAPI dev server on localhost:8000

### Production Considerations (Post-Hackathon)
- Containerization with Docker
- Kubernetes orchestration for scaling
- PostgreSQL for patient data
- Managed vector database (Pinecone, Weaviate)
- Load balancing for API endpoints
- Redis caching for frequent queries
- Monitoring with Prometheus + Grafana

## Security & Privacy Design

### Data Protection
- All patient data is synthetic (no real PHI)
- Patient IDs are randomly generated UUIDs
- No external API calls with patient identifiers
- Local LLM deployment option for air-gapped environments

### Access Control (Future)
- JWT-based authentication for API
- Role-based access (clinician, admin, read-only)
- Audit logging for all prescription analyses
- Rate limiting to prevent abuse

### Compliance
- HIPAA-ready architecture (when using real data)
- Data encryption at rest and in transit
- Configurable data retention policies
- Export capabilities for regulatory review

## Performance Optimization

### Latency Targets
- Vector search: < 100ms
- LLM inference: < 1000ms
- Total end-to-end: < 2000ms

### Optimization Strategies
1. **Caching**
   - Cache frequent drug-gene pair results (Redis)
   - Cache patient genotype lookups (in-memory LRU)
   - Cache guideline embeddings (persistent ChromaDB)

2. **Batch Processing**
   - Batch embed multiple queries together
   - Async database queries
   - Parallel retrieval for multiple genes

3. **Model Selection**
   - Use smaller, faster embedding models
   - Consider quantized LLMs for local deployment
   - Implement streaming responses for UX

4. **Database Indexing**
   - Index patient_id in SQLite
   - Optimize ChromaDB collection settings
   - Use metadata filtering to reduce search space

## Testing Strategy

### Unit Tests
- API endpoint validation
- Genotype lookup logic
- Risk classification rules
- Output parsing

### Integration Tests
- End-to-end RAG pipeline
- Database connectivity
- LLM response validation
- Frontend-backend communication

### Validation Tests
- Compare system recommendations against gold-standard CPIC cases
- Test edge cases (missing genotypes, unknown drugs)
- Verify citation accuracy
- Measure response time under load

## Future Enhancements

### Phase 2 Features
- Multi-drug interaction analysis (polypharmacy)
- Drug-drug-gene interaction detection
- Patient medication history integration
- Pharmacist review workflow

### Phase 3 Features
- Real EHR integration (HL7 FHIR)
- Mobile clinician app
- Patient portal for genetic results
- Insurance formulary integration
- Automated guideline update pipeline

## References

- CPIC Guidelines: https://cpicpgx.org/
- PharmGKB: https://www.pharmgkb.org/
- LangChain Documentation: https://python.langchain.com/
- ChromaDB Documentation: https://docs.trychroma.com/
