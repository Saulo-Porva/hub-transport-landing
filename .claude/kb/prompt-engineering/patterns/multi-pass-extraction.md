# Multi-Pass Extraction

> Dividir extração em 3 passes (rough → validate → format) — aumenta precisão em documentos complexos.

## Problema

Uma única chamada LLM tenta parsear, validar e formatar ao mesmo tempo. Em documentos ambíguos (layouts irregulares, valores em formatos mistos, campos implícitos), isso resulta em extrações incorretas sem sinal de erro. O modelo "inventa" valores com aparente confiança.

## Solução

```python
from __future__ import annotations
from dataclasses import dataclass
from pydantic import BaseModel, Field
from typing import Any
import anthropic
import json

client = anthropic.Anthropic()

# --- Pass 1: Rough extraction (sem schema, texto livre) ---
ROUGH_PROMPT = """
Read this document carefully and describe what you find, field by field.
Do not format yet. Just describe what you observe.

Document:
{document}

Describe:
1. Who issued this document?
2. What is the document number/reference?
3. What is the date of issue?
4. What is the total amount? Exactly how is it written in the document?
5. Are there line items? List each one.
6. Any unusual or ambiguous aspects?
"""

def rough_extract(document: str) -> str:
    response = client.messages.create(
        model='claude-haiku-4-5-20251001',  # modelo rápido para pré-processamento
        max_tokens=1024,
        messages=[{'role': 'user', 'content': ROUGH_PROMPT.format(document=document)}],
        temperature=0,
    )
    return response.content[0].text

# --- Pass 2: Validation (detectar anomalias antes de formatar) ---
VALIDATION_PROMPT = """
Review this extraction analysis for accuracy and flag any issues.

Original document:
<document>
{document}
</document>

Extraction analysis:
<analysis>
{rough_extraction}
</analysis>

Validate:
1. Does the total match the sum of line items (if present)?
2. Is the date realistic?
3. Are there any inconsistencies or missing required fields?
4. What is your confidence level (0.0-1.0)?

Respond with JSON:
{{
  "is_valid": true|false,
  "confidence": 0.0-1.0,
  "issues": ["issue1", "issue2"],
  "corrections": {{"field": "corrected_value"}}
}}
"""

@dataclass
class ValidationResult:
    is_valid: bool
    confidence: float
    issues: list[str]
    corrections: dict[str, Any]

def validate_extraction(document: str, rough: str) -> ValidationResult:
    response = client.messages.create(
        model='claude-haiku-4-5-20251001',
        max_tokens=512,
        messages=[{'role': 'user', 'content': VALIDATION_PROMPT.format(
            document=document, rough_extraction=rough
        )}],
        temperature=0,
    )
    raw = response.content[0].text
    data = json.loads(raw)
    return ValidationResult(**data)

# --- Pass 3: Format (produzir JSON final tipado) ---
FORMAT_PROMPT = """
Using the analysis and corrections below, produce the final structured extraction.

Analysis: {rough}
Corrections to apply: {corrections}

Produce ONLY valid JSON matching this schema:
{schema}

Rules:
- Apply all corrections from the validation step
- Dates: YYYY-MM-DD
- Amounts: float (no symbols or separators)
- Missing fields: null
"""

class InvoiceResult(BaseModel):
    vendor_name: str
    invoice_number: str
    invoice_date: str
    total_amount: float
    confidence: float = Field(ge=0, le=1)
    issues: list[str] = Field(default_factory=list)

def format_extraction(rough: str, validation: ValidationResult) -> InvoiceResult:
    schema = json.dumps(InvoiceResult.model_json_schema(), indent=2)
    prompt = FORMAT_PROMPT.format(
        rough=rough,
        corrections=json.dumps(validation.corrections),
        schema=schema,
    )
    response = client.messages.create(
        model='claude-sonnet-4-6',
        max_tokens=1024,
        messages=[{'role': 'user', 'content': prompt}],
        temperature=0,
    )
    result = InvoiceResult.model_validate_json(response.content[0].text)
    result.issues = validation.issues  # propagar issues da validação
    return result

# --- Pipeline completo ---
def extract_invoice_multipass(document: str) -> InvoiceResult | None:
    import logging
    logger = logging.getLogger(__name__)

    rough = rough_extract(document)
    logger.debug('Rough extraction complete')

    validation = validate_extraction(document, rough)
    logger.info('Validation complete', extra={
        'confidence': validation.confidence,
        'issues': validation.issues,
    })

    if validation.confidence < 0.5:
        logger.error('Confidence too low to proceed', extra={'issues': validation.issues})
        return None

    result = format_extraction(rough, validation)
    logger.info('Extraction complete', extra={'confidence': result.confidence})
    return result
```

## Variações

**2-pass (sem validation)** — usar apenas rough + format quando documentos são simples e bem estruturados.

**Parallel passes** — rodar rough extraction em paralelo para múltiplos documentos com `asyncio.gather`.

**Modelo diferente por pass** — Haiku para rough (rápido/barato), Sonnet para format (preciso), omitir validation para documentos simples.

**Cache de rough** — armazenar rough extraction no banco; se o documento for reprocessado, pular pass 1.
