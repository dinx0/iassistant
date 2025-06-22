# IAssistant

**IAssistant** é uma extensão Chrome desenvolvida para auxiliar na análise de resultados de buscas por meio de tokenização. A extensão calcula percentuais de similaridade entre a consulta realizada e os resultados obtidos, permitindo a visualização de métricas e gráficos que podem ser integrados a um serviço em nuvem (como o BigQuery).

## Funcionalidades

- **Tokenização & Análise de Similaridade:**  
  A extensão utiliza tokenização para comparar a pesquisa realizada com os resultados obtidos, calculando percentuais de similaridade.

- **Interface Dual – Popup e Side Panel:**  
  Utiliza um arquivo `index.html` para exibir a interface principal, disponível tanto como popup quanto como painel lateral (side panel) no Chrome.

- **Comandos Rápidos e Omnibox:**  
  - **Atalho:** Pressione `Ctrl+B` (ou `Command+B` no Mac) para executar a ação principal.  
  - **Omnibox:** Digite `assistente` na barra de endereços para interagir com a extensão.

- **Visualização de Gráficos e Métricas:**  
  Ao executar os comandos `show graphics` ou `show metrics`, a extensão realiza requisições para exibir gráficos e métricas detalhadas. (_Nota: Caso os endpoints ainda estejam configurados para `localhost`, ocorrerão erros de conexão em ambientes de produção._)


IAssistant/ ├── images/ │   ├── icon-16.png │   ├── icon-32.png │   ├── icon-48.png │   └── icon-128.png ├── scripts/ │   └── iaServiceWorker.js ├── index.html └── manifest.json


## Estrutura do Projeto

A estrutura da aplicação segue o padrão de uma extensão Chrome utilizando Manifest V3:


- **manifest.json:**  
  Arquivo de configurações que define:
  - Versão da extensão e do Manifest (Manifest V3).
  - Permissões necessárias: `"tabs"`, `"activeTab"`, `"sidePanel"`, `"storage"`, `"unlimitedStorage"`.
  - Configuração do **side panel** e do **popup** (ambos usando `index.html` como ponto de interface).
  - Detalhes dos ícones e comandos de atalho (ex.: `Ctrl+B` e a palavra-chave `assistente` para o omnibox).

- **index.html:**  
  Responsável pela interface gráfica do popup e do painel lateral, onde os usuários podem visualizar resultados, gráficos e métricas.

- **scripts/iaServiceWorker.js:**  
  Service worker que atua como backend interno da extensão, gerenciando eventos, processando requisições e, possivelmente, comunicando-se com APIs externas para enviar dados e obter métricas.

- **images/:**  
  Diretório contendo os ícones da extensão em várias resoluções para diferentes contextos (barra de ferramentas, painel, etc.).

## Instalação e Execução

### Instalação Local
1. **Carregar a Extensão:**
   - Abra o Chrome e acesse `chrome://extensions`.
   - Habilite o **Modo de Desenvolvedor**.
   - Clique em **"Carregar sem compactação"** e selecione a pasta raiz do projeto (`IAssistant/`).

2. **Testar a Interface:**
   - Clique no ícone da extensão para abrir o popup e verificar a interface.
   - Utilize a funcionalidade do **side panel** (disponível através da configuração no manifesto).
   - Utilize os atalhos (`Ctrl+B`/`Command+B`) e a palavra-chave `assistente` na Omnibox para testar os comandos.

### Executando Comandos Específicos
- **Show Graphics / Show Metrics:**  
  Estes comandos acionam funções que fazem requisições a endpoints responsáveis por retornar gráficos e métricas.  
  **Atenção:** Certifique-se que os endpoints estejam configurados para um domínio público (não "localhost") em produção.

## Configuração do Backend

A extensão se integra a um serviço backend para:
- Registrar os dados de tokenização.
- Processar as informações e exibir gráficos e métricas.
- Enviar os dados analisados para o Google BigQuery ou outros serviços em nuvem.

**Recomendação:**  
- Se estiver utilizando `localhost` durante o desenvolvimento, altere para a URL pública do seu servidor na versão publicada para evitar erros de conexão (_net::ERR_CONNECTION_REFUSED_).

## Resolução de Problemas

- **Erro de Conexão com `localhost`:**  
  Verifique se os endpoints configurados nos comandos `show graphics` e `show metrics` apontam para um servidor público e ativo.  
  Caso contrário, ajustados os endpoints ou utilize variáveis de ambiente para alternar entre desenvolvimento e produção.

- **Recursos Não Encontrados (ERR_FILE_NOT_FOUND):**  
  Certifique-se de que os caminhos para arquivos e recursos (como imagens, scripts e estilos) estão corretos no pacote da extensão.

- **Problemas com o Service Worker:**  
  Utilize `chrome://extensions` para inspecionar o service worker e verifique os logs para identificar possíveis erros.

## Contribuição

Contribuições para melhorias e novas funcionalidades são bem-vindas!

1. Fork o projeto.
2. Crie uma branch com a nova funcionalidade ou correção:  
   `git checkout -b feature/nome-da-feature`
3. Faça os commits com suas alterações.
4. Envie a branch para o repositório e abra um Pull Request.

## Licença

Este projeto está licenciado sob a [MIT License](LICENSE).

# IAssistant

O **IAssistant** é uma ferramenta em Python desenvolvida para analisar conteúdos textuais de documentos (PDF, HTML, TXT, CSV) e calcular métricas de similaridade entre frases extraídas dos documentos e termos de busca informados pelo usuário. Além disso, o projeto integra uma interface interativa—por meio do Dash e Plotly—para visualizar frequências, estatísticas e gráficos que evidenciam os resultados da análise.

## Funcionalidades

- **Extração e Processamento de Conteúdo:**
  - Suporte à leitura de documentos em diversos formatos:
    - PDFs (usando o PyMuPDF, via o módulo `fitz`)
    - Arquivos HTML, TXT e CSV através de tratamento de conteúdo ou requisições HTTP
  - Tokenização do texto utilizando NLTK, com remoção de stopwords (para os idiomas inglês e português) e limpeza textual via expressões regulares.

- **Análise de Similaridade e Frequência:**
  - Cálculo de métricas de similaridade (por meio do **cosine similarity**) entre o conteúdo extraído e os termos de busca desejados.
  - Divisão do documento em partes (splits por frases) para facilitar a análise por página ou por trecho.
  - Geração de distribuições de frequência usando `FreqDist` do NLTK e construção de dataframes no Pandas, permitindo a visualização e o agrupamento dos dados.

- **Armazenamento e Manipulação de Dados:**
  - Classes dedicadas à manipulação de arquivos: salvando conteúdo em formatos HTML, TXT e PDF.
  - Integração com um sistema de gerenciamento de dados via `BaseManager` para compartilhamento de informações entre processos.

- **Dashboards e Visualização Interativa:**
  - Criação de interfaces gráficas interativas utilizando o [Dash](https://dash.plotly.com/) e [Plotly](https://plotly.com/python/), com elementos como gráficos de dispersão, barras e tabelas dinâmicas.
  - Componentes interativos (dropdowns, botões e popovers via dash-bootstrap-components) que permitem filtrar os dados e visualizar métricas detalhadas por página, palavra e frase.
  - Funções utilitárias para definir cores (baseado na intensidade dos valores) que auxiliam na representação visual dos níveis de similaridade (ex.: vermelho para baixa similaridade, verde para alta).

- **Configuração Dinâmica do Backend:**
  - Funções para obter configurações dinâmicas de endpoints e dados de busca (através de GET requests para URLs configuradas).  
    **Atenção:** Por padrão, esses endpoints utilizam `http://localhost:8080`. Em produção, é necessário atualizar esse valor para um domínio público e acessível, evitando erros de conexão.

## Estrutura do Projeto

A pasta do projeto conta com os seguintes arquivos e subdiretórios básicos:

