# 🎯 YouTube AI Minerador

> Ferramenta de pesquisa de oportunidades Low Ticket usando YouTube API + Gemini AI + n8n

![Status](https://img.shields.io/badge/status-funcionando-brightgreen) ![Versão](https://img.shields.io/badge/vers%C3%A3o-v22-blue)

## 🚀 O que faz

Digite um **nicho** (ex: diabetes, emagrecimento, marmita) e a ferramenta:
1. Busca os **3 vídeos mais vistos** no YouTube sobre o nicho
2. Envia para o **Gemini AI** que sugere um produto Low Ticket (R$27–R$197)
3. Exibe no **Dashboard** com título, canal, produto sugerido, preço e dor principal

## 🏗️ Arquitetura

```
[index.html local]
       │  POST { nicho: "diabetes" }
       ▼
[n8n Webhook] /webhook/youtube-minerador
       │
       ▼
[YouTube Data API v3] — busca 3 vídeos
       │
       ▼
[Code Node] — processa YouTube + chama Gemini via $helpers.httpRequest
       │
       ▼
[Resposta] { videos: [{titulo, canal, oportunidade_produto, preco_sugerido, dor_principal, url_video}] }
```

## 📁 Estrutura do Projeto

```
minerador-youtube/
├── index.html              # Dashboard web (abrir localmente)
├── workflow_v22.json       # Backup do workflow n8n (versão estável)
├── README.md               # Este arquivo
├── MANUAL_AGENTE.md        # Manual completo para novo agente IA
└── ERROS_E_SOLUCOES.md     # Log de todos os erros e soluções
```

## ⚙️ Configuração

### Pré-requisitos
- Conta no [Google Cloud Console](https://console.cloud.google.com)
- n8n rodando na VPS (https://n8n.agbotia.com.br)
- Chaves de API ativas

### Chaves de API necessárias
| API | Onde gerar | Status |
|-----|-----------|--------|
| YouTube Data API v3 | Google Cloud Console | ✅ Funcionando |
| Gemini 1.5 Flash | aistudio.google.com | ⚠️ Verificar quota |

### Como usar
1. Abra `index.html` no navegador (duplo-clique no arquivo)
2. Digite um nicho no campo de busca
3. Clique **MINERAR AGORA**
4. Aguarde ~10 segundos (YouTube + Gemini)
5. Veja os 3 cards com oportunidades

## 🔧 Workflow n8n

- **ID:** `Nu0APJiMrWXX2789`
- **Endpoint:** `POST https://n8n.agbotia.com.br/webhook/youtube-minerador`
- **Nós:** Webhook → YouTube HTTP GET → Code node (tudo em 1)

## ⚠️ Regras Críticas

1. **`json` nunca pode ser array** — use `{ json: { videos: [...] } }`
2. **`responseMode: lastNode`** — retorna apenas `.json` do primeiro item
3. **`$helpers.httpRequest()`** para HTTP dentro de Code nodes
4. Sempre `publish_workflow()` após `update_workflow()`

Veja `ERROS_E_SOLUCOES.md` e `MANUAL_AGENTE.md` para detalhes completos.
