# Especificação - Document Management System

> Especificação funcional e técnica para orientar o desenvolvimento do DMS.
> O plano ao final descreve trabalho futuro; este documento não implementa nem
> executa alterações nos arquivos do backend ou do frontend.

## 1. Objetivo

Disponibilizar uma aplicação web simples para que usuários enviem, consultem e
baixem seus documentos, mantendo os arquivos no filesystem local da aplicação
e seus metadados em memória.

## 2. Escopo

### Dentro do escopo

- Envio de um documento por requisição.
- Listagem dos documentos pertencentes ao usuário da requisição.
- Download de um documento pelo identificador, somente pelo seu dono.
- Identificação simples do dono para separar os documentos entre usuários.
- Interface React para envio, listagem, atualização da lista e download.
- Armazenamento de arquivos em `backend/storage` via Multer `diskStorage`.
- Armazenamento volátil dos metadados em memória.
- Configuração da aplicação por variáveis de ambiente.

### Fora do escopo

- Armazenamento em nuvem, banco de dados ou qualquer serviço externo.
- Persistência durável dos metadados; eles são perdidos ao reiniciar o processo.
- Versionamento, edição, compartilhamento ou exclusão de documentos.
- Cadastro de usuários, login, autenticação ou gestão de credenciais.
- Busca avançada, pastas, etiquetas, pré-visualização e conversão de arquivos.
- Garantia de recuperação dos metadados dos arquivos já gravados após reinício.

### Premissas e limites

- Na ausência de autenticação nesta versão, o cliente identifica o usuário pelo
  cabeçalho `X-User-Id`. O valor é um identificador não vazio, limitado a 128
  caracteres após remoção de espaços nas extremidades.
- Esse cabeçalho não comprova a identidade de quem o envia. A separação por
  usuário é apenas uma conveniência da versão inicial e não constitui controle
  de acesso seguro para uso exposto à Internet.
- É permitido qualquer tipo de arquivo nesta versão; a aplicação não valida
  conteúdo nem extensão. O limite máximo inicial é de 10 MiB por arquivo,
  configurável por ambiente.
- A aplicação atende um único processo backend. Metadados em memória não são
  compartilhados entre processos ou instâncias.

## 3. Requisitos funcionais

| ID    | Requisito |
| ----- | --------- |
| RF-01 | O usuário pode enviar exatamente um arquivo no campo `file` de uma requisição `multipart/form-data`. |
| RF-02 | O sistema deve rejeitar a requisição de envio se o arquivo estiver ausente, vazio ou acima do limite configurado. |
| RF-03 | O sistema deve exigir `X-User-Id` válido para envio, listagem e download. |
| RF-04 | Ao aceitar um envio, o sistema deve gravar o arquivo em `backend/storage` usando Multer com `diskStorage` e um nome físico único gerado pelo servidor, independente do nome fornecido pelo cliente. |
| RF-05 | Após a gravação, o sistema deve registrar os metadados do documento em memória e retornar os metadados criados. |
| RF-06 | O usuário pode listar os metadados de seus próprios documentos; a resposta não deve incluir documentos de outro usuário. |
| RF-07 | A listagem deve apresentar documentos em ordem decrescente de data de envio. |
| RF-08 | O usuário pode baixar um documento existente de sua propriedade pelo identificador. |
| RF-09 | O download deve entregar o conteúdo binário e sugerir o nome original como nome do arquivo baixado. |
| RF-10 | O sistema deve responder com erro apropriado quando o documento não existir, não pertencer ao usuário ou não estiver disponível no filesystem. A resposta para documento inexistente e documento de outro usuário deve ser indistinguível. |
| RF-11 | A interface deve permitir selecionar e enviar um arquivo, apresentar estado de carregamento e informar sucesso ou erro. |
| RF-12 | A interface deve carregar e exibir a lista do usuário, permitir atualizar a lista e iniciar o download de um documento. |
| RF-13 | A interface deve apresentar estados vazios e mensagens compreensíveis quando não houver documentos ou ocorrer uma falha de comunicação. |

## 4. Requisitos não funcionais

| ID     | Requisito |
| ------ | --------- |
| RNF-01 | Os arquivos devem ser armazenados exclusivamente no filesystem local, em `backend/storage`, usando Multer com `diskStorage`; não usar provedores externos. |
| RNF-02 | Os metadados devem ser mantidos em memória nesta fase. O sistema deve documentar que reiniciar o backend perde os metadados e que os arquivos no disco podem permanecer sem referência. |
| RNF-03 | A configuração operacional deve vir de variáveis de ambiente. |
| RNF-04 | O nome físico do arquivo deve ser gerado pelo servidor, sem usar diretamente caminhos ou nomes fornecidos pelo cliente. O nome original é somente metadado e nome sugerido no download. |
| RNF-05 | A aplicação deve limitar o upload a um único arquivo e a 10 MiB por padrão; `MAX_FILE_SIZE` poderá substituir o limite em bytes e deve ser validado na inicialização. |
| RNF-06 | Falhas de validação, Multer, leitura e escrita de arquivos devem resultar em respostas HTTP explícitas, sem expor stack traces ou caminhos internos. |
| RNF-07 | O backend deve manter a separação `routes -> controllers -> services -> repositories`; camadas internas não devem depender de Express ou de detalhes HTTP. |
| RNF-08 | A interface deve consumir a API com `fetch` pelo prefixo `/api`; em desenvolvimento, o proxy Vite existente encaminha `/api` ao backend e remove o prefixo. |
| RNF-09 | O backend deve ser testável pelo runner nativo `node:test`, incluindo testes de sucesso, validação, isolamento por usuário e erros de arquivo. |
| RNF-10 | A implementação deve usar dependências existentes sempre que possível e manter JavaScript puro: CommonJS no backend e ESM no frontend. |

## 5. Modelo de dados

### Metadados do documento

| Campo        | Tipo   | Obrigatório | Descrição |
| ------------ | ------ | ----------- | --------- |
| `id`         | string | sim         | Identificador único, opaco e gerado pelo servidor. |
| `originalName` | string | sim       | Nome original informado para o arquivo; nunca usado como caminho de armazenamento. |
| `size`       | number | sim         | Tamanho em bytes; deve ser maior que zero e não exceder o limite configurado. |
| `uploadedAt` | string | sim         | Data/hora do recebimento bem-sucedido, em ISO 8601 UTC. |
| `owner`      | string | sim         | Valor normalizado do cabeçalho `X-User-Id`. |
| `storageName` | string | sim        | Nome físico aleatório gerado pelo servidor, relativo a `backend/storage`; não exposto na API. |

Os metadados são registros em memória, acessados por um repositório. O modelo
público de resposta contém somente `id`, `originalName`, `size`, `uploadedAt` e
`owner`; `storageName` é interno. Não há entidade persistida para usuários.

### Armazenamento do arquivo

- Diretório: `backend/storage`.
- Escrita: Multer configurado com `diskStorage`.
- Nome físico: identificador aleatório/UUID gerado no servidor, sem incorporar
  o nome original.
- O repositório de documentos relaciona `id` e `storageName` para localizar o
  arquivo no download.
- Caso a gravação tenha sucesso, mas o registro dos metadados falhe, remover o
  arquivo recém-gravado e retornar erro; não criar uma resposta de sucesso sem
  metadados.
- Arquivos existentes sem metadados após reinício não são recuperados nem
  listados automaticamente. Limpeza desses arquivos não faz parte desta versão.

## 6. Contratos de API

As rotas do backend são `/upload`, `/documents` e
`/documents/:id/download`. No frontend, as chamadas usam `/api` à frente das
rotas; o proxy de desenvolvimento já remove esse prefixo. Respostas JSON usam
`Content-Type: application/json; charset=utf-8`, exceto o download.

### Cabeçalho comum

- `X-User-Id: <identificador>` é obrigatório em todas as rotas.
- Ausente, vazio ou maior que 128 caracteres após trim: `400 Bad Request`.
- A aplicação usa o valor normalizado para associar e filtrar documentos.

### POST `/upload`

- **Entrada:** `multipart/form-data`, campo de arquivo `file`, somente um
  arquivo. `X-User-Id` obrigatório.
- **Sucesso:** `201 Created`.

```json
{
  "document": {
    "id": "uuid",
    "originalName": "relatorio.pdf",
    "size": 15360,
    "uploadedAt": "2026-01-15T12:30:00.000Z",
    "owner": "usuario-123"
  }
}
```

- **Erros:**
  - `400 Bad Request`: cabeçalho de usuário inválido, arquivo ausente ou vazio,
    mais de um arquivo, ou formulário inválido.
  - `413 Payload Too Large`: arquivo acima de `MAX_FILE_SIZE`.
  - `500 Internal Server Error`: falha inesperada de gravação ou processamento.
- A falha de gravação não deve criar metadados. O arquivo não deve ser
  disponibilizado como enviado se a operação completa falhar.

### GET `/documents`

- **Entrada:** nenhum corpo; `X-User-Id` obrigatório.
- **Sucesso:** `200 OK`, inclusive quando a lista estiver vazia.

```json
{
  "documents": [
    {
      "id": "uuid",
      "originalName": "relatorio.pdf",
      "size": 15360,
      "uploadedAt": "2026-01-15T12:30:00.000Z",
      "owner": "usuario-123"
    }
  ]
}
```

- Documentos de outros usuários não devem aparecer.
- **Erros:** `400 Bad Request` para cabeçalho inválido; `500 Internal Server
  Error` para falha inesperada ao consultar o repositório.

### GET `/documents/:id/download`

- **Entrada:** identificador no caminho e `X-User-Id` obrigatório.
- **Sucesso:** `200 OK`, corpo binário, `Content-Type` conforme disponível para
  o arquivo (ou `application/octet-stream`) e `Content-Disposition:
  attachment` com o nome original devidamente tratado pelo framework.
- **Erros:**
  - `400 Bad Request`: identificador ausente/inválido ou cabeçalho de usuário
    inválido.
  - `404 Not Found`: documento inexistente, de outro usuário ou arquivo não
    disponível no filesystem. A resposta não deve revelar qual caso ocorreu.
  - `500 Internal Server Error`: falha inesperada durante a leitura ou envio.

### Formato das respostas de erro

Para erros JSON antes do início do envio de um download:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Não foi possível processar a solicitação."
  }
}
```

`code` deve ser estável e próprio para tratamento no cliente; `message` deve
ser seguro para exibição e não incluir detalhes internos. Códigos previstos:
`VALIDATION_ERROR`, `FILE_TOO_LARGE`, `DOCUMENT_NOT_FOUND` e
`INTERNAL_ERROR`. O status HTTP é a fonte de verdade para a classe do erro.

## 7. Decisões arquiteturais

### Backend

- `routes/`: registra caminhos e encaminha requisições aos controllers.
- `controllers/`: extrai cabeçalhos, parâmetros e arquivos; valida o limite
  HTTP básico; converte resultado/erro de serviço em status e resposta.
- `services/`: implementa regras de envio, listagem por dono e autorização de
  download por correspondência de `owner`.
- `repositories/`: grava/lê/remove arquivos locais e guarda/consulta metadados
  em memória. O repositório de arquivos e o de metadados podem ser separados
  se isso mantiver responsabilidades claras.
- A configuração Express, o middleware Multer e o tratamento centralizado de
  erros ficam na composição da aplicação; services e repositories não
  dependem de Express.
- `PORT` define a porta do servidor (padrão atual: `3000`).
- `STORAGE_DIR` pode definir o diretório de armazenamento, com
  `backend/storage` como padrão; o diretório deve existir ou ser criado na
  inicialização.
- `MAX_FILE_SIZE` define o máximo de bytes por arquivo (padrão: `10485760`).

### Frontend

- Componentes funcionais com React Hooks e responsabilidades de apresentação
  separadas do acesso HTTP.
- Um serviço em `services/` encapsula as chamadas `fetch` para a API.
- Componentes previstos: envio de arquivo, listagem e ação de download,
  conforme o seed do frontend.
- O identificador do usuário nesta primeira versão deve ser obtido de uma
  configuração simples da interface e enviado como `X-User-Id` em cada
  chamada. Não implementar autenticação disfarçada nem confiar que esse valor
  seja uma credencial.

### Observabilidade e segurança

- Registrar falhas operacionais no backend sem expor conteúdo do arquivo,
  credenciais, caminhos internos ou stack traces ao cliente.
- Não confiar no nome do arquivo, no identificador do usuário ou em qualquer
  metadado fornecido pelo cliente como caminho de filesystem.
- A identificação via `X-User-Id` é falsificável e só é aceitável para o
  contexto inicial/local do projeto. Antes de implantação multiusuário real,
  deve ser substituída por autenticação confiável e autorização derivada da
  identidade autenticada.

## 8. Plano de execução

As etapas abaixo são propostas de trabalho futuro. Nenhum arquivo de backend ou
frontend é criado ou alterado por este plano; o único entregável desta tarefa
é `docs/specs/dms-spec.md`.

### Etapa 1 - Preparar a configuração e a composição do backend

- **Áreas/arquivos previstos:** `backend/src/app.js` e futura configuração
  dedicada em `backend/src/`.
- **Trabalho:** compor middleware, rotas e tratamento de erros; definir
  `PORT`, `STORAGE_DIR` e `MAX_FILE_SIZE` com valores padrão e validação.
- **Aceite:** aplicação inicia com os padrões, atende `GET /health` existente
  e reporta claramente configuração inválida sem iniciar em estado parcial.

### Etapa 2 - Implementar o fluxo de upload e os repositórios

- **Áreas/arquivos previstos:** `backend/src/routes/`,
  `backend/src/controllers/`, `backend/src/services/`,
  `backend/src/repositories/` e testes em `backend/test/`.
- **Trabalho:** configurar Multer `diskStorage`, gerar nomes físicos seguros,
  validar um arquivo e limite, armazenar metadados em memória e limpar arquivo
  em caso de falha ao registrar metadados.
- **Aceite:** upload válido retorna `201` com metadados sem `storageName`;
  ausências, arquivo vazio, excesso de tamanho e erro de escrita retornam
  status apropriado e não produzem metadados inconsistentes.

### Etapa 3 - Implementar listagem e download

- **Áreas/arquivos previstos:** rotas, controllers, services e repositories
  do backend definidos na etapa anterior, além de testes em `backend/test/`.
- **Trabalho:** filtrar lista por dono, ordenar por data decrescente, verificar
  propriedade antes do download e transmitir o arquivo com nome original.
- **Aceite:** lista só contém documentos do usuário; download do dono funciona;
  documento inexistente, de outro usuário ou sem arquivo resulta em `404`
  indistinguível.

### Etapa 4 - Construir a interface do usuário

- **Áreas/arquivos previstos:** `frontend/src/App.jsx`,
  `frontend/src/components/`, `frontend/src/services/` e, se necessário,
  `frontend/src/pages/`.
- **Trabalho:** implementar seleção/envio de arquivo, identificação simples de
  usuário, listagem, atualização, download, carregamento, estado vazio e
  mensagens de erro, consumindo `/api` com `fetch`.
- **Aceite:** uma pessoa consegue enviar um documento, ver os próprios
  documentos e baixar um item; erros e lista vazia são comunicados sem
  travar a interface.

### Etapa 5 - Integrar e validar o fluxo completo

- **Áreas/arquivos previstos:** testes existentes em `backend/test/` e
  validação dos scripts já declarados em `backend/package.json` e
  `frontend/package.json`.
- **Trabalho:** testar contratos HTTP e isolamento entre usuários; executar
  testes backend e build frontend; validar proxy `/api` em desenvolvimento e
  revisar documentação de execução e limites de persistência.
- **Aceite:** testes relevantes e build passam; fluxo de upload-listagem-
  download funciona pelo proxy; reinício do backend não é apresentado como
  persistência garantida dos metadados..
