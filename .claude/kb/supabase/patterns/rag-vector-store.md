# RAG Vector Store

> Pipeline completo de RAG usando pgvector: ingestão de documentos, geração de embeddings, busca semântica e geração com LLM.

## Problema

LLMs não conhecem dados privados da empresa. RAG (Retrieval-Augmented Generation) resolve isso: busca os chunks mais relevantes para a pergunta e os envia como contexto para o LLM responder.

## Solução

```sql
-- 1. Schema
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE public.documents (
  id         bigserial PRIMARY KEY,
  title      text,
  content    text NOT NULL,         -- chunk de texto
  metadata   jsonb DEFAULT '{}',   -- source, page, section, etc.
  embedding  vector(1536),
  created_at timestamptz DEFAULT now()
);

ALTER TABLE public.documents ENABLE ROW LEVEL SECURITY;
CREATE POLICY "authenticated_read" ON public.documents
  FOR SELECT TO authenticated USING (true);

-- Índice após ingestão em bulk
CREATE INDEX ON public.documents
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Função de busca
CREATE OR REPLACE FUNCTION match_documents(
  query_embedding vector(1536),
  match_threshold float DEFAULT 0.75,
  match_count     int   DEFAULT 5
)
RETURNS TABLE (id bigint, content text, metadata jsonb, similarity float)
LANGUAGE sql STABLE AS $$
  SELECT id, content, metadata,
         1 - (embedding <=> query_embedding) AS similarity
  FROM public.documents
  WHERE 1 - (embedding <=> query_embedding) > match_threshold
  ORDER BY embedding <=> query_embedding
  LIMIT match_count;
$$;
```

```typescript
import OpenAI from 'openai'
import { createClient } from '@supabase/supabase-js'

const openai = new OpenAI()
const supabase = createClient(url, serviceRoleKey)  // admin para ingestão

// --- Ingestão ---
async function ingestDocument(title: string, text: string) {
  // Chunking simples — em produção usar LangChain RecursiveCharacterTextSplitter
  const chunks = chunkText(text, { size: 1000, overlap: 100 })

  for (const chunk of chunks) {
    const { data } = await openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: chunk,
    })
    const embedding = data[0].embedding

    await supabase.from('documents').insert({
      title,
      content: chunk,
      metadata: { source: title },
      embedding,
    })
  }
}

// --- Busca + Geração ---
async function ask(question: string): Promise<string> {
  // 1. Embed a pergunta
  const { data } = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: question,
  })
  const queryEmbedding = data[0].embedding

  // 2. Buscar chunks relevantes
  const { data: chunks } = await supabase.rpc('match_documents', {
    query_embedding: queryEmbedding,
    match_threshold: 0.75,
    match_count: 5,
  })

  if (!chunks?.length) return 'Não encontrei informações relevantes.'

  // 3. Montar contexto
  const context = chunks.map((c: any) => c.content).join('\n\n---\n\n')

  // 4. Gerar resposta
  const completion = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      {
        role: 'system',
        content: `Responda apenas com base no contexto fornecido. Se não souber, diga que não tem informação.\n\nContexto:\n${context}`,
      },
      { role: 'user', content: question },
    ],
  })

  return completion.choices[0].message.content ?? ''
}

// Chunking simples
function chunkText(text: string, { size, overlap }: { size: number; overlap: number }): string[] {
  const chunks: string[] = []
  let start = 0
  while (start < text.length) {
    chunks.push(text.slice(start, start + size))
    start += size - overlap
  }
  return chunks
}
```

## Variações

**Com RLS** — documents filtrados por org_id; embeddings gerados server-side via Edge Function.

**Streaming** — usar `openai.chat.completions.create({ stream: true })` + ReadableStream na Edge Function.

**Reranking** — após busca vetorial, reordenar chunks com cross-encoder (Cohere Rerank API) antes de passar ao LLM.

**Hybrid search** — combinar busca vetorial (`<=>`) com busca full-text (`to_tsvector`) e RRF (Reciprocal Rank Fusion) para melhor recall.
