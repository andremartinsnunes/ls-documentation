# Documentação Técnica — Chatbot Corporativo GenAI  
**Grupo Luz Saúde**

---

## 1. Introdução

### Contexto
O Grupo Luz Saúde reconhece a crescente importância e potencial impacto das soluções de inteligência artificial (IA) generativa no seu contexto operacional, pretendendo iniciar um projeto centrado na implementação de um chatbot corporativo, assente sobre uma Frameworkde Gen AI agnóstica eescalável. Esta solução visa disponibilizar um chatbot para os colaboradores do Grupo Luz Saúde que permita aceder a qualquer informação existente na base de conhecimento definida e explorá-la através de linguagem natural.

### Âmbito Técnico
A solução implementada consiste num sistema RAG para a documentação interna do Grupo Luz Saúde. Este sistema é baseado na Framework GenAI Closer, Maestro, com recurso ao LLM Gemini e implementado em Google Cloud. Os ficheiros são retirados do MS Sharepoint e transformados numa base de dados vetorial, armazenada no Mongo DB. O chatbot poderá ser acedido em qualquer browser habitual, tendo o front-end sido desenvolvido em React.

### Componentes Principais
- **Componente A:** Chatbot base baseado no Gemini 2.5 Pro  
- **Componente B:** Sistema de RAG com a documentação interna do Grupo Luz Saúde

---

## 2. Arquitetura Geral

### Visão Técnica
Descrever a estrutura do sistema, as camadas e as interações entre microserviços.

### Tópicos a abordar

- **Estrutura em Camadas**  
  O sistema está organizado em múltiplas camadas lógicas e funcionais, garantindo modularidade, escalabilidade e isolamento de responsabilidades:  
    - **Front-end:** interface web para interação com o utilizador. 
    - **LLM:** gere a interação com o Large Language Model (LLM) Gemini, utilizado na seleção de documentos e geração da resposta.
    - **Vector Database:** cria e armazena os embedding e metadados dos ficheiros selecionados e realiza a pesquisa híbrida (semântica + vetorial) dos documentos relevantes para a pergunta do utilizador.
    - **Root Container:** gera a imagem do base container com todos os requisitos necessários para o projeto.
    - **Back-end:** gere a lógica da conversa, processa as mensagens do utilizador e interage com os diferentes serviços (ex.: MongoDB, Gemini)

- **Comunicação entre Serviços**  
  A comunicação entre microserviços e componentes é efetuada através de:
  - **Azure Queue Storage** — utilizado para o envio e gestão de tarefas assíncronas (OCR, análise, notificações).
  - **HTTP** — utilizado nas interações diretas entre a **API**, o **Backoffice** e serviços externos (OCR, LLMs, notificações).  

- **Infraestrutura em Nuvem**  
  Todo o sistema está implementado na **Azure Cloud**, com os seguintes componentes principais:
  - **Azure Web Apps** — alojamento dos serviços de API e Backoffice.  
  - **Azure Queue Storage** — orquestração de tarefas e mensagens assíncronas.  
  - **Azure Blob Storage** — armazenamento de ficheiros originais e resultados intermédios (OCR e extração).  
  - **MongoDB Atlas** — base de dados central para documentos, logs, resultados e auditoria.  
  - **Redis Cache (Azure Cache for Redis)** — ??

- **Observabilidade e Rastreabilidade**  
  A monitorização e rastreabilidade são garantidas por um conjunto de práticas e ferramentas integradas:
  - **OpenTelemetry** — rastreamento distribuído de requests e métricas de performance.  
  - **Azure Monitor / Application Insights** — recolha centralizada de logs e métricas.  


- **Diagrama Geral da Arquitetura**  
  Será incluído posteriormente um diagrama técnico que ilustra:
  - o fluxo completo de processamento (**Upload → OCR/PdfParser/Docx → Classificação → Análise → Notificação**),  
  - a interação entre serviços (**API**, **Workers**, **MongoDB**, **Azure Services**),  
  - e os principais pontos de monitorização e rastreabilidade.


---

## 3. Tech Stack

### Tecnologias Utilizadas
> *(Slide 22 - proposta comercial)*

| Componente | Tecnologias |
|-------------|------------|
| Frontend | React |
| Embeddings and RAG libraries | LangGraph, LangChain, VertexAI | 
| Backend and Model access | FastAPI, LangChain, Ollama, Huggingface |
| Data and Retrieval | Azure AI Search, CosmosDB, Azure Storage Accounts, MongoDB, ChromaDB, PGVector |
| Large Language Models | Gemini |
| Infrastructure | AzureDevOps, GitHub |

### Principais Componentes
| Componente | Descrição |
|-------------|------------|
| Web API | Recebe documentos e inicia o processamento |
| Document Validator | Valida o tipo, formato e permissões |
| Content Extractor | Extrai texto via OCR, parser de PDF ou leitura direta de ficheiros DOCX (utilizando a biblioteca **docx2txt**) |
| Document Classifier | Classifica o documento com base no conteúdo textual |
| Document Analyzer | Analisa o conteúdo baseado em regras específicas de conformidade e contexto |
| Notification Service | Envia resultados via webhook |
| Dashboard | Interface de monitorização |

### Comunicação entre Microserviços
Incluir tabela com endpoints e formato de requests/responses.

---

## 4. Componente A — Classificação de Documentos

### Descrição Técnica
- Fluxo: **Upload → Validação → OCR/PDFParser/Docx → Classificação**  
- Ferramentas: **Azure Vision**, **LangChain**, **GPT-4.1**, etc.  
- Otimização do OCR: limitação de chamadas e cache de resultados  
- Estrutura de dados e persistência  

### Principais Desafios e Evolução
Inicialmente, verificou-se dificuldades na leitura de determinados documentos devido a:
- utilização de **fontes personalizadas** nos PDFs (ainda pode ocorrer se o OCR não reconhecer a fonte),  
- documentos com texto não selecionável (digitalizados).  
- Casos recorrentes de falha/`classification = null` foram identificados em documentos que exigem credenciais para abrir (conteúdo inacessível ao OCR) ou que contêm páginas em branco; ao validar incidentes, verificar primeiro se o ficheiro está protegido ou vazio antes de reprocessar.
- Documentos muito extensos (~200/300 páginas) podem demorar substancialmente mais tempo; em alguns casos só se obtém a classificação dentro da janela de execução e a análise fica `null` por timeout, exigindo reprocessamento ou aumento de tempo limite.

Para resolver estas limitações, o pipeline foi ajustado de forma a (cobrindo a maioria dos casos; o que restar depende tipicamente do lado do utilizador, como credenciais ou ficheiros vazios):
- efetuar **múltiplas chamadas ao OCR** com diferentes configurações linguísticas e de precisão,  
- aplicar **pré-processamento de imagem** para aumentar a taxa de sucesso.  

Estas melhorias resultaram numa **maior fiabilidade da leitura e classificação**,  
mas também num **aumento do número de chamadas OCR**, o que implica **custos superiores de processamento** em comparação com a versão inicial.

---

## 4.1. Tipologia de Documentos

O sistema foi concebido para **classificar e analisar automaticamente diferentes tipos de documentos institucionais** no âmbito do **Regime Geral de Prevenção da Corrupção (RGPC)**.  

Cada tipologia segue regras e critérios próprios de conformidade, definidos pelo **Mecanismo Nacional Anticorrupção (MENAC)**.  

### Tipos de Documentos Suportados

| Código | Designação Completa | Descrição |
|---------|--------------------|------------|
| **PPR** | Plano de Prevenção de Riscos de Corrupção e Infrações Conexas | Documento estratégico que define riscos identificados e medidas preventivas a adotar pela entidade. |
| **CDE** | Código de Conduta | Define princípios éticos, comportamentais e de integridade aplicáveis a todos os colaboradores e dirigentes. |
| **Formação** | Formação e Comunicação | Documento que descreve as ações de formação e comunicação interna relacionadas com ética e anticorrupção. |
| **RL-CDE** | Relatório do Código de Conduta | Relatório anual que demonstra o cumprimento, atualização e comunicação do Código de Conduta. |
| **RL-PPR-INTERCALAR** | Relatório intercalar do Plano de Prevenção de Riscos, focado em avaliação pontual (abril/outubro), medidas implementadas e riscos elevados/máximos. |
| **RL-PPR-ANUAL** | Relatório anual consolidado de execução do PPR, com avaliação global do ano civil, síntese de medidas, atualização de matriz de risco e propostas de revisão. |

### Observações
- Cada documento é classificado de forma **automática** com base no conteúdo textual extraído e analisado pelo **modelo de classificação**.  
- A classificação serve como **etapa prévia obrigatória** para o módulo de **análise de conformidade**, que aplica as regras específicas de cada tipologia.  

---------?????-------------------
- As tipologias estão **parametrizadas** na base de dados e podem ser **atualizadas** via configuração sem alterar o código da aplicação.  

## 4.2. Fluxo End-to-End (Resumo Operacional)

1) API recebe o ficheiro → valida tipo/tamanho → rejeita PDFs protegidos por password ou ficheiros vazios.  
2) Extração: OCR/PDF parser/Docx → se falhar por ficheiro protegido ou página em branco, sinalizar e não seguir para classificação.  
3) Classificação (LLM): usa prompt configurado em `prompts/classifier/variables.yaml`.  
4) Análise de compliance (LLM): baseada no tipo inferido, usando `prompts/analyzer_*/variables.yaml`.  
5) Persistência: resultados guardados em MongoDB (`classification`, `analysis`, `full_text`).  
6) Notificação.  
Timeouts/retries: documentos longos (200/300+ páginas) podem exceder o tempo de análise; a classificação pode concluir e a análise ficar `null`. Reprocessar ou aumentar o timeout.

## 4.3. Validação de Ficheiros e Erros Comuns

- PDFs com password/credenciais: OCR não lê → classificação/análise `null`; pedir ficheiro sem proteção.  
- Limite de páginas: documentos muito extensos podem exigir reprocessamento; monitorizar tempos.  
- Formatos suportados: PDF/DOCX; validar mimetype e tamanho antes de enviar ao pipeline.

## 4.4. Gestão de Timeouts para Documentos Longos

- Tempo limite atual: <preencher valor> para extração/classificação/análise.  
- Sintoma: classificação preenchida e análise `null` por timeout.  
- Mitigação: reprocessar com mais tempo, dividir o documento ou pedir versão reduzida ao utilizador.

## 4.5. YAMLs de Prompts — Carregamento e Extensão
- Origem: `prompts/classifier/variables.yaml` e `prompts/analyzer_*/variables.yaml`.  
- Como o código usa: módulos Python leem estes YAMLs para montar o prompt e validar o formato de resposta.  
- Como estender: adicionar categorias/critérios mantendo as chaves existentes (`categories_description`, `response_format`, `regra_decisao`, etc.) e alinhar com o que está parametrizado na base de dados.


## 5. Componente B — Análise de Compliance

### Descrição Técnica
- Motor de regras específicas e utilização de **LLMs** para contextualização.  
- Integração com base de dados **MongoDB Atlas**.  
- Estrutura e exemplos de regras (YAML/JSON/Python).  
- Fluxo: **Classificação → Extração → Validação → Relatório.**  
- Problemas e ajustes (ambiguidade linguística, afinação de regras, etc.).  

---

### 5.1. Estrutura de Dados — Resultado de Análise

Após o processamento do documento e execução da análise de conformidade, os resultados são armazenados na base de dados **MongoDB Atlas** no seguinte formato:
```json
{
  "_id": "bda1f6f9-b2a6-48e6-ba4e-932c0ef9412c",
  "id_document": "string",
  "type_document": "formacao",
  "created_at": "2025-10-27T15:03:15.788+00:00",
  "last_modified": "2025-10-27T15:03:28.881+00:00",
  "full_text": "Plano de Formação – Ano 2024 B-22.02.01_01/24",
  "full_text_len": 269,
  "file": {},
  "pages": [{}],
  "classification": {},
  "analysis": {
    "in_compliance": false,
    "confidence": 0.7,
    "reason": "O documento apresenta um plano de formação para 2024, mas não cumpre v…",
    "suggestions": [],
    "metadata": {},
    "status": []
  }
}
```



### 5.2. Descrição do Resultado em JSON

O resultado final do processamento de cada documento é armazenado em **formato JSON**, refletindo o estado completo da operação — desde a extração de texto até à análise de conformidade.  

Esta estrutura foi concebida para garantir **rastreabilidade**, **auditoria** e **interpretação automatizada** dos resultados, tanto pela API como pelo Backoffice.

O JSON contém três blocos principais de informação:

1. **Metadados Gerais do Documento**  
   Inclui o identificador único (`_id`), tipo de documento (`type_document`), datas de criação e modificação, e o texto integral extraído (`full_text`).  
   Estes campos permitem reconstituir o histórico de cada submissão e associá-la ao ficheiro original.

2. **Resultados de Classificação**  
   A chave `classification` contém o tipo de documento inferido pelo modelo de classificação (ex.: *formação*, *PPR*, *CDE*).  
   É nesta etapa que se define o contexto do documento, que depois servirá de base à análise de compliance.

3. **Resultados de Análise de Compliance**  
   A secção `analysis` agrega toda a informação resultante da verificação das regras de conformidade:  
   - `in_compliance`: indica se o documento cumpre (`true`/`false`) os critérios definidos;  
   - `confidence`: nível de confiança (entre `0.0` e `1.0`) atribuído automaticamente pelo modelo de IA;  
   - `reason`: justificação textual para o resultado obtido;  
   - `suggestions`: lista de melhorias ou ações corretivas recomendadas para cada regra;  
   - `metadata`: detalhes técnicos sobre o processo de análise (modelo LLM, tempo de execução, prompt usada, etc.);  
   - `status`: histórico de estados e transições durante o processamento.  

Esta estrutura JSON serve como **formato padrão de resposta da API** e é também **armazenada integralmente no MongoDB Atlas**, permitindo:  
- auditoria e rastreabilidade completa de cada documento,  
- e eventual reprocessamento com novas regras ou modelos de análise.  

---

### 6. Backoffice

### Descrição Técnica
- Estrutura do módulo `service_backoffice/`  
- Funcionalidades principais
- Comunicação com a API  
- Segurança e permissões  

---

## 7. Infraestrutura e Implementação (Not done yet)

### Descrição Técnica
- **Dockerização** e orquestração dos serviços  
- **Azure Pipelines / DevOps** para CI/CD  
- Gestão de **variáveis de ambiente e configurações**  
- Monitorização e logs distribuídos  

### 7.1. Pipelines e YAMLs no diretório `azure/`
- O diretório `azure/` contém os ficheiros de pipeline que constroem e publicam cada serviço:  
  - `service-api.yml` — build/push da API e publicação em Azure App Service/Functions.  
  - `service-analyzer.yml` — pipeline do motor de análise/compliance.  
  - `service-classifier.yml` — pipeline do classificador de documentos.  
  - `service-frontend.yml` — build e deploy do front-end.  
  - `service.backoffice.yml` — pipeline do backoffice.
- Cada YAML segue o mesmo padrão: o bloco `variables` define o Dockerfile a usar, o registry, grupo de recursos, subscrição, nome da app e `tag`; as tarefas seguintes fazem build/push da imagem e deploy no serviço Azure correspondente, atualizando também as `appSettings` (ex.: `ENVIRONMENT`, strings de conexão, endpoints de serviços internos).
- Criação/troca de ambientes (dev/qa/prod) é feita alterando apenas os valores das variáveis nesses ficheiros (ou via variable groups/templates no Azure DevOps): `azResourceGroupName`, `azSubscription`, `azAppName`, `containerRegistry`, `imageRepository`, `env`, conexões a Mongo/Redis/OpenAI/OCR, URLs internas, etc. O pipeline permanece igual, apenas com valores diferentes para apontar para o ambiente desejado.

### 7.2. YAMLs de módulos e prompts (LLM)
- O diretório `prompts/` contém os YAML usados pelos módulos de classificação e análise em runtime; cada subpasta representa um fluxo (`classifier`, `analyzer_plano`, `analyzer_formacao`, `analyzer_codigo`, etc.).  
- Cada `variables.yaml` define o contexto enviado ao LLM: categorias, formato de resposta esperado e regras de decisão/critério. Exemplos: o classificador descreve cada categoria e o JSON de saída; o analyzer lista critérios de compliance e a regra global (`regra_decisao`) que determina `in_compliance`.
- Para ajustar lógica (novas categorias, critérios ou campos de saída), edite apenas estes YAMLs mantendo as mesmas chaves esperadas pelo código Python que monta os prompts dinamicamente.
---

## 8. Testes e Qualidade

### Descrição Técnica
- Estrutura de testes (`tests/`)  
- Ferramentas: `pytest`, `coverage`  
- Tipos de teste: unitário, integração, carga  
- Métricas e resultados esperados  

--- 