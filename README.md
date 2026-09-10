# Chatbot Telegram — Temperatura no Brasil (N8N + Docker)

## Descrição do projeto

Chatbot no Telegram, orquestrado pelo **N8N** em Docker, que informa a **temperatura atual** de qualquer cidade do Brasil.

O usuário envia uma mensagem com **cidade e estado** (ex.: `Campinas, SP`). O workflow:

1. Recebe a mensagem via **Telegram Trigger**
2. Valida o formato `Cidade, UF` (ou nome completo do estado)
3. Consulta a API gratuita da **OpenWeather** (geocoding + clima atual)
4. Responde com uma mensagem curta, clara e amigável em português

### Estrutura do repositório

| Arquivo / pasta | Função |
|---|---|
| `docker-compose.yml` | Sobe o N8N e o túnel HTTPS (Cloudflare) |
| `.env.example` | Modelo das variáveis de ambiente |
| `workflows/telegram-temperatura-brasil.json` | Workflow pronto para importar |
| `scripts/up.sh` | Sobe o stack e configura o `WEBHOOK_URL` HTTPS |

### Pré-requisitos

- Docker e Docker Compose
- Conta no Telegram
- Conta gratuita na [OpenWeather](https://openweathermap.org/api)

---

## Variáveis esperadas

| Variável | Onde configurar | Descrição |
|---|---|---|
| `OPENWEATHER_API_KEY` | Arquivo `.env` (injetada no container N8N) | Chave da API OpenWeather (plano free) |
| `TELEGRAM_BOT_TOKEN` | Credencial **Telegram API** no N8N | Token do bot gerado pelo [@BotFather](https://t.me/BotFather) |
| `WEBHOOK_URL` | Arquivo `.env` (HTTPS obrigatório) | URL pública do N8N para o Telegram registrar o webhook |

### Como obter as chaves

**`TELEGRAM_BOT_TOKEN`**

1. Abra o [@BotFather](https://t.me/BotFather) no Telegram
2. Envie `/newbot` e siga as instruções
3. Copie o token (formato: `123456789:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw`)

**`OPENWEATHER_API_KEY`**

1. Cadastre-se em [openweathermap.org](https://home.openweathermap.org/users/sign_up)
2. Acesse [API keys](https://home.openweathermap.org/api_keys)
3. Copie a chave e aguarde alguns minutos até ela ser ativada

### Configurar o `.env`

```bash
cp .env.example .env
```

Edite o `.env` e preencha pelo menos:

```env
OPENWEATHER_API_KEY=sua_chave_openweather
```

> O `TELEGRAM_BOT_TOKEN` **não** vai no `.env` para autenticação do bot: ele é cadastrado como credencial dentro do N8N (passo abaixo).  
> O `WEBHOOK_URL` é preenchido automaticamente pelo script `./scripts/up.sh` (túnel HTTPS). O Telegram **não aceita** `http://localhost`.

---

## Como executar o N8N (Docker)

Com túnel HTTPS (recomendado em desenvolvimento local):

```bash
chmod +x scripts/up.sh
./scripts/up.sh
```

Isso sobe o N8N + túnel Cloudflare e grava no `.env` uma URL do tipo:

```text
https://xxxxx.trycloudflare.com/
```

A interface do N8N fica em: [http://localhost:5678](http://localhost:5678)

Na primeira execução, crie o usuário owner (admin) do N8N.

> Se alterar o `.env` (por exemplo a chave OpenWeather) ou o túnel cair após um restart, rode novamente `./scripts/up.sh` e **reative** o workflow no N8N.

---

## Como inserir as credenciais no N8N

### 1. Telegram (`TELEGRAM_BOT_TOKEN`)

1. No N8N, abra o menu lateral
2. Vá em **Credentials** → **Add credential**
3. Busque e selecione **Telegram API**
4. No campo **Access Token**, cole o valor de `TELEGRAM_BOT_TOKEN` (token do BotFather)
5. Salve com um nome fácil de reconhecer, por exemplo: `Telegram Bot`

### 2. OpenWeather (`OPENWEATHER_API_KEY`)

A chave OpenWeather **não** é cadastrada como credencial na UI do N8N neste projeto.

Ela fica no arquivo `.env` como `OPENWEATHER_API_KEY` e o workflow lê via `$env.OPENWEATHER_API_KEY` nos nós de consulta à API.

Depois de editar o `.env`:

```bash
./scripts/up.sh
```

Confirme que a variável entrou no container:

```bash
docker exec n8n-temperatura-telegram printenv OPENWEATHER_API_KEY
```

---

## Passos para importar o workflow no N8N

1. Abra o N8N em [http://localhost:5678](http://localhost:5678)
2. No menu, vá em **Workflows**
3. Clique em **⋯** (ou no botão de criar) → **Import from File** / **Import from URL**
4. Selecione o arquivo local:

   ```text
   workflows/telegram-temperatura-brasil.json
   ```

   (o mesmo arquivo também está montado no container em `/workflows/telegram-temperatura-brasil.json`)
5. Após importar, abra o workflow **Telegram Temperatura Brasil**
6. Em **todos** os nós do tipo **Telegram Trigger** e **Telegram**, selecione a credencial `Telegram Bot` criada anteriormente
7. Salve o workflow
8. Ative o workflow com o toggle **Active** no canto superior direito

Se aparecer erro de webhook HTTPS, o `WEBHOOK_URL` está inválido ou o túnel caiu. Rode `./scripts/up.sh`, depois desative e ative o workflow novamente.

---

## Como executar / testar o chatbot

1. No Telegram, abra o bot que você criou com o BotFather
2. Envie:

```text
/start
```

Resposta esperada: mensagem de boas-vindas explicando o formato `Cidade, UF`.

3. Envie uma cidade de teste:

```text
Campinas, SP
```

Resposta esperada (exemplo):

```text
Em Campinas (SP) está 23°C agora, com céu limpo.
```

### Outros exemplos válidos

| Mensagem enviada | O que esperar |
|---|---|
| `São Paulo, SP` | Temperatura atual de São Paulo (SP) |
| `Curitiba, PR` | Temperatura atual de Curitiba (PR) |
| `Belo Horizonte, Minas Gerais` | Aceita UF ou nome completo do estado |
| `Campinas` (sem vírgula/estado) | Mensagem pedindo o formato correto |
| `CidadeInexistente, SP` | Mensagem de cidade não encontrada |

### Formato da mensagem

```text
Cidade, UF
```

- Use vírgula entre cidade e estado
- UF com 2 letras (`SP`, `RJ`, `MG`…) **ou** nome completo (`São Paulo`, `Minas Gerais`)

---

## Comandos úteis

```bash
# Subir / atualizar com túnel HTTPS
./scripts/up.sh

# Logs do N8N
docker compose logs -f n8n

# Logs do túnel
docker compose logs -f tunnel

# Parar
docker compose down

# Parar e apagar dados persistentes do N8N
docker compose down -v
```

---

## Solução de problemas

| Problema | O que verificar |
|---|---|
| `An HTTPS URL must be provided for webhook` | `WEBHOOK_URL` em HTTP/localhost — rode `./scripts/up.sh` e reative o workflow |
| Bot não responde | Workflow **Active**? Credencial Telegram correta? Túnel no ar? |
| Cidade não encontrada | Formato `Cidade, UF`; geocoding usa `cidade,BR` + filtro pelo estado |
| Erro / chave OpenWeather | `OPENWEATHER_API_KEY` no `.env` e container recriado (`./scripts/up.sh`) |
| Webhook parou após restart | URL do túnel mudou — rode `./scripts/up.sh` e reative o workflow |

---

## Escopo

- Temperatura **atual** apenas (sem previsão)
- Cidades do **Brasil**
- Respostas em **pt-BR**
