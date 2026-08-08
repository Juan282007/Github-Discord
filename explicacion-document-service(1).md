# Document Service Database

## 1. Introducción

Este documento presenta el trabajo realizado en el repositorio **`design-software-document-db`**, correspondiente a la base de datos del microservicio **Document Service** del proyecto **Design Software**.

El repositorio fue construido para administrar de forma organizada y versionada la estructura de la base de datos del microservicio. Su alcance se concentra exclusivamente en PostgreSQL, Liquibase, scripts de migración, permisos y mecanismos de reversión.

No se trata de una aplicación backend. En este repositorio no se desarrollaron controladores, servicios REST, endpoints, lógica Java ni interfaces gráficas. El resultado obtenido es una base sólida para que, posteriormente, el backend de Document Service pueda almacenar la información relacionada con documentos, plantillas y versiones.

---

## 2. Propósito del repositorio

El propósito principal de `design-software-document-db` es permitir que la base de datos de Document Service pueda crearse de la misma manera en todos los ambientes del proyecto.

Esto significa que un integrante del equipo no necesita crear manualmente tablas, relaciones o permisos desde PostgreSQL. La estructura se encuentra descrita en archivos versionados y Liquibase se encarga de ejecutarlos en el orden correcto.

Con este enfoque se consiguió:

- Mantener un historial de los cambios realizados en la base de datos.
- Evitar modificaciones manuales difíciles de reproducir.
- Aplicar la misma estructura en Develop, QA, Staging y Main.
- Organizar las migraciones por tipo de operación.
- Permitir que los cambios puedan revertirse mediante rollbacks.
- Facilitar la integración del trabajo realizado por diferentes integrantes.

---

## 3. Objetivo alcanzado

El trabajo realizado dejó preparada la base de datos principal de Document Service con los siguientes elementos:

- Extensión de PostgreSQL para generar identificadores UUID.
- Esquema independiente llamado `document`.
- Tabla de plantillas de documentos.
- Tabla principal de documentos.
- Tabla para almacenar versiones de documentos.
- Relaciones entre las tablas.
- Roles para lectura, escritura y administración.
- Permisos sobre el esquema y sus tablas.
- Changelogs organizados para la ejecución de Liquibase.
- Scripts de rollback para los cambios estructurales implementados.
- Estructura de carpetas unificada con el estándar del proyecto.
- Integración con la infraestructura Docker suministrada para los ambientes.

El resultado permite levantar PostgreSQL y ejecutar las migraciones sin tener que construir manualmente la estructura de la base de datos.

---

## 4. Alcance de Document Service en este repositorio

Document Service necesita conservar los metadatos asociados con los documentos administrados por la solución.

La base de datos implementada se concentra en tres conceptos principales:

### 4.1 Plantilla de documento

Una plantilla representa la estructura reutilizable con la que se puede generar un documento.

La tabla implementada es:

```text
document.document_template
```

Esta tabla permite almacenar información como:

- Identificador de la plantilla.
- Código.
- Nombre.
- Contenido de la plantilla.
- Tipo de salida.
- Versión.
- Información de auditoría.
- Estado del registro.

### 4.2 Documento

Un documento representa el registro principal administrado por el microservicio.

La tabla implementada es:

```text
document.document
```

En ella se almacena información como:

- Identificador del documento.
- Plantilla utilizada.
- Título.
- Dominio de origen.
- Microservicio propietario.
- Entidad relacionada.
- Ruta de almacenamiento del archivo.
- Tipo MIME.
- Tamaño.
- Estado.
- Versión para control de concurrencia.
- Información de auditoría.

La base de datos conserva la ruta o clave del archivo mediante `storage_key`. El archivo físico no se guarda directamente dentro de PostgreSQL.

### 4.3 Versión de documento

La tabla implementada es:

```text
document.document_version
```

Su objetivo es mantener el historial de versiones de cada documento.

Almacena información como:

- Identificador de la versión.
- Documento al que pertenece.
- Número de versión.
- Ruta del archivo correspondiente.
- Notas.
- Información de auditoría.

---

## 5. Tecnologías utilizadas

### PostgreSQL

Es el motor utilizado para almacenar los metadatos del microservicio.

### Liquibase

Es la herramienta utilizada para controlar y ejecutar las migraciones.

Liquibase registra los cambios aplicados en las tablas internas:

```text
DATABASECHANGELOG
DATABASECHANGELOGLOCK
```

`DATABASECHANGELOG` mantiene el historial de los changesets ejecutados. `DATABASECHANGELOGLOCK` evita que dos procesos modifiquen la base de datos al mismo tiempo.

### Docker

La infraestructura Docker permite levantar el motor PostgreSQL y ejecutar Liquibase de forma reproducible, usando archivos de variables de entorno para cada ambiente.

### Git y GitHub

Git se utilizó para separar el trabajo por historias de usuario, controlar los cambios y promoverlos mediante Pull Requests hacia las ramas correspondientes.

---

## 6. Organización del repositorio

La estructura principal quedó organizada de la siguiente forma:

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

### `01_ddl`

Contiene la definición estructural de la base de datos.

```text
01_ddl/
├── 00_extensions/
├── 01_schemas/
├── 02_types/
├── 03_tables/
├── 04_alter/
├── 05_views/
├── 06_materialized_views/
├── 07_functions/
├── 08_procedures/
├── 09_triggers/
└── 10_indexes/
```

Dentro de esta categoría se implementaron la extensión, el esquema, las tablas y las claves foráneas.

Las demás carpetas hacen parte de la organización estándar del proyecto y se conservaron para mantener una estructura uniforme.

### `02_dml`

Está preparada para cambios relacionados con datos, como inserciones, actualizaciones, eliminaciones y cargas iniciales.

### `03_dcl`

Contiene la creación de roles y la asignación de permisos.

### `04_tcl`

Mantiene la organización destinada a operaciones transaccionales y marcas de versiones.

### `05_rollbacks`

Contiene la estructura utilizada para revertir los cambios implementados.

---

## 7. Flujo de ejecución de Liquibase

El punto de entrada es el changelog maestro:

```text
changelog/changelog-master.yaml
```

Este archivo incluye los changelogs principales en un orden controlado:

```text
DDL → DML → DCL → TCL
```

Dentro del DDL, la ejecución sigue el orden necesario para respetar las dependencias:

```text
Extensión
   ↓
Esquema
   ↓
Tablas
   ↓
Alteraciones y claves foráneas
```

El proceso realizado por Liquibase es el siguiente:

1. Lee el changelog maestro.
2. Localiza los changesets pendientes.
3. Consulta `DATABASECHANGELOG`.
4. Ejecuta solamente los changesets que todavía no han sido aplicados.
5. Registra cada ejecución exitosa.
6. Mantiene disponible el rollback definido para cada cambio.

Gracias a este flujo, la misma migración puede utilizarse en los diferentes ambientes sin ejecutar manualmente cada archivo SQL.

---

## 8. Cambios estructurales implementados

### 8.1 Extensión `pgcrypto`

Se habilitó la extensión:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

Su uso permite generar identificadores UUID mediante:

```sql
gen_random_uuid()
```

### 8.2 Esquema `document`

Se creó un esquema propio para aislar los objetos del microservicio:

```sql
CREATE SCHEMA IF NOT EXISTS document;
```

De esta forma, las tablas se identifican como:

```text
document.document_template
document.document
document.document_version
```

### 8.3 Tablas

Se implementaron las tres tablas principales del modelo:

```text
document_template
document
document_version
```

Cada tabla utiliza UUID como identificador principal y contiene los campos necesarios para representar su información dentro del microservicio.

### 8.4 Relaciones

Se implementaron las siguientes claves foráneas:

```text
document.template_id
    → document_template.id
```

```text
document_version.document_id
    → document.id
```

Estas relaciones permiten asociar un documento con su plantilla y mantener las versiones relacionadas con el documento principal.

### 8.5 Roles y permisos

Se crearon tres roles agrupadores:

| Rol | Responsabilidad |
|---|---|
| `document_reader` | Consulta de información. |
| `document_writer` | Lectura y operaciones de escritura. |
| `document_admin` | Administración de los objetos del esquema. |

También se asignaron permisos sobre el esquema, las tablas y los objetos futuros creados dentro de él.

---

# 9. Historias de usuario realizadas

Las capturas del historial de Git muestran la evolución del repositorio mediante cuatro historias de usuario principales.

Los propietarios indicados a continuación corresponden al autor del trabajo principal observado en las ramas y commits de cada HU.

## HU-01 — Creación de la estructura inicial

**Propietario:** `Juan282007`

**Ramas observadas:**

```text
feature/HU-01-dev
HU-01-qa
HU-01-stg
```

**Commits destacados:**

```text
Add project structure
fix structure
Add initial structure
```

### ¿Por qué se realizó?

El repositorio necesitaba una organización inicial común para que los cambios posteriores pudieran ubicarse correctamente y ser ejecutados por Liquibase.

Sin una estructura definida, cada integrante podía organizar scripts y changelogs de forma diferente, dificultando la integración y el mantenimiento.

### ¿Qué se hizo?

- Se creó la estructura inicial del proyecto.
- Se organizaron las categorías DDL, DML, DCL, TCL y rollbacks.
- Se prepararon los directorios para los diferentes objetos de base de datos.
- Se crearon los changelogs necesarios para conectar la estructura.
- Se corrigió la organización inicial mediante una rama de hotfix.
- Se promovió la estructura hacia QA y Staging.

### Resultado

Se obtuvo una base uniforme sobre la cual fue posible agregar tablas, relaciones, permisos y nuevas migraciones.

---

## HU-02 — Creación de las tablas de Document Service

**Propietario:** `JohanAceroSalazar`

**Ramas observadas:**

```text
feature/HU-02-document-service
HU-02-qa
HU-02-stg
```

**Commits destacados:**

```text
feat(document-service): create document service database tables
Upload document service database tables to qa
Upload document service database tables to staging
```

### ¿Por qué se realizó?

Después de contar con la estructura inicial, era necesario representar en PostgreSQL la información principal administrada por Document Service.

La base de datos requería objetos para almacenar las plantillas, los documentos y el historial de versiones.

### ¿Qué se hizo?

- Se creó la tabla `document_template`.
- Se creó la tabla `document`.
- Se creó la tabla `document_version`.
- Se definieron los identificadores UUID.
- Se incorporaron los campos principales y de auditoría.
- Se agregaron los archivos SQL a los changelogs de Liquibase.
- Se promovieron las tablas hacia QA y Staging mediante sus ramas correspondientes.

### Resultado

El microservicio quedó con una estructura de persistencia capaz de registrar sus tres entidades principales.

---

## HU-03 — Creación de las claves foráneas

**Propietario:** `Juan282007`

**Ramas observadas:**

```text
feature/HU-03-foreigns
HU-03-qa
HU-03-stg
```

**Commits destacados:**

```text
Add foreign keys in constraints
fix wrong direction
Add foreign keys in constraints
```

### ¿Por qué se realizó?

Las tablas creadas en HU-02 necesitaban relaciones explícitas para garantizar la integridad de la información.

Sin claves foráneas, se podían registrar documentos relacionados con plantillas inexistentes o versiones asociadas con documentos que no existieran.

### ¿Qué se hizo?

- Se agregó la relación entre `document` y `document_template`.
- Se agregó la relación entre `document_version` y `document`.
- Se ubicaron las relaciones en la sección de alteraciones o constraints.
- Se corrigió la dirección de una relación durante el desarrollo.
- Se prepararon los rollbacks correspondientes.
- Se promovieron las relaciones hacia QA y Staging.

### Resultado

La base de datos quedó con integridad referencial entre sus tablas principales.

---

## HU-04 — Reorganización del repositorio e integración de infraestructura

**Propietario:** `Juan282007`

**Ramas observadas:**

```text
feature/HU-04-dev
HU-04-qa
HU-04-stg
```

**Commit principal:**

```text
HU-04: reorganize repository structure, add docker-infra, and rename 04_constraints to 04_alter
```

### ¿Por qué se realizó?

El repositorio necesitaba ajustarse al estándar general establecido para los microservicios del proyecto y separar claramente la infraestructura de la definición de la base de datos.

También era necesario corregir la ubicación y los nombres de las carpetas para que los changelogs funcionaran de forma consistente.

### ¿Qué se hizo?

- Se reorganizó la estructura completa del repositorio.
- Se integró la carpeta `docker-infra` suministrada para la ejecución con Docker y Liquibase.
- Se separó la infraestructura del repositorio de migraciones.
- Se cambió el nombre de `04_constraints` a `04_alter`.
- Se actualizó la estructura equivalente dentro de `05_rollbacks`.
- Se modificaron los changelogs para utilizar las nuevas rutas.
- Se preparó la configuración para Develop, QA, Staging y Main.
- Se promovió el cambio mediante ramas específicas de QA y Staging.
- Se integró el trabajo dentro de las ramas de release de las iteraciones.

### Resultado

El repositorio quedó alineado con el estándar del proyecto, con una estructura más clara y con la infraestructura necesaria para reproducir la base de datos en distintos ambientes.

---

## 10. Correcciones e integración posteriores

Las capturas también muestran actividades de integración posteriores a las historias de usuario:

### Hotfix de la estructura inicial

Se utilizó la rama:

```text
hotfix/HU-01-structure
```

Esta rama corrigió problemas identificados después de crear la primera estructura.

### Corrección de rutas del changelog

En el historial más reciente aparece el commit:

```text
File path correction in the changelog of the alter folder
```

Autor observado:

```text
JohanAceroSalazar
```

Esta corrección permitió que Liquibase localizara correctamente los archivos de la carpeta `04_alter` después de la reorganización.

### Releases de integración

Se utilizaron las ramas:

```text
release/iteration-01
release/iteration-02
```

Estas ramas reunieron los cambios aprobados de las historias de usuario antes de alinearlos con las ramas principales del proyecto.

También se observan commits de alineación para:

```text
develop
staging
main
```

Esto demuestra que el trabajo no quedó únicamente en ramas individuales, sino que fue integrado progresivamente en los ambientes definidos.

---

## 11. Flujo de ramas utilizado

El historial muestra un flujo basado en ramas hijas para cada etapa.

Ejemplo general:

```text
feature/HU-XX-dev
        │
        ▼
     develop
        │
        ▼
    HU-XX-qa
        │
        ▼
        qa
        │
        ▼
   HU-XX-stg
        │
        ▼
     staging
        │
        ▼
release/iteration
        │
        ▼
       main
```

Cada Pull Request permitió revisar e integrar el trabajo realizado antes de promoverlo al siguiente ambiente.

Las capturas muestran, entre otros, los siguientes Pull Requests:

- PR de HU-01 hacia sus ramas de integración.
- PR de HU-02 para llevar las tablas a QA y Staging.
- PR de HU-03 para llevar las claves foráneas a QA y Staging.
- PR de HU-04 para llevar la reorganización a Develop, QA y Staging.
- PR de las ramas `release/iteration-01` y `release/iteration-02`.

---

## 12. Organización de la carpeta de trabajo

Después de HU-04, la organización general quedó planteada de esta manera:

```text
carpeta-de-trabajo/
├── docker-infra/
└── design-software-document-db/
```

### `docker-infra`

Contiene los elementos necesarios para ejecutar el entorno:

- Dockerfile.
- `docker-compose.yml`.
- Archivos `.env` de los ambientes.
- Configuración de Liquibase.
- Driver JDBC de PostgreSQL.
- Instrucciones de ejecución.

### `design-software-document-db`

Contiene únicamente:

- Migraciones de base de datos.
- Changelogs.
- Scripts SQL.
- Roles y permisos.
- Rollbacks.
- Documentación del repositorio.

Esta separación permite que la infraestructura pueda encargarse de ejecutar el repositorio sin mezclar archivos operativos con las migraciones del microservicio.

---

## 13. Funcionamiento dentro de la carpeta de trabajo

El flujo de trabajo es el siguiente:

```text
Docker inicia PostgreSQL
          │
          ▼
Liquibase se conecta utilizando las variables del ambiente
          │
          ▼
Se lee changelog/changelog-master.yaml
          │
          ▼
Se ejecutan los changesets pendientes
          │
          ▼
Se crean extensión, esquema, tablas y relaciones
          │
          ▼
Se crean roles y se asignan permisos
          │
          ▼
Liquibase registra los cambios en DATABASECHANGELOG
```

Al cambiar de ambiente no se crean manualmente las tablas de nuevo. Se utiliza la configuración correspondiente y Liquibase reproduce las migraciones registradas en el repositorio.

---

## 14. Aporte de cada historia al resultado final

| Historia | Propietario | Aporte realizado |
|---|---|---|
| HU-01 | `Juan282007` | Creó y corrigió la estructura inicial del repositorio. |
| HU-02 | `JohanAceroSalazar` | Implementó las tablas principales de Document Service. |
| HU-03 | `Juan282007` | Implementó las claves foráneas y la integridad referencial. |
| HU-04 | `Juan282007` | Reorganizó el repositorio, integró Docker Infra y actualizó las rutas y carpetas. |

Las cuatro historias se complementan:

```text
HU-01: organiza el proyecto
          ↓
HU-02: crea las tablas
          ↓
HU-03: conecta las tablas
          ↓
HU-04: normaliza y prepara la ejecución por ambientes
```

---

## 15. Resultado final del trabajo realizado

El repositorio quedó preparado como una solución de migraciones para la base de datos de Document Service.

El trabajo realizado consiguió:

- Una estructura de proyecto organizada.
- Un esquema aislado para el microservicio.
- Las tres tablas principales.
- Relaciones entre documentos, plantillas y versiones.
- Generación de UUID desde PostgreSQL.
- Roles y permisos diferenciados.
- Rollbacks para los cambios estructurales implementados.
- Integración con Liquibase.
- Integración con Docker Infra.
- Configuración orientada a varios ambientes.
- Historial de Git separado por historias de usuario.
- Promoción de cambios mediante Pull Requests.
- Integración mediante ramas de release.

La base de datos implementada constituye la capa de persistencia sobre la cual puede funcionar el microservicio Document Service. Su principal valor es que la estructura puede reproducirse, verificarse y versionarse sin depender de configuraciones manuales realizadas directamente en PostgreSQL.

---

## 16. Conclusión

El desarrollo de `design-software-document-db` se enfocó en construir y organizar la base de datos de Document Service mediante un proceso incremental.

Primero se creó la estructura del repositorio; después se implementaron las tablas; posteriormente se agregaron las relaciones; y finalmente se reorganizó el proyecto para alinearlo con el estándar general, integrar la infraestructura Docker y preparar su ejecución en los diferentes ambientes.

Las capturas del historial de Git evidencian que cada historia de usuario fue trabajada en ramas separadas, revisada mediante Pull Requests y promovida progresivamente hacia QA, Staging, las ramas de release y las ramas principales.

El resultado es un repositorio de base de datos organizado, versionado y reproducible, preparado para servir como persistencia del microservicio Document Service dentro de la arquitectura de Design Software.
