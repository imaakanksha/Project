# GeneRx-AI: Requirements Document

## Project Overview

GeneRx-AI is a Clinical Decision Support System (CDSS) designed to prevent Adverse Drug Reactions (ADRs) by acting as a real-time "prescription interceptor." The system bridges the gap between static genetic reports and active clinical workflows by using Retrieval-Augmented Generation (RAG) to cross-reference a patient's structured genetic profile against official Clinical Pharmacogenetics Implementation Consortium (CPIC) guidelines at the point of prescribing.

### Mission
Transform pharmacogenomic data from passive lab reports into active clinical safeguards that protect patients from preventable drug-gene interactions.

## User Personas

### Dr. Alix Chen (Clinician)
- **Role**: Primary Care Physician in a busy urban clinic
- **Pain Points**: 
  - No time to read lengthy pharmacogenomic reports or guideline PDFs
  - Needs instant, actionable alerts at the point of prescribing
  - Requires confidence that recommendations are evidence-based
- **Needs**: 
  - Clear "stop-sign" alerts for dangerous drug-gene combinations
  - Quick access to source guidelines for verification
  - Minimal workflow disruption

### Sam Rodriguez (Patient)
- **Profile**: 32-year-old with CYP2D6 Poor Metabolizer status (unknown to them)
- **Risk**: Hidden genetic metabolic defect makes certain medications ineffective or dangerous
- **Example Scenario**: Prescribed Codeine for post-surgical pain, but cannot convert it to active morphine
- **Needs**: 
  - Protection from ineffective or harmful medications
  - Proactive identification of drug-gene interactions
  - Alternative medication recommendations

## Functional Requirements

### FR-1: Input Parsing
**Priority**: Critical  
**Description**: The system must accept structured prescription data for analysis.

**Acceptance Criteria**:
- Accept JSON payload containing:
  - `patient_id` (string): Unique synthetic patient identifier
  - `drug_name` (string): Medication being prescribed
  - `dosage` (string): Proposed dosage and frequency
- Validate input format and return clear error messages for malformed requests
- Support common drug name variations and aliases

### FR-2: Genotype Lookup
**Priority**: Critical  
**Description**: The system must retrieve patient-specific pharmacogenomic data.

**Acceptance Criteria**:
- Query patient database using `patient_id`
- Retrieve relevant diplotypes for drug-metabolizing genes (e.g., CYP2D6 *4/*4, CYP2C19 *2/*3)
- Return structured genotype data including:
  - Gene name
  - Diplotype (allele combination)
  - Phenotype classification (e.g., Poor Metabolizer, Rapid Metabolizer)
- Handle cases where patient genetic data is unavailable

### FR-3: RAG Retrieval
**Priority**: Critical  
**Description**: The system must query a vector database to find relevant CPIC guidelines.

**Acceptance Criteria**:
- Construct semantic query combining drug name and patient genotype
- Query ChromaDB vector store for relevant guideline chunks
- Retrieve top 3-5 most relevant context passages
- Include metadata (source document, page number, guideline version)
- Return empty result gracefully if no relevant guidelines exist

### FR-4: Risk Analysis
**Priority**: Critical  
**Description**: The system must classify drug-gene interactions using an LLM.

**Acceptance Criteria**:
- Analyze retrieved CPIC guidelines in context of patient genotype
- Classify interaction into three risk levels:
  - **Green (Safe)**: No significant interaction, standard dosing appropriate
  - **Yellow (Caution)**: Interaction present, dosage adjustment or monitoring recommended
  - **Red (Danger)**: Contraindicated or high-risk interaction, alternative drug recommended
- Provide specific reasoning for classification
- Suggest alternative medications for Yellow/Red classifications
- Ground all recommendations strictly in retrieved context (no hallucinations)

### FR-5: Output Generation
**Priority**: Critical  
**Description**: The system must display concise, actionable alerts with source citations.

**Acceptance Criteria**:
- Display risk level with visual indicator (color-coded)
- Provide 2-3 sentence summary of the interaction
- Include specific dosing recommendations or alternatives
- Provide "Click-to-Source" link to original CPIC guideline PDF
- Show confidence level or strength of evidence
- Format output for easy clinical interpretation

### FR-6: Audit Trail
**Priority**: High  
**Description**: The system must log all prescription analyses for compliance and review.

**Acceptance Criteria**:
- Log timestamp, patient_id, drug, genotype, and recommendation
- Store retrieved guideline chunks used in analysis
- Enable retrospective review of system recommendations
- Maintain logs for minimum 7 years (HIPAA compliance)

## Non-Functional Requirements

### NFR-1: Latency
**Priority**: Critical  
**Target**: End-to-end response time under 2 seconds

**Rationale**: Clinical workflows require near-instant feedback to avoid disrupting prescribing process.

**Acceptance Criteria**:
- 95th percentile response time < 2 seconds
- Average response time < 1 second
- Timeout and graceful degradation after 3 seconds

### NFR-2: Accuracy
**Priority**: Critical  
**Target**: Zero hallucinations, 100% citation grounding

**Rationale**: Patient safety depends on recommendations being strictly evidence-based.

**Acceptance Criteria**:
- All recommendations must cite specific retrieved guideline passages
- LLM responses must be constrained to retrieved context only
- System must refuse to answer if insufficient context is available
- Regular validation against gold-standard CPIC recommendations

### NFR-3: Privacy & Security
**Priority**: Critical  
**Target**: Zero PHI exposure to external APIs

**Rationale**: HIPAA compliance and patient privacy protection.

**Acceptance Criteria**:
- Use synthetic patient IDs only (no real PHI)
- No patient data sent to external LLM APIs
- Local LLM deployment or anonymized API calls only
- Encrypted data storage for patient genotypes
- Access controls and authentication for system users

### NFR-4: Reliability
**Priority**: High  
**Target**: 99.9% uptime during clinical hours

**Acceptance Criteria**:
- Graceful error handling for all failure modes
- Fallback mechanisms if vector DB or LLM unavailable
- Clear error messages for clinicians
- System health monitoring and alerting

### NFR-5: Scalability
**Priority**: Medium  
**Target**: Support 1000+ queries per day

**Acceptance Criteria**:
- Handle concurrent requests from multiple clinicians
- Efficient vector search with sub-100ms query time
- Caching for frequently queried drug-gene pairs
- Horizontal scaling capability for future growth

### NFR-6: Maintainability
**Priority**: Medium  
**Target**: Easy guideline updates as CPIC releases new versions

**Acceptance Criteria**:
- Modular architecture for swapping guideline versions
- Automated ingestion pipeline for new CPIC PDFs
- Version tracking for guidelines in vector database
- Clear documentation for system maintenance

## Out of Scope (V1)

- Integration with real EHR systems (Epic, Cerner)
- Multi-drug interaction analysis (polypharmacy)
- Patient-facing mobile application
- Real-time genetic testing integration
- Insurance coverage recommendations
- Pediatric dosing calculations

## Success Metrics

- **Clinical Impact**: Number of high-risk prescriptions flagged
- **Adoption**: Percentage of prescriptions screened through system
- **Performance**: Average response time and system uptime
- **Accuracy**: Agreement rate with manual pharmacogenomics review
- **User Satisfaction**: Clinician feedback scores and workflow integration

## Regulatory Considerations

- System is a Clinical Decision Support tool (CDS), not a diagnostic device
- Recommendations are advisory; final prescribing decision remains with clinician
- Compliance with 21st Century Cures Act CDS provisions
- HIPAA compliance for all patient data handling
- Documentation of clinical validation and evidence basis
