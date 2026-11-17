# AI Resume Matcher — Matching Methodology & Algorithm Specification

This document details the matching methodology, NLP pipeline, feature extraction, scoring formulation, and business rules implemented in the **AI-Based Resume Matching & Candidate Ranking System** for Avom Consultants.

---

## 1. Overview & Philosophy

The system uses a **hybrid matching approach** combining:
1. **Explicit Skill Matching**: Interpretability, transparent evidence, and canonical keyword normalization.
2. **Semantic Similarity**: Deep contextual representation using sentence embeddings to capture paraphrasing, domain alignment, and experience depth beyond literal keywords.
3. **Structured Attribute Comparison**: Experience years, educational qualification tiers, and additional job requirements.

Scores are intended as **decision-support signals** for recruiters, not automated hiring decisions.

---

## 2. Weighted Scoring Formula

Every candidate is evaluated on a normalized $0–100$ scale across five independent components:

$$\text{Overall Score} = 0.40 \cdot S + 0.25 \cdot M + 0.20 \cdot E + 0.10 \cdot Q + 0.05 \cdot A$$

| Component | Symbol | Proposed Weight | Definition |
| :--- | :---: | :---: | :--- |
| **Skills Matching** | $S$ | **40%** | Normalized coverage of required and preferred JD skills identified in the candidate's resume. |
| **Semantic Similarity** | $M$ | **25%** | Cosine similarity between dense embeddings of the JD and the candidate's resume, calibrated to 0–100. |
| **Relevant Experience** | $E$ | **20%** | Proportional comparison of candidate's total/relevant years of experience vs. minimum required experience. |
| **Education / Qualifications** | $Q$ | **10%** | Alignment between candidate's extracted degree level/field and the stated JD qualification requirements. |
| **Additional Requirements** | $A$ | **5%** | Coverage of secondary stated criteria (certifications, agile workflows, architectural paradigms). |

---

## 3. Component Details & Formulation

### 3.1. Skills Matching ($S$ — 40%)

1. **Extraction & Canonical Normalization**:
   - Skills are parsed from dedicated resume sections (`SKILLS`, `TECHNICAL SKILLS`, `CORE COMPETENCIES`) and extracted across the document text using a canonical alias dictionary.
   - Example normalizations:
     - `FastAPI`, `fast api`, `FastAPI framework` $\rightarrow$ `fastapi`
     - `React`, `reactjs`, `react.js`, `react js` $\rightarrow$ `react`
     - `PostgreSQL`, `postgres`, `psql`, `pg` $\rightarrow$ `postgresql`
     - `Docker container`, `containerization` $\rightarrow$ `docker`
2. **Formula**:
   $$S = \min\left(100, \frac{|K_{\text{matched}}|}{|K_{\text{JD}}| + \epsilon} \times 100\right)$$
   Where:
   - $K_{\text{JD}}$ is the set of required and preferred skills extracted from the job description.
   - $K_{\text{matched}}$ is the subset of $K_{\text{JD}}$ found in the candidate's extracted skill set.
   - If the JD specifies no skills, $S = 100.0$ (neutral applicability; no penalty for unspecified requirements).

### 3.2. Semantic Similarity ($M$ — 25%)

1. **Embedding Generation**:
   - Uses the Sentence Transformers model `all-MiniLM-L6-v2` (384-dimensional dense vectors).
   - The model is loaded once at backend startup (`preload_model`) and kept in memory for high-throughput inference.
   - To optimize performance and satisfy NFR-004, the Job Description embedding $\vec{v}_{\text{JD}}$ is precomputed once per batch request and reused across all candidate evaluations.
2. **Raw Cosine Similarity**:
   $$\text{cosine\_sim}(\vec{v}_{\text{JD}}, \vec{v}_{\text{resume}}) = \frac{\vec{v}_{\text{JD}} \cdot \vec{v}_{\text{resume}}}{\|\vec{v}_{\text{JD}}\| \|\vec{v}_{\text{resume}}\|}$$
3. **Calibration to 0–100 Scale**:
   Raw text cosine similarity between domain-specific texts typically falls in the range $[0.1, 0.8]$. Presenting raw cosine similarity directly as a percentage would distort human expectations. Therefore, a piecewise monotonic calibration function maps raw cosine to an intuitive 0–100 scale:
   - $[0.0, 0.1) \rightarrow [0, 10]$ (negligible relevance)
   - $[0.1, 0.3) \rightarrow [10, 40]$ (low relevance)
   - $[0.3, 0.5) \rightarrow [40, 65]$ (moderate relevance)
   - $[0.5, 0.7) \rightarrow [65, 85]$ (strong relevance)
   - $[0.7, 1.0] \rightarrow [85, 100]$ (exceptional relevance)

### 3.3. Experience Score ($E$ — 20%)

1. **Rules (SRS §6.2 compliance)**:
   - If JD specifies no minimum experience requirement: $E = 100.0$ (no arbitrary penalty applied).
   - If candidate's experience $Y_{\text{cand}} \ge Y_{\text{required}}$: $E = 100.0$.
   - If candidate's experience $Y_{\text{cand}} < Y_{\text{required}}$:
     $$E = \max\left(0, \frac{Y_{\text{cand}}}{Y_{\text{required}}} \times 100\right)$$
   - If candidate's experience cannot be identified from the resume: $E = 50.0$ (neutral baseline with an explicit note in the explanation).

### 3.4. Education / Qualifications Score ($Q$ — 10%)

1. **Hierarchy Matching**:
   - Degree levels are hierarchically categorized: Ph.D. (5) > Master/M.Tech/MS/MBA (4) > Bachelor/B.Tech/BS/BE (3) > Diploma/Associate (2) > Certificate (1).
   - If candidate level $\ge$ JD level: Base score = 100.
   - If candidate level is one level below JD level (e.g., Bachelor applying for Master): Base score = 75.
   - If candidate has an identifiable degree but below requirement: Base score = 50.
   - If no specific degree is requested by the JD: $Q = 100.0$ if candidate holds any degree, or $70.0$ otherwise.
   - Field bonus: +10 points (capped at 100) if discipline keywords (Computer Science, Software, IT, Engineering) align between JD and candidate education.

### 3.5. Additional Requirements Score ($A$ — 5%)

1. Stated secondary criteria such as Agile/Scrum, microservices architecture, cloud platforms, or specific methodologies.
2. Evaluated through keyword density and presence across the candidate's projects and employment records.
3. If no additional requirements are stated in the JD: $A = 100.0$.

---

## 4. Handling Unspecified JD Criteria & Edge Cases

As mandated by SRS §6.2:
- **No Unfair Penalization**: If a job description omits a criterion (such as experience or specific degree), the candidate is not penalized; the component is scored neutrally/fully ($100$).
- **"Not Identified" $\ne$ "Lacks Skill"**: If a skill or contact detail is not extracted, the system explicitly labels it as `"Not identified"` and notes that absence in the resume does not constitute proof that the candidate does not possess the capability.
- **Independent Parsing**: A corrupted or unreadable file (e.g., unsupported format, scanned image-only PDF without OCR) produces a per-file error entry and does not crash or interrupt the remaining batch.
