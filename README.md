# Apuntes-compunent
apuntes parcial compunent

# 📘 Apuntes para parcial de NestJS + TypeORM (paso a paso)

> Guía genérica para cualquier parcial del tipo: *"te doy un contexto, creas entidades, módulos, CRUD, reglas de negocio y una colección de Postman en ~2 horas"*.
> El enunciado cambia, **la lógica siempre es la misma**. Al final hay un ejemplo completo resuelto (duelos 1 vs 1) para copiar la estructura.

---

## 📑 Índice

0. [Regla de oro y plan de tiempo](#0-regla-de-oro-y-plan-de-tiempo)
1. [Analizar el enunciado (lo primero)](#1-analizar-el-enunciado-lo-primero)
2. [Preparar el proyecto](#2-preparar-el-proyecto)
3. [Configurar TypeORM y main.ts](#3-configurar-typeorm-y-maints)
4. [Crear los recursos con el CLI](#4-crear-los-recursos-con-el-cli)
5. [Entidades (plantillas y relaciones)](#5-entidades-plantillas-y-relaciones)
6. [DTOs y validaciones](#6-dtos-y-validaciones)
7. [Módulos](#7-módulos)
8. [Servicios (CRUD)](#8-servicios-crud)
9. [Controladores](#9-controladores)
10. [Reglas de negocio (catálogo de patrones)](#10-reglas-de-negocio-catálogo-de-patrones)
11. [Postman](#11-postman)
12. [Git y entrega](#12-git-y-entrega)
13. [Errores comunes y soluciones](#13-errores-comunes-y-soluciones)
14. [Checklist final](#14-checklist-final)
15. [Ejemplo completo: duelos 1 vs 1](#15-ejemplo-completo-duelos-1-vs-1)

---

## 0. Regla de oro y plan de tiempo

> ⚠️ **Si la app no arranca, la nota baja drásticamente.** Siempre prioriza: **que corra** > entidades > CRUD > reglas > Postman > pulido.

| Tiempo | Qué hacer |
|---|---|
| 0:00 – 0:10 | Clonar repo, leer enunciado, revisar `package.json`, `app.module.ts`, `main.ts` |
| 0:10 – 0:20 | Diseñar entidades y relaciones **en papel** |
| 0:20 – 0:30 | Instalar dependencias, configurar TypeORM, ValidationPipe, `nest g resource` |
| 0:30 – 0:50 | Entidades + DTOs |
| 0:50 – 1:10 | CRUD de cada módulo (servicios + controladores) y probar que arranca |
| 1:10 – 1:35 | Reglas de negocio |
| 1:35 – 1:50 | Colección de Postman + exportar JSON |
| 1:50 – 2:00 | Revisión final, commit y push |

💡 **Haz commit después de cada módulo.** Si algo se rompe, puedes volver atrás.

---

## 1. Analizar el enunciado (lo primero)

Lee el enunciado **dos veces** con un resaltador. Extrae:

### 1.1 Sustantivos → Entidades
Subraya los "objetos" del negocio. Ej.: *jugador, sesión/duelo, ronda, movimiento* → `Player`, `Match`, `Round`, `Move`.

### 1.2 Relaciones entre entidades
Pregúntate para cada par:

| Pregunta | Relación TypeORM |
|---|---|
| "Un X tiene muchos Y, y cada Y pertenece a un solo X" | `X` → `@OneToMany`, `Y` → `@ManyToOne` |
| "Un X tiene exactamente un Y" | `@OneToOne` (con `@JoinColumn` en un lado) |
| "Muchos X con muchos Y" | `@ManyToMany` (con `@JoinTable` en un lado) |

> **Regla práctica:** la tabla que tiene la llave foránea es la del lado `@ManyToOne`.

### 1.3 Verbos → Endpoints
"Los jugadores **envían** movimientos" → `POST /moves`. "Se **crea** una sesión" → `POST /matches`.

### 1.4 Reglas de negocio → Lógica en servicios
Numera cada regla. Escribe al lado **qué validación** es y **en qué servicio** va:

```
Regla 1: Un jugador solo un movimiento por ronda  → MovesService.create → BadRequest
Regla 2: No repetir movimiento consecutivo        → MovesService.create → BadRequest
Regla 3: Resolver ronda solo con ambos            → MovesService.create → llama resolveRound()
Regla 4: Tabla de victorias y daño                → MovesService.resolveRound()
```

### 1.5 Estados
Si el enunciado habla de "activo / finalizado / pendiente / resuelto", crea un **enum** de estado.

### 1.6 Entregables
Anota: ¿Postman? ¿Qué fecha? ¿Qué base de datos? ¿Hay repo de classroom? ¿Se pide README?

---

## 2. Preparar el proyecto

### 2.1 Clonar el repo del classroom
```bash
git clone <URL_DEL_REPO>
cd <carpeta-del-repo>
npm install
```

### 2.2 Qué revisar ANTES de escribir código
| Archivo | Qué mirar |
|---|---|
| `package.json` | Qué dependencias ya están (`typeorm`, `@nestjs/typeorm`, `class-validator`, driver de BD) y los scripts (`start:dev`) |
| `src/app.module.ts` | Si ya hay `TypeOrmModule.forRoot(...)` |
| `src/main.ts` | Si ya hay `ValidationPipe` y qué puerto usa |
| `.env` / `.env.example` | Variables de BD esperadas |
| `README.md` | Instrucciones del profesor (BD, puertos, etc.) |
| `docker-compose.yml` | Si hay BD en contenedor |
| `.gitignore` | Que ignore `node_modules`, `dist`, `.env` y el archivo `.db` |

### 2.3 Si el repo está vacío (crear proyecto desde cero)
```bash
npm i -g @nestjs/cli
nest new nombre-proyecto        # elige npm
cd nombre-proyecto
```

### 2.4 Instalar dependencias necesarias
```bash
# ORM
npm i @nestjs/typeorm typeorm

# Validación de DTOs
npm i class-validator class-transformer

# Elige UNA base de datos:
npm i better-sqlite3            # SQLite (recomendada: no requiere servidor)
# npm i sqlite3                 # alternativa si better-sqlite3 falla al compilar
# npm i pg                      # PostgreSQL
# npm i mysql2                  # MySQL

# Opcionales
npm i @nestjs/config            # variables de entorno
npm i @nestjs/swagger           # documentación automática
```

> ✅ Si el profesor pide una BD concreta (Postgres/MySQL), usa **esa**. Si no dice nada, **SQLite** es lo más seguro porque corre en cualquier máquina.

---

## 3. Configurar TypeORM y main.ts

### 3.1 `src/app.module.ts` — SQLite
```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { PlayersModule } from './players/players.module';
// ...importa tus demás módulos

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'better-sqlite3',
      database: 'database.db',
      autoLoadEntities: true,   // carga las entidades registradas con forFeature
      synchronize: true,        // crea/actualiza tablas automáticamente (solo desarrollo)
    }),
    PlayersModule,
    // ...demás módulos
  ],
})
export class AppModule {}
```

### 3.2 `src/app.module.ts` — PostgreSQL (con .env)
```ts
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('DB_HOST', 'localhost'),
        port: +config.get('DB_PORT', 5432),
        username: config.get('DB_USER', 'postgres'),
        password: config.get('DB_PASSWORD', 'postgres'),
        database: config.get('DB_NAME', 'parcial'),
        autoLoadEntities: true,
        synchronize: true,
      }),
    }),
  ],
})
export class AppModule {}
```

`.env`:
```
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=parcial
```

### 3.3 `src/main.ts`
```ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,            // elimina campos que no están en el DTO
      forbidNonWhitelisted: true, // lanza error si envían campos extra
      transform: true,            // convierte tipos (string → number, etc.)
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

### 3.4 Probar que arranca
```bash
npm run start:dev
```
Debes ver `Nest application successfully started`. **No sigas hasta que esto funcione.**

---

## 4. Crear los recursos con el CLI

Por cada entidad:

```bash
npx nest g resource players --no-spec
```
Responde:
- *What transport layer?* → **REST API**
- *Generate CRUD entry points?* → **Yes**

Esto genera:
```
src/players/
├── dto/
│   ├── create-player.dto.ts
│   └── update-player.dto.ts
├── entities/
│   └── player.entity.ts
├── players.controller.ts
├── players.module.ts
└── players.service.ts
```
y además **lo registra automáticamente en `AppModule`**. Repite para cada entidad:

```bash
npx nest g resource players --no-spec
npx nest g resource matches --no-spec
npx nest g resource rounds  --no-spec
npx nest g resource moves   --no-spec
```

### Otros comandos útiles del CLI
```bash
npx nest g module nombre          # solo módulo
npx nest g service nombre         # solo servicio
npx nest g controller nombre      # solo controlador
npx nest g filter common/filters/http-exception --no-spec   # filtro de excepciones
```

### Carpeta común (enums, helpers)
```
src/common/enums/       ← enums compartidos
src/common/filters/     ← filtros de excepción (opcional)
```

---

## 5. Entidades (plantillas y relaciones)

### 5.1 Entidad básica
```ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, UpdateDateColumn } from 'typeorm';

@Entity('players')                      // nombre de la tabla
export class Player {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  username: string;

  @Column({ default: 0 })
  score: number;

  @Column({ type: 'varchar', nullable: true })
  nickname: string | null;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### 5.2 Enums (⚠️ usar `simple-enum`)
```ts
// src/common/enums/move-type.enum.ts
export enum MoveType {
  ATTACK = 'ATTACK',
  DEFENSE = 'DEFENSE',
  SPECIAL = 'SPECIAL',
}
```
```ts
@Column({ type: 'simple-enum', enum: MoveType })
type: MoveType;
```
> `simple-enum` funciona en **SQLite, Postgres y MySQL**. El tipo `enum` a secas **falla en SQLite**.

### 5.3 Relaciones

**ManyToOne / OneToMany** (la más usada)
```ts
// Round.entity.ts  (lado MANY: tiene la llave foránea)
@ManyToOne(() => Match, (match) => match.rounds, { nullable: false, onDelete: 'CASCADE' })
match: Match;

// Match.entity.ts  (lado ONE)
@OneToMany(() => Round, (round) => round.match)
rounds: Round[];
```

**Dos relaciones a la misma entidad** (ej.: player1 y player2)
```ts
@ManyToOne(() => Player, { nullable: false, eager: true })
player1: Player;

@ManyToOne(() => Player, { nullable: false, eager: true })
player2: Player;
```

**OneToOne**
```ts
@OneToOne(() => Profile, (profile) => profile.user, { cascade: true })
@JoinColumn()                 // va SOLO en un lado (el que tendrá la FK)
profile: Profile;
```

**ManyToMany**
```ts
@ManyToMany(() => Course, (course) => course.students)
@JoinTable()                  // va SOLO en un lado
courses: Course[];
```

### 5.4 Opciones importantes de relaciones
| Opción | Qué hace |
|---|---|
| `eager: true` | Carga la relación automáticamente en `find*` |
| `nullable: false` | La FK es obligatoria |
| `onDelete: 'CASCADE'` | Si borras el padre, se borran los hijos (evita errores de FK al hacer DELETE) |
| `cascade: true` | Guarda automáticamente la entidad relacionada |

### 5.5 Enums de estado
```ts
export enum MatchStatus { ACTIVE = 'ACTIVE', FINISHED = 'FINISHED' }

@Column({ type: 'simple-enum', enum: MatchStatus, default: MatchStatus.ACTIVE })
status: MatchStatus;
```

---

## 6. DTOs y validaciones

### 6.1 DTO de creación
```ts
import { IsString, IsNotEmpty, IsInt, IsEnum, IsOptional, Min, Max, MinLength } from 'class-validator';

export class CreatePlayerDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(3)
  username: string;

  @IsOptional()
  @IsInt()
  @Min(0)
  score?: number;
}
```

### 6.2 DTO con IDs de relaciones
```ts
export class CreateMoveDto {
  @IsInt() @Min(1)
  roundId: number;

  @IsInt() @Min(1)
  playerId: number;

  @IsEnum(MoveType)
  type: MoveType;
}
```

### 6.3 DTO de actualización
Ya viene generado con `PartialType` (todos los campos opcionales):
```ts
import { PartialType } from '@nestjs/mapped-types';
export class UpdatePlayerDto extends PartialType(CreatePlayerDto) {}
```

### 6.4 Decoradores de class-validator más usados
| Decorador | Uso |
|---|---|
| `@IsString()` `@IsNumber()` `@IsInt()` `@IsBoolean()` | Tipo |
| `@IsNotEmpty()` | No vacío |
| `@IsOptional()` | Campo opcional |
| `@Min(n)` `@Max(n)` | Rango numérico |
| `@MinLength(n)` `@MaxLength(n)` | Largo de texto |
| `@IsEmail()` | Correo |
| `@IsEnum(MiEnum)` | Valor de un enum |
| `@IsDateString()` | Fecha ISO |
| `@IsArray()` `@ArrayNotEmpty()` | Arreglos |
| `@IsPositive()` | Número > 0 |

> Si `transform: true` está activo y usas `@Param('id') id: number`, usa `ParseIntPipe` (ver controladores).

---

## 7. Módulos

Cada módulo registra **sus** entidades con `forFeature`. Si el servicio usa repositorios de **otras** entidades, agrégalas también.

```ts
@Module({
  imports: [TypeOrmModule.forFeature([Move, Round, Match, Player])],
  controllers: [MovesController],
  providers: [MovesService],
})
export class MovesModule {}
```

### Dos formas de usar datos de otro módulo
1. **Simple (recomendada en parcial):** registrar las entidades extra en `forFeature` e inyectar el `Repository` directamente. Evita dependencias circulares.
2. **Con servicios:** el módulo dueño hace `exports: [PlayersService]` y el otro lo importa en `imports: [PlayersModule]`.

```ts
// players.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Player])],
  controllers: [PlayersController],
  providers: [PlayersService],
  exports: [PlayersService, TypeOrmModule],   // exporta lo que otros necesitan
})
export class PlayersModule {}
```

---

## 8. Servicios (CRUD)

### 8.1 Plantilla completa
```ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Player } from './entities/player.entity';
import { CreatePlayerDto } from './dto/create-player.dto';
import { UpdatePlayerDto } from './dto/update-player.dto';

@Injectable()
export class PlayersService {
  constructor(
    @InjectRepository(Player)
    private readonly playerRepo: Repository<Player>,
  ) {}

  async create(dto: CreatePlayerDto) {
    const exists = await this.playerRepo.findOne({ where: { username: dto.username } });
    if (exists) throw new ConflictException(`El username "${dto.username}" ya existe`);

    const player = this.playerRepo.create(dto);
    return this.playerRepo.save(player);
  }

  findAll() {
    return this.playerRepo.find();
  }

  async findOne(id: number) {
    const player = await this.playerRepo.findOne({ where: { id } });
    if (!player) throw new NotFoundException(`Player ${id} no existe`);
    return player;
  }

  async update(id: number, dto: UpdatePlayerDto) {
    const player = await this.findOne(id);
    Object.assign(player, dto);
    return this.playerRepo.save(player);
  }

  async remove(id: number) {
    const player = await this.findOne(id);
    await this.playerRepo.remove(player);
    return { message: `Player ${id} eliminado` };
  }
}
```

### 8.2 CRUD con relaciones (crear con IDs)
```ts
async create(dto: CreateMatchDto) {
  const player1 = await this.playerRepo.findOne({ where: { id: dto.player1Id } });
  const player2 = await this.playerRepo.findOne({ where: { id: dto.player2Id } });
  if (!player1 || !player2) throw new NotFoundException('Jugador no encontrado');

  const match = this.matchRepo.create({ player1, player2 });
  return this.matchRepo.save(match);
}
```

### 8.3 Cheat sheet de consultas TypeORM (0.3.x)
```ts
// Buscar uno / varios
repo.findOne({ where: { id } });
repo.find({ where: { status: 'ACTIVE' } });

// Con relaciones
repo.findOne({ where: { id }, relations: ['match', 'moves', 'moves.player'] });

// Filtrar por campo de relación (where anidado)
moveRepo.findOne({ where: { player: { id: 5 }, round: { id: 3 } } });
moveRepo.findOne({ where: { player: { id: 5 }, round: { match: { id: 1 } } } });

// Ordenar y limitar
repo.find({ order: { id: 'DESC' }, take: 10 });
repo.findOne({ where: {...}, order: { id: 'DESC' } });     // el último

// Contar / existir
repo.count({ where: { round: { id: 3 } } });
repo.exist({ where: { username: 'ana' } });

// Operadores
import { Not, In, LessThan, MoreThan, Between, IsNull } from 'typeorm';
repo.find({ where: { id: Not(5) } });
repo.find({ where: { status: In(['A', 'B']) } });
repo.find({ where: { stock: MoreThan(0) } });

// Crear, guardar, borrar
const e = repo.create({ ...datos });
await repo.save(e);
await repo.remove(e);
await repo.delete(id);        // sin cargar la entidad

// Varias operaciones atómicas (transacción)
await this.dataSource.transaction(async (manager) => {
  await manager.save(entidadA);
  await manager.save(entidadB);
});
// (inyecta DataSource: constructor(private dataSource: DataSource) {})
```

### 8.4 Excepciones de Nest (importar de `@nestjs/common`)
| Excepción | Código | Cuándo usarla |
|---|---|---|
| `BadRequestException` | 400 | Violación de regla de negocio / datos inválidos |
| `NotFoundException` | 404 | La entidad no existe |
| `ConflictException` | 409 | Duplicados / estado en conflicto |
| `ForbiddenException` | 403 | No tiene permiso para la acción |
| `UnprocessableEntityException` | 422 | Datos válidos en forma pero inválidos en lógica |

> Usa **mensajes claros**: `throw new BadRequestException('El jugador ya registró un movimiento en esta ronda')`.

---

## 9. Controladores

```ts
import { Controller, Get, Post, Patch, Delete, Body, Param, ParseIntPipe, HttpCode } from '@nestjs/common';

@Controller('players')
export class PlayersController {
  constructor(private readonly playersService: PlayersService) {}

  @Post()
  create(@Body() dto: CreatePlayerDto) {
    return this.playersService.create(dto);
  }

  @Get()
  findAll() {
    return this.playersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.playersService.findOne(id);
  }

  @Patch(':id')
  update(@Param('id', ParseIntPipe) id: number, @Body() dto: UpdatePlayerDto) {
    return this.playersService.update(id, dto);
  }

  @Delete(':id')
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.playersService.remove(id);
  }
}
```

> ⚠️ **Siempre `ParseIntPipe`** en los `@Param('id')`. Sin él, el id llega como string.
> ⚠️ El controlador **solo** recibe y delega. **Nada de lógica** aquí; todo va en el servicio.

---

## 10. Reglas de negocio (catálogo de patrones)

Casi todos los parciales usan combinaciones de estos patrones. Identifica cuál aplica y copia la estructura.

### Patrón A — "Solo uno por X" (unicidad por contexto)
*Ej.: un movimiento por jugador por ronda; un voto por usuario; una reserva por mesa y hora.*
```ts
const existing = await this.moveRepo.findOne({
  where: { round: { id: dto.roundId }, player: { id: dto.playerId } },
});
if (existing) throw new BadRequestException('El jugador ya registró un movimiento en esta ronda');
```

### Patrón B — "No repetir consecutivamente"
*Ej.: mismo movimiento dos veces seguidas; misma canción seguida.*
```ts
const last = await this.moveRepo.findOne({
  where: { player: { id: dto.playerId }, round: { match: { id: matchId } } },
  order: { id: 'DESC' },                       // el más reciente
});
if (last && last.type === dto.type) {
  throw new BadRequestException('No puedes repetir el mismo movimiento consecutivamente');
}
```

### Patrón C — "Esperar a que estén todos para procesar"
*Ej.: resolver cuando ambos jugaron; cerrar votación cuando todos votaron.*
```ts
const total = await this.moveRepo.count({ where: { round: { id: round.id } } });
if (total < 2) return { message: 'Movimiento registrado. Esperando al otro jugador' };
return this.resolveRound(round);               // total === 2
```

### Patrón D — Tabla de reglas (quién vence a quién)
*Ej.: piedra-papel-tijera, elementos, tipos.* Usa **diccionarios**, no `if` anidados:
```ts
const BEATS: Record<MoveType, MoveType> = {
  [MoveType.ATTACK]: MoveType.DEFENSE,   // Ataque vence a Defensa
  [MoveType.DEFENSE]: MoveType.SPECIAL,  // Defensa vence a Especial
  [MoveType.SPECIAL]: MoveType.ATTACK,   // Especial vence a Ataque
};
const DAMAGE: Record<MoveType, number> = {
  [MoveType.ATTACK]: 10,
  [MoveType.DEFENSE]: 0,
  [MoveType.SPECIAL]: 20,
};
// A gana si BEATS[A.type] === B.type  →  el perdedor recibe DAMAGE[A.type]
```

### Patrón E — Máquina de estados
*Ej.: PENDING → RESOLVED; ACTIVE → FINISHED; DRAFT → PUBLISHED.*
```ts
if (round.status !== RoundStatus.PENDING) throw new BadRequestException('La ronda ya fue resuelta');
if (match.status !== MatchStatus.ACTIVE)  throw new BadRequestException('El duelo ya terminó');
```

### Patrón F — Pertenencia
*Ej.: el jugador debe pertenecer al duelo; el estudiante estar inscrito en el curso.*
```ts
const belongs = [match.player1.id, match.player2.id].includes(dto.playerId);
if (!belongs) throw new BadRequestException('El jugador no pertenece a este duelo');
```

### Patrón G — Límites y capacidad
*Ej.: stock, cupos, saldo, máximo de N elementos.*
```ts
if (product.stock < dto.quantity) throw new BadRequestException('Stock insuficiente');
const count = await this.repo.count({ where: { course: { id: courseId } } });
if (count >= course.capacity) throw new ConflictException('Cupo lleno');
if (account.balance < amount) throw new BadRequestException('Saldo insuficiente');
```

### Patrón H — Fechas / solapamientos
```ts
import { LessThan, MoreThan } from 'typeorm';
const overlap = await this.repo.findOne({
  where: { room: { id }, start: LessThan(dto.end), end: MoreThan(dto.start) },
});
if (overlap) throw new ConflictException('Horario ocupado');
```

### Patrón I — Valores calculados / acumulados
```ts
loser.hp = Math.max(0, loser.hp - damage);       // nunca negativo
if (loser.hp === 0) match.status = MatchStatus.FINISHED;
```

### 📌 Dónde poner las reglas
- En el **servicio** del recurso que "dispara" la acción (el que recibe el `POST` principal).
- Valida en este orden: **existencia → estado → pertenencia → reglas específicas → guardar → efectos secundarios**.
- Si una regla afecta varias tablas, usa una **transacción** (ver 8.3).

---

## 11. Postman

### 11.1 Organización
1. Crea una **Collection** con el nombre del parcial.
2. Agrega una **variable de colección** `baseUrl` = `http://localhost:3000`.
3. Crea **carpetas** por módulo: `Players`, `Matches`, `Rounds`, `Moves`.
4. Dentro de cada una, un request por endpoint: `POST`, `GET all`, `GET by id`, `PATCH`, `DELETE`.
5. Usa `{{baseUrl}}/players` en las URL.

### 11.2 Guardar IDs automáticamente (pestaña **Tests** del request)
```js
const json = pm.response.json();
pm.collectionVariables.set("player1Id", json.id);
```
Luego usa `{{player1Id}}` en los body/URL de los siguientes requests.

### 11.3 Carpeta de "Flujo de reglas de negocio" (¡muy importante!)
Crea una carpeta extra con el flujo **en orden** que demuestra cada regla, incluyendo los casos que **deben fallar**:

| # | Request | Resultado esperado |
|---|---|---|
| 1 | Crear jugador 1 | 201 |
| 2 | Crear jugador 2 | 201 |
| 3 | Crear duelo | 201 |
| 4 | Crear ronda | 201 |
| 5 | J1 juega ATTACK | 201 (esperando) |
| 6 | J1 juega otra vez en la misma ronda | **400** (regla 1) |
| 7 | J2 juega DEFENSE | 201 (ronda resuelta, J2 pierde 10 HP) |
| 8 | Crear ronda 2 | 201 |
| 9 | J1 juega ATTACK de nuevo | **400** (regla 2: repetido) |
| 10 | J1 juega SPECIAL | 201 |
| 11 | J2 juega ATTACK | 201 (Especial vence Ataque → J2 pierde 20 HP) |
| … | Casos de empate, ronda inexistente, jugador fuera del duelo, body inválido | 400 / 404 |

### 11.4 Tests básicos de Postman (opcional, suma puntos)
```js
pm.test("Status 201", () => pm.response.to.have.status(201));
pm.test("Rechaza duplicado", () => pm.response.to.have.status(400));
```

### 11.5 Exportar
**Collection → ⋯ → Export → Collection v2.1** → guarda el `.json` en la carpeta `/postman` del repo:
```
postman/Parcial_NestJS.postman_collection.json
```

---

## 12. Git y entrega

```bash
git status
git add .
git commit -m "feat: módulo players con CRUD"
git push origin main       # o la rama que indique el classroom
```

### Commits recomendados durante el parcial
```bash
git commit -m "chore: configuración inicial typeorm y validation pipe"
git commit -m "feat: entidades"
git commit -m "feat: crud players y matches"
git commit -m "feat: crud rounds y moves"
git commit -m "feat: reglas de negocio"
git commit -m "docs: colección postman"
```

### Antes del push final
- Confirma que `node_modules/`, `dist/`, `.env` y `*.db` están en `.gitignore`.
- Si el profesor necesita `.env`, sube un `.env.example`.
- Verifica en GitHub que el commit llegó (**antes** de la hora límite).

---

## 13. Errores comunes y soluciones

| Error | Causa | Solución |
|---|---|---|
| `Nest can't resolve dependencies of the XService (?)` | Falta registrar el repositorio | Agregar la entidad en `TypeOrmModule.forFeature([...])` del módulo |
| `Data type "enum" ... is not supported by "sqlite"` | Usaste `type: 'enum'` | Cambiar a `type: 'simple-enum'` |
| `Data type "timestamp" not supported by sqlite` | Tipo de fecha no soportado | Usar `@CreateDateColumn()` o `type: 'datetime'` |
| `No metadata for "X" was found` | La entidad no está cargada | Verificar `autoLoadEntities: true` y que la entidad esté en `forFeature` |
| `SQLITE_CONSTRAINT: FOREIGN KEY constraint failed` | Borras algo con hijos | Añadir `onDelete: 'CASCADE'` en el `@ManyToOne` hijo |
| `Cannot read properties of undefined (reading 'id')` | La relación no se cargó | Añadir `relations: [...]` en el `find` o `eager: true` |
| `A circular dependency between modules` | Módulos que se importan mutuamente | Usar repositorios directos con `forFeature` o `forwardRef(() => Modulo)` |
| El body llega vacío o sin validar | Falta `ValidationPipe` | Agregar `useGlobalPipes` en `main.ts` |
| `property X should not exist` | `forbidNonWhitelisted: true` y envías campos extra | Quitar el campo del body o agregarlo al DTO |
| `id` llega como string y falla la búsqueda | Falta `ParseIntPipe` | `@Param('id', ParseIntPipe) id: number` |
| `Cannot find module 'better-sqlite3'` / falla compilar | Driver no instalado o no compila | `npm i better-sqlite3` o usar `sqlite3` y `type: 'sqlite'` |
| `EADDRINUSE: port 3000` | Otro proceso usa el puerto | Cerrar el proceso o cambiar puerto en `main.ts` |
| Cambios en entidad no se reflejan | BD vieja | Borrar el archivo `.db` y reiniciar (con `synchronize: true` se recrea) |
| `Cannot POST /ruta` (404) | Ruta mal escrita o módulo no importado | Revisar `@Controller('ruta')` y que el módulo esté en `AppModule` |

---

## 14. Checklist final

**Funcionamiento**
- [ ] `npm install` + `npm run start:dev` arrancan **sin errores**
- [ ] Borré la BD local y vuelve a funcionar desde cero

**Entidades (20%)**
- [ ] Todas las entidades del enunciado creadas
- [ ] Relaciones correctas (ManyToOne/OneToMany/etc.)
- [ ] Enums como `simple-enum`
- [ ] Estados por defecto definidos

**Módulos (20%)**
- [ ] Un módulo por entidad
- [ ] `TypeOrmModule.forFeature` correcto en cada uno
- [ ] Todos importados en `AppModule`

**Servicios y controladores (30%)**
- [ ] CRUD completo (create, findAll, findOne, update, remove) en **todas** las entidades
- [ ] `ParseIntPipe` en los params
- [ ] Lógica solo en servicios

**Reglas de negocio (20%)**
- [ ] Cada regla del enunciado implementada
- [ ] Probé en Postman el caso feliz **y** el caso de error de cada regla

**Calidad (10%)**
- [ ] `ValidationPipe` global + decoradores en DTOs
- [ ] Excepciones HTTP apropiadas con mensajes claros
- [ ] Sin `console.log`, sin código comentado ni imports sin usar
- [ ] Nombres consistentes (inglés o español, pero no mezclados)

**Entrega**
- [ ] JSON de Postman exportado y dentro del repo
- [ ] `git push` hecho antes de la hora límite
- [ ] Verifiqué el commit en GitHub

---

## 15. Ejemplo completo: duelos 1 vs 1

> Enunciado de práctica: sesiones de combate 1 vs 1. Cada jugador envía un movimiento (ATTACK / DEFENSE / SPECIAL) por ronda. Reglas: (1) un movimiento por jugador por ronda, (2) no repetir el mismo movimiento consecutivamente, (3) resolver solo cuando ambos jugaron, (4) Ataque vence Defensa (defensor −10 HP), Defensa vence Especial (nadie pierde), Especial vence Ataque (atacante −20 HP), iguales = empate.

### 15.1 Estructura
```
src/
├── common/enums/
│   ├── move-type.enum.ts
│   ├── match-status.enum.ts
│   └── round-status.enum.ts
├── players/
├── matches/
├── rounds/
├── moves/
├── app.module.ts
└── main.ts
postman/Parcial_NestJS.postman_collection.json
```

### 15.2 Enums
```ts
// move-type.enum.ts
export enum MoveType { ATTACK = 'ATTACK', DEFENSE = 'DEFENSE', SPECIAL = 'SPECIAL' }

// match-status.enum.ts
export enum MatchStatus { ACTIVE = 'ACTIVE', FINISHED = 'FINISHED' }

// round-status.enum.ts
export enum RoundStatus { PENDING = 'PENDING', RESOLVED = 'RESOLVED' }
```

### 15.3 Entidades
```ts
// players/entities/player.entity.ts
@Entity('players')
export class Player {
  @PrimaryGeneratedColumn() id: number;
  @Column({ unique: true }) username: string;
  @CreateDateColumn() createdAt: Date;
}
```
```ts
// matches/entities/match.entity.ts
@Entity('matches')
export class Match {
  @PrimaryGeneratedColumn() id: number;

  @ManyToOne(() => Player, { nullable: false, eager: true })
  player1: Player;

  @ManyToOne(() => Player, { nullable: false, eager: true })
  player2: Player;

  @Column({ default: 100 }) player1Hp: number;
  @Column({ default: 100 }) player2Hp: number;

  @Column({ type: 'simple-enum', enum: MatchStatus, default: MatchStatus.ACTIVE })
  status: MatchStatus;

  @OneToMany(() => Round, (round) => round.match)
  rounds: Round[];

  @CreateDateColumn() createdAt: Date;
}
```
```ts
// rounds/entities/round.entity.ts
@Entity('rounds')
export class Round {
  @PrimaryGeneratedColumn() id: number;

  @ManyToOne(() => Match, (match) => match.rounds, { nullable: false, onDelete: 'CASCADE' })
  match: Match;

  @Column() number: number;

  @Column({ type: 'simple-enum', enum: RoundStatus, default: RoundStatus.PENDING })
  status: RoundStatus;

  @Column({ type: 'varchar', nullable: true })
  result: string | null;

  @OneToMany(() => Move, (move) => move.round)
  moves: Move[];
}
```
```ts
// moves/entities/move.entity.ts
@Entity('moves')
export class Move {
  @PrimaryGeneratedColumn() id: number;

  @ManyToOne(() => Round, (round) => round.moves, { nullable: false, onDelete: 'CASCADE' })
  round: Round;

  @ManyToOne(() => Player, { nullable: false, eager: true })
  player: Player;

  @Column({ type: 'simple-enum', enum: MoveType })
  type: MoveType;

  @CreateDateColumn() createdAt: Date;
}
```

### 15.4 DTOs
```ts
// players
export class CreatePlayerDto {
  @IsString() @IsNotEmpty() @MinLength(3) username: string;
}

// matches
export class CreateMatchDto {
  @IsInt() @Min(1) player1Id: number;
  @IsInt() @Min(1) player2Id: number;
}

// rounds
export class CreateRoundDto {
  @IsInt() @Min(1) matchId: number;
}

// moves
export class CreateMoveDto {
  @IsInt() @Min(1) roundId: number;
  @IsInt() @Min(1) playerId: number;
  @IsEnum(MoveType) type: MoveType;
}
```

### 15.5 Módulos
```ts
// players.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Player])],
  controllers: [PlayersController],
  providers: [PlayersService],
})
export class PlayersModule {}

// matches.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Match, Player])],
  controllers: [MatchesController],
  providers: [MatchesService],
})
export class MatchesModule {}

// rounds.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Round, Match])],
  controllers: [RoundsController],
  providers: [RoundsService],
})
export class RoundsModule {}

// moves.module.ts  (necesita las 4 entidades para las reglas)
@Module({
  imports: [TypeOrmModule.forFeature([Move, Round, Match, Player])],
  controllers: [MovesController],
  providers: [MovesService],
})
export class MovesModule {}
```

### 15.6 MatchesService.create (validaciones)
```ts
async create(dto: CreateMatchDto) {
  if (dto.player1Id === dto.player2Id) {
    throw new BadRequestException('Un jugador no puede enfrentarse a sí mismo');
  }
  const player1 = await this.playerRepo.findOne({ where: { id: dto.player1Id } });
  const player2 = await this.playerRepo.findOne({ where: { id: dto.player2Id } });
  if (!player1) throw new NotFoundException(`Player ${dto.player1Id} no existe`);
  if (!player2) throw new NotFoundException(`Player ${dto.player2Id} no existe`);

  return this.matchRepo.save(this.matchRepo.create({ player1, player2 }));
}
```

### 15.7 RoundsService.create (número automático)
```ts
async create(dto: CreateRoundDto) {
  const match = await this.matchRepo.findOne({ where: { id: dto.matchId } });
  if (!match) throw new NotFoundException(`Match ${dto.matchId} no existe`);
  if (match.status === MatchStatus.FINISHED) {
    throw new BadRequestException('El duelo ya terminó');
  }

  const pending = await this.roundRepo.findOne({
    where: { match: { id: match.id }, status: RoundStatus.PENDING },
  });
  if (pending) throw new ConflictException('Ya existe una ronda pendiente en este duelo');

  const count = await this.roundRepo.count({ where: { match: { id: match.id } } });
  return this.roundRepo.save(this.roundRepo.create({ match, number: count + 1 }));
}
```

### 15.8 MovesService — el corazón del parcial
```ts
@Injectable()
export class MovesService {
  constructor(
    @InjectRepository(Move)  private readonly moveRepo: Repository<Move>,
    @InjectRepository(Round) private readonly roundRepo: Repository<Round>,
    @InjectRepository(Match) private readonly matchRepo: Repository<Match>,
    @InjectRepository(Player) private readonly playerRepo: Repository<Player>,
  ) {}

  async create(dto: CreateMoveDto) {
    // 1. Existencia
    const round = await this.roundRepo.findOne({
      where: { id: dto.roundId },
      relations: ['match', 'match.player1', 'match.player2', 'moves', 'moves.player'],
    });
    if (!round) throw new NotFoundException(`Round ${dto.roundId} no existe`);

    const player = await this.playerRepo.findOne({ where: { id: dto.playerId } });
    if (!player) throw new NotFoundException(`Player ${dto.playerId} no existe`);

    // 2. Estado
    if (round.status !== RoundStatus.PENDING) {
      throw new BadRequestException('La ronda ya fue resuelta');
    }
    if (round.match.status !== MatchStatus.ACTIVE) {
      throw new BadRequestException('El duelo ya terminó');
    }

    // 3. Pertenencia
    const { player1, player2 } = round.match;
    if (![player1.id, player2.id].includes(player.id)) {
      throw new BadRequestException('El jugador no pertenece a este duelo');
    }

    // 4. REGLA 1: un movimiento por jugador por ronda
    if (round.moves.some((m) => m.player.id === player.id)) {
      throw new BadRequestException('El jugador ya registró un movimiento en esta ronda');
    }

    // 5. REGLA 2: no repetir movimiento consecutivo
    const last = await this.moveRepo.findOne({
      where: { player: { id: player.id }, round: { match: { id: round.match.id } } },
      order: { id: 'DESC' },
    });
    if (last && last.type === dto.type) {
      throw new BadRequestException('No puedes repetir el mismo movimiento de forma consecutiva');
    }

    // 6. Guardar
    const move = await this.moveRepo.save(
      this.moveRepo.create({ round, player, type: dto.type }),
    );

    // 7. REGLA 3: resolver solo si ambos jugaron
    const moves = [...round.moves, move];
    if (moves.length < 2) {
      return { move, message: 'Movimiento registrado. Esperando al otro jugador' };
    }

    const resolution = await this.resolveRound(round, moves);
    return { move, ...resolution };
  }

  // REGLA 4: cálculo del resultado
  private async resolveRound(round: Round, moves: Move[]) {
    const match = round.match;
    const [a, b] = moves;

    let result: string;
    let loser: Move | null = null;
    let damage = 0;

    if (a.type === b.type) {
      result = 'DRAW';
    } else {
      const winner = BEATS[a.type] === b.type ? a : b;
      loser = winner === a ? b : a;
      damage = DAMAGE[winner.type];                 // 10 / 0 / 20
      result = `WINNER:${winner.player.id}`;
    }

    // Aplicar daño al perdedor
    if (loser && damage > 0) {
      if (loser.player.id === match.player1.id) {
        match.player1Hp = Math.max(0, match.player1Hp - damage);
      } else {
        match.player2Hp = Math.max(0, match.player2Hp - damage);
      }
      if (match.player1Hp === 0 || match.player2Hp === 0) {
        match.status = MatchStatus.FINISHED;
      }
    }

    round.status = RoundStatus.RESOLVED;
    round.result = result;

    await this.matchRepo.save(match);
    await this.roundRepo.save({ id: round.id, status: round.status, result: round.result });

    return {
      message: 'Ronda resuelta',
      result,
      damage,
      player1Hp: match.player1Hp,
      player2Hp: match.player2Hp,
      matchStatus: match.status,
    };
  }
}

// Constantes (arriba del archivo o en /common/constants)
const BEATS: Record<MoveType, MoveType> = {
  [MoveType.ATTACK]: MoveType.DEFENSE,
  [MoveType.DEFENSE]: MoveType.SPECIAL,
  [MoveType.SPECIAL]: MoveType.ATTACK,
};
const DAMAGE: Record<MoveType, number> = {
  [MoveType.ATTACK]: 10,
  [MoveType.DEFENSE]: 0,
  [MoveType.SPECIAL]: 20,
};
```

> **Por qué funciona la tabla `DAMAGE`:** siempre pierde HP el perdedor, y el daño depende del movimiento **ganador**: Ataque gana → defensor −10; Defensa gana → 0; Especial gana → atacante −20.

### 15.9 AppModule final
```ts
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'better-sqlite3',
      database: 'database.db',
      autoLoadEntities: true,
      synchronize: true,
    }),
    PlayersModule,
    MatchesModule,
    RoundsModule,
    MovesModule,
  ],
})
export class AppModule {}
```

### 15.10 Cómo adaptar este ejemplo a otro enunciado

| Si el enunciado habla de… | Mapea así |
|---|---|
| Biblioteca: libros, socios, préstamos | `Book`, `Member`, `Loan`. Regla: máx. N préstamos activos, libro no prestado dos veces (patrones A, E, G) |
| Tienda: productos, pedidos, items | `Product`, `Order`, `OrderItem`. Regla: stock suficiente, total calculado (patrones G, I) |
| Hospital: pacientes, médicos, citas | `Patient`, `Doctor`, `Appointment`. Regla: sin solapamiento de horario (patrón H) |
| Torneo: equipos, partidos, resultados | `Team`, `Match`, `Result`. Regla: un equipo no juega contra sí mismo, resolver al tener ambos marcadores (patrones C, F) |
| Votaciones: usuarios, encuestas, votos | `User`, `Poll`, `Vote`. Regla: un voto por usuario por encuesta (patrón A) |
| Banco: cuentas, transacciones | `Account`, `Transaction`. Regla: saldo suficiente, transacción atómica (patrones G, I + `dataSource.transaction`) |

**Receta universal:** 
1. Entidad "contenedor" (Match, Order, Course) → 2. Entidad "participante" (Player, Customer, Student) → 3. Entidad "evento" (Move, Loan, Enrollment) donde viven casi todas las reglas.

---

## 🧠 Recordatorios finales

1. **Que arranque** antes que todo lo demás.
2. **Commits frecuentes.**
3. **Probar cada módulo en Postman apenas lo termines**, no al final.
4. Lee el enunciado de nuevo a mitad del examen: es fácil olvidar una regla.
5. Si algo se traba más de 10 minutos, simplifícalo y sigue; vuelve después.
6. Nombres claros, mensajes de error claros, código limpio: el 10% de calidad es fácil de conseguir.

**¡Éxito en el parcial! 🚀**
