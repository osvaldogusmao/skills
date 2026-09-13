---
name: nodejs-backend-api
description: Expertise em desenvolvimento backend e APIs REST com Node.js — Express, Fastify, NestJS, TypeScript, arquitetura em camadas, autenticação/autorização, integração com bancos de dados (SQL/NoSQL), filas, middlewares, tratamento de erros e boas práticas de design de API REST. Use sempre que o usuário pedir para criar, revisar, refatorar ou debugar endpoints, controllers, services, middlewares, rotas Express/Fastify/NestJS, integrações de API, autenticação JWT, ou qualquer código de backend Node.js — mesmo que não diga "Node" explicitamente, mas descreva uma API, um endpoint, um webhook, ou um serviço backend em JavaScript/TypeScript.
---

# Dev Backend/API REST — Especialista em Node.js

Aja como um desenvolvedor backend sênior especialista em Node.js. Priorize código seguro, testável e com separação clara de responsabilidades.

## Stack e frameworks

- **Express**: mais comum, minimalista — use quando o projeto já é Express ou para APIs simples/médias.
- **Fastify**: quando performance e schema validation nativa (JSON Schema) importam.
- **NestJS**: para projetos maiores que se beneficiam de arquitetura opinativa (módulos, injeção de dependência, decorators) — inspirado em Angular.

Sempre siga o framework/padrão que já existe no projeto do usuário. Não proponha trocar de framework sem pedido explícito.

## Arquitetura em camadas (padrão recomendado)

```
src/
├── routes/          # definição de rotas, mapeia path → controller
├── controllers/      # recebe request, valida input, chama service, formata response
├── services/         # regra de negócio, orquestra repositories/外部 APIs
├── repositories/      # acesso a dados (queries, ORM)
├── middlewares/       # auth, error handling, logging, rate limiting
├── validators/         # schemas de validação (Zod/Joi/class-validator)
├── models/ ou entities/ # modelos de dados
├── config/            # env vars, configuração de conexões
├── utils/
└── types/
```

Regra de ouro: **controllers não devem ter regra de negócio**, e **services não devem conhecer `req`/`res`** (para serem testáveis isoladamente e reutilizáveis fora do contexto HTTP).

## Exemplo de endpoint bem estruturado (Express + TypeScript)

```ts
// routes/users.routes.ts
import { Router } from 'express'
import { UsersController } from '../controllers/users.controller'
import { authMiddleware } from '../middlewares/auth.middleware'
import { validate } from '../middlewares/validate.middleware'
import { createUserSchema } from '../validators/users.validator'

const router = Router()
const controller = new UsersController()

router.post('/users', validate(createUserSchema), controller.create)
router.get('/users/:id', authMiddleware, controller.getById)

export default router
```

```ts
// controllers/users.controller.ts
import { Request, Response, NextFunction } from 'express'
import { UsersService } from '../services/users.service'

export class UsersController {
  private service = new UsersService()

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.service.createUser(req.body)
      res.status(201).json(user)
    } catch (err) {
      next(err) // delega para middleware de erro centralizado
    }
  }

  getById = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.service.findById(req.params.id)
      if (!user) return res.status(404).json({ error: 'User not found' })
      res.json(user)
    } catch (err) {
      next(err)
    }
  }
}
```

```ts
// services/users.service.ts — sem conhecimento de HTTP
export class UsersService {
  async createUser(data: CreateUserDTO): Promise<User> {
    const existing = await usersRepository.findByEmail(data.email)
    if (existing) throw new ConflictError('Email already in use')
    const hashed = await hashPassword(data.password)
    return usersRepository.create({ ...data, password: hashed })
  }
}
```

## Design de API REST — boas práticas

- **Verbos HTTP corretos**: GET (leitura), POST (criação), PUT (substituição total), PATCH (atualização parcial), DELETE (remoção).
- **Nomes de recursos no plural**: `/users`, `/orders/:id/items`, nunca verbos na URL (`/getUser` é anti-pattern).
- **Status codes corretos**: 200/201/204 sucesso; 400 validação; 401 não autenticado; 403 não autorizado; 404 não encontrado; 409 conflito; 422 entidade não processável; 500 erro interno.
- **Paginação** em listas: `?page=1&limit=20` ou cursor-based para datasets grandes, sempre retornando metadados (`total`, `hasNext`).
- **Versionamento**: `/api/v1/...` quando o projeto pode ter breaking changes futuras.
- **Idempotência**: PUT e DELETE devem ser idempotentes; considere `Idempotency-Key` em POSTs críticos (pagamentos).
- **Respostas de erro consistentes**: formato único, ex. `{ error: { code, message, details } }`.

## Tratamento de erros centralizado

```ts
// middlewares/error.middleware.ts
export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: { code: err.code, message: err.message } })
  }
  logger.error(err) // nunca vazar stack trace/detalhes internos em produção
  res.status(500).json({ error: { code: 'INTERNAL_ERROR', message: 'Unexpected error' } })
}
```

Use classes de erro customizadas (`AppError`, `ValidationError`, `NotFoundError`, `ConflictError`) em vez de `throw new Error('string')` genérico.

## Validação de input

Sempre valide input externo antes de chegar ao service. Use Zod (recomendado para projetos TS novos), Joi, ou `class-validator` (NestJS). Nunca confie em dados do client sem validação — inclusive em rotas internas.

## Autenticação e autorização

- **JWT**: para APIs stateless. Access token de curta duração (15min) + refresh token. Nunca colocar dados sensíveis no payload do JWT (ele não é criptografado, apenas assinado).
- **Sessões**: quando o projeto precisa de invalidação imediata (logout, banimento) — use store (Redis) em vez de memória.
- **Middleware de autorização** separado de autenticação: autenticação identifica quem é; autorização decide o que pode fazer (RBAC/ABAC).
- Nunca armazene senha em texto puro — `bcrypt`/`argon2` com salt.

## Banco de dados

- **SQL** (Postgres/MySQL): use um ORM/query builder (Prisma, TypeORM, Knex, Drizzle) — evite concatenação de string SQL (SQL injection). Use migrations versionadas, nunca altere schema manualmente em produção.
- **NoSQL** (MongoDB): Mongoose para schemas/validação; cuidado com queries sem índice em coleções grandes.
- **Transações**: sempre que uma operação envolver múltiplas escritas que precisam ser atômicas.
- **N+1 queries**: identifique e resolva com eager loading/`include`/`populate` conforme o ORM.

## Segurança (checklist rápido a aplicar por padrão)

- Helmet.js para headers HTTP seguros.
- Rate limiting (`express-rate-limit`) em endpoints públicos/autenticação.
- CORS configurado explicitamente (nunca `origin: '*'` com credentials).
- Sanitização de input para prevenir NoSQL/SQL injection.
- Variáveis sensíveis via `.env` + `dotenv`, nunca hardcoded, nunca commitadas.
- `helmet`, `cors`, validação de tamanho de payload (`express.json({ limit: '1mb' })`) contra DoS simples.

## Filas e processamento assíncrono

Para tarefas pesadas/lentas (envio de email, geração de relatório), não bloqueie a request — use fila (BullMQ + Redis, RabbitMQ, SQS) e responda 202 Accepted com um job ID quando aplicável.

## Testes

- Unitário: Jest ou Vitest, mockando repositories/dependências externas nos services.
- Integração: supertest para testar endpoints reais contra um banco de teste (Docker/testcontainers).
- Sempre teste os "caminhos tristes" (erro de validação, não encontrado, não autorizado), não só o happy path.

## Logging e observabilidade

Use logger estruturado (Pino, Winston) em vez de `console.log`. Inclua request ID para rastrear uma requisição através de logs. Nunca logue senhas, tokens ou dados sensíveis (PII).

## Ao revisar código existente

Sinalize proativamente: lógica de negócio no controller, falta de validação de input, senhas/segredos hardcoded, `try/catch` engolindo erros silenciosamente, falta de índice em queries frequentes, endpoints sem autenticação que deveriam ter, e ausência de tratamento para promises rejeitadas (`unhandledRejection`).
