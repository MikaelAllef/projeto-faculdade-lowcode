# Automação n8n · Orçamentos do site

Cada orçamento enviado no site dispara este fluxo:

```
Site (/api/quote) ──► Webhook ──► Pontuar lead ──► IA (Gemini) ──► Montar lead ──┬──► Planilha "Leads"
                     (chave       (nota por        (quente/morno/   (textos)      ├──► Aviso por e-mail (você)
                      secreta)     regras)          frio + resumo)                 └──► E-mail de confirmação (cliente)
```

- **Se a IA falhar** (sem chave, fora do ar), o fluxo continua com a nota calculada por regras.
- **Se um dos 3 destinos falhar**, os outros dois continuam funcionando.
- Se o n8n estiver fora do ar, o cliente recebe o protocolo do mesmo jeito; só a automação deixa de rodar.

## Arquivos

| Arquivo | O que é |
|---|---|
| `orvian-orcamentos.json` | O fluxo pronto para importar no n8n |
| `codigo/1-pontuar-lead.js` | Código do nó "Pontuar lead" (regras de nota + pedido à IA) |
| `codigo/2-montar-lead.js` | Código do nó "Montar lead" (planilha e e-mails) |
| `exemplo-orcamento.json` | Um orçamento real de exemplo, no formato que o site envia |
| `testar-webhook.sh` | Dispara o exemplo direto no n8n |

> Os arquivos `codigo/*.js` são a versão legível do que já está dentro do JSON. Se editar um deles, cole o conteúdo no nó correspondente.

## Configuração (≈ 20 min)

### 1. Importar
n8n → **Workflows → Import from File** → `orvian-orcamentos.json`.

### 2. Planilha
1. Suba `Base_Leads_atualizada.xlsx` no Google Drive e abra como Planilhas Google (ela já tem a aba **Leads** com os cabeçalhos certos).
2. Copie o ID da URL: `docs.google.com/spreadsheets/d/`**`ESTE_TRECHO`**`/edit`.
3. No nó **Salvar na planilha**, cole o ID no lugar de `COLE_AQUI_O_ID_DA_PLANILHA`.

### 3. Credenciais (4)

| Nó | Tipo de credencial | Como preencher |
|---|---|---|
| Orçamento do site | **Header Auth** | Name: `X-Orvian-Secret` · Value: uma senha longa à sua escolha (a mesma vai nas variáveis do Netlify) |
| Classificar com IA | **Query Auth** | Name: `key` · Value: sua chave do Google AI Studio (Gemini) |
| Salvar na planilha | **Google Sheets OAuth2** | Login com a conta Google da planilha |
| E-mail para o cliente e Avisar a ORVIAN por e-mail | **Gmail OAuth2** | Login com o e-mail que vai enviar (ex.: contato@…) |

No nó **Avisar a ORVIAN por e-mail**, troque o destinatário (`sendTo`) pelo e-mail onde você quer receber os leads.

### 4. Testar
1. Abra o nó **Orçamento do site** → **Listen for test event**.
2. No terminal: `./testar-webhook.sh <Test URL> <sua chave secreta>`
3. Confira: linha na planilha, e-mail de aviso para você e e-mail em `ana@clinicabella.com.br` (troque o e-mail em `exemplo-orcamento.json` pelo seu para receber o teste).

### 5. Ligar no site (Netlify)
1. **Ative** o fluxo (botão *Active*) e copie a **Production URL** do webhook (não a *Test URL*).
2. No Netlify: **Site configuration → Environment variables → Add a variable**:

   | Key | Value |
   |---|---|
   | `QUOTE_WEBHOOK_URL` | `https://SEU-N8N/webhook/orvian-orcamento` |
   | `QUOTE_WEBHOOK_SECRET` | a mesma senha do Header Auth |
   | `GEMINI_API_KEY` | (opcional) chave do Gemini para o chat da Orvia |

3. **Deploys → Trigger deploy → Deploy site** (variáveis novas só valem após um novo deploy).
4. Faça um orçamento de teste no site e confira a planilha e o seu e-mail. Em caso de dúvida: **Logs → Functions → quote** mostra `n8n: ok` ou o erro.

> Rodando localmente (`npm run dev`), as mesmas variáveis vão no arquivo `.env`.
>
> O n8n precisa estar acessível pela internet (n8n Cloud ou servidor próprio com HTTPS) — um n8n rodando só no seu computador não é alcançado pelo Netlify.

## Ajustes comuns

- **Mudar critérios da nota:** edite as regras no início do nó *Pontuar lead*.
- **Outro modelo de IA:** troque o nome do modelo na URL do nó *Classificar com IA*.
