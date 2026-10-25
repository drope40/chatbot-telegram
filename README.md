# 🌤️ Bot do Clima — Chatbot no Telegram com N8N + OpenWeather

Chatbot do Telegram, construído no [N8N](https://n8n.io), que informa a **temperatura atual de qualquer cidade do Brasil**. O usuário envia o nome da cidade (de preferência no formato `Cidade,UF,BR`), o workflow consulta a API gratuita da [OpenWeather](https://openweathermap.org/current) e responde com uma mensagem curta e amigável:

```
🌤️ A temperatura em Belo Horizonte é de 25°C.
```

Se a cidade não for encontrada (ou a resposta da API for inválida), o bot responde:

```
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

---

## 📁 Arquivos do repositório

| Arquivo | Descrição |
| --- | --- |
| `workflow-chatbot-telegram.json` | Workflow exportado do N8N (sem credenciais ou tokens embutidos) |
| `README.md` | Esta documentação |
| `docker-compose.yml` | (Opcional) Sobe o N8N localmente já com as variáveis de ambiente necessárias |

---

## 🧩 Como o workflow funciona

```
Telegram Trigger → Capturar Mensagem (queue) → Normalizar Cidade → OpenWeather - Clima Atual → Resposta Válida? (IF)
                                                                                                  ├─ true  → Formatar Mensagem → Enviar Temperatura
                                                                                                  └─ false → Enviar Erro
```

1. **Telegram Trigger** — inicia o workflow sempre que o bot recebe uma mensagem de texto.
2. **Capturar Mensagem (queue)** (Set node) — guarda o texto recebido na variável `queue` (e o `chatId` da conversa).
3. **Normalizar Cidade** (Code node) — ajusta o texto às regras da OpenWeather: remove acentos, converte para minúsculas, remove espaços extras e espaços ao redor das vírgulas, remove símbolos e garante o país `br` no final.
   Exemplos: `São Paulo, SP, BR` → `sao paulo,sp,br` · `Florianópolis` → `florianopolis,br`. Comandos como `/start` são tratados como entrada inválida.
4. **OpenWeather - Clima Atual** (HTTP Request) — `GET https://api.openweathermap.org/data/2.5/weather` com **Send Query Parameters** ativado:

   | Parâmetro | Valor |
   | --- | --- |
   | `q` | `{{ $json.queue }}` (cidade formatada) |
   | `queue` | `{{ $json.queue }}` (mesma cidade formatada) |
   | `units` | `metric` (graus Celsius) |
   | `lang` | `pt_br` (português brasileiro) |
   | `appid` | `{{ $env.OPENWEATHER_API_KEY }}` |

   > ℹ️ **Por que `q` e `queue`?** O enunciado chama o parâmetro de `queue`, mas o parâmetro oficial da OpenWeather para busca por nome de cidade é `q`. O workflow envia os dois com o mesmo valor: `q` é o que a API realmente usa, e `queue` atende literalmente ao enunciado (a API ignora parâmetros desconhecidos).

   O nó está configurado com *Full Response* e *Never Error*, para que respostas como `404 city not found` sigam para o IF em vez de interromper o workflow.
5. **Resposta Válida?** (IF node) — segue pelo caminho de sucesso somente se `statusCode` for `200` **e** o JSON contiver `main.temp` (numérico) e `name`.
6. **Formatar Mensagem** (Code node) — extrai a temperatura, arredonda com `Math.round` e monta a string final.
7. **Enviar Temperatura / Enviar Erro** (Telegram Send Message) — responde ao usuário no mesmo chat.

---

## ✅ Pré-requisitos

- **N8N** (testado na linha 1.x; funciona também na 2.x com a observação sobre `$env` abaixo) — via Docker, npm ou N8N Cloud.
- **Bot do Telegram** e seu token, criados com o [@BotFather](https://t.me/BotFather) (`/newbot`).
- **Chave de API da OpenWeather** (plano gratuito): crie em <https://home.openweathermap.org/api_keys>.
  > Chaves novas podem levar de alguns minutos a algumas horas para serem ativadas. Enquanto isso, a API responde `401`.
- **URL pública com HTTPS para o N8N** — o Telegram Trigger usa *webhook*, e o Telegram só envia mensagens para endereços públicos com HTTPS. Em ambiente local, use `n8n start --tunnel`, ngrok ou Cloudflare Tunnel e informe essa URL em `WEBHOOK_URL`.

---

## 🔐 Variáveis esperadas

| Variável | Onde é usada | Exemplo (fictício) |
| --- | --- | --- |
| `OPENWEATHER_API_KEY` | Variável de ambiente do N8N, lida no nó HTTP Request via `{{ $env.OPENWEATHER_API_KEY }}` | `0123456789abcdef0123456789abcdef` |
| `TELEGRAM_BOT_TOKEN` | Token do bot, cadastrado como credencial **Telegram API** no N8N | `123456789:AAAbbbCCCdddEEEfff` |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | Deve ser `false` para permitir `$env` nas expressões (no N8N 2.x o padrão é bloquear) | `false` |
| `WEBHOOK_URL` | URL pública (HTTPS) do seu N8N, usada para registrar o webhook do Telegram | `https://meu-n8n.exemplo.com/` |

> ⚠️ **Nunca** coloque valores reais dessas variáveis neste repositório. O JSON do workflow não contém tokens nem chaves: a chave da OpenWeather vem do ambiente e o token do Telegram fica armazenado (criptografado) nas credenciais do próprio N8N.

---

## 🚀 Passo a passo

### 1. Subir o N8N com as variáveis de ambiente

**Opção A — Docker Compose (recomendado):** crie um arquivo `.env` ao lado do `docker-compose.yml` (o `.env` **não** deve ir para o GitHub):

```env
OPENWEATHER_API_KEY=sua_chave_openweather
TELEGRAM_BOT_TOKEN=seu_token_do_telegram
WEBHOOK_URL=https://sua-url-publica/
```

```bash
docker compose up -d
```

**Opção B — Docker sem Compose:**

```bash
docker run -it --rm --name n8n -p 5678:5678 \
  -e OPENWEATHER_API_KEY=sua_chave_openweather \
  -e N8N_BLOCK_ENV_ACCESS_IN_NODE=false \
  -e WEBHOOK_URL=https://sua-url-publica/ \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

**Opção C — npm com túnel (teste rápido local):**

```bash
export OPENWEATHER_API_KEY=sua_chave_openweather
export N8N_BLOCK_ENV_ACCESS_IN_NODE=false
npx n8n start --tunnel
```

> Se `$env` não estiver disponível (por exemplo, no N8N Cloud), substitua o valor do parâmetro `appid` pela chave diretamente **na sua instância** ou use uma credencial do tipo *Query Auth* — mas não exporte o workflow com a chave dentro.

### 2. Importar o workflow

1. Acesse o N8N (ex.: `http://localhost:5678`).
2. Clique em **Create workflow** (ou **+ → Workflow**).
3. No menu `⋯` (canto superior direito), escolha **Import from File…** e selecione `workflow-chatbot-telegram.json`.
   (Alternativa: copie o conteúdo do JSON e cole com `Ctrl+V` no editor vazio.)
4. Salve o workflow.

### 3. Cadastrar a credencial do Telegram

1. Abra o nó **Telegram Trigger** → em *Credential to connect with*, clique em **Create new credential**.
2. Em **Access Token**, cole o valor do seu `TELEGRAM_BOT_TOKEN`.
   - Em instâncias self-hosted, você também pode usar a expressão `{{ $env.TELEGRAM_BOT_TOKEN }}` no campo, desde que a variável esteja definida no ambiente do N8N.
3. Salve a credencial e selecione-a também nos nós **Enviar Temperatura** e **Enviar Erro**.

### 4. Conferir a chave da OpenWeather

Abra o nó **OpenWeather - Clima Atual** e confirme que o parâmetro `appid` está como `{{ $env.OPENWEATHER_API_KEY }}`. Nenhuma credencial precisa ser criada para esse nó.

### 5. Executar

- **Para testar:** clique em **Execute workflow** (o N8N fica aguardando uma mensagem) e envie uma cidade para o seu bot no Telegram.
- **Para deixar rodando:** ative o toggle **Active / Publish** no topo do workflow. A partir daí o bot responde automaticamente.

> Um token de bot só pode ter **um webhook ativo por vez**. Se o mesmo bot estiver ligado em outra instância do N8N, a última que registrar o webhook passa a receber as mensagens.

---

## 🧪 Testes sugeridos

| Mensagem enviada | Resposta esperada (temperatura varia) |
| --- | --- |
| `São Paulo,SP,BR` | 🌤️ A temperatura em São Paulo é de 22°C. |
| `Belo Horizonte,MG,BR` | 🌤️ A temperatura em Belo Horizonte é de 25°C. |
| `rio de janeiro, rj` | 🌤️ A temperatura em Rio de Janeiro é de 28°C. |
| `Florianópolis` | 🌤️ A temperatura em Florianópolis é de 19°C. |
| `Cidadeinexistente,XX,BR` | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |
| `/start` | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |

---

## 🛠️ Solução de problemas

| Sintoma | Causa provável | Solução |
| --- | --- | --- |
| Toda cidade retorna erro | `OPENWEATHER_API_KEY` ausente, chave ainda não ativada (401) ou `$env` bloqueado | Confira a variável, aguarde a ativação da chave e defina `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` |
| Bot não responde nada | Webhook do Telegram sem URL pública HTTPS | Configure `WEBHOOK_URL` ou use `--tunnel` |
| Erro *"access to env vars denied"* | N8N bloqueando `$env` | `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` e reinicie o N8N |
| Erro de conflito de webhook | Mesmo bot ativo em outra instância | Desative o workflow na outra instância |

---

## 🤖 Gemini (opcional)

Esta versão **não** utiliza o nó Google Gemini. A mensagem final é gerada de forma determinística pelo Code node **Formatar Mensagem**, que já devolve `{"message": "...", "ok": true}` — exatamente o formato que serviria de *fallback* caso um nó Gemini fosse adicionado entre **Formatar Mensagem** e **Enviar Temperatura**. Assim, o workflow funciona sem custos e sem credenciais além do Telegram e da OpenWeather.

---

## 🧱 Tecnologias

- [N8N](https://n8n.io/) — automação de workflows
- [Telegram Bot API](https://core.telegram.org/bots/api) — canal de mensagens
- [OpenWeather — Current Weather Data](https://openweathermap.org/current) — dados de clima
