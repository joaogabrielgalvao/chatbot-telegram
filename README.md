# Chatbot de Clima no Telegram com N8N

Bot do Telegram que informa a temperatura atual de uma cidade do Brasil. O usuário envia a cidade no formato `Cidade,UF,BR`, o N8N consulta a API da OpenWeather e responde no chat.

## Fluxo

1. **Telegram Trigger**: recebe a mensagem.
2. **Capturar e Formatar Cidade** (Set): guarda o texto na variável `queue`, sem acentos, em minúsculas e sem espaços sobrando.
3. **Consultar OpenWeather** (HTTP Request): consulta `https://api.openweathermap.org/data/2.5/weather` com `q`/`queue`, `units=metric`, `lang=pt_br` e `appid={{ $env.OPENWEATHER_API_KEY }}`.
4. **Resposta Válida?** (IF): confere se o status é 200 e se veio a temperatura.
5. **Formatar Mensagem** (Code): arredonda a temperatura e monta o texto.
6. **Enviar Temperatura / Enviar Mensagem de Erro** (Telegram): responde ao usuário.

## Variáveis

| Variável | Uso |
|---|---|
| `OPENWEATHER_API_KEY` | Chave da OpenWeather |
| `TELEGRAM_BOT_TOKEN` | Token do bot gerado no @BotFather |

Nenhuma chave ou token está neste repositório.

## Como importar e configurar

1. No N8N, crie um workflow em branco, abra o menu **...** (canto superior direito) e clique em **Import from File**. Selecione `workflow-chatbot-telegram.json`.
2. Abra o nó **Telegram Trigger**, clique em **Create New Credential** e cole o `TELEGRAM_BOT_TOKEN` no campo **Access Token**. Selecione essa credencial também nos nós **Enviar Temperatura** e **Enviar Mensagem de Erro**.
3. Defina a variável de ambiente `OPENWEATHER_API_KEY` no servidor do N8N (com `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`). No N8N Cloud, que não permite variáveis de ambiente, cole a chave direto no parâmetro `appid` do nó **Consultar OpenWeather**.
4. Salve e ative o workflow.

## Teste

| Mensagem | Resposta |
|---|---|
| `São Paulo,SP,BR` | 🌤️ A temperatura em São Paulo é de 23°C. |
| `Belo Horizonte,MG,BR` | 🌤️ A temperatura em Belo Horizonte é de 25°C. |
| `Cidadeinexistente,XX,BR` | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |
