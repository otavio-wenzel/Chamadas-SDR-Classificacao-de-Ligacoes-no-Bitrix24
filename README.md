````markdown
# Chamadas SDR – Classificação de Ligações no Bitrix24

Aplicação leve em HTML/JS que roda dentro do **CALL_CARD** do Bitrix24 e
permite que o usuário classifique cada ligação com um **status padronizado**,
salvando essa informação diretamente na **atividade de CRM** – pronta para uso
em relatórios e BI.

---

## 1. O que a aplicação faz

- Descobre automaticamente o **usuário logado** (`user.current`).
- Busca a **última ligação concluída** (`crm.activity.list` com `TYPE_ID = 2`,
  `COMPLETED = Y` e `RESPONSIBLE_ID = usuário`).
- Exibe no topo da tela:

  > Última ligação concluída: **NOME DO CONTATO – TELEFONE**

- Permite escolher um **status de resultado da chamada** em uma lista fixa:

  - `REUNIÃO AGENDADA`
  - `FALEI COM SECRETÁRIA`
  - `FOLLOW-UP`
  - `RETORNO POR E-MAIL`
  - `NÃO TEM INTERESSE`
  - `NÃO FAZ LOCAÇÃO`
  - `CAIXA POSTAL`

- Ao clicar em **“Salvar resultado desta ligação”**:
  - Atualiza a **mesma atividade** exibida no cabeçalho.
  - Grava:
    - `RESULT_VALUE` = status escolhido
    - `DESCRIPTION`  = status escolhido  
  - Ou seja, o BI pode ler **apenas o campo `DESCRIPTION`** para obter o
    resultado da chamada.

- Possui:
  - Botão **“Registrar / atualizar CALL_CARD”** (somente para administradores)
    para vincular o app ao cartão de chamadas.
  - Botão **“Atualizar última ligação”** para recarregar o cabeçalho.
  - **Auto-refresh** a cada 2 minutos (configurável no código).
  - Área de **log interno** para depuração.

---

## 2. Estrutura do projeto

Arquivo principal:

- `index.html` (ou outro nome, desde que usado como handler da app)

Principais blocos:

- `<style>` – estilos básicos (layout simples e responsivo).
- `<script src="https://api.bitrix24.com/api/v1/"></script>` – SDK JS do Bitrix24.
- `<script>` – toda a lógica da aplicação.

Elementos de UI:

- `#installBtn` – botão azul (registrar / atualizar CALL_CARD).
- `#lastCallInfo` – exibe a última ligação concluída.
- `#refreshBtn` – botão cinza para atualizar a última ligação.
- `#callResult` – `<select>` com os status possíveis.
- `#saveBtn` – botão verde para salvar o resultado.
- `#debug` – área de log (pode ser escondida/removida em produção).

---

## 3. Fluxo de funcionamento

### 3.1. Inicialização

1. Carrega o SDK do Bitrix:
   ```js
   BX24.init(function () { ... });
````

2. Obtém o usuário logado:

   ```js
   BX24.callMethod('user.current', {}, ...);
   ```

3. Verifica se o usuário é **admin**:

   ```js
   if (BX24.isAdmin && BX24.isAdmin()) { ... }
   ```

   * Se for admin, o botão azul `#installBtn` é exibido.
   * Se não for admin, o botão permanece oculto.

4. Chama `loadLastCallAndHeader()` para preencher o cabeçalho com a **última
   ligação concluída**.

### 3.2. Descoberta da última ligação concluída

A função `loadLastCallAndHeader` faz:

```js
BX24.callMethod('crm.activity.list', {
  order:  { "END_TIME": "DESC" },
  filter: {
    "TYPE_ID": 2,           // CALL
    "RESPONSIBLE_ID": userId,
    "COMPLETED": "Y"
  },
  select: ["ID", "SUBJECT", "END_TIME", "TYPE_ID", "RESPONSIBLE_ID", "COMPLETED"]
}, ...);
```

* Pega o **primeiro item** da lista (`data[0]`) como **última call concluída**.

* Guarda o ID em `LAST_ACTIVITY_ID`.

* Tenta descobrir o **nome** e o **telefone**:

  1. Primeira tentativa:
     `crm.activity.get` → `COMMUNICATIONS` → `ENTITY_TYPE` / `ENTITY_ID`
     → `crm.contact.get` / `crm.company.get` / `crm.lead.get`.
  2. Se não houver dados em `COMMUNICATIONS`:
     `voximplant.statistic.get` com `FILTER: { CRM_ACTIVITY_ID: activityId }`
     e repete a lógica de pegar nome / telefone a partir da entidade.

* Monta o texto:

  ```txt
  Última ligação concluída: Nome do contato – +550000000000
  ```

  e exibe em `#lastCallInfo`.

### 3.3. Registro do status da chamada

Quando o usuário clica no botão verde:

1. Lê o status selecionado em `#callResult`.

2. Dispara **novamente** `loadLastCallAndHeader` para garantir que está usando a
   última call concluída.

3. Com o `activityId` em mãos, executa:

   ```js
   BX24.callMethod('crm.activity.update', {
     id: activityId,
     fields: {
       RESULT_VALUE: result,
       DESCRIPTION: result
     }
   }, ...);
   ```

4. Exibe um `alert` informando que o status foi salvo com sucesso.

---

## 4. Botão azul – Registrar / atualizar CALL_CARD (somente admin)

O botão **“Registrar / atualizar CALL_CARD”**:

1. Executa um `placement.unbind` para limpar associações antigas:

   ```js
   BX24.callMethod('placement.unbind', {
     PLACEMENT: 'CALL_CARD'
   }, ...);
   ```

2. Em seguida, faz o bind do handler atual:

   ```js
   var handlerUrl = window.location.href.split('#')[0].split('?')[0];

   BX24.callMethod('placement.bind', {
     PLACEMENT: 'CALL_CARD',
     HANDLER: handlerUrl,
     TITLE: 'Resultado da chamada',
     DESCRIPTION: 'Classificação de cada ligação'
   }, ...);
   ```

> **Importante:**
> Esse botão só aparece se `BX24.isAdmin()` retornar `true`. Qualquer usuário
> comum não verá nem poderá disparar esse bind.

---

## 5. Integração com BI

Para facilitar o uso em relatórios:

* Cada ligação concluída terá:

  * `DESCRIPTION = <status escolhido>`
  * `RESULT_VALUE = <status escolhido>` (campo nativo da atividade)

Exemplo de dados gravados:

```txt
DESCRIÇÃO  = "REUNIÃO AGENDADA"
RESULT_VALUE = "REUNIÃO AGENDADA"
```

No BI, você pode:

* Ler apenas o campo `DESCRIPTION` das atividades de tipo **CALL (TYPE_ID = 2)**.
* Filtrar por `RESPONSIBLE_ID` (usuário).
* Opcionalmente cruzar com dados de **telefonia (voximplant.statistic)**.

---

## 6. Requisitos e permissões

A aplicação usa os seguintes métodos:

* `user.current`
* `crm.activity.list`
* `crm.activity.get`
* `crm.activity.update`
* `crm.contact.get`
* `crm.company.get`
* `crm.lead.get`
* `voximplant.statistic.get`
* `placement.bind`
* `placement.unbind`

Portanto, ao criar a aplicação no Bitrix24, certifique-se de conceder
permissões mínimas:

* **CRM** – leitura/escrita de atividades e entidades.
* **Telephony / Voximplant** – leitura das estatísticas de chamadas.
* **User** – leitura de dados básicos do usuário logado.
* **Placement** – uso do `CALL_CARD`.

---

## 7. Como instalar no Bitrix24 (visão geral)

1. **Hospede o arquivo HTML**
   Em um servidor acessível via HTTPS (por exemplo: `https://seu-dominio.com/chamadas-sdr/index.html`).

2. **Crie uma aplicação local no Bitrix24**

   * Vá em **Bitrix24 > Aplicações > Aplicações próprias / Local**.
   * Informe a URL do handler (o HTML hospedado).
   * Conceda as permissões citadas acima.
   * Salve.

3. **Abra a aplicação** dentro do Bitrix24:

   * Acesse o app.
   * Verifique se o painel sobe corretamente (sem erros de JS).

4. **Clique em “Registrar / atualizar CALL_CARD” (como admin)**

   * Isso vincula a aplicação ao **CALL_CARD**.
   * A partir daí, ao abrir uma chamada no Bitrix (janela de discagem / cartão
     de chamada), o app será carregado no painel.

5. **Teste o fluxo**

   * Faça uma ligação de teste.
   * Encerre a ligação.
   * Clique em “Atualizar última ligação”.
   * Verifique se o cabeçalho mostra o contato e telefone corretos.
   * Escolha um status e clique em “Salvar resultado desta ligação”.
   * Abra a atividade no CRM e confirme o valor em `DESCRIPTION`.

---

## 8. Customizações úteis

### 8.1. Alterar ou remover o auto-refresh

No final da inicialização há:

```js
setInterval(function () {
  log('Auto-refresh (2min) da última ligação concluída...');
  loadLastCallAndHeader(function (activityId) {
    log('Auto-refresh atualizado. Atividade ID = ' + activityId);
  });
}, 120000); // 2 minutos
```

* Para desativar o auto-refresh, basta **comentar** esse bloco.
* Para mudar o intervalo, altere `120000` para outro valor (em milissegundos).

### 8.2. Esconder/remover o painel de debug

A área de debug é o `<div id="debug">` e a função `log()`.

Opções:

1. **Esconder via CSS**:

   ```css
   #debug {
     display: none;
   }
   ```

2. **Remover completamente**:

   * Apagar a `<div id="debug"></div>` do HTML.
   * Na função `log()`, manter o `if (!dbg) return;` para que nada quebre.

### 8.3. Alterar a lista de status

A lista está no `<select id="callResult">`.
Basta editar as `<option>` existentes ou adicionar novas, lembrando que:

* O *texto* da opção é exatamente o que será gravado em `DESCRIPTION` e `RESULT_VALUE`.
* Se for mexer na padronização para BI, documente esses valores em algum lugar do projeto.

---

## 9. Segurança

* O código **não contém** credenciais estáticas, tokens ou segredos.
* Todas as chamadas à API são feitas via SDK `BX24`, reaproveitando a **sessão
  do usuário logado** no portal.
* Dado sensível só aparece:

  * Em **tempo de execução** (dentro dos logs).
  * Em prints de tela.
* Para produção, recomenda-se:

  * Minimizar ou ocultar o painel de debug.
  * Evitar expor logs em vídeos / prints públicos.

---

## 10. Licença

Adapte esta seção conforme a política da sua empresa (MIT, proprietária etc.).

```txt
Este projeto é de uso interno. A redistribuição externa deve seguir as
políticas de segurança e confidencialidade da organização.
```

---

```
::contentReference[oaicite:0]{index=0}
```
