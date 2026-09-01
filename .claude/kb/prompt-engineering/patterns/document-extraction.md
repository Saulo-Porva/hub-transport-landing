# Document Extraction

> Pipeline completo de extração de campos de documentos reais (invoices, contratos, formulários) com validação e fallback.

## Problema

Documentos chegam em formatos variados (PDF escaneado, NF-e XML, email, foto). O LLM precisa extrair campos específicos de forma confiável, tratar campos ausentes, e produzir saída validada que pode ser inserida no banco diretamente.

## Solução

```python
from __future__ import annotations
from pydantic import BaseModel, Field, model_validator
from typing import Literal
import anthropic
import logging

logger = logging.getLogger(__name__)
client = anthropic.Anthropic()

# --- Schema do domínio ---
class ExtractedInvoice(BaseModel):
    vendor_name: str
    vendor_document: str | None = None          # CNPJ ou CPF
    invoice_number: str
    invoice_date: str                            # YYYY-MM-DD
    due_date: str | None = None
    total_amount: float
    currency: Literal['BRL', 'USD', 'EUR'] = 'BRL'
    line_items: list[LineItem] = Field(default_factory=list)
    extraction_confidence: float = Field(ge=0, le=1)
    extraction_warnings: list[str] = Field(default_factory=list)

    @model_validator(mode='after')
    def validate_totals(self) -> 'ExtractedInvoice':
        if self.line_items:
            computed = sum(item.total for item in self.line_items)
            if abs(computed - self.total_amount) > 0.01:
                self.extraction_warnings.append(
                    f'Line items sum ({computed:.2f}) differs from total ({self.total_amount:.2f})'
                )
        return self

class LineItem(BaseModel):
    description: str
    quantity: float = 1.0
    unit_price: float
    total: float

# --- System prompt especializado ---
EXTRACTION_SYSTEM = """You are a specialist in extracting structured data from financial documents.

Rules:
- Return ONLY valid JSON. No markdown, no explanations.
- Dates: YYYY-MM-DD format
- Amounts: numbers only (no R$, $, EUR, commas as thousands separators)
- Brazilian format: 1.200,50 → 1200.50
- If a field is missing from the document: null
- confidence: your overall confidence 0.0-1.0
- warnings: list any anomalies or uncertainties found"""

EXTRACTION_PROMPT = """Extract all fields from this document.

<document>
{document_text}
</document>

JSON schema to follow:
{schema}

Return complete valid JSON:"""

def extract_invoice(document_text: str) -> ExtractedInvoice:
    import json
    schema = json.dumps(ExtractedInvoice.model_json_schema(), indent=2)
    prompt = EXTRACTION_PROMPT.format(document_text=document_text, schema=schema)
    last_error = None

    for attempt in range(3):
        try:
            response = client.messages.create(
                model='claude-sonnet-4-6',
                max_tokens=2048,
                system=EXTRACTION_SYSTEM,
                messages=[{'role': 'user', 'content': prompt}],
                temperature=0,
            )
            raw = response.content[0].text.strip()

            # Limpar markdown fences
            if '```' in raw:
                raw = raw.split('```')[1]
                if raw.startswith('json'):
                    raw = raw[4:]

            result = ExtractedInvoice.model_validate_json(raw)

            if result.extraction_confidence < 0.7:
                logger.warning('Low confidence extraction',
                               extra={'confidence': result.extraction_confidence,
                                      'warnings': result.extraction_warnings})

            return result

        except Exception as e:
            last_error = e
            logger.warning(f'Attempt {attempt + 1} failed: {e}')
            prompt += f'\n\nAttempt {attempt + 1} failed: {e}\nPlease fix the JSON and try again.'

    logger.error('All extraction attempts failed', extra={'error': str(last_error)})
    raise ValueError(f'Extraction failed after 3 attempts: {last_error}')

# --- Multimodal: PDF/imagem via Gemini ---
def extract_from_image(image_bytes: bytes, mime_type: str = 'image/jpeg') -> ExtractedInvoice:
    import google.generativeai as genai
    import base64, json

    model = genai.GenerativeModel('gemini-1.5-flash')
    schema = json.dumps(ExtractedInvoice.model_json_schema(), indent=2)

    response = model.generate_content([
        f'Extract invoice fields as JSON matching this schema:\n{schema}\nReturn ONLY valid JSON.',
        {'mime_type': mime_type, 'data': base64.b64encode(image_bytes).decode()},
    ], generation_config={'temperature': 0, 'response_mime_type': 'application/json'})

    return ExtractedInvoice.model_validate_json(response.text)
```

## Variações

**NF-e XML** — parsear o XML diretamente com `xml.etree.ElementTree` antes de passar ao LLM; enviar só os campos relevantes para reduzir tokens.

**Batch processing** — paralelizar com `asyncio.gather` ou `ThreadPoolExecutor`; rate limiting com `asyncio.Semaphore`.

**LangFuse observability** — wrappear `extract_invoice` com span do LangFuse para rastrear latência, tokens e confidence por documento.
