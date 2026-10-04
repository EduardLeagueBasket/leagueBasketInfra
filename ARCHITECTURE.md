# League Basket — arquitectura de microservicios

Documento para onboarding de personas o IAs. Describe el sistema **tal como está en el código**, no un diseño ideal.

**Raíz del workspace:** `/Users/eduard/www/league-basket/microservice`

Cada carpeta de servicio es un **repositorio git independiente**. No hay monorepo npm (no uses `package.json` en la raíz).

---

## 1. Qué es el producto

Backend de una plataforma de básquet:

- **Backoffice web** (staff: ligas, equipos, selecciones, contenido, tienda, tickets).
- **App móvil / cliente final** (fan: resultados, favoritos, stories, artículos, merch, tickets).

El único borde HTTP público previsto es `client_gateway` (`/api/...`). Los microservicios NestJS **no exponen HTTP**; hablan por **NATS** (request/reply). `content-ms` (Go) también tiene HTTP propio, pero el gateway es quien usa el resto de clientes.

---

## 2. Estilo arquitectónico

```
[Backoffice] ──HTTP──┐
                     ├──► client_gateway ──NATS send/emit──► microservicios
[App móvil]  ──HTTP──┘         │
                               ├── Cloudinary (upload de imágenes)
                               └── NATS broker :4222
```

Reglas:

1. El gateway traduce HTTP → patrón NATS (`ClientProxy.send` / `emit`).
2. Hay **un solo cliente NATS** en el gateway (`NATS_SERVICE`). El ruteo no es por nombre de servicio Nest, sino por **string del message pattern**.
3. Los microservicios **casi nunca se llaman entre sí**. La orquestación multi-servicio vive en el gateway.
4. Las relaciones entre DBs son **IDs lógicos (UUID) sin FK** (`competitionId`, `teamId`, `managerUserId`, `countryId`).
5. Cada servicio tiene **su propia base de datos**.

---

## 3. Mapa de piezas

| Carpeta | Stack | Escucha | Puerto compose | Base de datos | Dominio |
|---------|--------|---------|----------------|---------------|---------|
| `client_gateway` | NestJS HTTP | HTTP `:3000` + cliente NATS | 3000 | — | BFF / API gateway |
| `auth-ms` | NestJS + Prisma | NATS (`auth-queue`) | 3004 | PostgreSQL `league-auth` | Identidad **staff** (users, perfiles, JWT, mail) |
| `customer-auth-ms` | NestJS + Prisma | NATS (`customer-auth-queue`) | 3006 | PostgreSQL `customer_auth` | Identidad **fan/app** (customer, device, favoritos) |
| `sports-catalog-service` | NestJS + Prisma | NATS | 3001 | PostgreSQL `sports_catalog` | Países y competencias (catálogo maestro) |
| `competition-engine-service` | NestJS + Prisma | NATS | 3002 | PostgreSQL `competition_engine` | Ligas: equipos, grupos, partidos, tabla, temporadas |
| `national-teams-ms` | NestJS + Prisma | NATS | 3003 | PostgreSQL `national_teams` | Selecciones: equipos, partidos, stats |
| `retail-ms` | NestJS + Prisma | NATS | 3005 | **MongoDB** `retail` | Tienda merch (productos, carrito, pedidos) |
| `tickets-service` | NestJS + Prisma | NATS | 3007 | **MongoDB** `tickets` | Tickets, venues, eventos, puntos, sorteos |
| `content-ms` | **Go** + Gin + pgx | HTTP `:3008` **y** NATS | 3008 | PostgreSQL `content_db` | Stories, artículos, publishers, categorías |
| `basket-league-infra` | Docker Compose | — | — | NATS + Mongo en compose; Postgres suele ir en el host | Orquestación local |

Infra compartida en compose:

- **NATS** `4222`
- **MongoDB 7** `27017` (retail + tickets)
- **Postgres** no está en el compose: los NestJS apuntan a `host.docker.internal:5432` (Postgres local del desarrollador).

---

## 4. Cómo fluye un request

### Admin (protegido)

```
POST /api/auth/login
  → gateway send('auth.login')
  → auth-ms valida credenciales y firma JWT

Luego GET/POST /api/admin/...
  → NatsJwtAuthGuard: Authorization Bearer
  → send('auth.user.authenticate') a auth-ms
  → AdminRolesGuard compara profile.name con @Roles o DEFAULT_BACKOFFICE_PROFILES
  → controller del gateway hace send('<pattern>', payload)
  → microservicio responde
```

El gateway **no** verifica la firma JWT localmente. Delega en `auth-ms`.

### Móvil / app (hoy mayormente abierto en el gateway)

```
POST /api/mobile/auth/login
  → send('customer-auth.login') → customer-auth-ms → JWT de customer

GET /api/app/content/stories?countryCode=CO
  → send('stories.feed-by-country') → content-ms
```

Importante: la mayoría de rutas `/api/mobile/*`, `/api/retail/*`, `/api/tickets/*`, `/api/app/*` **no tienen guard JWT en el gateway**. Varios endpoints reciben `userId` en query/body.

### Evento (no request/reply)

Al crear/actualizar competencias, equipos de liga o selecciones, el gateway hace:

```
natsService.emit('publisher.sync', { ... })
```

`content-ms` lo recibe y sincroniza un **publisher** (entidad que puede publicar stories/artículos).

Uploads de imagen **no van por NATS**: el gateway sube a **Cloudinary** y manda la URL en el payload NATS.

---

## 5. Audiencias HTTP (`client_gateway`)

Prefijo global: `/api`.

| Prefijo | Audiencia | Auth real en código |
|---------|-----------|---------------------|
| `/api/auth/*` | Login/registro/verify de staff | Público |
| `/api/admin/*` | Backoffice | Bearer → `auth.user.authenticate` + roles |
| `/api/app/content/*` | Feed móvil de stories/artículos | Público (sin guard) |
| `/api/mobile/*` | Customer auth, devices, favoritos | Público en gateway |
| `/api/sports-catalog`, `/api/competition/*`, `/api/teams/*`, `/api/national-teams/*` | Lectura móvil de catálogo/liga/selecciones | Público en gateway |
| `/api/retail/*`, `/api/tickets/*` | Tienda y tickets móvil | Público en gateway |

`docs/API-AUDIENCES.md` habla de `/api/app/health` y `/api/app/me`; **no existen** en el código actual.

### Perfiles staff (`auth-ms`)

`SUPER_ADMIN`, `MANAGER_LEAGUE`, `MANAGER_TEAM`, `PLAYER`, `USER`, `ASISTANT_TEAM`, `ASISTANT_LEAGUE`, `COACH`, `MANAGER_NATIONAL_TEAM`, `ASISTANT_NATIONAL_TEAM`, `MANAGER_REGIONAL_TEAM`, `ASISTANT_REGIONAL_TEAM`.

El gateway también acepta `ADMIN` en `DEFAULT_BACKOFFICE_PROFILES`. `USER` no entra al backoffice.

El scoping de liga/equipo se hace en el gateway (p. ej. `LeagueScopeService`, `get-competition-by-manager-user`, `get-team-by-manager-user`, affiliation de publishers).

---

## 6. Contratos NATS (cómo se rutea)

No hay un registry. Convención observada:

| Prefijo de pattern | Servicio |
|--------------------|----------|
| `auth.*`, `profile-generate`, `login`, `user.authenticate`, `register`, … | `auth-ms` (algunos patterns legacy sin prefijo) |
| `customer-auth.*` | `customer-auth-ms` |
| `create-country`, `get-all-countries`, `list-competitions`, `get-competition`, … | `sports-catalog-service` (sin prefijo de servicio) |
| `competition-engine.*` | `competition-engine-service` |
| `create-national-team`, `list-national-matches`, … | `national-teams-ms` |
| `stories.*`, `articles.*`, `article-category.*`, `publisher.*` | `content-ms` |
| `retail.*` | `retail-ms` |
| `tickets.*`, `tickets.mobile.*` | `tickets-service` |

El gateway inyecta `ClientProxy` como `NATS_SERVICE` (`src/config/service.ts`). Servers: `NATS_URL` (default `nats://localhost:4222`), `maxPayload` 10MB.

---

## 7. Dominio por servicio (entidades)

### auth-ms (Postgres)

`Profile`, `User`, `UserHistory`.  
`User.competitionId` es UUID de `Competition` en sports-catalog (sin FK). Soft-delete vía status / deactivate.

### customer-auth-ms (Postgres)

`Customer`, `Address`, `Device`, `FavoriteTeam`.  
`FavoriteTeam.teamId` / `competitionId` son IDs externos.

### sports-catalog-service (Postgres) — catálogo maestro

- `Country` (`countryCode`, logo, region).
- `Competition` (`type`: `league | national_team | women | 3x3`, `managerUserId`, `countryId`).

Casi todo lo demás referencia `competitionId` de aquí.

### competition-engine-service (Postgres) — motor de liga

`Group`, `Team`, `Match`, `Standing`, `Season`, `SeasonTeam`.  
Enums: `MatchStatus` (SCHEDULED/LIVE/FINISHED/POSTPONED), `MatchType`, `SeasonStatus`.  
`Team.managerUserId`, `Team.sellsTickets`. Visitante de partido puede ser un nombre libre (`awayTeamName`).

### national-teams-ms (Postgres)

`NationalTeam` (gender, category, region, `competitionId`, `countryId`, `managerUserId`), `NationalMatch`, `NationalMatchStats`.

### retail-ms (Mongo)

`Producto` (imágenes, variantes, descuento, `teamId` + `teamType` LIGA|NACIONAL), `Carrito`/`CarritoItem`, `Pedido`/`PedidoItem`. Categorías por enum.

### tickets-service (Mongo)

`Venue`, `Section`, `Event`, `EventSection`, `Promotion`, `Ticket`, `Order`, `TicketCart`/`TicketCartItem`, `Transfer`, `PointsWallet`, `PointsTx`, `Raffle`, `Prize`.  
`Event.teamId` y `Ticket.ownerId` son IDs lógicos.

### content-ms (Postgres, migraciones SQL)

`publishers`, `stories` (+ views), `articles`, `article_categories`. Feeds por `countryCode` / país destino. Publishers se sincronizan desde ligas/equipos/selecciones.

---

## 8. Estructura interna típica (NestJS)

Servicios “maduros” (auth, sports-catalog, competition-engine, national-teams, retail):

```
src/<feature>/
  presentation/controllers   @MessagePattern
  application/use-cases + dto + ports
  domain/entities
  infrastructure/data-sources  Prisma
```

Más planos: `customer-auth-ms`, `tickets-service` (controller + dto + prisma).

Gateway: solo controllers HTTP que reenvían a NATS. Split `src/admin/*` vs `src/mobile/*`.

Go (`content-ms`): `cmd/api/main.go` + `internal/{publisher,stories,articles,article_categories}` (handler HTTP + subscriber NATS + service + repository).

---

## 9. Infra local

Carpeta: `basket-league-infra/`

- `docker-compose.yaml` — builds con rutas `./../<servicio>`
- `.env` / `.env.template` — puertos y `DATABASE_URL*`
- Arranque: desde `basket-league-infra` (`docker compose up`)

Postgres: bases esperadas en el host (`sports_catalog`, `competition_engine`, `national_teams`, `league-auth`, `customer_auth`, `content_db`). Mongo en el contenedor o Atlas (según `.env`).

Variables típicas (nombres, no valores):

- Gateway: `PORT`, `NATS_URL`
- Nest MS: `NATS_SERVERS`, `DATABASE_URL`, `PORT` (el puerto HTTP de compose no sirve de listen; los Nest son solo NATS)
- auth-ms extra: `SALT`, `JWT_SECRET`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_USER`, `MAIL_PASS`, `MAIL_FROM`
- customer-auth-ms extra: `SALT`, `JWT_SECRET`
- content-ms: `SERVER_PORT`, `NATS_URL`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`

---

## 10. Clientes conocidos (fuera de este workspace)

En el mismo árbol `league-basket/` hay frontends hermanos, no parte de estos microservicios:

- `basket-league-backoffice` — panel admin → `/api/admin` + `/api/auth`
- app móvil / `league_basket` — cliente fan → `/api/mobile`, `/api/app`, catálogo, retail, tickets

---

## 11. Convenciones para implementar o cambiar código

1. **No agregar `package.json` en la raíz de `microservice/`.** Cada servicio instala sus deps.
2. **Nuevas APIs de backoffice:** `client_gateway/src/admin/...` bajo `/api/admin/...` con `NatsJwtAuthGuard` + `AdminRolesGuard` + `@Roles` si aplica.
3. **Nuevas APIs de cliente final:** preferir `/api/app/...` (hoy conviven `/api/mobile` y rutas sueltas).
4. **Nuevo message pattern:** definirlo en el controller NATS del microservicio y llamarlo igual desde el gateway. Prefijar con el servicio (`competition-engine.*`, `auth.*`, …) en código nuevo.
5. **No crear FK entre bases.** Guardar UUID y documentar de qué servicio sale.
6. **Uploads:** Cloudinary en el gateway; persistir URL.
7. **Entidades publicables** (liga/equipo/selección): emitir `publisher.sync` para content-ms.
8. **Prisma:** schema y migraciones viven **dentro de cada servicio**.
9. **content-ms:** migraciones `golang-migrate` en `content-ms/migrations/`.
10. El gateway **orquesta**; no pongas reglas de negocio pesadas en un MS que pertenezcan a otro bounded context.

---

## 12. Deuda / trampas conocidas

- Docs de audiencias desactualizados (`/api/app/health`, `/api/app/me`, Bearer en `/app`).
- Rutas móviles del gateway sin autenticación real.
- Patterns NATS inconsistentes (unos con prefijo de servicio, sports-catalog y national-teams a menudo sin él).
- `auth-ms` acepta patterns legacy (`login`) y nuevos (`auth.login`).
- Credenciales Cloudinary hardcodeadas en `client_gateway/src/helpers/upload-media.ts`.
- `content-ms` expone HTTP directo además del bus.
- Compose publica puertos 3001–3007 de servicios que no escuchan HTTP.

---

## 13. Archivos de entrada rápidos

| Para entender… | Empieza aquí |
|----------------|--------------|
| Bootstrap HTTP | `client_gateway/src/main.ts`, `app.module.ts` |
| Cliente NATS | `client_gateway/src/nats/nats.module.ts` |
| Guards admin | `client_gateway/src/admin/auth/guards/` |
| Audiencias (doc) | `client_gateway/docs/API-AUDIENCES.md` |
| Compose | `basket-league-infra/docker-compose.yaml` |
| Identidad staff | `auth-ms/src/main.ts`, `auth-ms/prisma/schema.prisma` |
| Identidad fan | `customer-auth-ms/prisma/schema.prisma` |
| Catálogo | `sports-catalog-service/prisma/schema.prisma` |
| Motor de liga | `competition-engine-service/prisma/schema.prisma` |
| Contenido | `content-ms/cmd/api/main.go` |
