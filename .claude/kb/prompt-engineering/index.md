# Prompt Engineering KB — Index

> Técnicas para maximizar consistência, precisão e formato de saída em LLMs.

---

## Domain Overview

| Área | Escopo | KB File |
|------|--------|---------|
| Chain-of-Thought | CoT, self-consistency, tree-of-thought, step-back | `concepts/chain-of-thought.md` |
| Few-Shot Prompting | Seleção de exemplos, ordem, negative examples | `concepts/few-shot-prompting.md` |
| Structured Extraction | XML tags, JSON mode, campos, confidence scores | `concepts/structured-extraction.md` |
| Output Formatting | Pydantic em prompt, markdown vs JSON, enums | `concepts/output-formatting.md` |
| System Prompts | Persona, role, constraints, output format | `concepts/system-prompts.md` |
| Document Extraction | Pipeline de extração de documentos reais | `patterns/document-extraction.md` |
| Multi-Pass Extraction | Extração rough → validação → formatação | `patterns/multi-pass-extraction.md` |
| Validation Prompts | Auto-critic, verifier, confidence threshold | `patterns/validation-prompts.md` |

---

## Quick Navigation

- **LLM retornando JSON inválido?** → `concepts/output-formatting.md` → JSON mode + Pydantic
- **Extrair campos de documento?** → `patterns/document-extraction.md`
- **Melhorar raciocínio em tarefas complexas?** → `concepts/chain-of-thought.md`
- **Saída inconsistente?** → `concepts/few-shot-prompting.md` + `patterns/validation-prompts.md`
- **Classificação com confiança?** → `concepts/structured-extraction.md`

---

## Modelo Mental: Técnica por Objetivo

| Objetivo | Técnica Principal | Técnica de Backup |
|----------|------------------|-------------------|
| Extrair campos de documento | Structured extraction + XML tags | Multi-pass |
| Classificar em categorias fixas | Few-shot + Literal enum | Chain-of-thought |
| Raciocínio complexo | CoT com "think step by step" | Self-consistency |
| Geração de texto consistente | System prompt + few-shot | Temperature 0 |
| JSON sempre válido | JSON mode + Pydantic schema no prompt | Retry com parse error |
| Alta confiança | Verifier pattern + threshold | Multi-pass com auto-critic |

---

## Princípios Gerais

1. **Especificidade bate generalidade** — "extraia o CNPJ no formato XX.XXX.XXX/XXXX-XX" > "extraia dados da empresa"
2. **Exemplos > instruções** — mostrar 2-3 exemplos do formato esperado supera qualquer instrução
3. **Output format no início** — declarar o formato antes do conteúdo, não depois
4. **Temperatura 0 para extração** — consistência é mais importante que criatividade
5. **Nunca confie cegamente** — sempre validar saída LLM com Pydantic antes de usar downstream
