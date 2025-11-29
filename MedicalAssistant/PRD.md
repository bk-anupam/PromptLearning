
# **Product Requirements Document (PRD)**

Project Name: "Swasthya-Sahayak" (Working Title)

Version: 1.0

Owner: Solo Developer

Status: Draft

## **1. Executive Summary**

A local-first, offline-capable AI agent that helps Indian patients understand their medical history. It acts as a "Medical Translator," converting complex English medical reports (PDFs, Images) into simple, vernacular explanations (Hindi/Tamil/English) without sending private data to the cloud.

## **2. Problem Statement**

- **The Gap:** Indian patients often possess physical files of medical reports but lack the literacy or medical knowledge to understand them.
    
- **The Pain:** Doctors are too rushed to explain details. Patients are anxious and resort to Google, which often misleads them.
    
- **The Constraint:** Existing AI solutions require high-speed internet and cloud privacy trade-offs, which are barriers for the target demographic.
    

## **3. User Personas**

- **Persona A: "Ramesh" (The Patient)**
    
    - _Profile:_ 45-year-old, semi-urban, comfortable with WhatsApp but not complex apps. Speaks Hindi/English mix.
        
    - _Goal:_ Wants to know if his blood sugar report is "bad" and what he should eat.
        
- **Persona B: "Priya" (The Caregiver)**
    
    - _Profile:_ 25-year-old daughter managing her elderly father's chronic heart condition.
        
    - _Goal:_ Needs to organize 5 years of fragmented paper records into a chronological timeline to show the new doctor.
        

---

## **4. Functional Requirements (The "What")**

### **4.1 Module 1: The "Digital Vault" (Ingestion)**

- **FR-1.1 (File Upload):** System must accept images (`.jpg`, `.png`) and documents (`.pdf`).
    
- **FR-1.2 (Auto-Classification):** System must automatically tag uploaded files into categories: `Prescription`, `Lab Report`, `Radiology (MRI/CT)`, or `Discharge Summary`.
    
- **FR-1.3 (Privacy-Preserving OCR):** Text extraction must happen locally on the CPU/GPU using Tesseract/PaddleOCR. No data leaves the device.
    

### **4.2 Module 2: The "Timeline Engine" (Storage)**

- **FR-2.1 (Chronological Sorting):** System must extract the _Date of Visit_ from the document text to order records, not the _Date of Upload_.
    
- **FR-2.2 (structured_summary):** Every document must have a 3-line AI-generated summary stored in the SQL database for fast retrieval.
    

### **4.3 Module 3: The "Vernacular Doctor" (Interaction)**

- **FR-3.1 (Q&A Interface):** A chat window where users can ask questions in natural language (Hinglish supported).
    
- **FR-3.2 (Context Awareness):** The agent must "remember" previous reports in the current session (e.g., _"Is this worse than my report from last year?"_).
    
- **FR-3.3 (Hallucination Guard):** If the model is unsure of a medical term, it must output: _"I cannot read this clearly/I do not know"_ rather than guessing.
    

---

## **5. Technical Specifications (The "How")**

|**Component**|**Technology Selection**|**Rational**|
|---|---|---|
|**Frontend**|**Streamlit** (Python)|Fastest way to build UI with file uploaders and chat bubbles.|
|**Orchestrator**|**LangGraph**|Required to manage the "looping" logic (e.g., _Check Report -> Missing Value -> Ask User_).|
|**LLM Inference**|**Ollama**|Runs the GGUF model as a local API service.|
|**Model**|**MedGemma-4B (Q4_K_M)**|Optimized for medical reasoning; fits in 6GB VRAM.|
|**Database**|**SQLite**|Zero-config, single-file database. Easy to backup.|
|**OCR Engine**|**PaddleOCR**|Better than Tesseract at reading tabular data (tables) in blood reports.|

### **5.1 Data Schema (Draft)**

- **Table:** `medical_docs`
    
    - `id` (PK), `patient_id`, `event_date` (ISO8601), `category`, `summary`, `raw_text`, `file_path`, `is_abnormal` (Boolean flag).
        

---

## **6. Success Metrics (KPIs)**

Since you are a solo dev, don't overcomplicate this. Focus on **Trust** and **Speed**.

- **Extraction Accuracy:** Can the system correctly identify the "HbA1c" value from 10 different lab report formats? (Target: >90%).
    
- **Latency:** Time from "Upload" to "Summary Ready." (Target: <30 seconds on your GTX 1060).
    
- **Safety:** Zero occurrences of "Hallucinated Normalcy" (i.e., telling a patient a critical value is normal when it is not).
    

---

## **7. Risk & Mitigation Strategies**

|**Risk**|**Impact**|**Mitigation Strategy**|
|---|---|---|
|**Hallucination**|Critical (Patient harm)|**1.** Use a "Sanity Check" layer (Python script checks if extracted numbers are within valid human ranges).<br><br>  <br><br>**2.** Force the UI to display the _original image crop_ next to the AI's text answer.|
|**Slow Performance**|High (User frustration)|Use the **2-Tier Architecture**: Use `Llama-3-8B` for fast chatting and `MedGemma-27B` (offloaded) only for deep analysis of complex scans.|
|**Language Nuance**|Medium|Use "Hinglish" prompts. If the model outputs pure Hindi, it might be too formal. Prompt it to "Speak like an Indian family doctor."|

---

## **8. Roadmap (Phasing)**

- **Phase 1: The "Reader" (Weeks 1-2)**
    
    - Build Streamlit UI.
        
    - Implement Drag-and-Drop + OCR.
        
    - Display raw text extraction.
        
- **Phase 2: The "Thinker" (Weeks 3-4)**
    
    - Connect LangGraph + Ollama.
        
    - Implement the "Summarize" node.
        
    - Store metadata in SQLite.
        
- **Phase 3: The "Advisor" (Weeks 5-6)**
    
    - Build the Chat interface.
        
    - Add the "Medical Disclaimer" footer.
        
    - Test with 5 real anonymized patient histories.


# Claude Sonnet 4.5 enhanced PRD:

# Product Requirements Document (PRD)

**Project Name:** "Swasthya-Sahayak" (Health Assistant)  
**Version:** 2.0 (Enhanced)  
**Owner:** Solo Developer  
**Status:** Requirements Definition  
**Last Updated:** November 2025

---

## 1. Executive Summary

A **privacy-first, offline-capable AI medical assistant** designed for Indian patients to understand their medical records. The system acts as a "Medical Translator," converting complex English medical reports (PDFs, images, prescriptions) into simple, vernacular explanations (Hindi/Tamil/English) **without sending private health data to the cloud**.

**Core Value Proposition:** Empowers patients to understand their own health data, organize fragmented medical histories, and ask informed questions to their doctors—all while maintaining complete data privacy.

---

## 2. Problem Statement

### The Gap
Indian patients often possess physical medical reports but lack:
- Medical literacy to interpret clinical terminology
- English proficiency to read reports
- Organized chronological records spanning multiple hospitals

### The Pain Points
- **Information Asymmetry**: Doctors spend <5 minutes per patient; no time for detailed explanations
- **Misinformation Risk**: Patients resort to Google searches that cause anxiety or misguidance
- **Fragmented Records**: Paper reports across multiple hospitals make continuity of care difficult
- **Language Barriers**: Most reports are in English; patient speaks Hindi/regional languages

### The Constraint
- Existing AI solutions (ChatGPT, Gemini) require:
  - Stable high-speed internet (not available in Tier 2/3 cities)
  - Uploading sensitive health data to cloud (privacy concerns)
  - Subscription fees (barrier for middle-income families)

### Target Market Size
- **Primary:** 50M+ urban/semi-urban Indians with chronic conditions (diabetes, hypertension, joint issues)
- **Secondary:** 200M+ smartphone users seeking health information

---

## 3. User Personas

### Persona A: "Ramesh" (The Patient)
**Profile:**
- Age: 45, small business owner
- Location: Kanpur (Tier 2 city)
- Education: 10th grade, comfortable with WhatsApp but not complex apps
- Language: Hindi primary, basic English
- Health: Type 2 diabetes, hypertension

**Goals:**
- Understand if his HbA1c value (7.8%) is "bad"
- Know what foods to avoid based on lipid profile
- Compare today's report with last month's

**Pain Points:**
- Forgets to bring old reports to doctor appointments
- Can't read medical terms like "dyslipidemia"
- Anxious about numbers but doesn't know what's normal

**Quote:** *"Doctor said my 'triglycerides' are high. What does that even mean? Can I eat ghee or not?"*

---

### Persona B: "Priya" (The Caregiver)
**Profile:**
- Age: 28, IT professional
- Location: Bangalore
- Education: Engineering degree
- Language: English/Hindi bilingual
- Situation: Managing elderly father's chronic heart condition remotely

**Goals:**
- Organize 5 years of fragmented paper records into a digital timeline
- Track medication adherence and side effects
- Prepare a comprehensive medical summary for new cardiologist consultation

**Pain Points:**
- Father's reports scattered across 3 different hospitals
- No centralized record; each doctor sees only partial history
- Difficult to spot trends (is BP control improving or worsening?)

**Quote:** *"I need to show the new doctor my father's complete cardiac history, but I have 47 PDFs with no dates or labels."*

---

### Persona C: "Dr. Sharma" (The Overworked Physician - Secondary User)
**Profile:**
- Age: 38, general practitioner
- Location: Delhi government hospital
- Situation: Sees 80+ patients per day

**Pain Point:**
- Patients arrive with no prior records or incomplete histories
- Spends first 5 minutes just figuring out patient's background

**Potential Value:**
- If patients arrive with an **AI-generated chronological summary**, consultation quality improves

---

## 4. Core Principles (Non-Negotiables)

1. **Privacy-First Architecture**: All data processing happens locally; zero cloud uploads
2. **Offline-Capable**: Must work without internet after initial setup
3. **Hallucination Prevention**: Never fabricate medical information; admit uncertainty
4. **Vernacular-Native**: Hindi should feel natural, not Google-translated
5. **Transparency**: Always show original document alongside AI interpretation

---

## 5. Functional Requirements

### 5.1 Module 1: The "Digital Vault" (Document Ingestion)

**FR-1.1: Multi-Format File Upload**
- **Accepted Formats**: JPG, PNG, PDF, HEIC (iPhone photos)
- **Upload Methods**: 
  - Drag-and-drop in web UI
  - Bulk upload (select multiple files)
  - *(Future)* WhatsApp-style camera capture
- **File Size Limit**: 25MB per file
- **Validation**: Reject non-medical files (e.g., memes, screenshots)

**FR-1.2: Intelligent Auto-Classification**
- **Categories**: 
  - Lab Report (Blood Test, Urine Test, Lipid Profile)
  - Radiology (X-Ray, MRI, CT, Ultrasound)
  - Prescription
  - Discharge Summary
  - Consultation Notes
  - Vaccination Record
- **Confidence Score**: Display classification confidence (e.g., "85% confident this is a Blood Test")
- **Manual Override**: User can correct misclassifications

**FR-1.3: Privacy-Preserving OCR**
- **Engine**: PaddleOCR (primary), Tesseract (fallback)
- **Language Support**: English, Hindi, Tamil, Telugu, Marathi
- **Table Detection**: Must correctly extract tabular lab results (columns for Test Name, Value, Reference Range)
- **Handwriting Recognition**: Best-effort for handwritten prescriptions (common in India)
- **Processing Location**: 100% local CPU/GPU; no external API calls

**FR-1.4: Document Quality Feedback**
- **Image Quality Check**: Warn if image is too blurry/dark (OCR will fail)
- **Suggestion**: "Please retake photo in better lighting"

---

### 5.2 Module 2: The "Timeline Engine" (Storage & Organization)

**FR-2.1: Chronological Intelligence**
- **Date Extraction**: Auto-detect "Report Date" from document text
  - Handle multiple date formats: DD/MM/YYYY, DD-Mon-YYYY, YYYY-MM-DD
  - Prefer "Report Date" over "Collection Date" or "Upload Date"
- **Date Ambiguity Handling**: If date unclear, prompt user: "When was this test done?"
- **Timeline View**: Visual timeline (like Facebook timeline) showing all medical events

**FR-2.2: AI-Generated Structured Summaries**
- **3-Line Summary**: Store in database for fast retrieval
  - Example: "Blood test from Apollo Hospital. HbA1c: 7.8% (High). Cholesterol: Normal."
- **Key Abnormalities Flagged**: Automatically tag records with critical findings
- **Comparison Metadata**: Store previous values for trend analysis

**FR-2.3: Multi-Patient Support**
- **Family Accounts**: One user can manage records for 4-5 family members
- **Patient Switching**: Dropdown to switch between "Self", "Father", "Mother", "Child"
- **Data Isolation**: Each patient's data stored in separate encrypted folders

**FR-2.4: Smart Tagging System**
- **Auto-Tags**: Body part (Heart, Liver, Kidney), Condition (Diabetes, Hypertension)
- **Custom Tags**: User can add tags like "Before Surgery", "Post-Diet"
- **Search by Tag**: "Show me all kidney-related reports from 2024"

---

### 5.3 Module 3: The "Vernacular Doctor" (AI Interaction)

**FR-3.1: Conversational Q&A Interface**
- **Input Methods**: 
  - Text chat (Hinglish supported: "Mera sugar kitna hai?")
  - *(Future)* Voice input in Hindi
- **Example Queries**:
  - "Is my cholesterol normal?"
  - "Compare my last 3 HbA1c reports"
  - "What does 'mild hepatomegaly' mean?"
  - "Can I eat bananas with this sugar level?"

**FR-3.2: Session-Level Context Awareness**
- **Memory Scope**: Remember last 10 conversational turns
- **Cross-Document References**: "Is this worse than last year?" → Agent fetches 2023 report
- **Clarification Questions**: "Which report are you asking about—the one from January or March?"

**FR-3.3: Hallucination Prevention (Critical)**
- **Uncertainty Handling**: 
  - If medical term is unclear: *"I cannot read this term clearly. Please verify with your doctor."*
  - If question is beyond scope: *"I am not trained to answer this. Please consult a healthcare professional."*
- **No Diagnosis Claims**: Never say "You have X disease"; only explain what the report shows
- **No Treatment Advice**: Never prescribe medications or dosages

**FR-3.4: Explainability & Citations**
- **Source Attribution**: Every AI response must cite the source document
  - Example: *"According to your blood test from 15-Jan-2024, your HbA1c is 7.8%."*
- **Side-by-Side View**: Show original report snippet alongside AI explanation
- **Confidence Scores**: "I am 90% confident this value is 7.8"

**FR-3.5: Vernacular Output Quality**
- **Natural Hindi**: Avoid overly formal/Sanskritized Hindi
  - Good: "Aapka sugar level thoda zyada hai" (Your sugar level is a bit high)
  - Bad: "Aapka rakta sharkara star uchcha hai" (Overly formal)
- **Mixed Code**: Support Hinglish naturally (how real Indians speak)
- **Tone**: Empathetic family doctor, not robotic medical encyclopedia

**FR-3.6: Medical Disclaimer (Always Visible)**
- **Footer on Every Response**: *"This is an AI assistant, not a replacement for your doctor. Always consult a healthcare professional for medical decisions."*

---

### 5.4 Module 4: "Report Comparison & Trend Analysis" (New)

**FR-4.1: Automatic Trend Detection**
- **Longitudinal Analysis**: If user has 3+ reports of same test, auto-generate trend chart
  - Example: HbA1c over time (7.2% → 7.5% → 7.8%) with up-arrow
- **Alert on Deterioration**: "Your kidney function (eGFR) has declined by 15% in 6 months. Discuss with your doctor."

**FR-4.2: Visual Comparisons**
- **Side-by-Side Reports**: Place two reports next to each other with differences highlighted
- **Change Summary**: "Cholesterol improved by 20 mg/dL since last test"

---

## 6. Technical Architecture

### 6.1 Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Frontend** | Streamlit | Fastest MVP; built-in file uploader and chat UI |
| **Agent Orchestration** | LangGraph | Handles conditional logic, loops, memory; perfect for medical decision trees |
| **LLM Inference** | Ollama + llama.cpp | Local API server for GGUF models; GPU acceleration |
| **Model (Primary)** | MedGemma-4B (Q4_K_M) | Medical reasoning; fits in 6GB VRAM |
| **Model (Fallback)** | Llama-3.2-3B | For fast non-medical chat; conserve VRAM |
| **Database** | SQLite | Zero-config; single-file; easy backup |
| **OCR Engine** | PaddleOCR (primary), Tesseract (fallback) | Better at Indian medical report formats; table extraction |
| **Document Processing** | PyMuPDF, Pillow | PDF text extraction, image manipulation |
| **Encryption** | Cryptography (Fernet) | AES encryption for patient data at rest |

### 6.2 System Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                 Streamlit Frontend                   │
│  (File Upload, Chat UI, Timeline Viewer)            │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│              LangGraph Agent Controller              │
│  ┌──────────────┬───────────────┬─────────────┐    │
│  │Ingestion Node│Analysis Node  │Chat Node    │    │
│  └──────────────┴───────────────┴─────────────┘    │
└────────┬────────────────┬─────────────────┬─────────┘
         │                │                 │
         ▼                ▼                 ▼
┌────────────────┐ ┌──────────────┐ ┌──────────────┐
│  OCR Pipeline  │ │ Ollama Server│ │SQLite Database│
│ (PaddleOCR +   │ │ (MedGemma-4B)│ │(Patient Data) │
│  Tesseract)    │ │              │ │+ Vector Store │
└────────────────┘ └──────────────┘ └──────────────┘
         │                │                 │
         ▼                ▼                 ▼
┌─────────────────────────────────────────────────────┐
│           Local File System (Encrypted)              │
│   /patient_001/reports/*.pdf                        │
│   /patient_001/scans/*.jpg                          │
└─────────────────────────────────────────────────────┘
```

### 6.3 Data Schema (Enhanced)

```sql
-- Patients table
CREATE TABLE patients (
    patient_id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    date_of_birth DATE,
    gender TEXT,
    preferred_language TEXT DEFAULT 'hindi', -- NEW
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Medical records table
CREATE TABLE medical_records (
    record_id TEXT PRIMARY KEY,
    patient_id TEXT NOT NULL,
    record_type TEXT NOT NULL, -- 'lab_test', 'radiology', 'prescription'
    record_subtype TEXT, -- 'blood_test', 'mri_spine', 'lipid_profile'
    event_date DATE NOT NULL, -- Extracted from document
    upload_date DATE DEFAULT CURRENT_TIMESTAMP,
    body_part TEXT, -- 'heart', 'liver', 'spine'
    facility_name TEXT, -- Hospital/lab name
    
    -- File storage
    file_path TEXT NOT NULL,
    file_type TEXT, -- 'pdf', 'jpg'
    thumbnail_path TEXT,
    
    -- OCR output
    extracted_text TEXT, -- Full OCR text
    structured_data JSON, -- Parsed key-value pairs (e.g., {"HbA1c": "7.8%"})
    
    -- AI-generated metadata
    ai_summary TEXT, -- 3-line summary (NEW)
    abnormal_findings JSON, -- List of flagged abnormalities (NEW)
    confidence_score REAL, -- OCR confidence (0-1)
    
    -- Tags
    tags TEXT, -- JSON array: ['diabetes', 'pre_surgery']
    
    -- Processing status
    processing_status TEXT DEFAULT 'pending', -- 'pending', 'processed', 'failed'
    ocr_language TEXT, -- 'eng', 'hin', 'mixed'
    
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id)
);

CREATE INDEX idx_patient_date ON medical_records(patient_id, event_date DESC);
CREATE INDEX idx_abnormal ON medical_records(patient_id, abnormal_findings) 
    WHERE abnormal_findings IS NOT NULL; -- NEW: Fast retrieval of flagged reports

-- Timeline events (extracted key medical events)
CREATE TABLE timeline_events (
    event_id TEXT PRIMARY KEY,
    patient_id TEXT NOT NULL,
    event_date DATE NOT NULL,
    event_type TEXT, -- 'diagnosis', 'surgery', 'medication_change', 'hospitalization'
    event_description TEXT,
    severity TEXT, -- 'routine', 'concern', 'critical'
    related_record_ids TEXT, -- JSON array of record_ids
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id)
);

-- Chat history (NEW)
CREATE TABLE chat_history (
    chat_id TEXT PRIMARY KEY,
    patient_id TEXT NOT NULL,
    user_message TEXT NOT NULL,
    ai_response TEXT NOT NULL,
    context_record_ids TEXT, -- Which documents were referenced
    language TEXT, -- 'hindi', 'english', 'hinglish'
    feedback_rating INTEGER, -- 1-5 stars (optional user feedback)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id)
);

-- Analysis history (LLM interpretations)
CREATE TABLE analysis_history (
    analysis_id TEXT PRIMARY KEY,
    patient_id TEXT NOT NULL,
    analysis_type TEXT, -- 'report_interpretation', 'trend_analysis', 'comparison'
    input_record_ids TEXT, -- JSON array
    llm_model TEXT, -- 'medgemma-4b-it'
    prompt_hash TEXT, -- For deduplication
    llm_response TEXT,
    processing_time_seconds REAL, -- NEW: Performance tracking
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id)
);
```

### 6.4 LangGraph Workflow Design

```python
from langgraph.graph import StateGraph, END

# Define agent states
class AgentState(TypedDict):
    patient_id: str
    uploaded_file: bytes
    ocr_text: str
    classification: str
    extracted_date: str
    ai_summary: str
    user_query: str
    chat_history: List[dict]
    final_response: str

# Define nodes
def ocr_node(state):
    """Extract text from uploaded document"""
    text = run_paddleocr(state['uploaded_file'])
    return {"ocr_text": text}

def classify_node(state):
    """Classify document type using MedGemma"""
    prompt = f"Classify this medical document: {state['ocr_text'][:500]}"
    classification = call_llm(prompt)
    return {"classification": classification}

def extract_metadata_node(state):
    """Extract date, hospital name, test values"""
    # Use regex + LLM hybrid approach
    date = extract_date(state['ocr_text'])
    structured_data = extract_key_values(state['ocr_text'])
    return {"extracted_date": date, "structured_data": structured_data}

def summarize_node(state):
    """Generate 3-line summary"""
    prompt = f"Summarize this {state['classification']} in 3 lines: {state['ocr_text']}"
    summary = call_llm(prompt)
    return {"ai_summary": summary}

def store_node(state):
    """Save to SQLite database"""
    save_to_database(state)
    return {"status": "saved"}

def chat_node(state):
    """Handle user query with full context"""
    context = retrieve_patient_context(state['patient_id'])
    prompt = build_chat_prompt(state['user_query'], context, state['chat_history'])
    response = call_llm(prompt)
    return {"final_response": response}

# Build workflow
workflow = StateGraph(AgentState)
workflow.add_node("ocr", ocr_node)
workflow.add_node("classify", classify_node)
workflow.add_node("extract", extract_metadata_node)
workflow.add_node("summarize", summarize_node)
workflow.add_node("store", store_node)
workflow.add_node("chat", chat_node)

# Define edges (flow)
workflow.add_edge("ocr", "classify")
workflow.add_edge("classify", "extract")
workflow.add_edge("extract", "summarize")
workflow.add_edge("summarize", "store")
workflow.set_entry_point("ocr")

# For chat mode, use separate entry
workflow.add_conditional_edges(
    "chat",
    lambda state: "clarify" if needs_clarification(state) else END
)
```

---

## 7. Non-Functional Requirements

### 7.1 Performance
- **Upload to Summary**: <30 seconds on GTX 1060 (6GB VRAM)
- **Chat Response Latency**: <5 seconds for simple queries, <15 seconds for complex analysis
- **Concurrent Processing**: Handle 1 file upload + 1 chat query simultaneously
- **Memory Footprint**: <8GB RAM usage at peak

### 7.2 Security & Privacy
- **Encryption at Rest**: AES-256 encryption for all patient files
- **No Cloud Sync**: Absolutely zero data transmission to external servers
- **Audit Logging**: Track all access to patient records (who viewed what, when)
- **Session Timeout**: Auto-lock after 10 minutes of inactivity

### 7.3 Reliability
- **OCR Failure Handling**: If OCR confidence <70%, prompt user to retake photo
- **LLM Timeout**: If model doesn't respond in 30s, show error and retry
- **Database Corruption Recovery**: Daily automated backups; one-click restore

### 7.4 Usability
- **Mobile Responsive**: UI must work on tablets (many users will use iPads)
- **Low Digital Literacy**: Maximum 3 clicks to upload and view summary
- **Error Messages**: In vernacular ("फ़ाइल अपलोड नहीं हुई" not "Upload failed")

### 7.5 Accessibility
- **Font Size**: Minimum 16px for body text (elderly users)
- **High Contrast Mode**: For visually impaired users
- **Screen Reader Support**: *(Future)* For blind users

---

## 8. Success Metrics (KPIs)

### Phase 1 (MVP): Technical Validation
| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| **OCR Accuracy** | >90% for printed reports, >70% for handwritten | Manual verification on 100 sample reports |
| **Classification Accuracy** | >85% correct auto-tagging | Test on 50 diverse reports |
| **Hallucination Rate** | <2% (AI inventing information) | Red-team testing with adversarial prompts |
| **Processing Latency** | <30s for PDF, <15s for image | Automated benchmarking |

### Phase 2 (User Testing): Adoption & Trust
| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| **User Activation** | 5 users upload ≥3 reports each | Usage analytics |
| **Repeat Usage** | 70% return within 7 days | Retention analysis |
| **Safety Incidents** | Zero cases of harmful medical advice | User interviews + feedback |
| **Perceived Accuracy** | 4/5 stars on "Did this help you understand?" | In-app survey |

### Phase 3 (Scale): Impact
| Metric | Target (6 months) | Measurement Method |
|--------|--------|-------------------|
| **Active Users** | 500 families | Registration data |
| **Reports Processed** | 10,000 documents | Database count |
| **Doctor Consultation Improvement** | 60% users report "better prepared for doctor visit" | Post-consultation survey |

---

## 9. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation Strategy |
|------|-----------|--------|-------------------|
| **Hallucination (AI invents medical facts)** | Medium | **CRITICAL** | 1. Implement "Sanity Check" layer (Python validation of extracted values against human ranges)<br>2. Always display original document crop next to AI response<br>3. Add prominent disclaimer on every response<br>4. Red-team testing with adversarial inputs |
| **Slow Performance on Low-End Hardware** | High | High | 1. Use 2-tier LLM strategy: Llama-3-3B for chat, MedGemma-4B only for deep analysis<br>2. Implement aggressive caching of summaries<br>3. Optimize with quantization (Q4_K_M) |
| **OCR Failure on Poor-Quality Images** | High | Medium | 1. Pre-upload image quality check (blur detection)<br>2. Guide user: "Retake photo in better lighting"<br>3. Allow manual text entry as fallback |
| **Language Nuance (Hindi output sounds unnatural)** | Medium | Medium | 1. Few-shot prompting with colloquial examples<br>2. User feedback loop: "Was this explanation clear?"<br>3. Hinglish mode as default (more natural) |
| **User Uploads Non-Medical Files** | Medium | Low | 1. Pre-classification filter (reject if no medical keywords)<br>2. Warning: "This doesn't look like a medical report" |
| **Data Loss (Disk Failure)** | Low | High | 1. Daily automated backups to external drive<br>2. One-click export to USB/Google Drive (encrypted)<br>3. Checksum validation on startup |
| **Regulatory Scrutiny (Practicing Medicine Without License)** | Low | **CRITICAL** | 1. Prominent disclaimers: "Not a diagnostic tool"<br>2. Never provide treatment recommendations<br>3. Terms of Service: "For educational purposes only"<br>4. Consult legal expert before public release |

---

## 10. Ethical & Legal Considerations

### 10.1 Medical Disclaimers
**Every screen must display:**
> ⚕️ **Important:** Swasthya-Sahayak is an educational tool, not a replacement for professional medical advice. Always consult a qualified healthcare provider for diagnosis and treatment.

### 10.2 Scope Boundaries (What We DON'T Do)
- ❌ Diagnose diseases
- ❌ Prescribe medications or dosages
- ❌ Recommend specific treatments
- ❌ Interpret complex imaging (MRI/CT beyond text reports)
- ❌ Provide mental health counseling
- ❌ Handle emergency medical situations

### 10.3 Data Governance
- **Ownership**: User owns their data; can delete anytime
- **Portability**: One-click export to JSON/PDF
- **Anonymization**: Remove PII (name, phone) before sharing for debugging
- **Consent**: Explicit opt-in for any data usage (even anonymized)

### 10.4 Regulatory Compliance
- **Not a Medical Device**: Positioned as "document reader," not diagnostic tool (avoids FDA/CDSCO registration)
- **Consumer Software**: Marketed as organization/education tool, not clinical decision support
- **Legal Review**: Consult healthcare lawyer before public launch

---

## 11. Development Roadmap

### Phase 1: "The Reader" (Weeks 1-3) - MVP
**Goal:** Upload a report, see extracted text and AI summary

**Deliverables:**
- ✅ Streamlit UI with drag-and-drop upload
- ✅ PaddleOCR integration (English + Hindi)
- ✅ Display raw OCR text
- ✅ Basic SQLite schema (patients + medical_records tables)
- ✅ File storage in local encrypted folder

**Success Criteria:**
- Can upload 5 different report formats (Apollo, Max, Thyrocare, etc.)
- OCR accuracy >80% on printed text

---

### Phase 2: "The Thinker" (Weeks 4-6) - Intelligence Layer
**Goal:** AI-generated summaries and auto-classification

**Deliverables:**
- ✅ LangGraph workflow (OCR → Classify → Summarize)
- ✅ Ollama + MedGemma-4B integration
- ✅ 3-line summary generation
- ✅ Auto-tagging (body part, test type)
- ✅ Store summaries in database

**Success Criteria:**
- Classification accuracy >85%
- Summaries are factually correct (verified by manual review of 20 reports)

---

### Phase 3: "The Advisor" (Weeks 7-9) - Conversational Interface
**Goal:** Users can ask questions in natural language

**Deliverables:**
- ✅ Chat interface in Streamlit
- ✅ Context-aware responses (agent retrieves relevant past reports)
- ✅ Hinglish support ("Mera cholesterol kitna hai?")
- ✅ Hallucination guard (uncertainty handling)
- ✅ Medical disclaimer footer

**Success Criteria:**
- 5 beta users test with real reports
- Zero safety incidents (harmful advice)
- 4/5 user satisfaction rating

---

### Phase 4: "The Timeline" (Weeks 10-12) - Organization
**Goal:** Chronological view of medical history

**Deliverables:**
- ✅ Timeline UI (visual representation of all reports)
- ✅ Date extraction from documents
- ✅ Trend charts for repeated tests (HbA1c over time)
- ✅ Compare two reports side-by-side
- ✅ Export timeline to PDF

**Success Criteria:**
- Timeline correctly orders 10+ reports from different years
- Trend detection works for 3+ sequential tests

---

### Phase 5: "The Explainer" (Weeks 13-15) - Vernacular Excellence
**Goal:** High-quality Hindi/Tamil explanations

**Deliverables:**
- ✅ Improved Hindi prompts (colloquial, not formal)
- ✅ Language switcher (English ↔ Hindi ↔ Tamil)
- ✅ Medical term glossary (hover to see meaning)
- ✅ Audio output (text-to-speech for low-literacy users)

**Success Criteria:**
- Native Hindi speakers rate explanations as "natural"
- Supports 3 languages (English, Hindi, Tamil)

---

### Phase 6: "The Guardian" (Weeks 16-18) - Safety & Scale
**Goal:** Production-ready with robust error handling

**Deliverables:**
- ✅ Sanity check layer (validate extracted values)
- ✅ Rate limiting (prevent abuse)
- ✅ Automated backup system
- ✅ Crash reporting and logging
- ✅ User authentication (optional PIN for privacy)

**Success Criteria:**
- Zero data loss in stress testing
- Handles 100 reports per user without slowdown
- Recovery from crash within 10 seconds

---

## 12. Out of Scope (Future Enhancements)

**Not in V1.0, but consider later:**
- ❌ Medication reminder notifications
- ❌ Telemedicine integration (video calls with doctors)
- ❌ Wearable device sync (Apple Watch, Fitbit)
- ❌ Community forum (patients s


