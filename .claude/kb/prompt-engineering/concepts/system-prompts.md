# System Prompts

> A mensagem de sistema define a identidade, capacidades, restrições e formato de saída do modelo para toda a conversação.

## O que é

O system prompt é a instrução mais influente: define quem o modelo "é", o que pode e não pode fazer, e como deve formatar respostas. É avaliado antes de qualquer mensagem do usuário e permanece ativo por toda a sessão. Para Claude, tem mais peso que mensagens do usuário.

## Quando usar

- Toda aplicação que usa LLM em produção deve ter um system prompt
- Definir persona especialista aumenta qualidade da saída
- Restrições de segurança (não revelar sistema, não sair do escopo) pertencem aqui
- Formato de saída padrão (sempre JSON, sempre markdown, idioma fixo) vai no system

## Sintaxe / API

```python
# --- Template de system prompt para extração ---
EXTRACTION_SYSTEM = """You are an invoice data extraction specialist.

## Your role
Extract structured data from invoice documents with high accuracy.
When uncertain about a field, use null rather than guessing.

## Rules
- Always respond with valid JSON. No markdown, no explanations outside the JSON.
- Dates must be in YYYY-MM-DD format
- Amounts must be numbers (no currency symbols, no thousands separators)
- If a field is absent from the document, return null for that field
- Set confidence < 0.7 for any field you are not certain about

## Output schema
{
  "vendor_name": string | null,
  "total_amount": number | null,
  "invoice_date": "YYYY-MM-DD" | null,
  "confidence": 0.0 to 1.0
}"""

# --- Template de system prompt para classificação ---
CLASSIFICATION_SYSTEM = """You are a document classification expert for a logistics company.

## Categories
- INVOICE: Supplier invoices requesting payment
- RECEIPT: Proof of payment already made
- CONTRACT: Service or supply agreements
- DELIVERY_NOTE: Shipping and delivery documents
- OTHER: Anything that doesn't fit the above

## Rules
- Classify based on the document content, not the filename
- When in doubt between two categories, choose the primary purpose of the document
- Always include a confidence score (0.0-1.0)
- Respond only in JSON: {"category": "...", "confidence": 0.0, "signals": ["reason1", "reason2"]}"""

# --- Template de system prompt para assistente conversacional ---
ASSISTANT_SYSTEM = """You are a helpful assistant for PR Trasporti, a Brazilian freight transport company.

## You can help with
- Answering questions about active deliveries and routes
- Calculating estimated arrival times
- Explaining invoice discrepancies
- Driver and vehicle assignment queries

## You cannot
- Create, modify, or delete records (read-only access)
- Provide financial forecasts or commitments
- Discuss competitor pricing or internal costs

## Communication style
- Portuguese (Brazil) only
- Professional but friendly
- Short responses (under 150 words unless detail is explicitly requested)
- If unsure, say "Não tenho essa informação" — never guess

## Context
Today is {date}. Current user role: {role}."""

# --- Injetar contexto dinâmico no system ---
import anthropic
from datetime import date

def build_system(role: str) -> str:
    return ASSISTANT_SYSTEM.format(
        date=date.today().isoformat(),
        role=role,
    )

client = anthropic.Anthropic()

def chat(user_message: str, role: str = 'driver') -> str:
    response = client.messages.create(
        model='claude-haiku-4-5-20251001',
        max_tokens=512,
        system=build_system(role),
        messages=[{'role': 'user', 'content': user_message}],
    )
    return response.content[0].text

# --- Prefill (Claude): forçar início da resposta ---
# Útil para garantir que a resposta começa com {
response = client.messages.create(
    model='claude-sonnet-4-6',
    max_tokens=512,
    system='Extract fields as JSON.',
    messages=[
        {'role': 'user', 'content': f'Extract from: {document}'},
        {'role': 'assistant', 'content': '{'},  # prefill — Claude continua a partir daqui
    ],
)
raw = '{' + response.content[0].text  # recompor JSON completo
```

## Armadilhas comuns

- System prompt muito longo (> 2K tokens) — dilui atenção; priorizar as 5-10 regras mais críticas
- Instruções contraditórias entre system e user — system deve ganhar, mas alguns modelos seguem user
- "Never do X" sem alternativa — especificar o que fazer em vez do que não fazer é mais eficaz
- Não testar o system prompt com inputs adversariais — usuários podem tentar contornar as restrições
- Contexto dinâmico (data, user) hardcoded — usar template strings para injetar em runtime
