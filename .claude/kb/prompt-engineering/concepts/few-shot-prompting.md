# Few-Shot Prompting

> Incluir exemplos de input/output no prompt para ensinar o modelo o formato e comportamento desejado — mais eficaz que instruções.

## O que é

Few-shot prompting fornece 2-5 exemplos de entrada/saída antes da tarefa real. O modelo aprende o padrão por indução, sem fine-tuning. É mais robusto que instruções verbais porque demonstra o comportamento em vez de descrevê-lo.

## Quando usar

- Saída com formato específico difícil de descrever (ex: estrutura de dados personalizada)
- Tarefa onde "o tom certo" é mais fácil de mostrar que explicar
- Classificação com fronteiras ambíguas entre categorias
- Extração onde o campo pode estar em diferentes posições/formatos no documento

## Sintaxe / API

```python
# --- Formato básico: Input → Output ---
FEW_SHOT_TEMPLATE = """
Classify the expense category from the description.
Categories: TRAVEL, MEALS, SOFTWARE, HARDWARE, OTHER

{examples}

Description: {input}
Category:"""

def build_few_shot_prompt(examples: list[tuple[str, str]], input_text: str) -> str:
    example_block = '\n\n'.join(
        f'Description: {inp}\nCategory: {out}'
        for inp, out in examples
    )
    return FEW_SHOT_TEMPLATE.format(examples=example_block, input_text=input_text)

# Exemplos bem escolhidos (cobrem casos limite)
EXPENSE_EXAMPLES = [
    ('Uber from airport to hotel', 'TRAVEL'),
    ('GitHub Copilot annual subscription', 'SOFTWARE'),
    ('Team lunch at Outback', 'MEALS'),
    ('MacBook Pro 14"', 'HARDWARE'),
    ('Office birthday cake', 'OTHER'),
]

# --- Negative examples: mostrar o que NÃO fazer ---
NEGATIVE_EXAMPLE_PROMPT = """
Extract only the vendor name. Do NOT include suffixes like Ltd, S.A., LTDA, Inc.

CORRECT:
Input: "ACME Corp. Ltd - Invoice #123"
Output: ACME Corp.

Input: "Tecnologia S.A. - NF 456"
Output: Tecnologia

WRONG (do not do this):
Input: "XYZ Solutions Inc - Invoice #789"
Output: XYZ Solutions Inc.  ← includes "Inc." which is wrong

Now extract:
Input: {text}
Output:"""

# --- Seleção dinâmica de exemplos (k-NN por similaridade) ---
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

class FewShotSelector:
    def __init__(self, examples: list[tuple[str, str]], embeddings: list[list[float]]) -> None:
        self._examples = examples
        self._embeddings = np.array(embeddings)

    def select(self, query_embedding: list[float], k: int = 3) -> list[tuple[str, str]]:
        query = np.array(query_embedding).reshape(1, -1)
        similarities = cosine_similarity(query, self._embeddings)[0]
        top_k = np.argsort(similarities)[-k:][::-1]
        return [self._examples[i] for i in top_k]

# --- Ordem dos exemplos importa ---
# Regra: exemplo mais relevante por último (mais próximo do input real)
# O modelo tende a seguir mais o padrão do último exemplo

def ordered_few_shot(examples: list[tuple[str, str]], most_relevant_last: bool = True) -> str:
    if most_relevant_last:
        # Ordena do menos relevante para o mais relevante
        pass  # reordenar conforme similaridade
    return '\n\n'.join(f'Input: {inp}\nOutput: {out}' for inp, out in examples)

# --- Few-shot com CoT (exemplos com raciocínio) ---
COT_EXAMPLES = """
Invoice: "NF-e 001.2024 - Fornecedor: ABC LTDA - CNPJ 12.345.678/0001-90 - Total: R$ 1.200,50"
Thinking: I see "NF-e" indicating a Brazilian electronic invoice. The total is R$ 1.200,50 which in Brazilian format means 1200.50 (period as thousands separator, comma as decimal).
JSON: {"type": "nfe", "vendor": "ABC LTDA", "total": 1200.50}

Invoice: "RECEIPT - Payment for consulting services - $500 USD - March 2024"
Thinking: This is a receipt (not an invoice). Amount is $500 in USD. Date is March 2024 but no specific day given.
JSON: {"type": "receipt", "vendor": null, "total": 500.0}
"""
```

## Armadilhas comuns

- Exemplos inconsistentes entre si — o modelo aprende a média e fica confuso
- Mais de 6 exemplos raramente ajuda — aumenta token count sem ganho; 3-4 é o ponto ótimo
- Exemplos não cobrindo casos extremos — o modelo interpola entre exemplos; inclua bordas
- Formato diferente entre exemplos e input real — alinhar exatamente (espaços, pontuação, case)
- Exemplos desatualizados — exemplos antigos ensinam comportamento antigo; manter sincronizados com requisitos
