# Validation Prompts

> Usar um segundo LLM call para verificar e criticar a saída do primeiro — detecta erros antes de persistir dados.

## Problema

LLMs falham silenciosamente: extraem um valor plausível mas incorreto, classificam com confiança errada, ou produzem JSON sintaticamente válido mas semanticamente inconsistente. O primeiro call não sabe que errou.

## Solução

```python
from __future__ import annotations
from pydantic import BaseModel, Field
from typing import Any
import anthropic
import json

client = anthropic.Anthropic()

# --- Auto-critic: o mesmo modelo revisa sua própria saída ---
CRITIC_PROMPT = """
You extracted data from a document. Now review your own extraction for errors.

Original document:
<document>
{document}
</document>

Your extraction:
<extraction>
{extraction}
</extraction>

Check:
1. Is every value actually present in the document? (no hallucination)
2. Are amounts in the correct numeric format?
3. Is the date format YYYY-MM-DD?
4. Does the vendor name match exactly what's in the document?

Respond with JSON:
{{
  "is_correct": true|false,
  "errors": ["specific error 1", "specific error 2"],
  "confidence": 0.0-1.0
}}
"""

class CriticResult(BaseModel):
    is_correct: bool
    errors: list[str] = Field(default_factory=list)
    confidence: float = Field(ge=0, le=1)

def auto_critic(document: str, extraction: dict[str, Any]) -> CriticResult:
    response = client.messages.create(
        model='claude-haiku-4-5-20251001',
        max_tokens=512,
        messages=[{'role': 'user', 'content': CRITIC_PROMPT.format(
            document=document,
            extraction=json.dumps(extraction, indent=2, ensure_ascii=False),
        )}],
        temperature=0,
    )
    return CriticResult.model_validate_json(response.content[0].text)

# --- Verifier pattern: modelo separado, prompt diferente ---
# Usar modelo diferente evita os mesmos vieses do extrator

VERIFIER_SYSTEM = """You are a strict data quality auditor. Your job is to find errors, not approve work."""

VERIFIER_PROMPT = """
An automated system extracted the following data from a financial document.
Verify each field against the source document.

Source document:
{document}

Extracted data:
{extraction}

For each field, verify: present in document? correct format? correct value?

Respond with JSON:
{{
  "field_checks": {{
    "vendor_name": {{"correct": true, "note": ""}},
    "total_amount": {{"correct": false, "note": "Document shows R$ 1.200,50 = 1200.50 but extraction has 120050"}}
  }},
  "overall_confidence": 0.0-1.0,
  "must_fix": ["list of fields that are definitely wrong"]
}}
"""

class FieldCheck(BaseModel):
    correct: bool
    note: str = ''

class VerifierResult(BaseModel):
    field_checks: dict[str, FieldCheck]
    overall_confidence: float = Field(ge=0, le=1)
    must_fix: list[str] = Field(default_factory=list)

def verify_extraction(document: str, extraction: dict[str, Any]) -> VerifierResult:
    response = client.messages.create(
        model='claude-sonnet-4-6',
        max_tokens=1024,
        system=VERIFIER_SYSTEM,
        messages=[{'role': 'user', 'content': VERIFIER_PROMPT.format(
            document=document,
            extraction=json.dumps(extraction, indent=2, ensure_ascii=False),
        )}],
        temperature=0,
    )
    return VerifierResult.model_validate_json(response.content[0].text)

# --- Pipeline com threshold e retry ---
from dataclasses import dataclass

CONFIDENCE_THRESHOLD = 0.85

@dataclass
class ExtractionPipeline:
    extractor_model: str = 'claude-sonnet-4-6'
    verifier_model: str = 'claude-haiku-4-5-20251001'
    threshold: float = CONFIDENCE_THRESHOLD
    max_retries: int = 3

    def run(self, document: str, schema: type[BaseModel]) -> BaseModel | None:
        import logging
        logger = logging.getLogger(__name__)

        extract_prompt = build_extract_prompt(document, schema)

        for attempt in range(self.max_retries):
            # Extract
            raw = call_model(self.extractor_model, extract_prompt)
            try:
                extracted = schema.model_validate_json(raw)
            except Exception as e:
                logger.warning(f'Parse failed attempt {attempt + 1}: {e}')
                extract_prompt += f'\n\nPrevious JSON was invalid: {e}\nFix and retry.'
                continue

            # Verify
            critic = auto_critic(document, extracted.model_dump())

            if critic.confidence >= self.threshold and critic.is_correct:
                logger.info('Extraction passed verification',
                            extra={'confidence': critic.confidence, 'attempt': attempt + 1})
                return extracted

            if not critic.errors:
                logger.info('No errors found but low confidence',
                            extra={'confidence': critic.confidence})
                return extracted  # aceitar com baixa confiança se sem erros

            logger.warning('Verification failed, retrying',
                           extra={'errors': critic.errors, 'attempt': attempt + 1})
            extract_prompt += f'\n\nVerification found errors:\n' + '\n'.join(critic.errors) + '\nFix these and extract again.'

        logger.error('Pipeline failed after max retries')
        return None

def build_extract_prompt(document: str, schema: type[BaseModel]) -> str:
    return f'Extract fields as JSON matching this schema:\n{schema.model_json_schema()}\n\nDocument:\n{document}'

def call_model(model: str, prompt: str) -> str:
    response = client.messages.create(
        model=model, max_tokens=1024,
        messages=[{'role': 'user', 'content': prompt}],
        temperature=0,
    )
    return response.content[0].text
```

## Variações

**Threshold adaptativo** — abaixar threshold (0.70) para documentos de baixo risco; manter 0.90 para faturas fiscais.

**Human-in-the-loop** — quando confidence < threshold após max_retries, enfileirar para revisão manual em vez de descartar.

**LangFuse scoring** — registrar `critic.confidence` como score no trace LangFuse para monitorar degradação ao longo do tempo.
