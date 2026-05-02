# 🔴 Log de Erros e Soluções — YouTube AI Minerador

> 22 versões do workflow foram necessárias para estabilizar. Este documento registra todos os erros para não repetir.

---

## Erro 1 — `Unexpected end of JSON input`
**Versões:** v1–v5  
**Causa:** Workflow retornava body vazio (`responseMode: responseNode` sem chegar ao RespondToWebhook)  
**Solução:** Usar `responseMode: lastNode` OU garantir que todos os nós intermediários não quebrem

---

## Erro 2 — `d.forEach is not a function`
**Versões:** v6–v8  
**Causa:** n8n retornou objeto `{}` mas frontend chamou `.forEach()` direto  
**Solução:**
```javascript
if (Array.isArray(data)) {
  videos = data;
} else {
  videos = [data]; // fallback para objeto único
}
```

---

## Erro 3 — `Sem Título` / `Sem Análise`
**Versões:** v9–v15  
**Causa 1:** Nomes de campos diferentes entre workflow e frontend  
**Causa 2:** Chaves de API expiradas/quota zerada  
**Diagnóstico:** Criar "teste de vida" que retorna dados fixos sem chamar APIs externas  
**Solução:** Alinhar campos; usar fallback hardcoded para isolar se é problema de API ou de conexão

---

## Erro 4 — `{"message": "Error in workflow"}` (HTTP 500)
**Versões:** v16–v21  
**Causas identificadas:**
- Code node com `return [{ json: [array] }]` — n8n não aceita array como valor de `json`
- Regex com backtick dentro de template literal do SDK
- `responseMode: responseNode` sem RespondToWebhook alcançável

**Solução:**
```javascript
// ❌ Causa o erro 500
return [{ json: [item1, item2] }];

// ✅ Correto
return [{ json: { videos: [item1, item2] } }];
```

---

## Erro 5 — Resposta vazia (sem erro, sem dados)
**Versões:** v17–v18  
**Causa:** `responseMode: responseNode` + Code node quebrando silenciosamente antes do RespondToWebhook  
**Solução:** Usar `responseMode: lastNode` — responde automaticamente com o último nó executado

---

## Erro 6 — Apenas 1 vídeo em vez de 3
**Versão:** v20  
**Causa:** `responseMode: lastNode` retorna apenas o `.json` do **primeiro item**. Code node retornava 3 itens separados mas n8n entregava só o primeiro.  
**Solução:**
```javascript
// ❌ 3 itens separados — lastNode entrega só o primeiro
return resultados.map(r => ({ json: r }));

// ✅ 1 item com array dentro — lastNode entrega o objeto completo
return [{ json: { videos: resultados } }];
```

---

## Erro 7 — `A 'json' property isn't an object [item 0]`
**Versão:** v21 (HTTP 500)  
**Regra descoberta:** O campo `json` em itens n8n DEVE ser sempre um **objeto** `{}`  
**Solução:** Encapsular arrays em objeto nomeado: `{ videos: [...] }`

---

## Erro 8 — `404 webhook not registered`
**Causa:** Workflow atualizado mas não republicado (draft ≠ active)  
**Solução:** Sempre `publish_workflow()` após `update_workflow()`

---

## Erro 9 — Gemini retorna JSON com markdown, `JSON.parse` falha
**Causa:** Gemini envolve JSON em \`\`\`json ... \`\`\`  
**Solução:**
```javascript
const inicio = texto.indexOf('[');
const fim = texto.lastIndexOf(']');
const parsed = JSON.parse(texto.substring(inicio, fim + 1));
```

---

## Erro 10 — Perda de dados entre nós Code separados
**Versão:** v17  
**Causa:** Após HTTP Request (Gemini), dados do nó Code anterior eram inacessíveis  
**Solução:** Consolidar tudo em **um único Code node** usando `$helpers.httpRequest()` para Gemini

---

## ✅ Arquitetura Final Estável (v22)

```
Webhook (POST, lastNode)
  → YouTube (HTTP GET, onError: continue)
  → Code node (runOnceForAllItems):
      - $input.first().json → YouTube response
      - $helpers.httpRequest() → Gemini
      - return [{ json: { videos: [...] } }]
```

**Regras de ouro:**
1. `json` sempre objeto `{}`
2. `lastNode` mais simples que `responseNode`
3. Um único Code node > múltiplos nós com dependências
4. `$helpers.httpRequest()` para HTTP dentro de Code
5. `indexOf/lastIndexOf` para extrair JSON de texto Gemini
6. Sempre `publish_workflow()` após `update_workflow()`
