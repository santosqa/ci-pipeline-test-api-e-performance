# Cadastro de Pets API

API REST simples para cadastro de usuários e pets, criada para estudos de testes e automação em nível de API.

O projeto usa Express, autenticação JWT, documentação Swagger e armazenamento em memória. Ao reiniciar a aplicação, os registros criados são perdidos.

## Tecnologias

- Node.js
- Express
- JWT
- Swagger UI
- Mocha, Chai, Sinon e Supertest

## Estrutura

```text
src/
  model/      Modelos e armazenamento em memória
  service/    Regras de negócio da aplicação

rest/
  app.js
  server.js
  auth.js
  swagger.js
  controller/

test/
  rest/
```

## Instalação

```powershell
npm install
```

## Como Rodar

```powershell
npm start
```

Também é possível usar o script específico da API REST:

```powershell
npm run start-rest
```

A API roda por padrão em:

```text
http://localhost:3000
```

A documentação Swagger fica disponível em:

```text
http://localhost:3000/api-docs
```

## Autenticação

As rotas públicas não exigem token:

- `POST /register`
- `POST /login`
- `GET /status`

As demais rotas exigem o header:

```text
Authorization: Bearer <token>
```

## Endpoints

### Status

```http
GET /status
```

Retorna o status da API e não exige autenticação.

Exemplo de resposta:

```json
{
  "status": "Operacional",
  "version": "1.0.0",
  "database": {
    "status": "Operacional",
    "maxConnections": 900
  }
}
```

### Usuários

```http
POST /register
POST /login
GET /users
```

### Pets

```http
POST /pets
GET /pets
GET /mypets
DELETE /pets/:id
```

O endpoint `GET /mypets` permite filtros opcionais por query string:

```http
GET /mypets?name=Rex
GET /mypets?type=Cachorro
GET /mypets?name=Rex&type=Cachorro
```

## Dados Iniciais

Ao iniciar a aplicação, alguns usuários e pets já vêm pré-cadastrados em memória.

### Usuários

| id | username | password |
| --- | --- | --- |
| 1 | `tiago` | `123456` |
| 2 | `ricardo` | `456789` |

As senhas são armazenadas no model em formato de hash usando `bcryptjs`, mas podem ser usadas em texto puro no login.

### Pets

| id | userId | name | type |
| --- | --- | --- | --- |
| 1 | 1 | `Luke` | `Cão` |
| 2 | 1 | `Maurício` | `Jabuti` |
| 3 | 2 | `Pandora` | `Gato` |

O campo `userId` indica a qual usuário cada pet pertence.

## Testes

O projeto possui testes de controller e testes externos para a API REST.

Para testes externos, crie um arquivo `.env` na raiz do projeto:

```env
BASE_URL_REST=http://localhost:3000
```

Comandos disponíveis:

```powershell
npm run test-rest-controller
npm run test-rest-external
```

Antes de executar os testes externos, suba a API REST.

## Testes de Performance com JMeter

O plano [pets-performance.jmx](./test/performance/jmeter/pets-performance.jmx) executa os mesmos cenários autenticados em dois modelos de carga, em sequência:

- **Carga fechada:** 5 usuários concorrentes, rampa de 5 segundos e 5 iterações por usuário.
- **Carga aberta:** 1 chegada por segundo, rampa de 5 segundos e manutenção por 30 segundos, com limite de 20 threads.

Cada iteração faz `POST /login`, extrai o JWT da resposta e usa esse token em `POST /pets` e `GET /pets`. O nome do pet inclui um UUID para evitar respostas `409` por duplicidade. As respostas esperadas são verificadas: `200` no login, `201` ao cadastrar e `200` ao consultar.

Use o JMeter 5.6.3 ou superior. A carga aberta usa o **Arrivals Thread Group** do plugin **Custom Thread Groups** (ID `jpgc-casutg`), disponivel pelo JMeter Plugins Manager.

Com a API em execução em `http://localhost:3000`, abra o JMeter em modo gráfico e carregue o plano:

```sh
jmeter -t test/performance/jmeter/pets-performance.jmx
```

Inicie o teste pelo botão **Start**. Os ouvintes mostram:

- **View Results Tree - Detalhes das requisições:** selecione cada amostra para ver status, requisição, resposta, cabeçalhos e falhas das validações.
- **Summary Report - Resumo:** quantidade de amostras e falhas, tempo médio e throughput por requisição.
- **Aggregate Report - Agregado:** estatísticas por requisição, incluindo percentis e taxa de erro.

Para executar em modo não-GUI e salvar os resultados em JTL:

```sh
jmeter -n -t test/performance/jmeter/pets-performance.jmx -l pets-performance.jtl
```

Os valores de carga e conexão ficam fixos diretamente no plano para facilitar a leitura e edição no JMeter:

| Grupo | Configuração fixa | O que significa |
| --- | --- | --- |
| Carga fechada | 5 usuários, rampa de 5 s, 5 iterações | Mantém até 5 usuários virtuais ativos; cada usuário repete o fluxo 5 vezes. A rampa distribui a inicialização dos usuários por 5 segundos. |
| Carga aberta | 1 chegada/s, rampa de 5 s, 1 etapa, duração de 30 s, limite de 20 threads | Inicia novas iterações pelo ritmo de chegada configurado, aumentando até 1 por segundo; mantém esse ritmo por 30 segundos. O limite evita criar mais de 20 threads. |
| Conexão e login | `http://localhost:3000`, usuário `tiago` | Altere os campos correspondentes no plano se a API ou as credenciais forem diferentes. |

As configurações não precisam ser variáveis para que o teste funcione. Os valores do plano são fixos: `localhost:3000`, usuário `tiago` com senha `123456`; carga fechada com 5 usuários, rampa de 5 segundos e 5 iterações; carga aberta com 1 chegada por segundo, rampa de 5 segundos, 1 etapa, duração de 30 segundos e limite de 20 threads. No JMeter, abra o grupo desejado e edite diretamente esses campos para mudar a carga. Aumente-a gradualmente e use um ambiente de teste.

Há duas substituições dinâmicas que são parte do próprio cenário, não configurações de carga: `${token}` recebe o JWT retornado pelo login; `${__UUID()}` gera um nome único para cada pet, evitando duplicidade. Essas devem permanecer dinâmicas para que as requisições protegidas funcionem e os cadastros não colidam.

O **View Results Tree** guarda os detalhes das respostas na memória; use-o em ensaios e diagnóstico, pois pode consumir recursos e influenciar medições maiores. Para cargas mais altas, prefira execução não-GUI e análise do arquivo JTL. Execute cargas maiores com cautela, preferencialmente contra um ambiente de teste.

### Executar testes manualmente no GitHub Actions

O workflow [manual-tests.yml](./.github/workflows/manual-tests.yml) permite escolher qual conjunto executar. Cada execução recebe um nome próprio no histórico: **API - testes funcionais** ou **Performance - JMeter**.

1. No GitHub, abra a aba **Actions** do repositório.
2. Selecione **Testes manuais** e clique em **Run workflow**.
3. Escolha `api` para executar os testes de controller e externos, ou `performance` para executar os dois grupos de carga do JMeter.
4. Clique em **Run workflow** para iniciar.

O workflow inicia a API automaticamente. A opção de API separa os testes de controller e externos e exibe no resumo a quantidade de testes, aprovações, falhas, pendências e duração; os relatórios HTML/JSON do Mochawesome ficam disponíveis nos artefatos. A opção de performance instala o JMeter e o plugin necessários; o resumo mostra amostras, falhas e tempos médios, P90, P95 e máximo por requisição/modelo de carga, e o arquivo JTL fica disponível para download nos artefatos da execução. A opção `performance` utiliza a carga fixa definida no plano.

## Observações

- O banco em memória deixa a aplicação leve e previsível para estudos.
- Como os dados são reiniciados a cada execução, os cenários de teste podem criar massa sem depender de um banco real.
- A camada `src/service` concentra as regras de negócio usadas pelos controllers REST.
