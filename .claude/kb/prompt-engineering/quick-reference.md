# Prompt Engineering Quick Reference

## Técnica por Objetivo

| Objetivo | Técnica | Snippet-chave |
|----------|---------|--------------|
| Extrair campos | Structured + XML tags | `<field>valor</field>` |
| JSON confiável | JSON mode + schema no prompt | `"response_format": {"type": "json_object"}` |
| Raciocínio complexo | CoT | `"Think step by step before answering."` |
| Consistência em batch | Few-shot (3 exemplos) | Input → Output no prompt |
| Classificação com confiança | Enum + confidence float | `{"category": "...", "confidence": 0.92}` |
| Validar saída LLM | Verifier | Segundo LLM call com checklist |
| Alta precisão | Multi-pass | Rough → Validate → Format |

## System Prompt Template

```
You are a {role} that {primary_capability}.

## Rules
- {constraint_1}
- {constraint_2}
- If uncertain, output confidence < 0.7 instead of guessing

## Output Format
Always respond with valid JSON matching this schema:
{schema_or_example}
```

## JSON Mode (OpenAI / Claude)

```python
# OpenAI
response = client.chat.completions.create(
    model='gpt-4o-mini',
    response_format={'type': 'json_object'},  # força JSON válido
    messages=[{'role': 'user', 'content': prompt}],
    temperature=0,
)

# Claude (structured output via tool use)
response = client.messages.create(
    model='claude-sonnet-4-6',
    tools=[{'name': 'extract', 'input_schema': schema}],
    tool_choice={'type': 'tool', 'name': 'extract'},
    messages=[{'role': 'user', 'content': prompt}],
)
result = response.content[0].input  # dict com os campos
```

## XML Tags (Claude — mais confiável que JSON livre)

```
Extract the following fields from the invoice below.

<invoice>
{invoice_text}
</invoice>

Respond in this exact format:
<extraction>
<vendor_name>...</vendor_name>
<total_amount>...</total_amount>
<invoice_date>YYYY-MM-DD</invoice_date>
<invoice_number>...</invoice_number>
</extraction>
```

## Few-Shot Template

```
Extract the vendor name from the invoice.

Example 1:
Invoice: "ACME Corp. - Invoice #123 - Total: $500"
Vendor: ACME Corp.

Example 2:
Invoice: "From: TechSupplies Ltd - Ref: INV-2024-001 - Amount Due: R$ 1.200,00"
Vendor: TechSupplies Ltd

Now extract from:
Invoice: {text}
Vendor:
```

## Confidence Score Pattern

```
Classify the sentiment. If you are not confident (< 0.85), say so.

Respond with JSON:
{"sentiment": "positive|negative|neutral", "confidence": 0.0-1.0, "reason": "one sentence"}

Text: {text}
```

## Retry com Parse Error como Feedback

```python
for attempt in range(3):
    raw = call_llm(prompt)
    try:
        return MyModel.model_validate_json(raw)
    except ValidationError as e:
        prompt += f'\n\nYour previous response had errors:\n{e}\nTry again with valid JSON.'
raise ValueError(f'Failed after 3 attempts. Last response: {raw}')
```
