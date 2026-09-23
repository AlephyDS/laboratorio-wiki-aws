## 🗺 Quest 1: O Mapa dos Arquivos Perdidos

### 1. Inventário de Formatos e Natureza dos Documentos
A análise da pasta `raw/` revelou três arquivos com estruturas e exigências de processamento completamente distintas:
*   **`ata_reuniao_vendas_sa.pdf` (Documento Digital Nativo):** Possui 5 páginas. Como já possui uma camada de texto digitalizada nativa, não requer a aplicação de OCR, permitindo uma extração direta de texto via código.
*   **`ata_resultados_vendas_novos_dados.png` (Documento Escaneado/Imagem):** Composto puramente por matrizes de pixels. Exige obrigatoriamente a aplicação de OCR para que os caracteres visuais sejam traduzidos em texto processável.
*   **`vendas_sa_dados_ficticios_laboratorio.csv` (Dados Estruturados):** Contém 240 oportunidades de CRM distribuídas em 19 colunas. Não se comporta como texto corrido, mas sim como uma tabela que demanda agregação e lógica relacional.

### 2. Desafios Mapeados no Terreno
*   **Documentos Longos:** O PDF de 5 páginas pode diluir contextos específicos se analisado de forma linear e inteiriça.
*   **Qualidade da Imagem e Anotações:** O arquivo PNG corre o risco de apresentar baixa resolução, textos manuscritos, anotações soltas ou desalinhadas nas margens.
*   **Tabelas e Lógica Estruturada:** O arquivo CSV perderia totalmente o sentido analítico se fosse concatenado como uma única string corrida de texto para leitura de uma inteligência artificial convencional.

### 3. Informações Críticas para Extração
O pipeline foi projetado para minerar as seguintes entidades de negócio do texto:
*   *Datas das reuniões, Participantes chaves, Temas centrais discutidos, Decisões tomadas, Responsáveis por ações, Prazos estipulados, Riscos identificados, Próximos passos/Pendências, Projetos citados e Departamentos envolvidos*.

### 4. Classificação sem Subpastas (S3 Object Tagging)
Como a regra de negócio impede a criação de subpastas na raiz de `raw/`, a classificação e a segregação lógica serão feitas usando **S3 Object Tagging**. No momento da ingestão ou triagem, o sistema injeta chaves de metadados no objeto S3 (Ex: `Key=DocType, Value=AtaDigital`, `Key=DocType, Value=AtaImagem` ou `Key=DocType, Value=CRMTabela`). Isso permite aplicar regras de ciclo de vida e permissões do IAM sem alterar a estrutura de arquivos flat.

---

## 🚀 Quest 2: O Portal de Entrada na AWS

### 1. Ingestão e Preservação dos Originais (Amazon S3)
Os arquivos locais da pasta `raw/` são transmitidos para um bucket central do **Amazon S3** chamado `wiki-corporativa-raw-data`. 
*   **Preservação:** O bucket possui o **S3 Versioning** ativado para evitar deleções acidentais e políticas de **Bucket Policy (IAM)** restritas que impedem qualquer modificação posterior nos arquivos originais. Os dados de auditoria permanecem imutáveis.

### 2. Pipeline de Processamento Orquestrado (AWS Step Functions)
Todo o fluxo de ingestão, tomada de decisão baseada no tipo de arquivo e tratamento de falhas é governado pelo **AWS Step Functions**. O upload de um arquivo gera um evento que inicia a máquina de estados:

[Arquivo Ingerido no S3]│▼[AWS Step Functions] ── (Identifica Extensão e Formato)│├──► Se .png ──────────────► [Amazon Textract] ──────────────┐│                                                            ▼├──► Se .pdf (Texto) ──────► [AWS Lambda (Direct Parse)] ──► [Texto Bruto Normalizado]│                                                            ▲└──► Se .csv (Tabular) ────► [AWS Glue Crawler & Catalog] ───┘

*   **Identificação do OCR:** Uma função **AWS Lambda** de triagem analisa os metadados do arquivo. Se a extensão for `.png`, o fluxo o classifica como documento que necessita de OCR.
*   **Tratamento do PNG (Amazon Textract):** O Step Functions envia o arquivo `.png` para o **Amazon Textract**. O Textract analisa a imagem e extrai o texto bruto contido nas linhas, blocos e possíveis tabelas desenhadas.
*   **Tratamento do PDF Nativo:** O arquivo `.pdf` é direcionado para uma função **AWS Lambda** dedicada que utiliza bibliotecas de extração direta de texto (como PyPDF ou PDFMiner). O texto é extraído de forma rápida e barata, sem consumir recursos de OCR do Textract.
*   **Tratamento do CSV (AWS Glue):** Sendo uma tabela estruturada, o arquivo `.csv` ignora a extração textual tradicional. O Step Functions aciona um **AWS Glue Crawler** que faz a leitura das 19 colunas e popula automaticamente o **AWS Glue Data Catalog**. A partir daí, o arquivo fica disponível para consultas relacionais robustas via SQL utilizando o **Amazon Athena**.

### 3. Armazenamento Intermediário e Registro de Erros
*   Os textos puros extraídos das atas (PDF e PNG) são gravados de forma padronizada em formato JSON em um bucket intermediário chamado `wiki-corporativa-processed-data`.
*   **Tratamento de Falhas:** Caso ocorra algum erro (arquivo corrompido, falha de OCR ou timeout), o bloco `Catch` do Step Functions desvia o fluxo, move o registro do arquivo com erro para a pasta lógica `failed/` dentro do S3 e publica alertas críticos no **Amazon CloudWatch**, registrando os logs de erro para auditoria.

---

## 🏛 Quest 3: A Relíquia dos Metadados

### 1. Limpeza, Normalização e Formato Padronizado
Os textos extraídos do Textract e do parsing do PDF são consolidados por uma função **AWS Lambda** de higienização. Esta função limpa ruídos de quebra de página, hifens órfãos causados pela justificação do texto, espaços duplicados e cabeçalhos repetitivos. O resultado final é salvo em um arquivo JSON padronizado com o esquema: `{"document_id": "UUID", "content": "Texto limpo...", "technical_metadata": {...}}`.

### 2. Enriquecimento de Metadados via IA (Amazon Bedrock)
Para extrair inteligência dos textos desorganizados, o AWS Lambda faz uma chamada assíncrona para o **Amazon Bedrock**, utilizando uma LLM avançada (como o Claude). O modelo recebe o texto limpo com instruções estritas para extrair as seguintes entidades e estruturá-las em um esquema de metadados de negócio:

*   `Nome do documento`: Mapeado a partir do arquivo original (Ex: `ata_reuniao_vendas_sa.pdf`).
*   `Tipo de documento`: Classificado pela IA (Ex: Ata de Reunião, Relatório de Resultados).
*   `Data identificada`: Extraída do contexto da ata.
*   `Tema principal / Participantes`: Resumo do assunto e lista das pessoas presentes.
*   `Decisões tomadas / Responsáveis / Próximos passos`: Mapeamento cirúrgico de planos de ação.
*   `Nível de confidencialidade`: Classificação automática com base na sensibilidade dos termos (Ex: Interno, Confidencial).

### 3. Armazenamento e Rastreabilidade (Amazon DynamoDB)
Esses metadados estruturados e higienizados pela IA são armazenados no **Amazon DynamoDB**, uma tabela NoSQL de baixa latência. Cada linha do DynamoDB contém o campo chave `s3_original_filepath` que armazena a URI exata do documento original no S3 (Ex: `s3://wiki-corporativa-raw-data/ata_reuniao_vendas_sa.pdf`). Isso vincula de forma definitiva os metadados de inteligência ao arquivo bruto original.

---

## 🔮 Quest 4: O Oráculo da Wiki Inteligente

### 1. Estratégia de Chunking e Geração de Embeddings
Para que os documentos textuais possam ser consultados via busca semântica, eles são configurados no **Amazon Bedrock Knowledge Bases**:
*   **Chunking (Divisão):** O texto limpo é fatiado em blocos menores (*chunks*) de tamanho fixo (ex: 512 tokens) com uma sobreposição (*overlap*) de 10% a 20%. Essa sobreposição assegura que informações localizadas na divisão de duas páginas ou blocos não percam o contexto semântico.
*   **Embeddings:** Cada *chunk* gerado é submetido ao modelo **Amazon Titan Text Embeddings** através do Bedrock, que traduz o significado do texto em vetores matemáticos de alta densidade.

### 2. Base Vetorial e Mecanismo de Busca Semântica
Os vetores numéricos de contexto são armazenados em uma coleção de busca vetorial no **Amazon OpenSearch Serverless**. 
*   Quando o usuário submete uma pergunta em linguagem natural (Ex: *“Quais foram as principais decisões tomadas sobre o projeto de expansão comercial?”*), essa pergunta é convertida em vetor pelo mesmo modelo de embeddings.
*   O OpenSearch Serverless realiza uma busca por similaridade de vetores (K-Nearest Neighbors via distância de cosseno) e extrai os fragmentos textuais (*chunks*) que possuem o significado mais próximo da dúvida do usuário, ignorando a necessidade de correspondência de palavras-chave exatas.

### 3. Geração de Respostas com RAG (Retrieval-Augmented Generation) e Citações
Os *chunks* textuais recuperados pelo OpenSearch são envelopados em um prompt enriquecido e enviados ao modelo de fundação no **Amazon Bedrock**. O modelo atua sob o fluxo de **RAG**: ele consolida uma resposta fluida e natural baseada exclusivamente nos trechos de documentos fornecidos como contexto.

*   **Rastreabilidade e Citações:** O prompt exige que a LLM cite os IDs das fontes fornecidas. A aplicação consome essa referência e cruza com a tabela do DynamoDB, exibindo para o usuário final o link para o arquivo original no S3, o nome do documento, as datas mapeadas e os responsáveis associados.
