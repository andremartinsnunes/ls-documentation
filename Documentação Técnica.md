# Documentação Técnica — Chatbot Corporativo GenAI  
**Grupo Luz Saúde**

---

## 1. Introdução

### Contexto
O Grupo Luz Saúde reconhece a crescente importância e potencial impacto das soluções de inteligência artificial (IA) generativa no seu contexto operacional, pretendendo iniciar um projeto centrado na implementação de um chatbot corporativo, assente sobre uma Frameworkde Gen AI agnóstica eescalável. Esta solução visa disponibilizar um chatbot para os colaboradores do Grupo Luz Saúde que permita aceder a qualquer informação existente na base de conhecimento definida e explorá-la através de linguagem natural.

### Âmbito Técnico
A solução implementada consiste num sistema RAG para a documentação interna do Grupo Luz Saúde. Este sistema é baseado na Framework GenAI Closer, Maestro, com recurso ao LLM Gemini e implementado em Google Cloud. Os ficheiros são armazenados num bucket no GCP, de onde são retirados e transformados numa base de dados vetorial, armazenada no Mongo DB. O chatbot poderá ser acedido em qualquer browser habitual, tendo o front-end sido desenvolvido em React.

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
  - **HTTP** — utilizado nas interações diretas entre os vários repositórios e serviços externos (OCR, LLMs, notificações).  

- **Infraestrutura em Nuvem**
  Todo o sistema está implementado com recurso a três infrastruturas em nuvem:
    **Google Cloud Platform** — construção e deploy da aplicação; armazenamento de ficheiros.
  - **Azure DevOps** — repositório de código e definição de pipelines. 
  - **MongoDB Atlas** — base de dados central para documentos e embeddings.

- **Observabilidade e Rastreabilidade**  
  A monitorização e rastreabilidade são garantidas por um conjunto de práticas e ferramentas integradas:
  - **Looker** — rastreamento distribuído de requests e métricas de performance.  
  - **Cloud Logging** — recolha centralizada de logs e métricas.  


- **Diagrama Geral da Arquitetura**  
  Será incluído posteriormente um diagrama técnico que ilustra:
  - o fluxo completo de processamento,  
  - a interação entre serviços,  
  - e os principais pontos de monitorização e rastreabilidade.


---

## 3. Tech Stack

### Tecnologias Utilizadas

| Componente | Tecnologias |
|-------------|------------|
| Frontend | React |
| Embeddings and RAG libraries | LangGraph, LangChain, VertexAI | 
| Backend and Model access | FastAPI, LangChain, Ollama, Huggingface |
| Data and Retrieval | Azure AI Search, CosmosDB, Azure Storage Accounts, MongoDB, ChromaDB, PGVector |
| Large Language Models | Gemini |
| Infrastructure | AzureDevOps, GitHub |

### Comunicação entre Microserviços
Incluir tabela com endpoints e formato de requests/responses.

---

## 4. Componente A — Chatbot

### Descrição Técnica
- Fluxo: **Pergunta do utilizador → Construção de prompt → Comunicação com o LLM → Resposta**  
- Ferramentas: **GCP**, **Vertex AI**, **Gemini 2.5 Pro**, etc.

### Principais Desafios e Evolução

---

## 4.1. Fluxo End-to-End (Resumo Operacional)

1) O utilizador faz uma pergunta no front-end.
2) É construída uma prompt para envio para o LLM com indicações para a reposta e histórico de conversas.
3) Comunicação com o LLM para obtenção de resposta. 
4) Envio da resposta ao utilizador através do front-end.

## 4.2. Ficheiros de Prompts — Carregamento e Extensão
- Origem: `\LS-chatbot-backend\app\orchestrators\resources\prompts`.
- Ficheiros: `prompt_rag.xml` e `prompt_relevant_docs.xml`
- Como o código usa: módulos Python leem estes ficheiros para montar o prompt e validar o formato de resposta.

## 5. Componente B — Sistema de RAG

### Descrição Técnica
- Sistema de RAG para obtenção de informação de ficheiros internos do Grupo Luz Saúde.
- Integração com base de dados **MongoDB Atlas**. 
- Fluxo: **Pergunta do utilizador → Pesquisa híbrida de informação relevante → Filtragem dos chunks obtidos → Construção de Prompt → Comunicação com o LLM → Resposta com base na informação obtida.**

## 5.1. Fluxo End-to-End (Resumo Operacional)

1) O utilizador faz uma pergunta no front-end.
2) Com base na pergunta, é feita uma pesquisa híbrida (vetorial + semântica) na base de dados do MongoDB para encontrar os chunks de documentos com informação relevante à pergunta.
3) Os chunks selecionados são avaliados com recurso a um LLM, que faz uma segunda seleção dos mesmos.
4) É construída uma prompt para envio para o LLM com instruções para a reposta, a pergunta do utilizador, histórico de conversas e os chunks selecionados.
5) Comunicação com o LLM para obtenção de resposta. 
6) Envio da resposta ao utilizador através do front-end.

## 5.2. Tipologia de Documentos

### Tipos de Documentos Suportados

1) PDF
2) docx
3) pptx
4) csv

### Bases de dados

Os ficheiros são guardados em duas bases de dados distintas, com objetivos diferentes.
  - **GCP Bucket:** os documentos são inicialmente guardados num bucket do Google Cloud Platform, que é gerido pela equipa do Grupo Luz Saúde.
  - **MongoDB:** os documentos no bucket do GCP são divididos em chunks de 500 palavras com overlap de 100 palavras. Os chunks são guardados na base de dados do Mongo DB como texto e como vetor após o processo de embedding.

### Principais Desafios e Evolução

---

## 6. Infraestrutura e Implementação (Not done yet)

### Descrição Técnica
- **Dockerização** e orquestração dos serviços  
- **Azure DevOps** para CI/CD  
- Gestão de **variáveis de ambiente e configurações**  
- Monitorização e logs distribuídos  

### 6.1. Pipelines e YAMLs no diretório `azure/`
- O diretório `azure/` contém os ficheiros de pipeline que constroem e publicam cada serviço:  
  - `service-api.yml` — build/push da API e publicação em Azure App Service/Functions.  
  - `service-analyzer.yml` — pipeline do motor de análise/compliance.  
  - `service-classifier.yml` — pipeline do classificador de documentos.  
  - `service-frontend.yml` — build e deploy do front-end.  
  - `service.backoffice.yml` — pipeline do backoffice.
- Cada YAML segue o mesmo padrão: o bloco `variables` define o Dockerfile a usar, o registry, grupo de recursos, subscrição, nome da app e `tag`; as tarefas seguintes fazem build/push da imagem e deploy no serviço Azure correspondente, atualizando também as `appSettings` (ex.: `ENVIRONMENT`, strings de conexão, endpoints de serviços internos).
- Criação/troca de ambientes (dev/qa/prod) é feita alterando apenas os valores das variáveis nesses ficheiros (ou via variable groups/templates no Azure DevOps): `azResourceGroupName`, `azSubscription`, `azAppName`, `containerRegistry`, `imageRepository`, `env`, conexões a Mongo/Redis/OpenAI/OCR, URLs internas, etc. O pipeline permanece igual, apenas com valores diferentes para apontar para o ambiente desejado.

### 6.2. YAMLs de módulos e prompts (LLM)
- O diretório `prompts/` contém os YAML usados pelos módulos de classificação e análise em runtime; cada subpasta representa um fluxo (`classifier`, `analyzer_plano`, `analyzer_formacao`, `analyzer_codigo`, etc.).  
- Cada `variables.yaml` define o contexto enviado ao LLM: categorias, formato de resposta esperado e regras de decisão/critério. Exemplos: o classificador descreve cada categoria e o JSON de saída; o analyzer lista critérios de compliance e a regra global (`regra_decisao`) que determina `in_compliance`.
- Para ajustar lógica (novas categorias, critérios ou campos de saída), edite apenas estes YAMLs mantendo as mesmas chaves esperadas pelo código Python que monta os prompts dinamicamente.
---

## 7. Testes e Qualidade
 

--- 