# Aplicação web para busca e comparação de preços de produtos
## Introdução

Aplicação web _full-stack_ que permite buscar e comparar preços entre produtos de múltiplos supermercados, utilizando recursos de `full-text search` e `autocomplete`. A aplicação capacita os usuários a identificarem as melhores ofertas, otimizarem suas compras semanais e, para usuários cadastrados, salvar listas de compras. O projeto foi desenvolvido utilizando *Clean Architecture* e *SOLID*.

## Funcionalidades e mapeamento para requisitos técnicos

- Busca de produtos e comparação de preços de produtos entre diferentes supermercados:
  - `elasticsearch` para `full-text search` e `autocomplete` iniciais (retornando `ids` dos produtos) e `PostgreSQL` para agregar demais dados de produtos com base nos `ids` resultantes da busca no `elasticsearch`.
  - `PostgreSQL`: desenho relacional para otimizar a busca de determinado produto com respectivos preços em diferentes supermercados.
  - Para busca de produtos, não havia necessidade de _cadastro/login_, o que tinha o objetivo de diminuir o atrito. Dessa forma, utilizou-se `reCAPTCHA v3` (com *fallback* para `v2`) para proteção contra bots.
  - _Web-crawling_ automatizado utilizando `Selenium`, `regex` e `Celery` para coleta, transformação e atualização de dados de produtos de diferentes supermercados.
 
- Cadastro e login tradicional e via Google:
  - cadastro tradicional (utilizando `hashing` e `salting` para armazenamento das senhas) e via Google (utilizando `OAuth 2.0` e `OIDC`).
  - No caso do cadastro tradicional, era feita a validação do e-mail através de link de verificação. Em ambos os casos, utilizou-se `reCAPTCHA v3` (com *fallback* para `v2`), para que se pudesse iniciar o processo de _cadastro/login_ e, após a autenticação, o _backend_ retornava um `JWT`, salvo em `localStorage`. Para cada requisição autenticada, o `token` era lido do `localStorage` e enviado no `header Authorization`. Este método (envio "manual" de `header`) previne ataques `CSRF` por design. Para mitigar o risco de `XSS` (a principal vulnerabilidade utilizando token `JWT` salvo `localStorage`), toda entrada e saída de dados é devidamente sanitizada.
    
- Criação e salvamento de listas de compras para usuários autenticados:
  - componentes `React.js` e rotas específicas autenticadas no backend para criação, atualização, leitura e deleção de listas de compras.

## Tecnologias Utilizadas

- **Backend**:
  - `Python`: Linguagem principal.
  - `Flask`: Framework web para _APIs REST_.
  - `SQLAlchemy`: _ORM_ para persistência em `PostgreSQL`.
  - `PostgreSQL`: persistência dos dados de forma relacional.
  - `Elasticsearch`: persistência otimizada para `full-text-search` e `autocomplete`.
  - `Selenium`, `regex` e `Celery`: _Web-crawling_, _ETL_, e processamento assíncrono para coleta e atualização de dados de produtos.
  - `Amazon S3`: Armazenamento de imagens de produtos.
    
- **Frontend**:
  - `React.js` como framework principal.
  - `Bootstrap` como framework `CSS`.
  - `AJAX`: Consumo das _APIs_ do backend (`JSON`).
    
- **Testes**:
  - `pytest`: Testes unitários e de integração no backend.
  - `Jest` + `React Testing Library`: Testes unitários e de integração no frontend.
    
- **Infraestrutura**:
  - `Nginx`: Servidor web.
  - `VPS (Ubuntu)`: Hospedagem da aplicação `Flask`, serviço de _web-crawling_, servidor do banco de dados `PostgreSQL` e servidor `elasticsearch`.
    
- **Segurança**:
  - `JWT`: Autenticação para rotas protegidas.
  - `reCAPTCHA v3` (com *fallback* para `v2`): Proteção contra bots.
  - Sanitização de dados para evitar ataques `XSS`.
    
- **Documentação**:
  - `OpenAPI (Swagger)`: Documentação da _REST_ _API_.

## Arquitetura

### Backend
**_REST APIs_** seguindo *Clean Architecture* e *SOLID*, com divisão clara de camadas e _dependency inversion_:

- `Flask` na camada mais externa (**web framework**), processando e respondendo (com `JSON`) aos respectivos `requests` de cada rota.
- **Controllers (Adapters)**: Convertem `requests` processados por cada rota do `Flask` em `Data Transfer Objects` (`DTOs`).
- **Use Cases**: Implementam a lógica de negócio, criando `Entities` (_domain core_) a partir dos `DTOs` e invocando os métodos dos respectivos repositórios que implementam as interfaces `I*Repository`.
- **Entities (_domain core_)**: camada central que define as entidades de negócio: ex.: `class Product`.
- **Repositories**: implementam as interfaces `I*Repository` utilizando tanto `SQLAlchemy` (_ORM_) para interação com os dados modelados de forma relacional em `PostgreSQL` quanto as _APIs_ da biblioteca `elasticsearch`, `client` oficial da _Elastic_ para `Python`, para os _cases_ de `autocomplete` e `full-text-search`. 
- **persitência de dados (camada mais externa)**: `PostgreSQL`, `Elasticsearch`, `Amazon S3`;

### Frontend
***Single Page Application*** (_SPA_) implementada em `React.js` com _Client Side Rendering_(_CSR_), utilizando `AJAX` (`fetch`) para consumir _APIs_ do _backend_ e atualizar o `DOM`. Não houve necessidade de utilizar uma biblioteca à parte para gerenciamento de estados, como `redux`. `Bootstrap` como _framework_ `CSS`. 

### _Web-Crawling_
Serviço automatizado de **_web-crawling_** utilizando `Selenium` e `Celery` (`Python`) para aquisição, normalização, deduplicação, categorização e atualização de dados de produtos, tanto na base de dados  `PostgreSQL` via `SQLAlchemy` quanto no `Elasticsearch`. Todo o pipeline de normalização, deduplicação e categorização usa apenas `regex`. Embora os `regex` utilizados em cada etapa permitissem um número grande de variações de palavras-chave, é um algoritmo completamente determinístico e, por isso, tem relativa fragilidade.

### Testes
- _backend_: testes unitários e de integração utilizando `pytest`.
- _frontend_:unitários e de integração utilizando `Jest` + `RTL`.

### DevOps
- `Git` e `Github` para controle de versionamento de código.
- Gerenciamento de dependências: `pip` para dos serviços em `Python` e `npm` para o frontend em `React.js`.
- `Github actions` para a execução automatizada de testes unitários e de integração após o `push` (_CI_). Não há deploy automatizado (_CD_).
- Deploys manuais para o servidor via `pull`.
- Configuração do servidor (incluindo a do `nginx`) via terminal `Linux` (`Ubuntu`).

### Segurança
- Para as rotas públicas (sem autenticação), utilização de `reCAPTCHA v3` (com _fallback_ para `v2`) para proteção contra bots.
- Utilização de `JWT` para rotas autenticadas (armazenado em `localStorage` e enviado via `header Authorization`, protegendo contra `CSRF`).
- _Rate limiting_ na camada do `Nginx` (com `limit_req_zone`).
- Todas as chaves secretas e endpoints eram armazenadas em variáveis de ambiente no servidor.

### Observabilidade
- Observabilidade e monitoramento (_logs_, _traces_, métricas de performance e de negócios): no _backend_, apenas geração de _logs_ de `INFO` e `ERROR` utilizando o módulo built-in `logging`. No _frontend_, `Sentry` para monitoramento de erros e observabilidade.

## Pontos de Melhoria

### Frontend
- Migrar para `Next.js` para combinar *Client-Side Rendering* (_CSR_) com *Server-Side Rendering* (_SSR_) e melhorar _performance_ e _SEO_.
- Utilizar *API Routes* do `Next.js` para criar *Backend for Frontends* (_BFFs_) e ocultar endpoints das _REST_ _API_.
- Melhorar acessibilidade da interface.

### Web-Crawling
- re-implementar a parte de transformação (normalização, deduplicação e categorização) de dados do _pipeline_ de _ETL_ de forma mais robusta, utilizando, por exemplo, `fuzzy matching` ou modelos de _NLP_.
- Substituir `Selenium` por `Playwright` para maior eficiência no _webscraping_.
  
### Testes
- Adicionar testes automatizados *End-to-End* (_E2E_) com `Playwright` como uma das etapas de testes dentro do processo de _CI_ com o `Github Actions`.
- Incluir análise de *code coverage*.

### DevOps
- Utilizar `Docker` para faciliar a paridade entre ambientes de desenvolvimento e produção e como um primeiro passo para a tansição para uma arquitetura _cloud-native_.
- Combinar `Docker` com *Containers as a Service* para facilitar a escalabilidade horizontal.
- Automatizar *deploy* com `GitHub Actions` (_CD_).

### Segurança
- Adicionar `Cloudflare` para `WAF` e mitigação de tentativas de `DoS`.
- Usar serviços como `Doppler` para gerenciamento seguro de chaves.

### Observabilidade
- Integrar `Sentry` no _backend_ para _logs_ de erros e alertas.
- Coletar e consolidar métricas de performance no _backend_ usando bibliotecas/serviços específicos (ex: `Prometheus` e `Grafana`).
- Capturar _traces_ no backend utilizando `Sentry`.
- Utilizar monitoramento de negócios (_Analytics_).

### Backup e DR
- Configurar backups automáticos (ex.: `pg_dump` da base de dados e armazenamento no `S3`).
