# Chatbot de Clima no Telegram com N8N

Chatbot do Telegram, construído no N8N, que informa a temperatura atual de qualquer cidade do Brasil. O usuário envia o nome da cidade no formato `Cidade,UF,BR`, o workflow consulta a API gratuita da OpenWeather e responde com uma mensagem curta e amigável.

```
Usuário:  São Paulo,SP,BR
Bot:      🌤️ A temperatura em São Paulo é de 23°C.

Usuário:  Cidadeinexistente,XX,BR
Bot:      ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

## Arquivos do repositório

| Arquivo | Descrição |
|---|---|
| `workflow-chatbot-telegram.json` | Workflow exportado do N8N (sem credenciais ou tokens) |
| `README.md` | Esta documentação |
| `docker-compose.yml` | Opcional: sobe o N8N localmente com as variáveis de ambiente |
| `.env.example` | Modelo das variáveis de ambiente (copie para `.env` e preencha) |

## Como o workflow funciona

```
Telegram Trigger
  -> Capturar e Formatar Cidade (Set: queue, chatId)
  -> Consultar OpenWeather (HTTP Request)
  -> Resposta Válida? (IF)
       true  -> Formatar Mensagem (Fallback) (Code)
                -> Gemini: Melhorar Mensagem (opcional, desativado por padrão)
                -> Selecionar Mensagem Final (Code)
                -> Enviar Temperatura (Telegram)
       false -> Enviar Mensagem de Erro (Telegram)
```

1. **Telegram Trigger**: recebe as mensagens de texto enviadas ao bot (`updates: message`).
2. **Capturar e Formatar Cidade** (Set node): grava o texto na variável `queue`, já normalizado conforme as regras da OpenWeather:
   - remove espaços nas pontas e ao redor das vírgulas e colapsa espaços duplicados;
   - remove a acentuação (`São Paulo` vira `sao paulo`);
   - converte para minúsculas.

   Exemplo: `"  São Paulo , SP , BR "` vira `"sao paulo,sp,br"`. Também guarda o `chatId` para a resposta.
3. **Consultar OpenWeather** (HTTP Request): `GET https://api.openweathermap.org/data/2.5/weather` com **Send Query Parameters** ativado:

   | Parâmetro | Valor |
   |---|---|
   | `q` | `{{ $json.queue }}` (nome do parâmetro exigido pela OpenWeather) |
   | `queue` | `{{ $json.queue }}` (a mensagem formatada, conforme o enunciado) |
   | `units` | `metric` (graus Celsius) |
   | `lang` | `pt_br` (português brasileiro) |
   | `appid` | `{{ $env.OPENWEATHER_API_KEY }}` |

   A OpenWeather só reconhece a cidade pelo parâmetro `q`; por isso o valor de `queue` é enviado nos dois parâmetros (a API ignora parâmetros desconhecidos). O nó está com **Full Response** e **Never Error** ativados, para que respostas 404/401 sigam o fluxo de validação em vez de interromper a execução.
4. **Resposta Válida?** (IF): segue pelo caminho de sucesso apenas se `statusCode = 200`, se `body.main.temp` existe e se `body.name` não está vazio. Qualquer outro caso (cidade não encontrada, chave inválida, falha de rede) vai para o nó de erro.
5. **Formatar Mensagem (Fallback)** (Code): extrai `main.temp`, arredonda com `Math.round` e monta a mensagem `🌤️ A temperatura em {cidade} é de {temp}°C.` O nome exibido vem do campo `name` da API, já com acentuação correta.
6. **Gemini: Melhorar Mensagem (opcional)**: reescreve a mensagem em tom natural (ver seção do Gemini abaixo). Vem **desativado** no JSON.
7. **Selecionar Mensagem Final** (Code): usa a mensagem do Gemini se ela vier no formato `{"message":"...","ok":true}`; caso contrário, usa a mensagem do fallback.
8. **Enviar Temperatura / Enviar Mensagem de Erro** (Telegram Send Message): respondem ao mesmo chat que enviou a mensagem.

## Pré-requisitos

- N8N 1.x (testado na versão 1.123) via Docker, npm ou N8N Cloud.
- Bot do Telegram criado no **@BotFather** (comando `/newbot`), com o token em mãos.
- Conta na OpenWeather com uma API Key ativa (https://home.openweathermap.org/api_keys). Chaves novas podem levar até 2 horas para serem ativadas.
- URL pública com HTTPS para o N8N. O Telegram Trigger usa webhook, então o Telegram precisa alcançar o seu N8N. Em ambiente local, use um túnel como o ngrok (instruções abaixo).

## Variáveis de ambiente esperadas

| Variável | Uso |
|---|---|
| `OPENWEATHER_API_KEY` | Chave da OpenWeather, lida no nó HTTP Request por `{{ $env.OPENWEATHER_API_KEY }}` |
| `TELEGRAM_BOT_TOKEN` | Token do bot gerado pelo BotFather, usado na credencial **Telegram API** do N8N |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | Deve ser `false` para permitir que os nós leiam `$env` |
| `WEBHOOK_URL` | URL pública HTTPS do N8N (ex.: a URL gerada pelo ngrok) |

Nenhum valor real dessas variáveis está no repositório. Use o arquivo `.env.example` como modelo.

## Passo a passo

### 1. Subir o N8N

**Opção A: Docker (recomendado)**

```bash
cp .env.example .env
# edite o .env e preencha OPENWEATHER_API_KEY, TELEGRAM_BOT_TOKEN e WEBHOOK_URL
docker compose up -d
```

Para obter a `WEBHOOK_URL` localmente, rode em outro terminal:

```bash
ngrok http 5678
```

Copie a URL `https://....ngrok-free.app` exibida, coloque em `WEBHOOK_URL` no `.env` e reinicie com `docker compose up -d`.

**Opção B: npm**

```bash
export OPENWEATHER_API_KEY="sua_chave"
export TELEGRAM_BOT_TOKEN="seu_token"
export N8N_BLOCK_ENV_ACCESS_IN_NODE=false
export WEBHOOK_URL="https://sua-url-publica"
npx n8n
```

**Opção C: N8N Cloud**: a URL pública já é fornecida. Como o Cloud não permite definir variáveis de ambiente próprias, substitua no nó HTTP Request o valor de `appid` pela sua chave **somente na sua instância** (nunca no JSON do repositório).

Acesse o editor em `http://localhost:5678`.

### 2. Importar o workflow

1. No N8N, clique em **Overview** no menu lateral esquerdo e depois no botão **Create Workflow** (canto superior direito).
2. No canto superior direito do editor, clique no menu de três pontos (**...**) e escolha **Import from File...**.
3. Selecione o arquivo `workflow-chatbot-telegram.json`.
4. Clique em **Save** (canto superior direito).

### 3. Configurar a credencial do Telegram

1. Dê dois cliques no nó **Telegram Trigger**.
2. No campo **Credential to connect with** (topo do painel), clique em **Create New Credential**.
3. No campo **Access Token**, use uma das opções:
   - cole o token do BotFather; ou
   - clique em **Expression** (acima do campo) e informe `{{ $env.TELEGRAM_BOT_TOKEN }}` para ler a variável de ambiente.
4. Clique em **Save** e feche a janela.
5. Abra os nós **Enviar Temperatura** e **Enviar Mensagem de Erro** e selecione a mesma credencial no campo **Credential to connect with**.

### 4. Configurar a OpenWeather

Nada a fazer no editor: o nó **Consultar OpenWeather** já lê a chave pela expressão `{{ $env.OPENWEATHER_API_KEY }}`. Basta a variável estar definida no ambiente do N8N (passo 1).

### 5. Ativar e executar

1. Salve o workflow e ative o toggle **Inactive / Active** no canto superior direito.
2. No Telegram, abra o seu bot e envie mensagens de teste:

| Mensagem enviada | Retorno esperado |
|---|---|
| `São Paulo,SP,BR` | `🌤️ A temperatura em São Paulo é de XX°C.` |
| `Belo Horizonte,MG,BR` | `🌤️ A temperatura em Belo Horizonte é de XX°C.` |
| `florianópolis, sc, br` | `🌤️ A temperatura em Florianópolis é de XX°C.` |
| `Cidadeinexistente,XX,BR` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

Para depurar, use o menu **Executions** (aba no topo do editor) e abra a execução para ver os dados de cada nó.

## Google Gemini (opcional)

O workflow já contém o nó **Gemini: Melhorar Mensagem (opcional)** (Basic LLM Chain) ligado ao sub-nó **Google Gemini Chat Model**, posicionados entre **Formatar Mensagem (Fallback)** e **Selecionar Mensagem Final**. Ambos vêm **desativados**, então o workflow funciona sem custo e sem credencial do Gemini.

Configuração aplicada:

- modelo `models/gemini-2.5-flash`, temperatura `0.1` (respostas determinísticas);
- prompt instrui saída em português, em uma frase, mantendo cidade e temperatura, no formato `{"message":"...","ok":true}`;
- o nó tem **On Error: Continue**, ou seja, se o Gemini falhar o fluxo segue.

**Fallback obrigatório:** o nó Code **Formatar Mensagem (Fallback)** sempre gera a mesma `message` de forma determinística. O nó **Selecionar Mensagem Final** só usa o texto do Gemini se ele for um JSON válido com `ok: true`; em qualquer outro caso (nó desativado, sem credencial, resposta fora do formato), envia a mensagem do fallback.

Para ativar:

1. Gere uma chave em https://aistudio.google.com/app/apikey.
2. Dê dois cliques no nó **Google Gemini Chat Model**, em **Credential to connect with** clique em **Create New Credential** e cole a chave no campo **API Key** (credencial *Google Gemini (PaLM) API*).
3. Selecione os dois nós do Gemini, clique com o botão direito e escolha **Activate** (ou use a tecla `D`).
4. Salve o workflow.

Não inclua a chave do Gemini no repositório.

## Segurança

- O JSON exportado não contém tokens, chaves nem IDs de credenciais: a chave da OpenWeather é referenciada por `$env.OPENWEATHER_API_KEY` e as credenciais do Telegram/Gemini são criadas na sua instância após a importação.
- O arquivo `.env` está no `.gitignore`. Suba apenas o `.env.example`.
