# 📧 Automação de Envio de E-mails (n8n)

![n8n](https://img.shields.io/badge/n8n-Workflow-FF6D5A?logo=n8n&logoColor=white)
![Status](https://img.shields.io/badge/Status-Pronto_para_Uso-success)
![License](https://img.shields.io/badge/License-MIT-blue)

Este repositório contém o workflow do n8n para automação de envio de e-mails em massa a partir de uma planilha Excel (`.xlsx`), com controle rigoroso de status, limite diário de disparos e execução agendada.

## 📑 Sumário
- [🎯 Objetivo do Workflow](#-objetivo-do-workflow)
- [📊 Estrutura da Planilha de Origem](#-estrutura-da-planilha-de-origem)
- [🔄 Funcionamento do Controle de Status](#-funcionamento-do-controle-de-status)
- [⏱️ Limite Diário de Envios](#️-limite-diário-de-envios)
- [⚙️ Configuração de Credenciais SMTP no n8n](#️-configuração-de-credenciais-smtp-no-n8n)
- [🚀 Passo a Passo para Implantação](#-passo-a-passo-para-implantação)
- [🛡️ Notas de Segurança e Manutenção](#️-notas-de-segurança-e-manutenção)

---

## 🎯 Objetivo do Workflow
Automatizar o envio de e-mails personalizados para uma base de contatos, garantindo que:
1. Nenhum contato receba e-mail duplicado.
2. O volume de disparos seja controlado (máximo de 200 por execução).
3. O arquivo de origem seja atualizado automaticamente com o resultado de cada tentativa (sucesso ou erro).
4. O processo rode de forma autônoma, sem necessidade de intervenção manual.

---

## 📊 Estrutura da Planilha de Origem
O arquivo `.xlsx` de entrada **deve** conter obrigatoriamente as seguintes colunas na primeira linha (cabeçalho):

| Coluna | Descrição | Exemplo de Conteúdo |
| :--- | :--- | :--- |
| `Nome` | Nome do destinatário para personalização da mensagem. | `Maria Silva` |
| `Email` | Endereço de e-mail válido para envio. | `maria@exemplo.com` |
| `Status` | Campo de controle do workflow. **Deve iniciar vazio** para novos contatos. | *(vazio)*, `Enviado em 18/07/26 às 09:00`, `Erro: Invalid recipient` |

> ⚠️ **Importante:** Não altere os nomes das colunas. O workflow depende exatamente de `Nome`, `Email` e `Status` (respeitando maiúsculas e minúsculas).

---

## 🔄 Funcionamento do Controle de Status
O workflow avalia a coluna `Status` de cada linha e segue a seguinte lógica:

1. **Vazio (nulo ou string vazia)**: O contato é elegível para envio. O workflow processa o e-mail.
2. **Enviado em dd/mm/aa às hh:mm**: O e-mail foi entregue com sucesso. O workflow **ignora** este contato em execuções futuras.
3. **Erro: [motivo do erro]**: O envio falhou (ex: e-mail inválido, rejeição do servidor). O motivo técnico é gravado na célula para diagnóstico. O workflow **ignora** este contato em execuções futuras para evitar loops de erro, permitindo correção manual posterior.

---

## ⏱️ Limite Diário de Envios
Para proteger a reputação do domínio de e-mail e respeitar limites do provedor SMTP, o workflow possui um nó de **Limit** configurado para processar **no máximo 200 contatos por execução**. 

- Se houver 500 contatos com `Status` vazio, apenas os primeiros 200 serão processados na execução atual.
- Os 300 restantes aguardarão a próxima execução agendada.
- Isso garante que nunca haja reenvio duplicado e que o limite diário seja estritamente respeitado.

---

## ⚙️ Configuração de Credenciais SMTP no n8n
Para que o nó "Send an Email" funcione, você deve configurar uma credencial SMTP válida no seu n8n:

1. No editor do workflow, clique no nó **Send an Email**.
2. No campo **Credential to connect with**, clique em **Create New** (ou selecione uma existente).
3. Preencha os dados do seu servidor de saída:
   - **Host**: `smtp.seudominio.com` (ex: `smtp.gmail.com`, `smtp.office365.com`)
   - **Port**: `587` (TLS) ou `465` (SSL)
   - **User**: Seu endereço de e-mail completo (ex: `supervisoras@didier.com.br`)
   - **Password**: Senha do e-mail ou "App Password" (recomendado para Gmail/Outlook).
4. Clique em **Save** e depois em **Test** para verificar a conexão.
5. Salve o workflow.

---

## 🚀 Passo a Passo para Implantação

1. **Prepare o ambiente**: Certifique-se de que o n8n tem acesso ao diretório onde a planilha será armazenada (ex: `/home/node/.n8n-files/contatos.xlsx` no Docker, ou ajuste o caminho no nó *Read/Write Files from Disk* para seu ambiente local).
2. **Importe o Workflow**: 
   - No n8n, clique em **Add Workflow** > **Import from File**.
   - Selecione o arquivo `Automação_de_Envio_de_Emails.json` deste repositório.
3. **Ajuste os Caminhos**: Verifique se o campo `File Selector` nos nós de leitura e escrita aponta para o local correto do seu arquivo `.xlsx`.
4. **Configure as Credenciais**: Siga o passo a passo de SMTP acima.
5. **Ative o Agendamento**: 
   - O workflow já possui um nó **Schedule Trigger** configurado para rodar 1x por dia (ex: às 09:00).
   - Clique no switch **Active** no canto superior direito do editor do n8n para ligar a automação.
6. **Teste Inicial**: Coloque 2 ou 3 contatos de teste na planilha com a coluna `Status` vazia e clique em **Execute Workflow** manualmente para validar o fluxo completo antes de confiar no agendamento.

---

## 📈 Fluxo de Execução (Diagrama)

```mermaid
graph TD
    A[📄 Planilha .xlsx Original] --> B{⚖️ Status está Vazio?}
    B -- Não --> F[⏭️ Ignorar e Preservar]
    B -- Sim --> C[🛑 Nó Limit: Máx 200 itens]
    C --> D[📧 Nó Send an Email SMTP]
    D --> E{✅ Envio com Sucesso?}
    E -- Sim --> G[📝 Atualiza Status: 'Enviado em dd/mm/aa às hh:mm']
    E -- Não --> H[📝 Atualiza Status: 'Erro: motivo da falha']
    G --> I[💾 Nó Write: Sobrescreve Planilha .xlsx]
    H --> I
    F --> I