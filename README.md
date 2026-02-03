# ServicoGestao — Fase 1 (Clean Architecture)

> **Escopo da fase:** modelagem de todo o sistema e **implementação apenas do serviço principal** (ServicoGestao) para gestão de **Clientes, Planos e Assinaturas** de uma operadora de internet. Microsserviços auxiliares (Faturamento e PlanosAtivos) ficam **fora do escopo** de implementação nesta fase, mas já foram considerados no desenho da arquitetura.

## Sumário
- [1. Visão geral](#1-visão-geral)
- [2. Arquitetura e princípios](#2-arquitetura-e-princípios)
- [3. Estrutura de pastas](#3-estrutura-de-pastas)
- [4. Banco de dados](#4-banco-de-dados)
- [5. Variáveis de ambiente](#5-variáveis-de-ambiente)
- [6. Como executar (passo a passo)](#6-como-executar-passo-a-passo)
- [7. Referência da API](#7-referência-da-api)
- [8. Dados de exemplo (seed)](#8-dados-de-exemplo-seed)
- [9. Postman Collection](#9-postman-collection)
- [10. Testes (opcional)](#10-testes-opcional)
- [11. Dúvidas comuns (FAQ)](#11-dúvidas-comuns-faq)

---

## 1. Visão geral
O **ServicoGestao** centraliza o cadastro e a consulta de **Clientes**, **Planos** e **Assinaturas**. Ele expõe uma API REST para:
- Listar clientes e planos
- Criar **assinaturas** com cálculo automático de **fim de fidelidade (365 dias)**
- Atualizar o **custo mensal** de um plano
- Consultar assinaturas por **tipo** (ATIVOS/CANCELADOS/TODOS), por **cliente** e por **plano**

**Base da API:** `http://localhost:3000/gerenciaplanos`

Stack principal:
- **Node.js 20+** · **TypeScript**
- **Express** (HTTP)
- **Prisma ORM** + **SQLite** (arquivo `dev.db`) — simples para correção/execução
- Coleção **Postman** para validação rápida

> Caso prefira **PostgreSQL + Docker**, veja a seção [4. Banco de dados](#4-banco-de-dados) para como alternar.

---

## 2. Arquitetura e princípios
### Clean Architecture (camadas)
- **domain/**: entidades de negócio, contratos (ports) e serviços de domínio (ex.: regra da fidelidade de 365 dias).
- **application/**: **use-cases** orquestram o fluxo (ex.: `CriarAssinatura`, `ListarClientes`, `AtualizarCustoPlano`).
- **infrastructure/**: adapters (repositórios Prisma, servidor HTTP, etc.).
- **interfaces (delivery)**: controladores/rotas HTTP (alocados em `infrastructure/http`).

### Padrões e SOLID
- **DIP**: casos de uso dependem de **interfaces** (`I*Repository`), injetadas por implementações Prisma.
- **SRP**: cada arquivo com responsabilidade única (entidade, caso de uso, repositório).
- **OCP**: novas consultas entram como **novos** casos de uso/rotas, sem alterar os existentes.
- **Repository Pattern**: isolamento do ORM (Prisma) atrás de interfaces.
- **DTO/Mapper** (quando necessário) para evitar vazamento de infraestrutura no domínio.

---

## 3. Estrutura de pastas
```
servico-gestao/
  src/
    domain/
      entities/              # Cliente, Plano, Assinatura
      repositories/          # IClienteRepository, IPlanoRepository, IAssinaturaRepository
      services/              # Regras (ex.: AssinaturaService - fidelidade)
    application/
      use-cases/             # Casos de uso (CriarAssinatura, Listar*, AtualizarCustoPlano)
      dtos/, mappers/        # (disponível para evoluções)
    infrastructure/
      http/                  # server.ts + routes.ts (controllers + rotas)
      repositories/          # *RepositoryPrisma (adapters)
      db/                    # (reservado para extras)
      messaging/             # (stub para fase 2)
    config/                  # env.ts (quando necessário)
    tests/                   # testes unitários/integrados (opcional)
  prisma/
    schema.prisma            # modelos Prisma
    seed.ts                  # popular DB com dados iniciais
  package.json
  tsconfig.json
  .env.example
  README.md                  # este arquivo
```

---

## 4. Banco de dados
### Padrão (recomendado para correção): **SQLite**
- Vantagem: zero-config, banco embarcado em arquivo (`dev.db`).

### Alternar para PostgreSQL (opcional)
1. **Altere** `prisma/schema.prisma`:
   ```prisma
   datasource db {
     provider = "postgresql"
     url      = env("DATABASE_URL")
   }
   ```
2. **Defina** `DATABASE_URL` no `.env`, por exemplo:
   ```env
   DATABASE_URL="postgresql://app:app@localhost:5432/gestao?schema=public"
   ```
3. (Opcional) Suba um Postgres com Docker:
   ```yaml
   services:
     db:
       image: postgres:16
       environment:
         POSTGRES_USER: app
         POSTGRES_PASSWORD: app
         POSTGRES_DB: gestao
       ports: ["5432:5432"]
   ```
4. Rode `npm run prisma:generate && npm run prisma:migrate && npm run prisma:seed`.

> Importante: o código das **rotas/casos de uso** não muda ao trocar de SQLite para PostgreSQL — só o provider/URL do Prisma.

---

## 5. Variáveis de ambiente
Arquivo: `.env` (criar a partir de `.env.example`):
```env
DATABASE_URL="file:./dev.db"
PORT=3000
BASE_PATH=/gerenciaplanos
```

---

## 6. Como executar (passo a passo)
```bash
# 1) Dentro de servico-gestao
cp .env.example .env

# 2) Instalar dependências
npm install

# 3) Gerar client Prisma e criar/migrar o banco
npm run prisma:generate
npm run prisma:migrate

# 4) Popular dados iniciais (clientes, planos e assinaturas)
npm run prisma:seed

# 5) Iniciar API (modo desenvolvimento)
npm run dev
```
- **Base URL:** `http://localhost:3000/gerenciaplanos`
- **Health-check rápido:** acesse `GET /gerenciaplanos/planos` após o seed.

---

## 7. Referência da API
**Headers gerais**: `Content-Type: application/json` quando houver corpo.

### 7.1 Listar clientes
`GET /gerenciaplanos/clientes`  
**200 OK** — array de clientes.

### 7.2 Listar planos
`GET /gerenciaplanos/planos`  
**200 OK** — array de planos.

### 7.3 Atualizar custo do plano
`PATCH /gerenciaplanos/planos/:idPlano`  
**Body:**
```json
{ "custoMensal": 129.90 }
```
**200 OK** — plano atualizado.  
**404** — quando o `idPlano` não existir.  
**400** — parâmetros inválidos.

### 7.4 Criar assinatura
`POST /gerenciaplanos/assinaturas`  
**Body:**
```json
{
  "codCli": 1,
  "codPlano": 1,
  "custoFinal": 79.90,
  "descricao": "Assinatura teste"
}
```
- Calcula automaticamente `fimFidelidade = dataContratacao + 365 dias`
- Status inicial: **ATIVO**  
**201 Created** — assinatura criada.  
**400** — parâmetros inválidos.

### 7.5 Assinaturas por tipo
`GET /gerenciaplanos/assinaturas/{TODOS|ATIVOS|CANCELADOS}`  
**200 OK** — array de assinaturas filtradas.

### 7.6 Assinaturas por cliente
`GET /gerenciaplanos/assinaturascliente/:codcli`

### 7.7 Assinaturas por plano
`GET /gerenciaplanos/assinaturasplano/:codplano`

#### Exemplos `curl`
```bash
curl -s http://localhost:3000/gerenciaplanos/clientes | jq
curl -s http://localhost:3000/gerenciaplanos/planos | jq

curl -s -X PATCH http://localhost:3000/gerenciaplanos/planos/1   -H "Content-Type: application/json"   -d '{"custoMensal": 129.9}' | jq

curl -s -X POST http://localhost:3000/gerenciaplanos/assinaturas   -H "Content-Type: application/json"   -d '{"codCli":1,"codPlano":1,"custoFinal":79.9,"descricao":"Assinatura teste"}' | jq

curl -s http://localhost:3000/gerenciaplanos/assinaturas/ATIVOS | jq
curl -s http://localhost:3000/gerenciaplanos/assinaturascliente/1 | jq
curl -s http://localhost:3000/gerenciaplanos/assinaturasplano/1 | jq
```

---

## 8. Dados de exemplo (seed)
O script `prisma/seed.ts` cria:
- **10 clientes** (`Cliente 1` a `Cliente 10`)
- **5 planos** (Básico 100, Intermediário 200, Avançado 300, Fiber 500, Gamer 600)
- **5 assinaturas** com datas variadas (algumas **ATIVAS**, uma **CANCELADA**) para facilitar testes dos filtros.

---

## 9. Postman Collection
Arquivo: `../{nome}_Desenvolvimento_de_Sistemas_backend_Fase-1.postman_collection.json`  
- Variável `{{base}}` padrão: `http://localhost:3000`  
- Requests prontos para todos os endpoints da seção 7.

**Como usar:** Importar o `.json` no Postman → definir `{{base}}` se necessário → executar requests.

---

## 10. Testes (opcional)
A pasta `src/tests` está preparada para inclusão de testes unitários/integrados. Sugestões:
- Casos de uso (`CriarAssinatura`, `AtualizarCustoPlano`) com repositórios *mockados*.
- Rotas (integração) utilizando **supertest**.

---

## 11. Dúvidas comuns (FAQ)
- **Erro de migração do Prisma**: apague `dev.db` e a pasta `prisma/migrations` (se for um ambiente de teste), depois rode `npm run prisma:migrate` novamente.
- **Porta ocupada (3000)**: altere `PORT` no `.env` e reinicie `npm run dev`.
- **Quero usar PostgreSQL**: veja a seção [4. Banco de dados](#4-banco-de-dados).

---

> **Fase 2 (próximos passos)**: microsserviços de **Faturamento** (pagamentos) e **PlanosAtivos** (validação de assinatura ativa), com **mensageria** e/ou comunicação síncrona via gateway.
