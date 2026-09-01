# Structured Extraction

> Técnicas para extrair campos específicos de texto não estruturado com saída confiável e validável.

## O que é

Structured extraction instrui o LLM a identificar e extrair campos específicos de texto livre (documentos, emails, notas) e retorná-los em formato estruturado (JSON, XML). A chave é combinar instruções precisas de formato com validação downstream via Pydantic.

## Quando usar

- Extrair campos de invoices, contratos, formulários digitalizados
- Processar emails para extrair datas, valores, nomes
- Parsear logs ou mensagens semiestruturadas
- Normalizar dados de fontes diversas para um schema único

## Sintaxe / API

```python
from __future__ import annotations
from pydantic import BaseModel, Field
from typing import Literal
import anthropic
import json

# --- Schema Pydantic define o contrato ---
class InvoiceExtraction(BaseModel):
    vendor_name: str = Field(description='Name of the company that issued the invoice')
    vendor_cnpj: str | None = Field(default=None, description='CNPJ in format XX.XXX.XXX/XXXX-XX')
    invoice_number: str = Field(description='Invoice number or reference code')
    invoice_date: str = Field(description='Issue date in YYYY-MM-DD format')
    total_amount: float = Field(description='Total amount due, as a number without currency symbol')
    currency: Literal['BRL', 'USD', 'EUR'] = Field(default='BRL')
    line_items: list[LineItem] = Field(default_factory=list)
    confidence: float = Field(ge=0, le=1, description='Extraction confidence 0-1')

class LineItem(BaseModel):
    description: str
    quantity: float
    unit_price: float
    total: float

# --- XML tags (Claude) — mais confiável que JSON livre ---
EXTRACTION_PROMPT = """Extract all fields from the invoice below. If a field is not present, use null.

<invoice>
{invoice_text}
</invoice>

Schema to follow:
{schema}

Respond ONLY with valid JSON matching the schema. No explanation, no markdown.
"""

client = anthropic.Anthropic()

def extract_invoice(invoice_text: str) -> InvoiceExtraction:
    schema = InvoiceExtraction.model_json_schema()
    prompt = EXTRACTION_PROMPT.format(
        invoice_text=invoice_text,
        schema=json.dumps(schema, indent=2),
    )

    for attempt in range(3):
        response = client.messages.create(
            model='claude-sonnet-4-6',
            max_tokens=1024,
            messages=[{
                'role': 'user',
                'content': prompt,
            }],
            temperature=0,
        )
        raw = response.content[0].text.strip()

        # Limpar markdown fence se presente
        if raw.startswith('```'):
            raw = raw.split('\n', 1)[1].rsplit('```', 1)[0].strip()

        try:
            return InvoiceExtraction.model_validate_json(raw)
        except Exception as e:
            prompt += f'\n\nPrevious attempt failed validation:\n{e}\nPlease fix and try again.'

    raise ValueError(f'Extraction failed after 3 attempts. Last response: {raw}')

# --- Tool use (Claude) — garante estrutura sem parse de JSON ---
def extract_with_tool(invoice_text: str) -> InvoiceExtraction:
    tool_schema = {
        'name': 'extract_invoice',
        'description': 'Extract structured data from an invoice',
        'input_schema': InvoiceExtraction.model_json_schema(),
    }

    response = client.messages.create(
        model='claude-sonnet-4-6',
        max_tokens=2048,
        tools=[tool_schema],
        tool_choice={'type': 'tool', 'name': 'extract_invoice'},
        messages=[{
            'role': 'user',
            'content': f'Extract all fields from this invoice:\n\n{invoice_text}',
        }],
        temperature=0,
    )

    tool_use = next(b for b in response.content if b.type == 'tool_use')
    return InvoiceExtraction.model_validate(tool_use.input)

# --- Confidence score inline ---
CONFIDENCE_PROMPT = """
For each field you extract, rate your confidence (0.0-1.0).
Low confidence (< 0.7) means the field was inferred or unclear.

{invoice_text}

Return JSON:
{
  "fields": {
    "vendor_name": {"value": "...", "confidence": 0.95},
    "total_amount": {"value": 1234.56, "confidence": 0.80},
    "invoice_date": {"value": null, "confidence": 0.0}
  }
}
"""
```

## Armadilhas comuns

- `temperature > 0` em extração — introduce variabilidade; usar sempre `temperature=0`
- Não incluir o schema JSON no prompt — modelo inventa campos ou omite obrigatórios
- Confiar no JSON sem validar — `model_validate_json()` pode lançar; sempre tratar `ValidationError`
- Campos obrigatórios sem fallback — usar `| None` com `default=None` para campos que podem estar ausentes
- Documentos muito longos — truncar ou usar sliding window; Gemini suporta contextos maiores para PDFs
