# Output Formatting

> Controlar o formato exato da saída do LLM — JSON, markdown, enums, Pydantic schema no prompt.

## O que é

Output formatting instrui o modelo sobre a estrutura exata da resposta. Envolve: especificar formato (JSON/markdown/texto), incluir o schema ou exemplo no prompt, restringir a valores válidos (enums), e usar JSON mode quando disponível. Sem isso, o modelo inventa formatos inconsistentes.

## Quando usar

- Sempre que o output será processado por código (parsear, validar, inserir no banco)
- Quando múltiplas chamadas precisam de saída idêntica em estrutura
- Quando os valores devem ser de um conjunto fixo (classificação)

## Sintaxe / API

```python
# --- Incluir schema Pydantic como JSON Schema no prompt ---
import json
from pydantic import BaseModel, Field
from typing import Literal

class ClassificationResult(BaseModel):
    category: Literal['invoice', 'receipt', 'contract', 'other']
    confidence: float = Field(ge=0, le=1)
    reason: str = Field(max_length=200)

schema = ClassificationResult.model_json_schema()

PROMPT = f"""
Classify the document type.

Respond ONLY with valid JSON matching this schema:
{json.dumps(schema, indent=2)}

Rules:
- confidence must be between 0 and 1
- reason must be under 200 characters
- Do not include markdown, code fences, or explanations

Document: {{document_text}}
"""

# --- JSON mode (OpenAI / compatível) ---
import openai
client = openai.OpenAI()

response = client.chat.completions.create(
    model='gpt-4o-mini',
    response_format={'type': 'json_object'},
    messages=[
        {'role': 'system', 'content': 'You are a document classifier. Always respond with valid JSON.'},
        {'role': 'user', 'content': PROMPT.format(document_text='...')},
    ],
    temperature=0,
)
raw = response.choices[0].message.content
result = ClassificationResult.model_validate_json(raw)

# --- Structured output (OpenAI gpt-4o-2024-08-06+) ---
from openai import OpenAI
from pydantic import BaseModel

class Step(BaseModel):
    explanation: str
    output: str

class MathSolution(BaseModel):
    steps: list[Step]
    final_answer: str

response = client.beta.chat.completions.parse(
    model='gpt-4o-2024-08-06',
    messages=[{'role': 'user', 'content': 'Solve: 2x + 5 = 11'}],
    response_format=MathSolution,
)
solution = response.choices[0].message.parsed  # MathSolution instance

# --- Claude tool use para saída estruturada ---
import anthropic

claude = anthropic.Anthropic()

def classify_document(text: str) -> ClassificationResult:
    response = claude.messages.create(
        model='claude-sonnet-4-6',
        max_tokens=512,
        tools=[{
            'name': 'classify',
            'description': 'Classify a document and return structured result',
            'input_schema': ClassificationResult.model_json_schema(),
        }],
        tool_choice={'type': 'tool', 'name': 'classify'},
        messages=[{'role': 'user', 'content': f'Classify this document:\n\n{text}'}],
        temperature=0,
    )
    tool_use = next(b for b in response.content if b.type == 'tool_use')
    return ClassificationResult.model_validate(tool_use.input)

# --- Formatação de markdown (quando output é para humanos) ---
MARKDOWN_PROMPT = """
Generate a structured report.

Format:
## Summary
[2-3 sentences]

## Key Findings
- [finding 1]
- [finding 2]

## Recommendation
[action to take]

Data: {data}
"""

# --- Enum restrito: listar valores explicitamente ---
ENUM_PROMPT = """
Classify the priority. You MUST respond with exactly one of these values:
- urgent
- high
- medium  
- low

Do not include any other text.

Task: {task}
Priority:"""

# --- Strip de artefatos comuns ---
def clean_json_response(raw: str) -> str:
    raw = raw.strip()
    if raw.startswith('```json'):
        raw = raw[7:]
    elif raw.startswith('```'):
        raw = raw[3:]
    if raw.endswith('```'):
        raw = raw[:-3]
    return raw.strip()
```

## Armadilhas comuns

- Dizer "respond in JSON" sem mostrar o schema — modelo inventa campos
- Não usar `temperature=0` em extração estruturada — formato varia entre chamadas
- Pedir JSON e texto de explicação na mesma resposta — difícil de parsear; usar campo `reason` dentro do JSON
- JSON mode sem system prompt pedindo JSON — alguns modelos precisam de instrução explícita no system
- Enum sem listar os valores no prompt — modelo inventa variações ("High" vs "high" vs "HIGH")
