---
name: document-intelligence
description: Intelligent Document Processing (IDP) with Vision Language Models (VLMs). Prompt engineering for Gemini Vision, Chain of Thought (CoT) for document interpretation, confidence scoring, Context Caching for cost optimization, bounding box extraction, and structured data output with Pydantic AI. Use when extracting data from documents using AI vision, building OCR-free document processing pipelines, implementing confidence scoring for HITL, optimizing VLM costs with context caching, or designing document review UIs with bounding boxes. Use PROACTIVELY whenever the project involves document extraction, VLM processing, or IDP. Do NOT use for simple text file reading, traditional OCR setup (Tesseract), or image classification without text extraction.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Document Intelligence

> **Extract meaning from documents, not just text.**
> Zero-template document processing with VLMs — understand structure, not just pixels.

---

## ⚠️ Core Principle

- VLMs reason about documents — they don't just OCR them
- Chain of Thought (CoT) is the differentiator — use it
- Confidence scoring is mandatory for production — never trust blind extraction
- Context Caching saves 90% on costs — always plan for it
- Pydantic AI guarantees structured output — no post-processing

---

## 1. VLM Selection

### Decision Tree

```
What VLM to use?
│
├── High volume + cost-sensitive
│   └── Gemini 1.5 Flash (1M tokens, ~$1/6000 pages)
│
├── Maximum accuracy + complex layouts
│   └── Gemini 1.5 Pro (or GPT-4o for comparison)
│
├── On-premise / privacy requirements
│   └── Self-hosted models (DocTR, LayoutLM)
│
└── Simple, well-structured documents
    └── Consider traditional OCR first (cheaper)
```

### Gemini Vision Capabilities

| Feature | Support |
|---------|---------|
| Multi-page PDFs | ✅ Up to 1M tokens (~1500 pages) |
| Handwritten text | ✅ With CoT prompting |
| Tables | ✅ With structure reasoning |
| Forms with checkboxes | ✅ Visual reasoning |
| Rotated/skewed docs | ✅ Auto-correction |
| Low quality scans | ⚠️ Needs CoT for ambiguous chars |

---

## 2. Prompt Engineering for Document Extraction

### The CoT Extraction Pattern

The key differentiator of VLM-based IDP is Chain of Thought reasoning. Instead of "extract field X", instruct the model to reason about what it sees.

```python
EXTRACTION_PROMPT = """
You are a document analysis expert. Analyze this document and extract
the requested fields.

## Instructions

For each field:
1. **LOCATE**: Describe where on the page you see the field label or context
2. **READ**: State exactly what characters/text you see in that location
3. **REASON**: If the value seems ambiguous, explain your reasoning
   - Example: A "O" in a price column is likely "0" (zero)
   - Example: A "l" next to numbers is likely "1" (one)
4. **EXTRACT**: Provide the final interpreted value
5. **CONFIDENCE**: Rate your confidence (0.0 to 1.0)

## Fields to Extract

{field_definitions}

## Output Format

Return a JSON object following this exact schema:
{json_schema}

## Important Rules

- If a field is not visible in the document, set its value to null
  and confidence to 0.0
- For numeric fields, consider common OCR-like confusions:
  O↔0, l↔1, S↔5, B↔8
- For dates, normalize to ISO 8601 (YYYY-MM-DD)
- Always include your reasoning in the 'reasoning' field
"""
```

### Field Definition Pattern

```python
FIELD_DEFINITIONS = """
| Field | Description | Type | Location Hint |
|-------|------------|------|---------------|
| nome_completo | Full name of the professional | string | Usually near top of document |
| cpf | Brazilian CPF number (###.###.###-##) | string | Near name or identification section |
| data_nascimento | Date of birth | date | Personal data section |
| endereco | Full address | string | Address section |
| numero_registro | Professional registration number | string | Header or registration section |
| instituicao | Education institution name | string | Education/title section |
| curso | Course/degree name | string | Education/title section |
"""
```

### Prompt Optimization Techniques

| Technique | When to Use | Impact |
|-----------|-------------|--------|
| Few-shot examples | First deployment, complex layouts | +15-30% accuracy |
| CoT reasoning | Always for ambiguous fields | +20-40% accuracy |
| Field location hints | Known document types | +10% accuracy |
| Negative examples | Common errors | -50% error rate |
| Schema enforcement | Always | Eliminates parsing errors |

---

## 3. Confidence Scoring

### Confidence Framework

```python
from pydantic import BaseModel, Field
from typing import Optional

class ExtractedField(BaseModel):
    value: Optional[str] = None
    confidence: float = Field(ge=0.0, le=1.0)
    reasoning: str = ""
    bounding_box: Optional[dict] = None  # {x, y, width, height} normalized

class ExtractionResult(BaseModel):
    nome_completo: ExtractedField
    cpf: ExtractedField
    data_nascimento: ExtractedField
    endereco: ExtractedField
    numero_registro: ExtractedField
    instituicao: ExtractedField
    curso: ExtractedField

    def fields_needing_review(self, threshold: float = 0.8) -> list[str]:
        """Return field names with confidence below threshold."""
        review = []
        for field_name, field_value in self:
            if isinstance(field_value, ExtractedField):
                if field_value.confidence < threshold:
                    review.append(field_name)
        return review

    @property
    def overall_confidence(self) -> float:
        """Weighted average confidence across all fields."""
        fields = [v for _, v in self if isinstance(v, ExtractedField)]
        if not fields:
            return 0.0
        return sum(f.confidence for f in fields) / len(fields)

    @property
    def needs_review(self) -> bool:
        """Whether this extraction needs human review."""
        return len(self.fields_needing_review()) > 0
```

### HITL Threshold Strategy

```
Confidence thresholds:
│
├── ≥ 0.95 → Auto-accept (high confidence)
│
├── 0.80 - 0.94 → Accept with flag (review later)
│
├── 0.50 - 0.79 → Pause for HITL (human must verify)
│   └── Show document + extracted value + reasoning
│
└── < 0.50 → Reject (likely wrong)
    └── Highlight as error, request manual input
```

### Confidence Calibration

```python
def calibrate_confidence(raw_confidence: float, field_type: str) -> float:
    """Adjust VLM-reported confidence based on empirical accuracy."""
    # VLMs tend to be overconfident — apply calibration
    calibration_factors = {
        "cpf": 0.85,       # Numeric — often confused chars
        "date": 0.90,      # Format-sensitive
        "name": 0.95,      # Usually accurate
        "address": 0.80,   # Long text — more room for error
        "number": 0.85,    # Digit confusion is common
    }
    factor = calibration_factors.get(field_type, 0.90)
    return min(raw_confidence * factor, 1.0)
```

---

## 4. Context Caching (Gemini)

### What is Context Caching

Context Caching allows you to pre-cache the system instruction and few-shot examples in Gemini's API, paying for that context only once instead of on every request. This can reduce costs by up to 90% for repetitive extraction tasks.

### When to Cache

```
Cache when:
├── System prompt is long (> 2000 tokens)
├── Few-shot examples are included (> 5000 tokens)
├── Processing many documents with same instructions
├── Running batch extractions
└── Instruction context >> per-document context

Don't cache when:
├── Instructions change per document
├── Processing only a few documents
├── Low token count in system prompt
└── Real-time, one-off extractions
```

### Caching Implementation

```python
import google.generativeai as genai
from google.generativeai import caching
import datetime

def create_extraction_cache(
    system_instruction: str,
    few_shot_examples: list[dict],
    ttl_minutes: int = 60,
) -> caching.CachedContent:
    """Create a cached context for document extraction."""

    # Build the cached content with system instruction + examples
    cached = caching.CachedContent.create(
        model="gemini-1.5-flash-001",
        display_name="document-extraction-cache",
        system_instruction=system_instruction,
        contents=[
            # Few-shot examples as conversation history
            *[
                {"role": "user", "parts": [ex["image"], ex["prompt"]]}
                for ex in few_shot_examples
            ],
            *[
                {"role": "model", "parts": [ex["response"]]}
                for ex in few_shot_examples
            ],
        ],
        ttl=datetime.timedelta(minutes=ttl_minutes),
    )
    return cached

def extract_with_cache(
    cached_content: caching.CachedContent,
    document_image: bytes,
    fields_to_extract: str,
) -> dict:
    """Extract data using cached context (90% cheaper)."""
    model = genai.GenerativeModel.from_cached_content(cached_content)

    response = model.generate_content([
        {"mime_type": "image/png", "data": document_image},
        f"Extract these fields from the document:\n{fields_to_extract}",
    ])

    return response.text
```

### Cost Comparison

| Approach | Cost per 1000 pages | Notes |
|----------|-------------------|-------|
| No caching | ~$0.17 | Full prompt every time |
| With context caching | ~$0.02 | 90% reduction on cached tokens |
| Batch + caching | ~$0.01 | Optimal for high volume |

### TTL Strategy

| Scenario | Recommended TTL |
|----------|----------------|
| Batch processing (same session) | 30-60 min |
| Daily recurring workflow | 4-8 hours |
| Always-on service | 24 hours (renew on use) |

---

## 5. Pydantic AI for Structured Output

### Integration Pattern

```python
from pydantic_ai import Agent
from pydantic import BaseModel, Field
from typing import Optional

class ProfessionalData(BaseModel):
    """Structured output for professional registration documents."""
    nome_completo: str = Field(description="Full legal name")
    cpf: str = Field(description="CPF in format ###.###.###-##")
    data_nascimento: Optional[str] = Field(description="Date of birth (YYYY-MM-DD)")
    endereco: str = Field(description="Complete address")
    cidade: str = Field(description="City")
    uf: str = Field(description="State abbreviation (2 letters)")
    cep: str = Field(description="CEP in format #####-###")
    instituicao: str = Field(description="Education institution name")
    curso: str = Field(description="Degree/course name")
    ano_conclusao: Optional[int] = Field(description="Graduation year")

agent = Agent(
    "gemini-1.5-flash",
    result_type=ProfessionalData,
    system_prompt=EXTRACTION_PROMPT,
)

async def extract_document(image_path: str) -> ProfessionalData:
    """Extract structured data from document image."""
    result = await agent.run(
        f"Extract professional data from this document.",
        # Pass image via message parts
    )
    return result.data  # Validated ProfessionalData
```

### Validation & Retry

```python
from pydantic_ai import Agent, ModelRetry

agent = Agent(
    "gemini-1.5-flash",
    result_type=ProfessionalData,
    retries=3,  # Auto-retry on validation failure
)

@agent.result_validator
async def validate_extraction(result: ProfessionalData) -> ProfessionalData:
    """Custom validation for extracted data."""
    # Validate CPF format
    import re
    if result.cpf and not re.match(r"\d{3}\.\d{3}\.\d{3}-\d{2}", result.cpf):
        raise ModelRetry("CPF format is invalid. Must be ###.###.###-##")

    # Validate UF
    valid_ufs = ["AC","AL","AP","AM","BA","CE","DF","ES","GO","MA",
                 "MT","MS","MG","PA","PB","PR","PE","PI","RJ","RN",
                 "RS","RO","RR","SC","SP","SE","TO"]
    if result.uf and result.uf.upper() not in valid_ufs:
        raise ModelRetry(f"'{result.uf}' is not a valid Brazilian state. Check again.")

    return result
```

---

## 6. Bounding Box Extraction

### Why Bounding Boxes

```
Bounding boxes enable:
├── Side-by-side document review UI
├── Click-to-verify workflow
├── Visual proof of extraction source
├── Confidence visualization (color-coded)
└── Training data generation
```

### Bounding Box Prompt Pattern

```python
BBOX_PROMPT_ADDITION = """
For each extracted field, also provide the bounding box of the source text
in the document. Return as normalized coordinates (0.0 to 1.0):

{
  "bounding_box": {
    "x": 0.15,      // Left edge (0=left, 1=right)
    "y": 0.32,      // Top edge (0=top, 1=bottom)
    "width": 0.45,  // Width of the box
    "height": 0.03  // Height of the box
  }
}

If you cannot precisely locate the field, set bounding_box to null.
"""
```

### UI Rendering Pattern

```typescript
// React component for side-by-side document review
interface BoundingBox {
  x: number;
  y: number;
  width: number;
  height: number;
}

interface FieldHighlight {
  fieldName: string;
  value: string;
  confidence: number;
  boundingBox: BoundingBox | null;
}

function DocumentReview({
  imageUrl,
  fields,
}: {
  imageUrl: string;
  fields: FieldHighlight[];
}) {
  // Render document image with overlay bounding boxes
  // Color-code by confidence: green (>0.8), yellow (0.5-0.8), red (<0.5)
  // Click on a field → scroll to + highlight in side panel
}
```

---

## 7. Pipeline Architecture

### End-to-End Pipeline

```
Document Input → Pre-processing → VLM Extraction → Validation → HITL → Output
                       │                │                │           │
                  ┌────┘          ┌─────┘          ┌────┘     ┌────┘
                  │               │                │          │
            PDF to Images    Context Cache    Pydantic    Review UI
            Deskew/Clean     CoT Prompting    Scoring     Approve/Fix
            Quality Check    Batch Process    Threshold   Audio Alert
```

### Pre-processing

```python
async def preprocess_document(file_path: str) -> list[bytes]:
    """Convert document to optimized images for VLM processing."""
    from pdf2image import convert_from_path

    # Convert PDF pages to images
    images = convert_from_path(
        file_path,
        dpi=200,          # Balance quality vs token cost
        fmt="png",
        grayscale=False,   # Keep color for VLM context
    )

    # Optimize each page
    optimized = []
    for img in images:
        # Resize if too large (token cost)
        max_dimension = 2048
        if max(img.size) > max_dimension:
            img.thumbnail((max_dimension, max_dimension))

        buffer = io.BytesIO()
        img.save(buffer, format="PNG", optimize=True)
        optimized.append(buffer.getvalue())

    return optimized
```

### Batch Processing

```python
async def process_batch(
    documents: list[str],
    cache: caching.CachedContent,
    concurrency: int = 5,
) -> list[ExtractionResult]:
    """Process multiple documents with controlled concurrency."""
    semaphore = asyncio.Semaphore(concurrency)
    results = []

    async def process_one(doc_path: str) -> ExtractionResult:
        async with semaphore:
            images = await preprocess_document(doc_path)
            result = await extract_with_cache(cache, images[0], FIELDS)
            return result

    tasks = [process_one(doc) for doc in documents]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

---

## 8. Decision Checklist

Before building document intelligence:

- [ ] **Analyzed sample documents?** (5+ examples of each type)
- [ ] **Identified all fields to extract?**
- [ ] **Chosen VLM model?** (Flash vs Pro based on accuracy needs)
- [ ] **Designed CoT prompt with reasoning steps?**
- [ ] **Defined confidence thresholds?** (auto-accept vs HITL)
- [ ] **Implemented Pydantic models for validation?**
- [ ] **Planned Context Caching strategy?** (TTL, when to refresh)
- [ ] **Set up bounding box extraction?** (for review UI)
- [ ] **Calculated cost per document?** (with and without caching)
- [ ] **Tested with edge cases?** (blurry scans, rotated pages, handwriting)

---

## 9. Anti-Patterns

| ❌ Don't | ✅ Do |
|----------|-------|
| Use simple "extract X" prompts | Use CoT reasoning prompts |
| Trust VLM confidence blindly | Calibrate and validate confidence |
| Send full system prompt every request | Use Context Caching |
| Parse VLM text output with regex | Use Pydantic AI for structured output |
| Process at maximum resolution | Optimize image size for cost |
| Skip few-shot examples | Include 2-3 examples in cached context |
| Ignore character confusion (O/0, l/1) | Explicitly address in prompt |
| Store raw VLM responses only | Store structured + raw for debugging |
| Process documents sequentially | Use batch with controlled concurrency |

---

> **Remember:** Document Intelligence is about understanding, not just reading. A good VLM prompt makes the model reason about what it sees — not just transcribe pixels. The Chain of Thought approach turns a $0.02 API call into insights that previously required $50/hour human operators.
