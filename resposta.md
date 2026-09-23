Transformação de documentos em uma base de conhecimento pesquisável usando serviços AWS
1. Visão geral do projeto
O projeto tem como objetivo transformar arquivos espalhados, como PDFs, imagens e planilhas, em
uma base de conhecimento organizada e pesquisável. A ideia final é permitir que uma pessoa faça
perguntas em linguagem natural e receba respostas baseadas nos documentos da empresa.
Fluxo conceitual: Arquivos - armazenamento - processamento - extração e organização - busca
semântica - inteligência artificial - resposta com referências.
2. Amazon S3 — armazenamento
O Amazon S3 é o armazenamento principal dos arquivos. Os documentos originais devem ser
preservados, enquanto os resultados processados podem ser armazenados separadamente. Essa
separação facilita a organização, auditoria e recuperação dos arquivos.
Conceito: raw = arquivos originais; processed = conteúdo processado; metadata = informações sobre
os documentos; failed = registros de processamento que apresentaram erro.
3. AWS Step Functions — orquestração
O AWS Step Functions funciona como um orquestrador. Ele controla a sequência das etapas,
permitindo definir decisões, condições, tentativas novamente e tratamento de falhas. Em vez de cada
serviço trabalhar isoladamente, o Step Functions coordena o fluxo.
Exemplo conceitual: receber arquivo - identificar formato - escolher processamento - extrair
conteúdo - limpar - gerar metadados - indexar.
4. Amazon Textract — OCR e extração
O Amazon Textract é utilizado para extrair informações de documentos, principalmente quando o
conteúdo está em imagens ou documentos digitalizados. Um PDF que já possui uma camada de texto
pode seguir por uma extração direta, enquanto uma imagem escaneada precisa de OCR.
Ideia principal: OCR transforma o texto que está visualmente em uma imagem em informação que
pode ser processada pelo sistema.
5. AWS Lambda — processamento
O AWS Lambda permite executar pequenas funções sem manter servidores permanentemente ativos.
No projeto, pode ser utilizado para tarefas como identificar arquivos, limpar conteúdos, transformar
dados e preparar informações para as próximas etapas.
6. Metadados
Metadados são informações que descrevem o documento. Além do texto, o sistema pode registrar
data, tipo do documento, participantes, temas, decisões, responsáveis, prazos, riscos, pendências,
projetos, departamentos e caminho do arquivo original.
Essas informações ajudam a organizar a base e tornam a recuperação dos documentos mais precisa.
7. Amazon Bedrock — inteligência artificial
O Amazon Bedrock fornece acesso a modelos de inteligência artificial generativa. No projeto, ele pode
ser utilizado para interpretar conteúdos, auxiliar na extração de informações e gerar respostas em
linguagem natural.
A IA, porém, não deve depender apenas do conhecimento geral do modelo. Para responder sobre
documentos internos, o projeto utiliza o conceito de RAG.
8. RAG — Retrieval-Augmented Generation
RAG significa Retrieval-Augmented Generation. A ideia é recuperar informações relevantes de uma
base de documentos antes de pedir ao modelo de IA que produza a resposta.
Fluxo: pergunta do usuário ® busca de informações relevantes ® recuperação dos trechos ® envio do
contexto ao modelo ® geração da resposta.
Isso permite que a resposta seja baseada no conteúdo disponível na base de conhecimento, podendo
também apresentar as fontes utilizadas.
9. Embeddings e busca semântica
Embeddings são representações numéricas que capturam características semânticas de um texto.
Com eles, o sistema consegue comparar o significado de uma pergunta com o significado de trechos
armazenados.
Por exemplo, frases com palavras diferentes podem representar uma ideia semelhante. A busca
semântica procura conteúdos relacionados pelo significado, e não apenas pela correspondência exata
de palavras.
10. Chunking — divisão dos documentos
Documentos grandes normalmente são divididos em partes menores chamadas chunks. Essa divisão
facilita a criação de embeddings e permite recuperar apenas os trechos mais relevantes para cada
pergunta.
Um documento de muitas páginas pode, por exemplo, ser dividido em dezenas de trechos. Quando
uma pergunta é feita, o sistema busca os chunks relacionados ao assunto.
11. Banco vetorial
Depois que os textos são transformados em embeddings, essas representações precisam ser
armazenadas em uma estrutura que permita buscas por similaridade. No contexto do desafio, podem
ser consideradas opções como Amazon S3 Vectors, Amazon OpenSearch Serverless ou Aurora
PostgreSQL com pgvector.
12. Amazon Bedrock Knowledge Bases
O Bedrock Knowledge Bases ajuda a construir uma base de conhecimento para aplicações de RAG.
Ele pode participar do processo de preparação dos documentos, criação de embeddings,
armazenamento vetorial e recuperação de conteúdo relevante.
13. Tratamento de arquivos CSV
Arquivos CSV são diferentes de documentos textuais porque representam dados estruturados em
linhas e colunas. Para esse tipo de conteúdo, serviços como AWS Glue e Amazon Athena podem ser
utilizados para catalogar e consultar os dados.
Assim, uma pergunta que exige cálculo ou filtragem de dados estruturados pode ser tratada de maneira
diferente de uma pergunta que procura informações em atas e documentos textuais.
14. Segurança
Serviço Função teórica
IAM Controle de identidades e permissões
KMS Gerenciamento de chaves e criptografia
CloudTrail Registro e auditoria de atividades
CloudWatch Logs, métricas, alertas e monitoramento
15. Arquitetura conceitual completa
Usuário
¯
Pergunta em linguagem natural
¯
Aplicação
¯
Knowledge Base / busca semântica
¯
Recuperação dos trechos relevantes
¯
Amazon Bedrock
¯
Resposta baseada nos documentos + referências
Paralelamente, o fluxo de ingestão pode seguir: S3 - Step Functions - Lambda/Textract/Glue -
normalização e metadados - embeddings - armazenamento vetorial.
16. O que acontece quando o usuário faz uma pergunta?
1. O usuário envia uma pergunta.
2. A pergunta é analisada e transformada para permitir a busca.
3. O sistema procura informações semanticamente relacionadas.
4. Os trechos mais relevantes são recuperados.
5. Esses trechos são usados como contexto para o modelo.
6. O Bedrock gera uma resposta baseada nesse contexto.
7. A aplicação pode apresentar as fontes, datas e documentos relacionados.
17. Resumo final
A essência do projeto é criar um fluxo no qual documentos que antes estavam espalhados passam a
ser armazenados, processados, organizados e pesquisáveis. O S3 guarda os arquivos, o Step
Functions coordena o processo, o Textract extrai informações de documentos digitalizados, Lambda
executa tarefas de processamento, Glue e Athena ajudam com dados estruturados, embeddings
permitem busca semântica e o Bedrock utiliza RAG para gerar respostas baseadas no conteúdo
recuperado.
