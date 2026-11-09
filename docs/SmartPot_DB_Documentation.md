<!-- portada
eyebrow: Component Technical Documentation
titulo: SmartPot-DB-Mongo (Legacy Stack A)
acento: DB
subtitulo: Legacy Document Database and Data Migration
bajada: MongoDB 8 with $jsonSchema validation across 10 collections, least-privilege application credentials, optional demo seed data, and idempotent schema migrations.
documento: SmartPot-DB-Mongo Documentation
version: 1.1 · October 2026
equipo: SmartPotTech
proyecto: smartpot.app
-->

# SmartPot-DB-Mongo (Legacy Stack A)

> [!WARNING]
> **Legacy Component (Stack A)**: This service has been superseded in production by the canonical relational engine in PostgreSQL 17 + TimescaleDB 2.17.2 ([`SmartPot-DB`](https://github.com/SmartPotTech/SmartPot-DB)). This document preserves the historical schema architecture and migration mechanics of the original MongoDB 8 implementation.

## Document Summary

| Field                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Proyecto                       | SmartPot · [smartpot.app](https://smartpot.app)                                                                                                                                                                                                                                                                                                                                                                                                         |
| Componente                     | [SmartPot-DB](https://github.com/SmartPotTech/SmartPot-DB)                                                                                                                                                                                                                                                                                                                                                                                              |
| Versión                        | 1.1 · octubre 2026                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Scope                        | Collections and validators, initialization, migration, demo seed fixtures, configuration, testing, and operations                                                                                                                                                                                                                                                                                                                                                    |
| Platform Documentation | [Technical Documentation](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Technical_Documentation.md), [Project Journey](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Project_Journey.md), [Software Lifecycle](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Software_Lifecycle.md), and [Global Diagrams](https://github.com/SmartPotTech/.github/blob/main/docs/README.md#global-diagrams) |
| Maintenance                  | Generated from `docs/` in this repository via `.github/docs/tools`; updated with every component change                                                                                                                                                                                                                                                                                                                   |

<!-- parte: PART I | Component Overview -->

## 1. Purpose

### In Plain Terms

La base guarda todo lo que SmartPot necesita recordar: las cuentas, los cultivos reales y virtuales con su forma, sus
lecturas, actuadores y órdenes, las notificaciones, los vínculos de Telegram y la configuración de las simulaciones.
Rechaza los documentos que no traen lo que la API necesita, y la API se conecta con un usuario que solo puede leer y
escribir su propia base.

## 2. Component Architecture

<!-- diagrama: SmartPot_DB_Global_Component | titulo=SmartPot-DB Component Architecture | lamina=H -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}, "layout": "elk", "elk": {"nodePlacementStrategy": "BRANDES_KOEPF", "mergeEdges": false, "cycleBreakingStrategy": "GREEDY"}}}%%
flowchart LR
  api["SmartPot-API-Java"]
  subgraph mongo["SmartPot-DB-Mongo · MongoDB 8.0/7.0"]
    direction TB
    auth["Authentication<br/>SCRAM-SHA-256"]
    wired["WiredTiger Storage Engine<br/>snappy compression"]
    schema["Schema Validation<br/>JSON schema collection validators"]
    init["01_init.js<br/>Database creation & index provisioning"]
  end
  api -->|"mongodb:// :27017"| auth
  auth --> schema --> wired
  init --> schema
  classDef core fill:#067A52,stroke:#0B3D2B,color:#FFFFFF
  classDef leaf fill:#DDF5EA,stroke:#067A52,color:#17261F
  class api core
  class mongo,auth,wired,schema,init leaf
```

## 3. Collections and Schemas

<!-- diagrama: SmartPot_DB_01_Collections | titulo=Colecciones y reglas principales -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}}}%%
classDiagram
  class users {
    +ObjectId _id
    +String email
    +String passwordHash
    +String role
  }
  class crops {
    +ObjectId _id
    +ObjectId ownerId
    +String name
    +String species
    +String kind
    +Document placement
    +Document device
    +Document health
  }
  class readings {
    +ObjectId _id
    +ObjectId cropId
    +Date timestamp
    +Double temperature
    +Double humidity
    +Double brightness
    +Double ph
    +Double tds
    +Double soilMoisture
  }
  class actuators {
    +ObjectId _id
    +ObjectId cropId
    +String type
    +String state
  }
  class commands {
    +ObjectId _id
    +ObjectId cropId
    +ObjectId actuatorId
    +String action
    +Integer durationSeconds
    +String status
  }
  users "1" --> "*" crops : owns
  crops "1" --> "*" readings : telemetry
  crops "1" --> "*" actuators : controls
  crops "1" --> "*" commands : dispatches
```

| Colección               | Campos obligatorios                                                               | Reglas                                                                                                                                   |
|-------------------------|-----------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `users`                 | `email`, `passwordHash`, `role`, `createdAt`                                      | Hash BCrypt; rol `USER` o `ADMIN`                                                                                                        |
| `crops`                 | `ownerId`, `name`, `type`, `automationEnabled`, `createdAt`                       | Seis especies; `kind` `REAL` o `VIRTUAL`; `form` `POT`, `NFT`, `TOWER` o `RAFT`; `placement` con lugar, exposición y coordenadas válidas |
| `readings`              | `cropId`, `measuredAt`, `measures`                                                | Origen `MQTT` o `HTTP`; TTL de un año que crea la API                                                                                    |
| `actuators`             | `cropId`, `type`, `active`                                                        | Seis tipos                                                                                                                               |
| `commands`              | `cropId`, `actuatorId`, `actuatorType`, `action`, `status`, `source`, `createdAt` | Estados y orígenes del catálogo; TTL de 180 días                                                                                         |
| `notifications`         | `userId`, `type`, `title`, `message`, `read`, `createdAt`                         | TTL de 90 días                                                                                                                           |
| `password_reset_tokens` | `tokenHash`, `userId`, `expiresAt`                                                | Solo el SHA-256 del token                                                                                                                |
| `channel_links`         | `userId`, `type`, `address`, `enabled`, `events`, `linkedAt`, `failures`          | Canal `TELEGRAM`                                                                                                                         |
| `crop_channels`         | `cropId`, `ownerId`, `type`, `enabled`, `delivery`                                | `INSTANT` o `DIGEST`; resumen cada 1 a 24 h; `HH:mm`; hasta 10 chats                                                                     |
| `virtual_devices`       | `cropId`, `ownerId`, `mode`, `intervalSeconds`, `createdAt`, `updatedAt`          | Solo cultivos virtuales; `active` en `false` durante la pausa                                                                            |

<!-- parte: PART II | Operations & Migration -->

## 4. Migration to PostgreSQL & TimescaleDB

<!-- diagrama: SmartPot_DB_02_Migration_Flow | titulo=Data Migration Pipeline from MongoDB to TimescaleDB -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}}}%%
sequenceDiagram
  autonumber
  participant M as Legacy MongoDB 8.0/7.0
  participant MIG as Migration Script (migrate.js)
  participant PG as PostgreSQL 17 + TimescaleDB
  MIG->>M: Query all user accounts
  M-->>MIG: Users collection cursor
  MIG->>PG: Batch insert into users (preserve ObjectIds & BCrypt hashes)
  MIG->>M: Query crops & nested documents
  M-->>MIG: Crops collection cursor
  MIG->>PG: Insert into crops & JSONB fields
  MIG->>M: Stream historical readings in batches
  M-->>MIG: Readings batches
  MIG->>PG: Binary COPY into readings hypertable
  MIG->>PG: Verify row counts and checksums
```

Los scripts de `init/` solo corren con el volumen vacío. Una base existente recibe los esquemas nuevos con la migración,
que el despliegue central ejecuta después de `up --wait`; a mano:
`docker exec smartpot-db mongosh --quiet /opt/smartpot/migrate.js`.

## 5. Demo Seed Data

Con `SMARTPOT_SEED_DEMO=true` se crea la cuenta `demo@smartpot.app` (contraseña pública de la demo) con dos cultivos *
*reales** que publica el simulador: lechugas en tubos NFT con modo automático y tomates cherry en maceta, cada uno con
48 horas de lecturas. Nunca se activa en producción.

## 6. Configuration & Environment

| Variable                                                           | Uso                                                   |
|--------------------------------------------------------------------|-------------------------------------------------------|
| `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD`         | Cuenta administradora, solo para inicializar y migrar |
| `SMARTPOT_DB_NAME`, `SMARTPOT_DB_USERNAME`, `SMARTPOT_DB_PASSWORD` | Base y usuario de la aplicación                       |
| `SMARTPOT_SEED_DEMO`                                               | Datos demo (`false` por defecto)                      |

## 7. Automated Testing

`sh tests/validate.sh smartpot-db:ci true` levanta la imagen y comprueba las diez colecciones con sus validadores (el lugar del cultivo y los avisos por cultivo incluidos), los
permisos del usuario de la aplicación, los datos demo y la migración sobre una base anterior (una colección sin
validador y otra con un documento antiguo), incluida su idempotencia. En Git Bash de Windows: `MSYS_NO_PATHCONV=1`.

## 8. Production Operations

| Tarea      | Cómo                                                                                                                         |
|------------|------------------------------------------------------------------------------------------------------------------------------|
| Imagen     | `ghcr.io/smartpottech/smartpot-db`: usuario `999`, solo lectura con `tmpfs` en `/tmp`                                        |
| Respaldo   | `mongodump` comprimido desde el servidor; restauración con `mongorestore --drop --archive --gzip`                            |
| Red        | Solo la red interna; nunca se publica el puerto 27017                                                                        |
| Despliegue | Cada cambio en `main` pasa por el CI, publica la imagen y pide el despliegue central de `.github`, que migra tras actualizar |
