# pgvector Fundamentals

> PostgreSQL extension para armazenar e buscar embeddings vetoriais — base de qualquer sistema RAG no Supabase.

## O que é

`pgvector` adiciona o tipo `vector(n)` ao PostgreSQL e operadores de distância (`<=>` cosine, `<->` L2, `<#>` inner product). Permite busca por similaridade semântica direto no banco, sem necessidade de um vector database externo (Pinecone, Qdrant).

## Quando usar

- RAG (Retrieval-Augmented Generation): buscar chunks relevantes antes de chamar o LLM
- Busca semântica em catálogo de produtos, documentos, FAQs
- Deduplicação por similaridade (encontrar registros quase idênticos)

## Sintaxe / API

```sql
-- 1. Habilitar extensão (uma vez por projeto)
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Criar tabela com coluna de embedding
CREATE TABLE documents (
  id        bigserial PRIMARY KEY,
  content   text NOT NULL,
  metadata  jsonb DEFAULT '{}',
  embedding vector(1536)   -- 1536 para text-embedding-ada-002
                           -- 768 para text-embedding-3-small
                           -- 384 para all-MiniLM-L6-v2
);

-- 3. Índice IVFFLAT (criar APÓS inserir dados em bulk)
-- lists = sqrt(número de linhas) é um bom ponto de partida
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- HNSW (mais preciso, mais memória — PostgreSQL 15+)
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- 4. Busca por similaridade (cosine distance)
-- 1 - distância = similaridade (0 a 1)
SELECT
  id,
  content,
  metadata,
  1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
WHERE 1 - (embedding <=> '[0.1, 0.2, ...]'::vector) > 0.7  -- threshold
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;

-- 5. Função reutilizável match_documents
CREATE OR REPLACE FUNCTION match_documents(
  query_embedding vector(1536),
  match_threshold float DEFAULT 0.7,
  match_count     int   DEFAULT 5
)
RETURNS TABLE (
  id         bigint,
  content    text,
  metadata   jsonb,
  similarity float
)
LANGUAGE sql STABLE AS $$
  SELECT
    id, content, metadata,
    1 - (embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE 1 - (embedding <=> query_embedding) > match_threshold
  ORDER BY embedding <=> query_embedding
  LIMIT match_count;
$$;
```

```typescript
// Chamar via SDK
const { data } = await supabase.rpc('match_documents', {
  query_embedding: embeddingArray,   // number[]
  match_threshold: 0.7,
  match_count: 5,
})
```

## Gerar embeddings (OpenAI)

```typescript
import OpenAI from 'openai'
const openai = new OpenAI()

async function embed(text: string): Promise<number[]> {
  const { data } = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  })
  return data[0].embedding
}
```

## Armadilhas comuns

- Criar o índice ANTES de inserir dados — índice em tabela vazia não beneficia queries posteriores
- Dimensão errada no `vector(n)` — deve coincidir exatamente com o modelo de embedding
- Não ter threshold de similaridade — resultados irrelevantes com baixa similaridade vão aparecer
- Embeddings gerados por modelos diferentes não são comparáveis (espaços vetoriais distintos)
- Embedding de texto longo (>8K tokens) trunca silenciosamente — chunking é obrigatório
