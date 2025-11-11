# Bemobi IFrame Checkout - Documentação de Eventos

Este documento descreve como implementar e reagir aos eventos enviados pelo iframe Bemobi durante o processo de checkout no TAA. Documentação baseada no arquivo index.html.

Em contrato.html temos o mesmo fluxo de index.html, mas com o código de instalação sendo informado via entrada de usuário.

## Visão Geral

A Bemobi utiliza um iframe para gerenciar o processo de ativação de dispositivo, resolução de pendências e checkout. A comunicação entre a aplicação host e o iframe é feita através de mensagens postMessage.

## Configuração Inicial

### URLs Base
```javascript
const BASE_URL = "https://totem-bemobi-web.vercel.app";
```

### Gerenciamento de Cookies
O exemplo utiliza cookies para armazenar o sessionToken (Mais explicações adiante - DEVICE_ACTIVATION_OK).

```javascript
// Salvar cookie
function setCookie(name, value, days = 30) {
  const expires = new Date();
  expires.setTime(expires.getTime() + days * 24 * 60 * 60 * 1000);
  document.cookie = `${name}=${value};expires=${expires.toUTCString()};path=/;SameSite=Strict`;
}

// Recuperar cookie
function getCookie(name) {
  const nameEQ = name + "=";
  const ca = document.cookie.split(";");
  for (let i = 0; i < ca.length; i++) {
    let c = ca[i];
    while (c.charAt(0) === " ") c = c.substring(1, c.length);
    if (c.indexOf(nameEQ) === 0)
      return c.substring(nameEQ.length, c.length);
  }
  return null;
}

// Remover cookie
function deleteCookie(name) {
  document.cookie = `${name}=;expires=Thu, 01 Jan 1970 00:00:00 UTC;path=/;SameSite=Strict`;
}
```

### Configuração do IFrame

É **essencial** que o elemento iframe inclua o atributo `allow="local-network-access"` para permitir que o conteúdo do iframe faça requisições para servidores locais (localhost, 127.0.0.1).

```html
<iframe 
  id="testIframe" 
  src="" 
  class="w-100 h-100 border-0" 
  allow="local-network-access">
</iframe>
```

#### Motivo Técnico

Navegadores modernos implementam a especificação **Private Network Access** (também conhecida como CORS-RFC1918), que bloqueia requisições de páginas HTTPS para endereços de rede local (localhost, 127.0.0.1, endereços privados) por questões de segurança.

Quando o iframe carregado a partir de uma origem HTTPS (como `https://totem-bemobi-web.vercel.app`) tenta fazer requisições para um servidor local (como `https://127.0.0.1/agente/clisitef/session`), o navegador bloqueia essas requisições e retorna um erro de CORS:

```
Access to fetch at 'https://127.0.0.1/agente/clisitef/session' from origin 'https://totem-bemobi-web.vercel.app' 
has been blocked by CORS policy: Permission was denied for this request to access the `unknown` address space.
```

O atributo `allow="local-network-access"` concede explicitamente permissão para que o iframe faça requisições para a rede local, resolvendo esse problema. Isso é especialmente necessário quando o iframe precisa se comunicar com serviços locais, como o SITEF rodando em `127.0.0.1`.

**Nota sobre compatibilidade:** Este atributo funciona em navegadores baseados em Chromium (Chrome, Edge) a partir de versões recentes. Para outros navegadores, pode ser necessário configurar o servidor local com CORS adequado ou usar outras soluções como túneis HTTPS.

## Fluxo de Inicialização

### Verificação de Session Token
```javascript
window.onload = function () {
  sessionToken = getCookie("sessionToken");

  if (sessionToken) {
    // Se o sessionToken existe, redirecionar para a URL com o token
    document.getElementById("testIframe").src =
      BASE_URL + "/bemobi/" + sessionToken + "/device";
  } else {
    // Se não existe sessionToken, redirecionar para ativação
    document.getElementById("testIframe").src =
      BASE_URL + "/activation/serial";
  }
};
```

## Eventos Recebidos do IFrame

### 1. DEVICE_ACTIVATION_OK
**Quando:** Dispositivo ativado com sucesso
**Ação:** Salvar o sessionToken e redirecionar para a tela principal. O sessionToken é quem identificará o TAA durante todo o processo de checkout. Armazene-o. Em caso de perda, será necessário reativar o TAA.

```javascript
if (e.data.event == "DEVICE_ACTIVATION_OK") {
  console.log("Dispositivo ativado com sucesso. Session Token: ", e.data.sessionToken);

  // Salvar o sessionToken em um cookie
  if (e.data.sessionToken) {
    setCookie("sessionToken", e.data.sessionToken, 30); // Cookie válido por 30 dias
    console.log("Session Token salvo em cookie com sucesso!");
    document.getElementById("testIframe").src =
      BASE_URL + "/bemobi/" + e.data.sessionToken + "/device";
  } else {
    console.warn("Session Token não foi recebido na mensagem DEVICE_ACTIVATION_OK");
  }
}
```

### 2. DEVICE_ACTIVATION_FAILED
**Quando:** Falha na ativação do dispositivo
**Ação:** Remover sessionToken e acionar suporte para ativação/reativação do totem

```javascript
if (e.data.event == "DEVICE_ACTIVATION_FAILED") {
  console.log("Falha na ativação do dispositivo. Removendo sessionToken do cookie.");
  deleteCookie("sessionToken");
  console.log("Session Token removido do cookie com sucesso!");
  // Acionar suporte para obter credenciais para o totem
}
```

### 3. SITEF_ISSUE_RESOLUTION_OK
**Quando:** Resolução de pendências concluída com sucesso
**Ação:** Redirecionar para a listagem.

```javascript
if (e.data.event == "SITEF_ISSUE_RESOLUTION_OK") {
  document.getElementById("testIframe").src =
    BASE_URL + "/bemobi/" + sessionToken + "/light";
}
```

### 4. SITEF_ISSUE_RESOLUTION_ERROR
**Quando:** Falha na resolução de pendências
**Ação:** Nada a fazer. Deixe que o terminal siga o fluxo normal. Trata-se apenas de um aviso.

```javascript
if (e.data.event == "SITEF_ISSUE_RESOLUTION_ERROR") {
  console.log("Falha na resolução de pendências");
  //Siga com a operação.
}
```

### 5. PAGE_LOADED
**Quando:** Página do iframe carregou completamente
**Ação:** Enviar dados para listagem após delay. Recomenda-se aguardar 1 segundo antes de enviar a mensagem, como no exemplo.

```javascript
if (e.data.event == "PAGE_LOADED") {
  setTimeout(enviarDadosParaListagem, 1000); // Aguarda 1 segundo para garantir que o iframe carregou
}
```

### 6. PAYMENT_IN_PROGRESS
**Quando:** Pagamento iniciado
**Ação:** Bloquear ações de saída da tela. Durante o fluxo de pagamento é imprescindível que o cliente permaneça na tela até que o Checkout termine com o evento adequado. Aguardar PAYMENT_SUCCESS, PAYMENT_ERROR ou PAYMENT_CANCELLED para liberar outras ações.

```javascript
if (e.data.event == "PAYMENT_IN_PROGRESS") {
  console.log("Pagamento em andamento...");
  // Remover qualquer ação que um usuário possa usar para sair da tela
  // Aguardar PAYMENT_SUCCESS, PAYMENT_ERROR ou PAYMENT_CANCELLED para liberar outras ações.
}
```

### 7. PAYMENT_OK
**Quando:** Pagamento finalizado com sucesso
**Ação:** Liberar ações de saída e processar dados da transação

```javascript
if (e.data.event == "PAYMENT_OK") {
  console.log("Pagamento finalizado com sucesso!!! Dados da transação: ", e.data.transaction);
  // Liberar as ações que um usuário poderia usar para sair da tela
  // Processar dados da transação conforme necessário
}
```

### 8. PAYMENT_ERROR
**Quando:** Pagamento finalizado com falha
**Ação:** Liberar ações de saída e tratar erro. O checkout ainda pode permitir retentativa. Recomenda-se que use esse evento apenas como informativo.

```javascript
if (e.data.event == "PAYMENT_ERROR") {
  console.log("Pagamento finalizado com falha na transação. Dados da transação: ", e.data.error);
  // Liberar as ações que um usuário poderia usar para sair da tela
  // O pagamento ainda poderá ser retentado dentro do IFrame
}
```

### 9. PAYMENT_CANCELLED
**Quando:** Pagamento cancelado pelo usuário
**Ação:** Retornar à uma tela a ser definida pelo integrador.

```javascript
if (e.data.event == "PAYMENT_CANCELLED") {
  console.log("Pagamento Cancelado: ", e.data.invoices, e.data.error);
  document.getElementById("testIframe").src = BASE_URL + "/bemobi/" + sessionToken + "/device";
}
```

### 9. PAYMENT_GO_BACK
**Quando:** Pagamento cancelado pelo usuário sem finalizar sessão
**Ação:** Retornar à uma tela a ser definida pelo integrador.

```javascript
if (e.data.event == "PAYMENT_GO_BACK") {
  console.log("Pagamento Voltou: ", e.data.invoices, e.data.error);
  document.getElementById("testIframe").src = BASE_URL + "/bemobi/" + sessionToken + "/device";
}
```

### 10. PAYMENT_TIMEOUT
**Quando:** Em casos de abandono do totem pelo cliente. Ocorre em 1 minuto de inatividade.
**Ação:** Retornar à tela inicial.

```javascript
if (e.data.event == "PAYMENT_TIMEOUT") {
  document.getElementById("testIframe").src = BASE_URL + "/bemobi/" + sessionToken + "/device";
}
```

## Envio de Dados para o IFrame

### Dados de Conta para Consulta
```javascript
function enviarDadosParaListagem() {
  const iframe = document.querySelector("iframe");
  const mensagem = {
    event: "SELECTED_INVOICES",//Mantenha esse evento.
    ids: [""], //IDs das faturas selecionadas.
    documentNumber: "01205793321", //CPF ou CNPJ do cliente
    installationCode: "3018353422", //Código de instalação
    contractCode: "3019024597", //Código do contrato
    protocol: "20251003000001", //EX_GUID do Protocolo pai
    protocol2: "20000200000000", //Protocolo pai
    partnerNumber: "30000050000000", //Número do parceiro de negócios
  };
  iframe.contentWindow.postMessage(mensagem, "*");
}
```

## Listener Principal
Receba mensagens do IFrame Bemobi com um código como abaixo: 
```javascript
window.onmessage = function (e) {
  if (e.origin === BASE_URL) {
    console.log("Mensagem recebida no iframe:", e.data);
  }
  
  if (e.data.event != undefined) {
    // Processar eventos conforme documentação acima
  }
};
```

## Fluxo Completo

O fluxo se divide em 3 partes:

### Ativação do dispositivo:
1. **Inicialização**: Verificar se existe sessionToken
2. **Ativação**: Se não houver token, redirecionar para `/activation/serial`. Ao final deste processo, se a ativação ocorrer com sucesso, o token de sessão será obtido e deverá ser salvo pela aplicação.

Enquanto o session token estiver válido e um evento `DEVICE_ACTIVATION_FAILED` não tiver sido recebido, esse procedimento não precisará ser realizado novamente.



### Inicialização do dispositivo
1. **Buscar dados do dispositivo**: Direcionar o iframe para `/bemobi/{token}/device`. Neste ponto, os dados do totem serão validados com o token obtido na ativação e então será iniciado o processo de resolução de pendências.
2. **Resolução de Pendências**: Aguardar `SITEF_ISSUE_RESOLUTION_OK`. Ao receber esse status, a aplicação pode ir para o checkout ou fazer outras atividades próprias.

### Checkout
1. **Listagem de faturas**: Redirecionar para `BASE_URL/bemobi/{token}/light`
6. **Envio de Dados**: Enviar dados das contas selecionadas, instalação e protocolo quando receber o evento `PAGE_LOADED`, conforme exemplo.
7. **Pagamento**: Gerenciar estados de pagamento conforme descrito anteriormente
8. **Finalização**: Processar resultado do pagamento, exibido a tela de sucesso de pagamento e imprimindo o comprovante da fatura. **TODO COMPROVANTE DEVE CONTER O COMPROVANTE DO SITEF AO FINAL. PODE SER OBTIDO NO OBJETO transaction NO EVENTO PAYMENT_OK**

## Considerações Importantes

- **Segurança**: Sempre verificar `e.origin === BASE_URL` antes de processar mensagens
- **Session Token**: Esse token não se renova com frequência. É gerado uma vez e não deve mudar até que seja recebida uma falha de ativação.
- **UI/UX**: Bloquear ações de saída durante pagamento em progresso (evento `PAYMENT_IN_PROGESS`)

## Estrutura de Mensagens

Todas as mensagens seguem o padrão:
```javascript
{
  event: "NOME_DO_EVENTO",
  // dados específicos do evento
}
```

### Exemplo de Mensagem de Sucesso:
```javascript
{
  event: "DEVICE_ACTIVATION_OK",
  sessionToken: "abc123def456"
}
```

### Exemplo de Mensagem de Erro:
```javascript
{
  event: "PAYMENT_ERROR",
  error:  "Descrição do erro"
}
```
