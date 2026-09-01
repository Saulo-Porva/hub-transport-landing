# Chain-of-Thought

> Técnica que instrui o LLM a raciocinar passo a passo antes de responder — melhora dramaticamente tarefas de lógica, matemática e planejamento.

## O que é

CoT (Chain-of-Thought) instrui o modelo a produzir raciocínio intermediário antes da resposta final. O raciocínio visível reduz erros porque o modelo "se compromete" com cada passo, e erros intermediários são detectáveis. Essencial para tarefas onde a resposta correta depende de múltiplas inferências.

## Quando usar

- Problemas matemáticos ou lógicos com múltiplos passos
- Classificação com critérios complexos
- Extração onde campos dependem de inferência (ex: "calcule o total sem o desconto")
- Qualquer tarefa onde respostas diretas são inconsistentes

## Sintaxe / API

```python
# --- Zero-shot CoT: adicionar a frase mágica ---
prompt = """
Analyze the following invoice and determine if it is a duplicate of any previous invoice.

Invoice to check:
{invoice}

Previous invoices:
{previous_invoices}

Think step by step before answering. Consider: same vendor, similar amount, same date range, same invoice number patterns.

After your analysis, respond with JSON:
{"is_duplicate": true|false, "confidence": 0.0-1.0, "reason": "explanation"}
"""

# --- Few-shot CoT: mostrar o raciocínio nos exemplos ---
prompt = """
Classify the document type and extract key fields.

Example 1:
Document: "NOTA FISCAL ELETRÔNICA - Série A - Número 12345 - Emitente: ABC LTDA - CNPJ: 12.345.678/0001-90"
Thinking: The document contains "NOTA FISCAL ELETRÔNICA" which is the Brazilian tax invoice. It has a series (A) and number (12345). The emitter is ABC LTDA with CNPJ.
Result: {"type": "nfe", "number": "12345", "series": "A", "vendor_name": "ABC LTDA", "cnpj": "12.345.678/0001-90"}

Example 2:
Document: "RECIBO DE PAGAMENTO - Referente ao aluguel do mês de Janeiro/2024 - Locador: João Silva"
Thinking: This is a payment receipt ("RECIBO"), not an invoice. It references rent payment (aluguel) for January 2024. The landlord (Locador) is João Silva.
Result: {"type": "receipt", "reference": "aluguel Janeiro/2024", "payee": "João Silva"}

Now classify:
Document: {document}
Thinking:
"""

# --- Self-consistency: múltiplas amostras e voto majoritário ---
import anthropic
from collections import Counter

client = anthropic.Anthropic()

def classify_with_consistency(text: str, n_samples: int = 5) -> str:
    prompt = f"Classify the sentiment of: '{text}'\nAnswer with one word: positive, negative, or neutral."
    results = []
    for _ in range(n_samples):
        response = client.messages.create(
            model='claude-haiku-4-5-20251001',
            max_tokens=10,
            messages=[{'role': 'user', 'content': prompt}],
        )
        results.append(response.content[0].text.strip().lower())
    return Counter(results).most_common(1)[0][0]

# --- Step-back prompting: abstrair antes de resolver ---
STEPBACK_PROMPT = """
Before answering the specific question, first answer a more general version.

Question: {specific_question}

Step 1 - General principle: What general principle or rule applies here?
Step 2 - Apply to specific case: How does that principle apply to this specific question?
Step 3 - Final answer: {specific_question}
"""

# --- Tree-of-Thought (ToT): explorar múltiplos ramos ---
TOT_PROMPT = """
You are solving: {problem}

Generate 3 different approaches to solve this:

Approach A: ...
Approach B: ...
Approach C: ...

Evaluate each approach:
- Approach A: [pros/cons]
- Approach B: [pros/cons]
- Approach C: [pros/cons]

Best approach: [select and explain why]

Final solution using the best approach:
"""
```

## Armadilhas comuns

- CoT com `temperature > 0.3` em extração — raciocínio varia, resposta final fica inconsistente
- Pedir raciocínio E resposta estruturada no mesmo campo — separar em `<thinking>` e depois JSON
- Few-shot CoT com exemplos ruins — exemplos com raciocínio errado ensinam o modelo a errar
- Self-consistency com n > 10 — custo alto sem ganho proporcional; 5 geralmente é suficiente
