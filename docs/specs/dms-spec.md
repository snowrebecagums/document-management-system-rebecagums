# Especificação - Document Management System

## 1. Objetivo

Permitir que usuários identifiquem, enviem, listem e baixem seus documentos, mantendo os arquivos no filesystem local da aplicação e os metadados em memória nesta fase inicial.

## 2. Escopo

### Dentro do escopo

- Upload de um documento por requisição.
- Listagem dos documentos associados ao usuário informado na requisição.
- Download de um documento pelo identificador, somente para o usuário associado.
- Armazenamento de arquivos em disco local com `multer` e `diskStorage`.
- Armazenamento em memória dos metadados necessários à listagem e ao download.
- Configuração do servidor e do armazenamento por variáveis de ambiente.

### Fora do escopo

- Armazenamento em nuvem ou em qualquer serviço externo.
- Autenticação, cadastro de contas ou gestão de credenciais.
- Persistência dos metadados em banco de dados ou recuperação automática dos metadados após reinício.
- Versionamento, exclusão, busca, compartilhamento, visualização prévia ou edição de documentos.
- Upload de múltiplos arquivos em uma única requisição.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O sistema deve aceitar um arquivo enviado como `multipart/form-data` no campo `file`. |
| RF-02 | O sistema deve rejeitar upload sem arquivo e arquivos acima do limite configurado, sem manter arquivo parcial no armazenamento. |
| RF-03 | Para cada upload aceito, o sistema deve gerar um identificador único, gravar o arquivo no armazenamento local com nome interno não controlado pelo cliente e registrar seus metadados em memória. |
| RF-04 | O sistema deve associar o documento ao identificador de usuário enviado no cabeçalho `X-User-Id`. |
| RF-05 | O sistema deve listar somente documentos pertencentes ao usuário indicado em `X-User-Id`, sem expor caminhos internos do filesystem. |
| RF-06 | O sistema deve permitir o download de um documento pelo identificador somente quando ele pertencer ao usuário indicado em `X-User-Id`. |
| RF-07 | O download deve ser entregue como anexo, usando o nome original do arquivo no cabeçalho `Content-Disposition`. |
| RF-08 | O sistema deve responder a `GET /health` com estado HTTP `200` e JSON `{ "status": "ok" }` quando estiver disponível. |
| RF-09 | O sistema deve retornar erros HTTP consistentes para entrada inválida, documento inexistente ou não pertencente ao usuário, arquivo acima do limite e falha interna. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos devem ser gravados exclusivamente no filesystem local da aplicação, por meio do `multer` com `diskStorage`, no diretório configurado. |
| RNF-02 | Os metadados devem permanecer em memória durante a execução do processo; um reinício limpa os metadados e pode deixar arquivos sem referência no disco. |
| RNF-03 | A configuração deve seguir 12-Factor: valores operacionais devem ser obtidos de variáveis de ambiente, com padrões documentados para desenvolvimento local. |
| RNF-04 | O nome recebido do cliente não deve ser usado como nome físico do arquivo. O caminho de armazenamento não deve ser retornado pela API. |
| RNF-05 | O limite de tamanho deve ser aplicado pelo middleware de upload antes de aceitar o arquivo. O padrão inicial é 10 MiB, configurável por `MAX_FILE_SIZE_BYTES`. |
| RNF-06 | A API deve usar JSON para respostas de metadados e erros; downloads são a exceção e retornam bytes do arquivo. |
| RNF-07 | O backend deve manter a separação `routes -> controllers -> services -> repositories`; as camadas internas não devem depender do Express. |
| RNF-08 | A interface web deve consumir o backend via `fetch` usando o prefixo `/api`; em desenvolvimento, o proxy do Vite remove esse prefixo antes de encaminhar a requisição. |

### Configuração inicial

| Variável | Padrão | Uso |
| --- | --- | --- |
| `PORT` | `3000` | Porta HTTP do backend. |
| `STORAGE_DIR` | `backend/storage` | Diretório local para os arquivos enviados. |
| `MAX_FILE_SIZE_BYTES` | `10485760` | Tamanho máximo por arquivo, em bytes (10 MiB). |

## 5. Modelo de dados

### Metadados públicos do documento

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | string | Identificador único, gerado pelo servidor (UUID). |
| `originalName` | string | Nome original fornecido pelo cliente, preservado apenas como metadado e para o nome do download. |
| `size` | number | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | Data e hora do upload em ISO 8601 UTC. |
| `owner` | string | Identificador recebido em `X-User-Id`. |
| `mimeType` | string | Tipo de mídia informado no upload; usar `application/octet-stream` quando ausente. |

### Dados internos de persistência

O repositório mantém, além dos metadados públicos, o caminho físico do arquivo. Esse dado é interno e nunca deve aparecer nas respostas da API. O nome físico deve ser gerado pelo servidor a partir do identificador, sem confiar no nome ou no caminho enviado pelo cliente.

## 6. Contratos de API

Todas as rotas de negócio usam os caminhos abaixo no backend. A aplicação web as acessa com `/api` na frente, por exemplo, `POST /api/upload`.

### Cabeçalho de usuário

As operações de upload, listagem e download exigem `X-User-Id`. O valor deve ser tratado como texto opaco: remover espaços em branco no início e no fim e rejeitar com `400` se, após essa normalização, estiver vazio. A comparação de propriedade é exata e diferencia maiúsculas de minúsculas. Nesta versão o valor é apenas um identificador de escopo, não uma credencial: o sistema não autentica nem verifica sua origem. Essa decisão atende ao escopo educacional/local e não deve ser considerada controle de acesso adequado para exposição pública.

### `POST /upload`

- Entrada: `multipart/form-data` com exatamente um arquivo no campo `file`; cabeçalho obrigatório `X-User-Id`. Requisições sem arquivo, com mais de um arquivo ou com arquivo em outro campo devem ser rejeitadas como inválidas. Campos de texto adicionais não fazem parte do contrato e não devem ser persistidos.
- Sucesso: HTTP `201`; corpo JSON com os metadados públicos do documento criado.
- Erros: `400` para cabeçalho de usuário ausente/vazio, campo de arquivo ausente ou requisição inválida; `413` para arquivo acima do limite; `500` para falha inesperada de armazenamento. Em caso de falha após gravar o arquivo, o arquivo deve ser removido para evitar órfão.

Exemplo de resposta:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "originalName": "relatorio.pdf",
  "size": 24576,
  "uploadedAt": "2026-11-11T12:00:00.000Z",
  "owner": "usuario-123",
  "mimeType": "application/pdf"
}
```

### `GET /documents`

- Entrada: cabeçalho obrigatório `X-User-Id`.
- Sucesso: HTTP `200`; corpo JSON é uma lista (possivelmente vazia) dos metadados públicos pertencentes ao usuário.
- Erros: `400` se o cabeçalho estiver ausente ou vazio; `500` para falha inesperada.

### `GET /documents/:id/download`

- Entrada: identificador UUID do documento no caminho e cabeçalho obrigatório `X-User-Id`.
- Sucesso: HTTP `200`; corpo binário do arquivo, `Content-Type` do documento e `Content-Disposition: attachment` com o nome original devidamente tratado para uso em cabeçalho HTTP.
- Erros: `400` para identificador que não seja UUID válido ou cabeçalho de usuário ausente/vazio; `404` quando o documento não existir, não pertencer ao usuário ou o arquivo não estiver disponível. A mesma resposta para inexistente e não pertencente evita revelar a existência de documentos de outros usuários; `500` para falha inesperada de leitura.

### `GET /health`

- Entrada: nenhuma.
- Sucesso: HTTP `200`; corpo JSON `{ "status": "ok" }`.

### Formato de erro

Erros da API devem usar JSON no formato `{ "error": { "code": "CODIGO_ESTAVEL", "message": "Descrição em português." } }`. Mensagens não devem incluir stack trace, caminhos internos ou detalhes sensíveis. Códigos previstos: `INVALID_REQUEST`, `FILE_TOO_LARGE`, `DOCUMENT_NOT_FOUND` e `INTERNAL_ERROR`.

## 7. Decisões arquiteturais

- Backend em CommonJS, Express e Clean Architecture simples: `routes` encaminham para `controllers`, que delegam regras a `services`; `services` usam `repositories` para metadados e filesystem.
- O middleware Multer configura `diskStorage` no limite HTTP de entrada. O repositório é responsável pelo acesso aos arquivos locais e pelo registro em memória dos metadados.
- O armazenamento físico usa nome gerado pelo servidor. O nome original é somente metadado e nome de apresentação no download.
- Os identificadores de usuário são fornecidos pelo cabeçalho `X-User-Id` até que autenticação seja definida em outro escopo. Esse cabeçalho não comprova identidade e não oferece proteção contra falsificação.
- O frontend é React e usa `fetch` para chamar `/api`; o proxy de desenvolvimento encaminha as chamadas para o backend local.
- Os metadados são voláteis. Persistência, limpeza automática de arquivos sem metadados e autenticação são decisões futuras, fora desta entrega.

## 8. Plano de execução

Este plano descreve etapas de trabalho futuras. Nesta atividade, somente este documento de especificação deve ser criado; não implementar nem alterar arquivos de backend ou frontend.

1. Revisar e aprovar contratos, limite de upload, comportamento de propriedade e riscos da identidade não autenticada.
2. Implementar a API backend conforme os contratos: armazenamento local com Multer, metadados em memória, isolamento por proprietário e tratamento de erros.
3. Implementar a experiência frontend de upload, listagem, download e apresentação de erros, consumindo a API pelo proxy `/api`.
4. Validar o fluxo integrado, incluindo upload válido e inválido, listagem por usuário, download autorizado, documento inexistente e limite de tamanho.
5. Registrar limitações da fase inicial e definir o escopo futuro para autenticação, persistência de metadados e limpeza de arquivos órfãos.

## 9. Critérios de aceite

- As rotas correspondem aos métodos, caminhos, cabeçalhos, respostas e erros descritos nesta especificação.
- Arquivos aceitos são gravados somente no diretório local configurado e não usam nomes físicos fornecidos pelo cliente.
- Uploads acima do limite ou inválidos não deixam arquivos parciais no armazenamento.
- Listagem e download não retornam documentos associados a outro valor de `X-User-Id`.
- Respostas de metadados não expõem caminhos internos.
- A interface consome a API pelo prefixo `/api` e oferece upload, listagem e download.
- Os limites conhecidos da persistência em memória e da ausência de autenticação estão documentados e não são apresentados como garantias de produção.
