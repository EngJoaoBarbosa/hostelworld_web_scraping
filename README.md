# Projeto Hostelworld Web Scraping
<img width="400" height="50" alt="image" src="https://github.com/user-attachments/assets/5c6f5309-81a7-4904-b209-8111a7c7bd7c" />

## 1. Visão Geral do Projeto
#### Objetivo/Propósito
Realizar a extração automatizada de dados de propriedades (Nome, Id, Links das imagens, Descrição, Rating total e Serviços oferecidos) e suas respectivas avaliações (reviews) da plataforma Hostelworld . Salvando o resultado em um arquivo csv.

#### Conceito e Escopo
O pipeline faz consultas a partir de suas APIs de backend da Hostelworld por meio de requisições HTTP.\
A extração abrange os **detalhes cadastrais** da propriedade (nome, imagens, descrição, nota geral e facilidades/serviços) e a **totalidade de avaliações** escritas pelos usuários.

#### Resultados Esperados
Gerar um arquivo .csv consolidado (dados_hostelworld.csv) contendo a união dos dados estruturados de propriedades com seus respectivos comentários de usuários para posterior análise de dados.

#### Fontes de Dados
Origem: APIs REST internas da Hostelworld (endpoints de microsserviços Apigee).
Formato: o resultado das requisições é um conteúdo em formato JSON.
Frequência/Acesso: Sob demanda via chamadas HTTP GET.
Estrutura: JSON aninhado/hierárquico contendo listas e dicionários.

## 2. Modelagem de Dados
O resultado final da extração junta os dois domínios (propriedade e avaliações) através do identificador id_prop.\
Estrutura do Dataset Consolidado (dados_hostelworld.csv):
- id_prop (Chave Primária/Estrangeira de junção): Identificador numérico da propriedade.
- notes: Texto com o comentário/avaliação deixado pelo usuário.
- name: Nome da propriedade.
- images: Lista de URLs completas das imagens da propriedade.
- description: Texto descritivo limpo (sem quebras de linha \n ou \r).
- ratings.overall: Nota geral de avaliação da propriedade.
- facilities: Lista formatada das categorias e serviços/comodidades oferecidos pela propriedade.

## 3. Fluxo de Dados (ELT / Extração e Tratamento)
#### 1º passo - Requisição do Perfil da Propriedade:
Consulta o endpoint de propriedades usando id_prop com cabeçalho User-Agent customizado.

#### 2º passo - Normalização e Tratamento da Propriedade:
- Achata a estrutura JSON com pandas.json_normalize.
- Reconstrói as URLs completas das imagens concatenando prefix + suffix.
- Higieniza o campo de descrição removendo espaços e quebras de linha.
- Concatena categorias de facilidades e seus respectivos serviços em strings formatadas ("Categoria: Serviço1, Serviço2").

#### 3º passo - Paginação e Requisição de Avaliações:
- Faz uma chamada inicial na página 1 de reviews para capturar a contagem total de páginas (pagination.numberOfPages).
- Itera sobre todas as páginas de avaliações trazendo até o último comentário.

#### 4º passo - Junção (Merge) e Persistência:
- Executa um pd.merge do tipo left associando todos os comentários aos dados cadastrais da propriedade correspondente via id_prop.
- Grava o resultado gradativamente no arquivo CSV local via modo append (mode='a')

## 4. Detalhes da Arquitetura e Infraestrutura
#### Linguagem & Bibliotecas:
Python 3 (Kernel IPython), requests, pandas, concurrent.futures, os, time.

#### Estratégia de Concorrência / Paralelismo:
Uso da classe ThreadPoolExecutor da biblioteca concurrent.futures.
- Pool de Trabalhadores: Múltiplas threads simultâneas (max_workers = 50) enviando requisições I/O.
- Loteamento (Batching): Processamento dividido em blocos (ex: 100 requisições por lote, totalizando até 1000 propriedades) para evitar estouro de memória RAM do Pandas ao criar DataFrames intermediários.

## 5. Desenvolvimento & Decisões de Projeto
#### Superação de Bloqueios de API:
Configuração explícita de cabeçalho HTTP (User-Agent) simulando um navegador Chrome real em Windows para evitar HTTP 403 (Forbidden) ou bloqueios por WAF/Cloudflare.

#### Resiliência e Tratamento de Erros:
- Retentativas (Retry Logic): Laço de até 3 tentativas por requisição em caso de timeouts ou erros de conexão de rede (requests.exceptions.Timeout, requests.exceptions.ConnectionError), acompanhado de um intervalo (backoff) de 2 segundos.
- Propriedades Inexistentes / Sem Reviews: Retorno de DataFrames vazios com estrutura segura para evitar que a quebra de um ID específico paralise a execução inteira do ThreadPoolExecutor.

#### Tratamento de Arquivos Local:
- Verificação com os.path.exists() para deletar arquivos de compilações anteriores antes de iniciar uma nova execução do zero.
- Escrita progressiva em modo append com condicional no parâmetro header (header=not arquivo_existe) para garantir que o cabeçalho seja escrito apenas na primeira iteração.
- Codificação utf-8-sig na geração do CSV para suportar acentuação e caracteres especiais em múltiplos idiomas de avaliações.
