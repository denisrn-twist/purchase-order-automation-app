# Epic 1: Purchase Order Automation (MuleSoft)

Este documento centraliza todas as Histórias de Usuário (User Stories) que estão atualmente **codificadas e funcionais** na aplicação \`purchase-order-automation-app\`.

---

## Story 1.1: Ingestão de Documentos em Lote (Scheduler)

> [!WARNING]
> **PENDÊNCIA DE AMBIENTE (AWS S3)**
> Estamos aguardando a definição final do cliente sobre o mapeamento exato dos buckets (Dev/QA/UAT/Prod) e o padrão de nomenclatura (ex: \`po_file_*.pdf\`).

### Visão Geral
- **Ação:** Um Agendador (Scheduler) roda a cada hora na aplicação MuleSoft.
- **Objetivo:** Varrer o bucket do AWS S3 em busca de novos PDFs de Purchase Orders (ignorando \`.zip\`).
- **Arquitetura (Assíncrona):** Ao encontrar os arquivos, o fluxo não os processa imediatamente. Ele publica o nome de cada arquivo em uma **Fila VM Persistente** (\`po-processing-queue\`) para que sejam processados de forma assíncrona, paralela e segura contra falhas de rede.

### Especificação Técnica
- **Source:** Agendador Mule (\`0 0 * * * ?\`)
- **AWS S3 Actions:** \`s3:ListObjects\`
- **Destino:** VM Queue (\`<vm:publish>\`)

---

## Story 1.2: Gatilho Sob Demanda via System API (Salesforce)

### Visão Geral
- **Ação:** O Salesforce pode fazer uma requisição \`GET /system/s3-file-lookup/v1/files/{fileName}\` buscando um arquivo específico.
- **Objetivo:** Verificar se o arquivo já chegou no S3. Se sim, responder instantaneamente com os metadados (HTTP 200) e **engatilhar** o processamento desse arquivo.
- **Arquitetura (Assíncrona):** Assim como o Agendador, a API apenas publica o nome do arquivo na **Fila VM** (\`po-processing-queue\`), unificando a esteira de processamento para arquivos vindos em lote ou sob demanda, evitando *Timeout* na tela do Salesforce.

### Especificação Técnica
- **Source:** HTTP Listener (\`8081\`)
- **AWS S3 Actions:** \`s3:ListObjects\` (com filtro)
- **Destino:** VM Queue (\`<vm:publish>\`)
- **Respostas:** \`200 OK\` (com JSON de metadados) ou \`404 Not Found\`.

---

## Story 1.3: Extração Inteligente e Validação (Consumidor)

> [!WARNING]
> **PENDÊNCIA DE REGRA DE NEGÓCIO (Salesforce)**
> O processamento dos cenários de ERRO (Dados Inválidos) está com um Mapeamento (DataWeave) genérico, aguardando o cliente fornecer os Nomes de API definitivos amanhã.

### Visão Geral
- **Ação:** O Consumidor (Listener) da fila VM pega um arquivo por vez e executa o processamento pesado.
- **Objetivo 1 (MuleSoft IDP):** Baixar o PDF do S3 e enviá-lo para o **Intelligent Document Processing (IDP)** para extração das informações via OCR (PO Number, Amount, QuoteId, etc).
- **Objetivo 2 (Salesforce):** Avaliar o \`ConfidenceScore\`. Se for maior que 90%, consultar a Quote no Salesforce. Se os dados da PO baterem exatamente com o CRM (Total, Endereços, Método de Pagamento e PO Number), a Quote é aprovada. Caso contrário, os dados de falha são enviados ao CRM.

### Mapa de Validação (MuleSoft -> Salesforce)
| MuleSoft Data (IDP) | Salesforce Target Object & Field | Condição |
| :--- | :--- | :--- |
| \`quoteId\` | \`Quote.Name\` | SOQL \`WHERE\` clause |
| \`totalAmount\` | \`Quote.GrandTotal\` | Regra de Match (Validação) |
| \`poNumber\` | \`Opportunity.Purchase_Order_Reference__c\` | Regra de Match (Validação) |
| \`billingAddress\` | \`Opportunity.FinServ__BillingAddress__c\` | Regra de Match (Validação) |
| \`shippingAddress\` | \`Opportunity.FinServ__ShippingAddress__c\` | Regra de Match (Validação) |
| \`paymentMethod\` | \`Opportunity.FinServ__PaymentMethod__c\` | Regra de Match (Validação) |
| **Status: "Approved"** | \`Quote.Status\` | Atualiza no Salesforce se **Passar** |
| **"Draft" (Placeholder)** | \`Quote.Status\` | Atualiza no Salesforce se **Falhar** |

### Especificação Técnica
- **Source:** VM Listener (\`po-processing-queue\`)
- **AWS S3 Actions:** \`s3:GetObject\` (Baixar PDF)
- **IDP Actions:** \`POST /executions\` e \`GET /executions/{id}\` (Polling via \`Until-Successful\`)
- **Salesforce Actions:** \`salesforce:query\`, \`salesforce:update\` (Sucesso e Erro), \`salesforce:create\` (Task de Exceção).
- **Tratamento de Erros:** O \`On-Error-Continue\` garante que se um arquivo quebrar em qualquer etapa, a transação vira uma Exception Task no Salesforce, e a fila prossegue para o próximo documento.
