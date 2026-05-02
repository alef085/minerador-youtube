# 📖 Manual para Novo Agente — YouTube AI Minerador

> Leia este documento COMPLETO antes de fazer qualquer alteração no projeto.

## 🎯 Contexto do Projeto

Este projeto foi construído ao longo de uma sessão intensa com **22 versões do workflow n8n** antes de estabilizar. O objetivo é minerar oportunidades de produtos Low Ticket a partir de vídeos do YouTube usando IA.

## 🔐 Credenciais e Acessos

| Recurso | URL / Valor |
|---------|-------------|
| n8n VPS | https://n8n.agbotia.com.br |
| Workflow ID | `Nu0APJiMrWXX2789` |
| Webhook endpoint | `POST /webhook/youtube-minerador` |
| YouTube API Key | `AIzaSyDjd__9uO3qZwc-_hGyTlbYQKSMJTzs5fo` |
| Gemini API Key | `AIzaSyDrrsBJB-gujJAPw85EpvcklPgWGfTMXE0` |
| GitHub repo | https://github.com/alef085/minerador-youtube |

## 🏗️ Arquitetura Atual (v22 — ESTÁVEL)

### Fluxo de dados
```
Usuário digita nicho no index.html
  → fetch POST para https://n8n.agbotia.com.br/webhook/youtube-minerador
  → n8n Webhook node (responseMode: lastNode)
  → HTTP Request node (YouTube Data API v3, maxResults=3, order=viewCount)
  → Code node (runOnceForAllItems):
      1. Extrai items[0..2] do YouTube
      2. Chama Gemini via $helpers.httpRequest()
      3. Parseia JSON do Gemini (indexOf/lastIndexOf, sem regex com backtick)
      4. Fallback: retorna dados do YouTube sem análise IA
      5. return [{ json: { videos: [...] } }]  ← CRÍTICO
  → n8n responde com { videos: [{...}, {...}, {...}] }
  → index.html lê data.videos e renderiza 3 cards
```

### Formato de resposta esperado pelo frontend
```json
{
  "videos": [
    {
      "titulo": "Título do vídeo",
      "canal": "Nome do canal",
      "visualizacoes": 50000,
      "oportunidade_produto": "Produto sugerido pela IA",
      "preco_sugerido": "R$47",
      "dor_principal": "Problema que o produto resolve",
      "url_video": "https://www.youtube.com/watch?v=VIDEO_ID"
    }
  ]
}
```

## ⚡ Regras Absolutas — NUNCA Viole Estas

### 1. O campo `json` do n8n NUNCA pode ser um array
```javascript
// ❌ ERRADO — causa erro 500 "A 'json' property isn't an object"
return [{ json: [item1, item2, item3] }];

// ✅ CORRETO — empacota o array dentro de um objeto
return [{ json: { videos: [item1, item2, item3] } }];
```

### 2. `responseMode: lastNode` retorna apenas o PRIMEIRO item
Com `lastNode`, o n8n devolve automaticamente o `.json` do primeiro item da saída do último nó. Por isso o Code node retorna apenas 1 item contendo o array inteiro dentro de um objeto.

### 3. Não use `responseMode: responseNode` sem garantir que o fluxo chega ao fim
Se qualquer nó intermediário quebrar silenciosamente, o webhook retorna **resposta vazia** (body vazio — sem erro, sem dado).

### 4. Use `$helpers.httpRequest()` para HTTP dentro de Code nodes
```javascript
// ✅ Correto
const resp = await $helpers.httpRequest({
  method: 'POST',
  url: 'https://generativelanguage.googleapis.com/...',
  headers: { 'Content-Type': 'application/json' },
  body: { contents: [{ parts: [{ text: prompt }] }] }
});
```

### 5. Nunca use regex com backtick dentro de template literals do SDK
```javascript
// ❌ ERRADO no SDK — causa erro de sintaxe
const cleaned = texto.replace(/```json/g, '');

// ✅ CORRETO — use indexOf/lastIndexOf
const inicio = texto.indexOf('[');
const fim = texto.lastIndexOf(']');
const jsonStr = texto.substring(inicio, fim + 1);
```

### 6. Sempre publicar após atualizar
```
1. validate_workflow(code)        → verifica sintaxe
2. update_workflow(id, code)      → salva como draft
3. publish_workflow(id)           → ativa para produção ← OBRIGATÓRIO
```

### 7. Leia o SKILL n8n-codando-javascript antes de escrever Code nodes
Path: `C:\Users\Paulo Aleixo\.gemini\antigravity\skills\n8n-code-javascript\SKILL.md`

## 🚨 Diagnóstico Rápido de Problemas

| Sintoma | Causa provável | Solução |
|---------|---------------|--------|
| `Error 500: Error in workflow` | Code node quebrando | Ver logs no n8n: Executions tab |
| Resposta vazia (body vazio) | responseMode: responseNode sem chegar ao fim | Trocar para `lastNode` |
| `Sem Título` / `Sem Análise` | Campos não mapeados ou APIs falhando | Verificar se frontend lê `data.videos` |
| Produto genérico ("Produto digital sobre...") | Gemini falhando silenciosamente | Renovar chave em aistudio.google.com |
| `d.forEach is not a function` | n8n retornou objeto, frontend esperava array | Frontend: adicionar `videos = [data]` como fallback |
| `Unexpected end of JSON input` | Resposta vazia do servidor | Verificar se workflow está publicado |
| `404 webhook not registered` | Workflow não está ativo | `publish_workflow(workflowId)` |
| Apenas 1 vídeo aparece | `return resultados.map(r => ({json:r}))` com lastNode | Usar `return [{ json: { videos: resultados } }]` |

## 🧪 Como Testar

### Teste de vida (sem APIs externas)
```
GET https://n8n.agbotia.com.br/webhook/minerador-v15
```
Se aparecer "CONEXÃO ESTABELECIDA" → n8n está funcionando.

### Teste completo via curl
```bash
curl -X POST https://n8n.agbotia.com.br/webhook/youtube-minerador \
  -H 'Content-Type: application/json' \
  -d '{"nicho": "diabetes"}'
```
Espera: `{"videos":[{"titulo":"...","canal":"..."}]}`

## 📝 Próximos Passos Sugeridos

1. **Renovar chave do Gemini** — [aistudio.google.com](https://aistudio.google.com)
2. **Buscar visualizações reais** — Adicionar chamada à API `videos?part=statistics`
3. **Hospedar o frontend** — Deploy no Vercel ou Netlify
4. **Histórico de pesquisas** — Integrar com n8n Data Tables
5. **Mais nichos por busca** — Permitir busca de múltiplos nichos simultâneos
