# Documento de Arquitetura --- Plataforma de Processamento de NF-e

## 1. Objetivo

Este documento descreve a arquitetura lógica de uma plataforma de
ingestão, validação fiscal, classificação contábil, conciliação e
integração de Notas Fiscais Eletrônicas (NF-e).

A arquitetura é organizada segundo o modelo **C4**, com foco nas visões
de **Contexto do Sistema (Nível 1)** e **Containers (Nível 2)**. O
detalhamento interno dos microsserviços (Nível 3 --- Componentes) não
está representado no diagrama de origem e deverá ser elaborado durante o
desenho técnico de cada serviço.

## 2. Escopo e fronteiras

O sistema recebe arquivos XML de NF-e por meio de uma interface web ou
de integração automatizada. Após a ingestão, os documentos são
armazenados, submetidos à validação junto à SEFAZ, classificados
contabilmente e conciliados com os dados de compras. Quando as
verificações são concluídas, os lançamentos são encaminhados ao ERP
corporativo.

A plataforma também mantém dados transacionais e de auditoria, utiliza
cache para idempotência e publica eventos em filas para desacoplar o
processamento entre os serviços.

### Capacidades de negócio

-   Receber XMLs de NF-e por interface web e integração automatizada.
-   Armazenar os XMLs originais.
-   Evitar processamento duplicado por meio de controles de
    idempotência.
-   Validar documentos fiscais junto à SEFAZ.
-   Classificar documentos conforme regras contábeis.
-   Conciliar NF-e com pedidos de compra e contratos.
-   Registrar divergências e trilhas de auditoria.
-   Integrar lançamentos ao livro razão do ERP corporativo.
-   Encaminhar falhas críticas para uma fila de mensagens não
    processadas (DLQ).

## 3. C4 --- Nível 1: Contexto do Sistema

### Sistema em foco

**Plataforma de Processamento de NF-e**

Responsável por orquestrar o ciclo de vida operacional e contábil das
NF-e, desde o recebimento do XML até a integração do lançamento contábil
no ERP.

### Atores e sistemas externos

  -----------------------------------------------------------------------
  Elemento externo                    Interação com a plataforma
  ----------------------------------- -----------------------------------
  Usuário da Web UI                   Envia e acompanha documentos
                                      fiscais por meio da interface web.

  Diretório/SFTP de entrada           Disponibiliza arquivos para
                                      ingestão automatizada pelo
                                      Directory Watcher / SFTP Worker.

  SEFAZ Web Services                  Recebe solicitações de validação
                                      fiscal via SOAP com autenticação
                                      mTLS.

  Sistema de Compras & Contratos      Fornece pedidos de compra (PO) e
                                      dados necessários à conciliação.

  ERP Corporativo --- Livro Razão     Recebe os lançamentos contábeis
                                      aprovados pela plataforma.

  Serviço de Notificação              Recebe eventos de rejeição ou
                                      invalidação para envio de e-mail ou
                                      webhook.
  -----------------------------------------------------------------------

### Diagrama de contexto

O diagrama de contexto identifica a plataforma como o sistema central e
evidencia suas integrações externas. Os detalhes de tecnologia e
mensageria são apresentados na visão de containers.

``` mermaid
flowchart LR
    USER["Usuário"]
    SFTP["Diretório / SFTP"]
    PLATFORM["Plataforma de Processamento de NF-e"]
    SEFAZ["SEFAZ Web Services"]
    PURCHASE["Sistema de Compras & Contratos"]
    ERP["ERP Corporativo — Livro Razão"]
    NOTIF["Serviço de Notificação"]

    USER -->|"Submete e acompanha NF-e"| PLATFORM
    SFTP -->|"Disponibiliza XML"| PLATFORM
    PLATFORM <-->|"Validação fiscal"| SEFAZ
    PLATFORM <-->|"Consulta pedidos / PO"| PURCHASE
    PLATFORM -->|"Envia lançamentos"| ERP
    PLATFORM -->|"Solicita notificações"| NOTIF
```

## 4. C4 --- Nível 2: Containers

A visão de containers apresenta as principais unidades executáveis,
aplicações, bancos de dados, mecanismos de mensageria e sistemas
externos que compõem a solução.

### 4.1 Canais de ingestão

  -----------------------------------------------------------------------
  Container               Responsabilidade        Tecnologia
  ----------------------- ----------------------- -----------------------
  Web UI                  Interface para envio e  React / Next.js
                          acompanhamento de NF-e. 

  API Gateway             Ponto de entrada das    Spring Cloud Gateway
                          requisições, roteamento 
                          e aplicação de          
                          políticas de API.       

  Directory Watcher /     Monitora diretórios ou  Worker de integração;
  SFTP Worker             servidores SFTP e       tecnologia de
                          encaminha XMLs para     implementação a
                          ingestão.               definir.
  -----------------------------------------------------------------------

### 4.2 Microsserviços de backend

  -----------------------------------------------------------------------------------
  Container                           Responsabilidade        Tecnologia indicada
  ----------------------------------- ----------------------- -----------------------
  `nfe-ingestion-service`             Recebe documentos,      Java 21 / Spring Boot 3
                                      armazena o XML          
                                      original, verifica      
                                      idempotência e publica  
                                      evento de ingestão.     

  `nfe-validator-service`             Executa a validação     Java 21 / Spring Boot 3
                                      fiscal da NF-e junto à  
                                      SEFAZ, atualiza o       
                                      status e encaminha      
                                      documentos autorizados  
                                      para classificação.     

  `accounting-engine-service`         Aplica regras contábeis Java 21 / Spring Boot 3
                                      e determina a           
                                      classificação e as      
                                      contas aplicáveis.      

  `matching-reconciliation-service`   Concilia os dados da    Java 21 / Spring Boot 3
                                      NF-e com pedidos de     
                                      compra e registra       
                                      divergências.           

  `ledger-integration-service`        Integra os lançamentos  Java 21 / Spring Boot 3
                                      aprovados ao livro      
                                      razão do ERP e persiste 
                                      os dados do lote e da   
                                      auditoria.              
  -----------------------------------------------------------------------------------

### 4.3 Mensageria

A comunicação assíncrona entre etapas é realizada pelo Apache Kafka. Os
tópicos representam os eventos e filas lógicas do fluxo.

  -----------------------------------------------------------------------
  Tópico                              Finalidade
  ----------------------------------- -----------------------------------
  `nfe.ingestion.events`              Eventos de documentos recebidos e
                                      prontos para validação.

  `nfe.sefaz.validation.queue`        Tópico reservado no desenho para a
                                      etapa de validação SEFAZ. Não há
                                      conexão explícita com ele no fluxo
                                      fornecido; confirmar se será
                                      utilizado como tópico de
                                      entrada/controle da validação.

  `nfe.accounting.classification`     Documentos autorizados e aguardando
                                      classificação contábil.

  `nfe.three-way-match`               Documentos classificados e
                                      aguardando conciliação com compras.

  `nfe.processing.dlq`                Eventos que falharam criticamente
                                      ou excederam as tentativas de
                                      processamento.
  -----------------------------------------------------------------------

### 4.4 Persistência e cache

  -----------------------------------------------------------------------
  Container                           Responsabilidade
  ----------------------------------- -----------------------------------
  PostgreSQL 16 Cluster               Persistência transacional, status
                                      de processamento, regras contábeis,
                                      divergências, lotes e dados de
                                      auditoria.

  Object Storage (S3 / MinIO)         Armazenamento dos XMLs originais
                                      das NF-e.

  Redis Cluster                       Cache, controle de idempotência e
                                      apoio à prevenção de processamento
                                      duplicado.
  -----------------------------------------------------------------------

### 4.5 Sistemas externos

  -----------------------------------------------------------------------
  Sistema                 Protocolo / integração  Finalidade
  ----------------------- ----------------------- -----------------------
  SEFAZ Web Services      SOAP com mTLS           Validação fiscal e
                                                  consulta de autorização
                                                  da NF-e.

  Sistema de Compras &    Integração a definir    Consulta de pedidos de
  Contratos                                       compra (PO) e
                                                  informações contratuais
                                                  para conciliação.

  ERP Corporativo ---     Integração a definir    Recebimento dos
  Livro Razão                                     lançamentos contábeis.

  Serviço de Notificação  E-mail / Webhook        Comunicação de
                                                  documentos inválidos ou
                                                  rejeitados.
  -----------------------------------------------------------------------

### Diagrama de containers e fluxo

``` mermaid
flowchart TD
    subgraph Ingestao ["Canais de Ingestão de NF-e"]
        UI["Web UI (React/Next.js)"]
        API_GW["API Gateway (Spring Cloud Gateway)"]
        SFTP_BOT["Directory Watcher / SFTP Worker"]
    end

    subgraph Messaging ["Mensageria & Filas (Apache Kafka)"]
        TOPIC_INGEST["nfe.ingestion.events"]
        TOPIC_SEFAZ["nfe.sefaz.validation.queue"]
        TOPIC_ACCOUNTING["nfe.accounting.classification"]
        TOPIC_MATCHING["nfe.three-way-match"]
        TOPIC_DLQ["nfe.processing.dlq"]
    end

    subgraph Services ["Microsserviços de Backend (Java 21 / Spring Boot 3)"]
        SVC_INGEST["nfe-ingestion-service"]
        SVC_VALIDATOR["nfe-validator-service (SEFAZ Engine)"]
        SVC_ACCOUNTING["accounting-engine-service"]
        SVC_MATCHING["matching-reconciliation-service"]
        SVC_LEDGER["ledger-integration-service"]
    end

    subgraph DataStorage ["Persistência & Cache"]
        PG[("PostgreSQL 16 Cluster\n(Transacional & Auditoria)")]
        S3[("Object Storage (S3 / MinIO)\n(XMLs Originais)")]
        REDIS[("Redis Cluster\n(Cache & Idempotência)")]
    end

    subgraph ExternalSystems ["Sistemas Externos & Corporativos"]
        SEFAZ["SEFAZ Web Services (SOAP mTLS)"]
        ERP_PURCHASE["Sistema de Compras & Contratos"]
        ERP_LEDGER["ERP Corporativo (Livro Razão)"]
        SMTP_NOTIF["Serviço de Notificação (Email/Webhook)"]
    end

    UI -->|HTTPS / REST| API_GW
    SFTP_BOT -->|REST Ingest| API_GW
    API_GW --> SVC_INGEST

    SVC_INGEST -->|Salva XML| S3
    SVC_INGEST -->|Verifica Idempotência| REDIS
    SVC_INGEST -->|Publica Evento| TOPIC_INGEST

    TOPIC_INGEST --> SVC_VALIDATOR
    SVC_VALIDATOR -->|mTLS / SOAP| SEFAZ
    SVC_VALIDATOR -->|Atualiza Status| PG
    SVC_VALIDATOR -->|Se Inválida / Rejeitada| SMTP_NOTIF
    SVC_VALIDATOR -->|Se Autorizada| TOPIC_ACCOUNTING

    TOPIC_ACCOUNTING --> SVC_ACCOUNTING
    SVC_ACCOUNTING <-->|Regras & Contas| PG
    SVC_ACCOUNTING -->|Classificada| TOPIC_MATCHING

    TOPIC_MATCHING --> SVC_MATCHING
    SVC_MATCHING <-->|Consulta Pedido / PO| ERP_PURCHASE
    SVC_MATCHING -->|Grava Divergência| PG
    SVC_MATCHING -->|Valor Conferido| SVC_LEDGER

    SVC_LEDGER -->|Lançamento Razão| ERP_LEDGER
    SVC_LEDGER -->|Persiste Lote/Auditoria| PG

    SVC_VALIDATOR -.->|Falha Crítica / Exaustão| TOPIC_DLQ
```

## 5. Fluxos principais

### 5.1 Ingestão da NF-e

1.  O usuário envia o XML pela Web UI, ou o worker identifica um arquivo
    no diretório/SFTP.
2.  A requisição é encaminhada ao API Gateway por HTTPS/REST.
3.  O `nfe-ingestion-service` recebe o documento.
4.  O XML original é salvo no Object Storage.
5.  O serviço verifica no Redis se o documento já foi
    recebido/processado, aplicando a chave de idempotência definida.
6.  Após a ingestão, publica um evento em `nfe.ingestion.events`.

### 5.2 Validação fiscal

1.  O `nfe-validator-service` consome eventos de `nfe.ingestion.events`.
2.  O serviço realiza a comunicação com a SEFAZ por SOAP com mTLS.
3.  O resultado da validação é persistido no PostgreSQL.
4.  Documentos inválidos ou rejeitados acionam o Serviço de Notificação.
5.  Documentos autorizados são publicados em
    `nfe.accounting.classification`.
6.  Falhas críticas ou esgotamento das tentativas devem ser encaminhados
    para `nfe.processing.dlq`.

### 5.3 Classificação contábil

1.  O `accounting-engine-service` consome documentos autorizados.
2.  Consulta regras e contas armazenadas no PostgreSQL.
3.  Executa a classificação contábil.
4.  Publica o resultado em `nfe.three-way-match`.

### 5.4 Conciliação de três vias

1.  O `matching-reconciliation-service` consome documentos
    classificados.
2.  Consulta o Sistema de Compras & Contratos para obter pedido de
    compra (PO) e dados relacionados.
3.  Compara as informações da NF-e com os dados de compra.
4.  Registra divergências no PostgreSQL.
5.  Quando os valores são conferidos, encaminha o processamento ao
    `ledger-integration-service`.

### 5.5 Integração contábil

1.  O `ledger-integration-service` recebe os documentos conciliados.
2.  Envia o lançamento ao ERP Corporativo --- Livro Razão.
3.  Persiste informações do lote e da auditoria no PostgreSQL.

## 6. Requisitos arquiteturais transversais

Os itens abaixo são diretrizes a serem detalhadas e validadas na
implementação:

-   **Segurança:** HTTPS nas APIs; mTLS na comunicação com a SEFAZ;
    gestão segura de certificados, credenciais e segredos.
-   **Idempotência:** definir uma chave estável por NF-e, considerando
    sua chave de acesso, e garantir comportamento idempotente nos
    consumidores e nas integrações.
-   **Resiliência:** estabelecer timeout, retry com backoff, circuit
    breaker e política de encaminhamento à DLQ.
-   **Rastreabilidade:** propagar um identificador de correlação entre
    API Gateway, microsserviços, eventos Kafka e integrações externas.
-   **Auditoria:** registrar eventos relevantes, alterações de status,
    resultados de validação, classificação, conciliação e lotes enviados
    ao ERP.
-   **Observabilidade:** disponibilizar logs estruturados, métricas e
    tracing distribuído.
-   **Versionamento de eventos:** definir contratos versionados para
    mensagens Kafka e estratégia de compatibilidade.
-   **Consistência:** definir estratégia para publicação confiável de
    eventos após gravações no banco, avaliando o padrão Transactional
    Outbox.
-   **Retenção:** estabelecer políticas de retenção para XMLs, eventos
    Kafka, logs e dados de auditoria.

## 7. Pontos de atenção e decisões pendentes

1.  **Tópico `nfe.sefaz.validation.queue`:** está declarado no diagrama,
    mas não participa das conexões. Confirmar se será utilizado ou se a
    validação consumirá diretamente `nfe.ingestion.events`.
2.  **DLQ:** o diagrama mostra o `nfe-validator-service` publicando
    falhas na DLQ. Definir também o tratamento de falhas nos demais
    consumidores e integrações.
3.  **Conciliação:** esclarecer se a comparação será estritamente de
    três vias (NF-e, pedido de compra e recebimento/entrada de
    mercadoria) e quais fontes fornecerão cada conjunto de dados.
4.  **Integração com ERP e Compras:** definir protocolos, autenticação,
    contratos, limites, timeouts e mecanismos de reprocessamento.
5.  **Persistência:** detalhar o modelo de dados, particionamento,
    índices, política de backup, recuperação e alta disponibilidade do
    PostgreSQL.
6.  **Object Storage:** definir criptografia, controle de acesso,
    versionamento, retenção e política de imutabilidade dos XMLs.
7.  **Redis:** definir TTL, política de expiração e comportamento em
    indisponibilidade, evitando que o cache seja a única fonte de
    verdade para impedir duplicidade.
8.  **Kafka:** definir partições, replication factor, estratégia de
    chaveamento, ordenação necessária e política de retry.
9.  **Orquestração:** definir como será controlado o estado global do
    processamento e como ocorrerá a retomada após falhas parciais.

## 8. Limites do documento

Este documento representa uma visão lógica inicial baseada no diagrama
fornecido. Ele não especifica infraestrutura física, topologia de rede,
dimensionamento, implantação, contratos de API, schemas de eventos,
modelo relacional ou diagramas internos de componentes.

Para completar o modelo C4, recomenda-se produzir diagramas de
**Componentes (Nível 3)** para os serviços com maior complexidade ---
especialmente validação fiscal, classificação contábil e conciliação ---
e, se necessário, diagramas de código (Nível 4) para módulos críticos.
