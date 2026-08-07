# Informe individual de contribuciones — Juan282007

## 1. Identificación

| Campo | Información |
|---|---|
| Proyecto | Design Software |
| Microservicio | Document Service |
| Repositorio | `design-software-document-db` |
| Colaborador | `Juan282007` |
| Área de trabajo | Base de datos, Liquibase, organización del repositorio e integración por ambientes |
| Tecnologías relacionadas | PostgreSQL, Liquibase, Docker, Git y GitHub |

---

## 2. Introducción

Este documento presenta de manera individual el trabajo realizado por **`Juan282007`** en el repositorio de base de datos del microservicio **Document Service**.

El repositorio tiene como responsabilidad administrar la estructura de PostgreSQL mediante migraciones versionadas con Liquibase. Por esta razón, el trabajo realizado no corresponde al desarrollo de controladores, endpoints, servicios Java o interfaces gráficas. Las contribuciones se concentraron en la preparación de la estructura del proyecto, la integridad referencial de las tablas, la reorganización del repositorio, la integración de la infraestructura y la promoción de los cambios entre los ambientes del proyecto.

Las contribuciones principales identificadas en el historial de Git corresponden a:

- **HU-01:** creación y corrección de la estructura inicial.
- **HU-03:** implementación de las claves foráneas.
- **HU-04:** reorganización del repositorio e integración de la infraestructura.
- Correcciones mediante hotfix.
- Promoción de cambios hacia QA y Staging.
- Participación en ramas de release e integración.

---

## 3. Contexto del repositorio

El repositorio **`design-software-document-db`** contiene la definición versionada de la base de datos utilizada por Document Service.

Su propósito es permitir que la misma estructura pueda reproducirse en los diferentes ambientes sin crear manualmente los objetos desde PostgreSQL.

La organización general del repositorio contempla:

```text
design-software-document-db/
├── 01_ddl/
├── 02_dml/
├── 03_dcl/
├── 04_tcl/
├── 05_rollbacks/
├── changelog/
└── README.md
```

Dentro de esta estructura se administran:

- Extensiones de PostgreSQL.
- Esquemas.
- Tablas.
- Alteraciones y relaciones.
- Roles y permisos.
- Changelogs de Liquibase.
- Scripts de rollback.

El trabajo de `Juan282007` permitió establecer y mejorar la estructura necesaria para que estos elementos pudieran organizarse, relacionarse y promoverse correctamente.

---

# 4. HU-01 — Creación de la estructura inicial

## Propietario

**`Juan282007`**

## Ramas observadas

```text
feature/HU-01-dev
HU-01-qa
HU-01-stg
hotfix/HU-01-structure
```

## Commits observados

```text
Add project structure
fix structure
Add initial structure
hotfix from initial structure
```

## Motivo de la historia de usuario

El proyecto necesitaba una estructura inicial que permitiera organizar de forma uniforme los cambios de base de datos.

Antes de agregar las tablas y sus relaciones, era necesario definir dónde se almacenarían los scripts, los changelogs y los rollbacks. Esto evitó que cada integrante trabajara con una organización diferente y facilitó la integración posterior de nuevas historias de usuario.

## Trabajo realizado

En esta historia se realizaron las siguientes actividades:

- Creación de la estructura inicial del repositorio.
- Organización de los cambios por categorías DDL, DML, DCL y TCL.
- Creación de la estructura destinada a rollbacks.
- Preparación de carpetas para los diferentes objetos de PostgreSQL.
- Preparación de los changelogs que conectan las categorías del repositorio.
- Corrección de inconvenientes encontrados en la estructura inicial.
- Creación de una rama `hotfix` para ajustar la organización.
- Promoción de la estructura hacia QA y Staging.

## Estructura preparada

```text
01_ddl/
├── 00_extensions/
├── 01_schemas/
├── 02_types/
├── 03_tables/
├── 04_constraints/
├── 05_views/
├── 06_materialized_views/
├── 07_functions/
├── 08_procedures/
├── 09_triggers/
└── 10_indexes/
```

Esta fue la organización inicial. Posteriormente, en HU-04, la carpeta `04_constraints` fue reemplazada por `04_alter` para alinearla con el estándar actualizado del proyecto.

## Resultado de HU-01

La historia dejó una base organizada para incorporar los siguientes cambios del microservicio.

Gracias a esta estructura fue posible agregar posteriormente:

- Las tablas de Document Service.
- Las claves foráneas.
- Los permisos y roles.
- Los rollbacks.
- La reorganización definitiva del repositorio.

La existencia de ramas para Develop, QA y Staging también permitió promover progresivamente el trabajo realizado.

---

# 5. HU-03 — Implementación de claves foráneas

## Propietario

**`Juan282007`**

## Ramas observadas

```text
feature/HU-03-foreigns
HU-03-qa
HU-03-stg
```

## Commits observados

```text
Add foreign keys in constraints
fix wrong direction
Add foreign keys in constraints
```

## Motivo de la historia de usuario

Las tablas principales de Document Service ya habían sido creadas, pero necesitaban relaciones que garantizaran la integridad de la información.

Sin claves foráneas, la base de datos podía aceptar registros inconsistentes, como:

- Documentos asociados con plantillas inexistentes.
- Versiones asociadas con documentos inexistentes.

La historia se realizó para conectar correctamente las entidades principales del modelo.

## Trabajo realizado

Se implementaron las siguientes relaciones:

```text
document.template_id
    → document_template.id
```

```text
document_version.document_id
    → document.id
```

Las relaciones se agregaron mediante scripts de alteración, manteniendo separada la creación inicial de las tablas y la creación de sus dependencias.

También se realizaron las siguientes actividades:

- Creación del archivo SQL de claves foráneas.
- Registro del archivo dentro del changelog correspondiente.
- Preparación de los scripts de rollback.
- Corrección de la dirección de una relación detectada durante el desarrollo.
- Validación de las dependencias entre tablas.
- Promoción del cambio hacia QA.
- Promoción del cambio hacia Staging.

## Comportamiento de las relaciones

La relación entre documento y plantilla permite que cada documento se encuentre asociado con una plantilla válida.

La relación entre versión y documento permite que cada versión pertenezca a un documento existente.

La implementación también definió el comportamiento de eliminación necesario para proteger la integridad de la información:

```text
document → document_template: RESTRICT
document_version → document: CASCADE
```

Esto significa que:

- Una plantilla utilizada por documentos no puede eliminarse directamente.
- Al eliminar un documento, también se eliminan sus versiones relacionadas.

## Corrección realizada

En el historial aparece el commit:

```text
fix wrong direction
```

Este ajuste corrigió la orientación de una relación para que la clave foránea partiera de la tabla dependiente y apuntara hacia la tabla principal correspondiente.

## Resultado de HU-03

Después de esta historia, las tablas principales dejaron de ser objetos aislados y pasaron a formar un modelo relacionado.

El resultado fue una base de datos con integridad referencial entre:

```text
document_template
        ↑
     document
        ↑
document_version
```

Esto evita la creación de documentos o versiones que no tengan una referencia válida.

---

# 6. HU-04 — Reorganización del repositorio e integración de infraestructura

## Propietario

**`Juan282007`**

## Ramas observadas

```text
feature/HU-04-dev
HU-04-qa
HU-04-stg
```

## Commit principal observado

```text
HU-04: reorganize repository structure, add docker-infra, and rename 04_constraints to 04_alter
```

## Motivo de la historia de usuario

El repositorio necesitaba ajustarse al estándar definitivo establecido para los microservicios del proyecto.

También era necesario separar claramente:

- La infraestructura utilizada para levantar PostgreSQL y Liquibase.
- El repositorio que contiene las migraciones de Document Service.

Además, la carpeta denominada `04_constraints` debía convertirse en `04_alter`, porque esa ubicación no solamente podía contener claves foráneas, sino también otras modificaciones realizadas sobre objetos existentes.

## Trabajo realizado

En esta historia se llevaron a cabo las siguientes actividades:

- Reorganización completa de la carpeta de trabajo.
- Integración de la carpeta `docker-infra` suministrada para el proyecto.
- Separación de la infraestructura y las migraciones del microservicio.
- Cambio de nombre de `04_constraints` a `04_alter`.
- Actualización de la estructura equivalente dentro de `05_rollbacks`.
- Actualización de rutas en los changelogs.
- Preparación de archivos de configuración para los ambientes.
- Organización de la ejecución con PostgreSQL, Docker y Liquibase.
- Promoción de la reorganización hacia QA.
- Promoción de la reorganización hacia Staging.
- Integración del trabajo en las ramas de release.

## Organización resultante

```text
carpeta-de-trabajo/
├── docker-infra/
└── design-software-document-db/
```

### `docker-infra`

Esta carpeta quedó destinada a los componentes de infraestructura, entre ellos:

- `Dockerfile`.
- `docker-compose.yml`.
- Variables de entorno.
- Configuración de Liquibase.
- Driver JDBC de PostgreSQL.
- Guía de instalación y ejecución.

### `design-software-document-db`

Esta carpeta quedó destinada exclusivamente a:

- Migraciones SQL.
- Changelogs.
- Definición de objetos de PostgreSQL.
- Roles y permisos.
- Scripts de rollback.

## Cambio de `04_constraints` a `04_alter`

La estructura anterior era:

```text
01_ddl/
└── 04_constraints/
```

Después de la reorganización quedó:

```text
01_ddl/
└── 04_alter/
```

El mismo cambio se aplicó en los rollbacks:

```text
05_rollbacks/
└── 01_ddl/
    └── 04_alter/
```

Este cambio permitió utilizar la carpeta para cualquier modificación posterior a la creación de un objeto, no únicamente para constraints.

## Preparación de ambientes

La reorganización dejó preparada la infraestructura para trabajar con los ambientes:

```text
Develop
QA
Staging
Main
```

Cada ambiente puede utilizar su configuración de conexión, mientras que los mismos changelogs de Liquibase conservan el historial común de migraciones.

## Resultado de HU-04

El repositorio quedó:

- Alineado con el estándar del proyecto.
- Separado correctamente de la infraestructura.
- Preparado para ejecutarse con Docker y Liquibase.
- Organizado para diferentes ambientes.
- Con una estructura más clara para alteraciones y rollbacks.
- Integrado mediante ramas de desarrollo, QA, Staging y release.

---

# 7. Hotfix de la estructura inicial

## Rama observada

```text
hotfix/HU-01-structure
```

## Propietario

**`Juan282007`**

## Commit observado

```text
hotfix from initial structure
```

## Motivo del hotfix

Después de crear la estructura inicial se identificaron ajustes necesarios para que la organización funcionara correctamente con las siguientes historias de usuario.

En lugar de mezclar la corrección con una nueva funcionalidad, se utilizó una rama de hotfix dedicada.

## Trabajo realizado

- Corrección de la estructura creada inicialmente.
- Ajuste de la organización de carpetas o changelogs afectados.
- Integración de la corrección mediante Pull Request.

## Resultado

La estructura inicial quedó estabilizada antes de continuar con nuevas implementaciones.

---

# 8. Participación en el flujo de ambientes

El historial muestra que `Juan282007` trabajó con ramas asociadas a diferentes etapas de integración.

## Develop

En Develop se realizaron las implementaciones principales mediante ramas `feature`:

```text
feature/HU-01-dev
feature/HU-03-foreigns
feature/HU-04-dev
```

## QA

Se utilizaron ramas destinadas a promover y validar los cambios:

```text
HU-01-qa
HU-03-qa
HU-04-qa
```

## Staging

Después de QA, los cambios fueron promovidos mediante:

```text
HU-01-stg
HU-03-stg
HU-04-stg
```

## Release

En el historial aparecen las ramas:

```text
release/iteration-01
release/iteration-02
```

Estas ramas agruparon cambios aprobados de las historias de usuario antes de su integración final en las ramas principales.

## Flujo general observado

```text
Rama feature
     ↓
  Develop
     ↓
 Rama de QA
     ↓
     QA
     ↓
Rama de Staging
     ↓
  Staging
     ↓
   Release
     ↓
    Main
```

Este flujo permitió que los cambios pasaran por diferentes niveles de validación antes de llegar a la rama principal.

---

# 9. Uso de Git y Pull Requests

El trabajo se administró mediante ramas independientes y Pull Requests.

El historial evidencia merges relacionados con:

- La estructura inicial.
- Las correcciones de HU-01.
- Las claves foráneas de HU-03.
- La promoción de HU-03 hacia QA y Staging.
- La reorganización de HU-04.
- La promoción de HU-04 hacia QA y Staging.
- La integración de las iteraciones mediante releases.

El uso de Pull Requests permitió:

- Separar cada historia de usuario.
- Revisar los cambios antes de integrarlos.
- Mantener trazabilidad de los responsables.
- Corregir errores sin modificar directamente las ramas principales.
- Promover los mismos cambios entre ambientes.

---

# 10. Relación de las contribuciones con Liquibase

Las historias desarrolladas por `Juan282007` permitieron que Liquibase tuviera una organización y una secuencia de ejecución coherentes.

El changelog maestro funciona como punto de entrada:

```text
changelog/changelog-master.yaml
```

Desde allí se integran las categorías principales:

```text
DDL → DML → DCL → TCL
```

Dentro del DDL, el orden necesario es:

```text
Extensión
   ↓
Esquema
   ↓
Tablas
   ↓
Alteraciones y claves foráneas
```

Las contribuciones de Juan tuvieron impacto directo en este flujo:

| Contribución | Impacto en Liquibase |
|---|---|
| HU-01 | Preparó la estructura y los changelogs iniciales. |
| HU-03 | Incorporó las relaciones después de la creación de tablas. |
| HU-04 | Reorganizó rutas y cambió `04_constraints` por `04_alter`. |
| Hotfix HU-01 | Corrigió problemas de la estructura inicial. |

---

# 11. Aportes técnicos principales

## Organización

Se creó y posteriormente se reorganizó una estructura uniforme para administrar las migraciones del microservicio.

## Integridad referencial

Se conectaron las tres tablas principales mediante claves foráneas.

## Rollbacks

Se preparó la estructura correspondiente para revertir las relaciones y cambios estructurales implementados.

## Infraestructura

Se incorporó la organización necesaria para utilizar Docker, PostgreSQL y Liquibase desde una carpeta de infraestructura separada.

## Ambientes

Se promovieron historias de usuario mediante ramas de Develop, QA y Staging.

## Corrección de errores

Se utilizaron commits y ramas específicas para corregir la estructura y la dirección de una relación.

## Integración

Las contribuciones se integraron mediante Pull Requests y ramas de release.

---

# 12. Resumen de historias de usuario de Juan282007

| Historia | Responsabilidad | Resultado |
|---|---|---|
| HU-01 | Crear la estructura inicial del repositorio | Base organizada para migraciones, changelogs y rollbacks |
| HU-03 | Crear las claves foráneas | Integridad referencial entre plantillas, documentos y versiones |
| HU-04 | Reorganizar el repositorio e integrar infraestructura | Separación entre Docker Infra y migraciones, nuevas rutas y soporte para ambientes |
| Hotfix HU-01 | Corregir la estructura inicial | Organización estabilizada antes de continuar el desarrollo |

---

# 13. Resultados alcanzados por el colaborador

Como resultado del trabajo realizado por `Juan282007`:

- El repositorio obtuvo una estructura inicial organizada.
- Se corrigieron problemas encontrados en dicha estructura.
- Las tablas de Document Service quedaron relacionadas correctamente.
- Se protegió la integridad referencial de los datos.
- Se prepararon rollbacks para las relaciones implementadas.
- La carpeta de constraints fue reorganizada como carpeta de alteraciones.
- La estructura de rollbacks fue actualizada de acuerdo con la reorganización.
- La infraestructura quedó separada de las migraciones.
- El proyecto quedó preparado para su ejecución con Docker y Liquibase.
- Las contribuciones fueron promovidas por Develop, QA y Staging.
- Los cambios fueron integrados mediante Pull Requests y releases.

---

# 14. Evidencia observada en el historial de Git

En las capturas suministradas se identifican las siguientes referencias asociadas con `Juan282007`:

```text
feature/HU-01-dev
HU-01-qa
HU-01-stg
hotfix/HU-01-structure
feature/HU-03-foreigns
HU-03-qa
HU-03-stg
feature/HU-04-dev
HU-04-qa
HU-04-stg
release/iteration-01
release/iteration-02
```

También se identifican commits como:

```text
Add project structure
fix structure
Add initial structure
hotfix from initial structure
Add foreign keys in constraints
fix wrong direction
HU-04: reorganize repository structure, add docker-infra, and rename 04_constraints to 04_alter
fix case-sensitivity Linux error
Add final proyect
```

Estas referencias muestran participación en la creación, corrección, promoción e integración del repositorio de base de datos.

---

