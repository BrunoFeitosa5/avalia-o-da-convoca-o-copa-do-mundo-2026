# Shared Ratings — Design Spec
**Data:** 2026-06-09  
**Projeto:** Avaliação da Lista de Convocação 2026  
**Status:** Aprovado pelo usuário

---

## Objetivo

Substituir o sistema de avaliação local (localStorage isolado por dispositivo) por um sistema compartilhado onde todos os visitantes veem a média e contagem de votos da comunidade, mantendo a capacidade de cada pessoa votar e editar seu próprio voto.

---

## Banco de Dados — Supabase

### Tabela: `ratings`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `id` | uuid (PK) | Gerado automaticamente |
| `player_id` | text | ID do jogador (ex: `'vini'`, `'raphinha'`) |
| `voter_id` | text | UUID anônimo do visitante |
| `stars` | integer (1–5) | Nota dada |
| `updated_at` | timestamptz | Atualizado a cada UPSERT |

**Restrição:** UNIQUE em `(player_id, voter_id)` — um voto por jogador por visitante.

### Row Level Security (RLS)

- **SELECT:** público — qualquer pessoa lê todos os votos
- **INSERT/UPDATE:** permitido apenas quando `voter_id = auth.uid()` via anon key — cada visitante só escreve na própria linha

---

## Frontend — Alterações no `index.html`

### 1. SDK Supabase
Carregado via CDN no `<head>`. Sem npm, sem build step.

### 2. Identidade anônima
- Chave localStorage: `copa2026-voter-id`
- Na primeira visita: gera UUID v4 e persiste
- Nas visitas seguintes: reutiliza o mesmo UUID

### 3. Carregamento inicial (duas queries paralelas)
- **Query A:** votos do próprio visitante → pré-preenche as estrelas
- **Query B:** `AVG(stars)` e `COUNT(*)` agrupados por `player_id` → exibe média da comunidade

### 4. Interação — clique nas estrelas
- UPSERT no Supabase: `INSERT ... ON CONFLICT (player_id, voter_id) DO UPDATE SET stars = ..., updated_at = now()`
- Atualiza a média local otimisticamente sem aguardar resposta

### 5. Exibição por card
- Estrelas interativas = nota do próprio visitante (comportamento atual mantido)
- Nova linha abaixo do clube: `Comunidade: 4.2★ · 127 votos`

### 6. Painel de aprovação + resumo final
- Passa a usar médias da comunidade em vez de localStorage
- Contagens (aprovados/regulares/rejeitados) baseadas na média da comunidade por jogador

### 7. Migração de dados locais
- Na primeira visita com o novo código, exporta votos do localStorage e faz UPSERT em lote no Supabase
- Garante que quem já avaliou não perde os votos anteriores

### 8. Fallback offline
- Se Supabase indisponível, usa localStorage como fallback silencioso
- Tentativa de sync quando a conexão retornar (na próxima abertura da página)

---

## Fluxo de dados

```
Visitante abre a página
  → gera/recupera voter_id do localStorage
  → busca votos próprios no Supabase      → preenche estrelas
  → busca médias da comunidade            → exibe "Comunidade: X★ · N votos"
  → migra votos locais pendentes (se houver)

Visitante clica numa estrela
  → UPSERT no Supabase
  → atualiza display local imediatamente
  → recalcula painel de aprovação
```

---

## Segurança

- A `anon key` do Supabase fica exposta no JS do cliente — isso é seguro porque é pública por design
- RLS impede que um visitante sobrescreva votos de outro
- Sem autenticação real: o `voter_id` é uma proteção por obscuridade — não impede votos em massa deliberados, mas é adequado para o contexto de um site pessoal/fan

---

## Fora do escopo

- Autenticação real (login/cadastro)
- Moderação de votos
- Proteção anti-bot robusta
- Histórico de mudanças de voto
