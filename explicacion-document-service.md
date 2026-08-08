# Document Service: arquitectura definida e implementación actual de la base de datos

## 1. Introducción

Este documento explica detalladamente el contenido de la carpeta arquitectónica **`07-document-service`** y cómo esa definición se relaciona con la implementación existente en la **carpeta de trabajo** del repositorio **`design-software-document-db`**.

Es fundamental diferenciar ambos elementos:

- **`07-document-service`** contiene la documentación de arquitectura y diseño del microservicio completo. Allí se describen sus responsabilidades, componentes, contratos REST, eventos, modelo de datos, almacenamiento, operación y decisiones técnicas.
- **`design-software-document-db`**, representado en la carpeta de trabajo suministrada, no contiene la aplicación del microservicio. Contiene exclusivamente la definición versionada de su base de datos PostgreSQL mediante **Liquibase**.

Por lo tanto, la carpeta de trabajo implementa solamente una parte de todo lo descrito en `07-document-service`: la capa de persistencia de metadatos del Document Service.

---

## 2. ¿Qué es Document Service?

Document Service es un microservicio transversal del proyecto **Design Software** encargado de administrar documentos generados o cargados por otros servicios.

Su responsabilidad funcional contempla:

- Crear registros de documentos.
- Administrar plantillas reutilizables.
- Mantener versiones inmutables de los documentos.
- Registrar la ubicación de los archivos.
- Permitir carga, consulta, descarga y archivado.
- Generar archivos PDF o Excel de manera asíncrona.
- Aplicar políticas de retención, expiración y limpieza.
- Publicar eventos cuando un documento se genera o cuando se crea una nueva versión.

El servicio es **agnóstico al dominio de negocio**. Esto significa que no está diseñado exclusivamente para horarios, matrículas, fichas o certificados. Otros microservicios pueden solicitarle documentos indicando el dominio, la entidad propietaria, la plantilla y los datos requeridos.

Ejemplos de documentos que podría administrar:

- Horarios académicos en PDF.
- Constancias.
- Certificados.
- Reportes en Excel.
- Documentos cargados manualmente.

---

## 3. Alcance real del repositorio `design-software-document-db`

El repositorio de la carpeta de trabajo **no es el backend de Document Service**.

No contiene:

- Código Java.
- Spring Boot.
- Controladores REST.
- Servicios de aplicación.
- Repositorios JPA.
- Entidades Java.
- Consumidores de Kafka.
- Workers.
- Generación de PDF.
- Integración con MinIO o Amazon S3.
- Validación de JWT.
- Interfaces gráficas.

Su responsabilidad es exclusivamente:

> Crear, modificar, asegurar, versionar y revertir la estructura de la base de datos utilizada por Document Service.

Esto se realiza mediante:

- PostgreSQL como motor de base de datos.
- Liquibase como herramienta de migraciones.
- Archivos SQL para ejecutar los cambios.
- Archivos YAML para ordenar y registrar los Changesets.
- Docker Compose, desde una infraestructura externa, para levantar PostgreSQL y ejecutar Liquibase de forma reproducible.

---

## 4. Contenido de la carpeta `07-document-service`

La carpeta de arquitectura contiene los siguientes documentos principales:

```text
07-document-service/
├── README.md
├── data-model.md
├── decisions.md
├── events.md
├── runbook.md
├── storage-adapters.md
└── components/
    ├── document-api/
    │   ├── README.md
    │   └── contract.md
    ├── template-api/
    │   ├── README.md
    │   └── contract.md
    ├── pdf-renderer-worker/
    │   ├── README.md
    │   └── contract.md
    └── document-lifecycle-worker/
        ├── README.md
        └── contract.md
```

Cada archivo cubre una parte distinta del servicio.

### 4.1 `README.md`

Define la visión general del microservicio.

Establece que Document Service gestiona:

- Creación de documentos.
- Versionado.
- Almacenamiento.
- Exportación.
- Generación solicitada por otros servicios.

También define cuatro conceptos principales del contexto:

| Concepto | Propósito |
|---|---|
| Documento | Representa un archivo generado o cargado junto con sus metadatos. |
| Versión de documento | Conserva una versión inmutable del archivo. |
| Plantilla de documento | Contiene el formato reutilizable para generar documentos. |
| Firma digital | Representa una firma o sello aplicado a un documento. |

La implementación actual de base de datos incluye documento, versión y plantilla. **No existe actualmente una tabla de firma digital** en la carpeta de trabajo.

El README también especifica que:

- La base de datos lógica se denomina `document_db`.
- PostgreSQL almacena solamente metadatos.
- Los archivos binarios deben almacenarse en object storage.
- Los binarios nunca deben guardarse directamente en PostgreSQL.

### 4.2 `data-model.md`

Describe el modelo de datos esperado para el servicio.

Las entidades principales son:

- `document_template`.
- `document`.
- `document_version`.

También documenta convenciones transversales, como:

- UUID para identificadores.
- Auditoría de creación y actualización.
- Borrado lógico.
- Bloqueo optimista mediante `row_version`.
- Restricciones `CHECK` para valores cerrados.
- Índices para consultas frecuentes.
- Relaciones con acciones explícitas de actualización y eliminación.

Este archivo representa el **modelo objetivo o esperado**, pero no todo está implementado todavía en los SQL de la carpeta de trabajo.

### 4.3 `decisions.md`

Registra decisiones técnicas internas del microservicio.

Las principales son:

1. Guardar los binarios exclusivamente en object storage.
2. Generar documentos de manera asíncrona con un worker y una cola.
3. Mantener el historial en `document_version` y la versión vigente en `document.storage_key`.
4. Usar plantillas HTML/Handlebars en lugar de formatos PDF codificados directamente.
5. Utilizar MinIO en desarrollo y S3 en producción mediante un patrón Adapter.

Estas decisiones explican por qué la base de datos solo necesita almacenar rutas, estados y metadatos, en lugar del contenido completo de los archivos.

### 4.4 `events.md`

Describe la comunicación asíncrona del servicio.

#### Eventos publicados

- `document.document.generated`: se publica cuando un documento fue generado y está disponible.
- `document.version.created`: se publica cuando se crea una nueva versión.

#### Eventos consumidos

El servicio está diseñado para reaccionar a eventos de otros dominios, por ejemplo la publicación de un horario académico.

#### Flujo asíncrono esperado

```text
Otro microservicio publica un evento o solicita un documento
                        │
                        ▼
Se crea un registro con estado GENERATING
                        │
                        ▼
Se envía un mensaje a document-generation-queue
                        │
                        ▼
pdf-renderer-worker genera el archivo
                        │
                        ▼
El archivo se guarda en MinIO o S3
                        │
                        ▼
Se actualizan storage_key, mime_type, size_bytes y status
                        │
                        ▼
Se publica document.document.generated
```

La carpeta de trabajo prepara las tablas donde se registrarían esos cambios, pero **no contiene la lógica que consume o publica los eventos**.

### 4.5 `storage-adapters.md`

Define la abstracción del almacenamiento físico.

Los proveedores planteados son:

- `MinioStorageAdapter` para desarrollo local.
- `S3StorageAdapter` para otros ambientes o producción.

Operaciones esperadas:

- Subir un archivo.
- Descargar un archivo.
- Crear una URL temporal firmada.
- Eliminar un archivo.

Ejemplo de una clave de almacenamiento:

```text
horario/2026/06/ficha-abc123/horario-v1.pdf
```

La base de datos guarda esa ruta en `storage_key`. El archivo no se encuentra dentro de la tabla.

### 4.6 `runbook.md`

Es una guía operativa prevista para soporte y despliegue.

Incluye:

- Verificaciones de salud del servicio.
- Variables de entorno.
- Conectividad con PostgreSQL.
- Conectividad con MinIO o S3.
- Manejo de documentos fallidos.
- Recuperación de metadatos perdidos.
- Diagnóstico de workers y colas.
- Procedimientos ante fallas.

Los endpoints de salud y los procedimientos descritos pertenecen al backend futuro o esperado. No están implementados en el repositorio de migraciones.

---

## 5. Componentes desplegables definidos en la arquitectura

### 5.1 `document-api`

Es la API REST encargada de administrar los documentos y sus versiones.

Los contratos plantean operaciones para:

- Listar documentos.
- Consultar metadatos.
- Generar una URL de descarga.
- Solicitar generación asíncrona.
- Consultar versiones.
- Cargar un documento manualmente.
- Archivar un documento.

Ejemplo conceptual:

```http
POST /api/v1/documents/generate
```

La respuesta esperada es `202 Accepted`, porque el archivo no se genera inmediatamente. Primero se crea el registro con un estado equivalente a `GENERATING` y luego un worker completa el proceso.

### 5.2 `template-api`

Administra las plantillas utilizadas para crear documentos.

Operaciones previstas:

- Listar plantillas activas.
- Consultar una plantilla.
- Crear una plantilla.
- Actualizarla.
- Desactivarla.

La tabla `document.document_template` de la carpeta de trabajo sirve como persistencia para este componente.

### 5.3 `pdf-renderer-worker`

Es un consumidor de la cola interna de generación.

Su función esperada es:

1. Recibir el identificador del documento, la plantilla y los datos.
2. Obtener la plantilla.
3. Renderizar el documento.
4. Generar PDF o Excel.
5. Subir el resultado al object storage.
6. Actualizar el registro en PostgreSQL.
7. Publicar un evento de finalización.

La arquitectura plantea tres intentos antes de declarar un fallo definitivo.

Cuando falla permanentemente, el documento debería quedar con estado `GENERATION_FAILED`.

### 5.4 `document-lifecycle-worker`

Gestiona políticas de ciclo de vida.

Las acciones descritas incluyen:

- Marcar documentos como expirados.
- Mover documentos antiguos a almacenamiento frío.
- Eliminar objetos huérfanos.
- Aplicar políticas de retención por tipo documental.

La documentación presenta cierta diferencia interna sobre este worker: un contrato lo describe como tarea periódica, mientras que el flujo de eventos también le asigna funciones de consumo y orquestación. Esto debe validarse cuando se implemente el backend.

---

## 6. Estructura real de la carpeta de trabajo

La copia analizada contiene directamente el repositorio de migraciones:

```text
carpeta-de-trabajo/
├── 01_ddl/
├── 02_dml/
├── 03_dcl/
├── 04_tcl/
├── 05_rollbacks/
├── changelog/
├── README.md
└── .git/
```

El repositorio remoto configurado es:

```text
https://github.com/code-sena/design-software-document-db.git
```

En el contexto de trabajo completo, este repositorio se usa junto con una carpeta externa `docker-infra`. Sin embargo, **la copia comprimida de la carpeta de trabajo analizada no contiene físicamente `docker-infra`**. Sí contiene las instrucciones para ejecutarse desde dicha infraestructura.

La organización conceptual esperada es:

```text
carpeta-de-trabajo/
├── docker-infra/
└── design-software-document-db/
```

La separación busca que:

- `docker-infra` orqueste PostgreSQL y Liquibase.
- `design-software-document-db` almacene únicamente las migraciones del dominio Document.

---

## 7. Organización por tipos de cambios

### 7.1 `01_ddl`

DDL significa **Data Definition Language**.

Contiene la definición estructural de la base de datos:

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

Implementaciones actuales:

- Extensión `pgcrypto`.
- Esquema `document`.
- Tres tablas.
- Dos claves foráneas.

Carpetas preparadas, pero sin implementaciones:

- Tipos.
- Vistas.
- Vistas materializadas.
- Funciones.
- Procedimientos.
- Triggers.
- Índices adicionales.

Sus `changelog.yaml` contienen `databaseChangeLog: []`, lo cual indica que Liquibase puede incluirlos sin ejecutar ningún Changeset todavía.

### 7.2 `02_dml`

DML significa **Data Manipulation Language**.

Está organizado para:

- Inserts.
- Updates.
- Deletes.
- Upserts.
- Patches.

Actualmente no existen scripts DML activos. Por tanto, el repositorio no inserta catálogos, plantillas iniciales ni datos semilla.

### 7.3 `03_dcl`

DCL significa **Data Control Language**.

Esta sección sí contiene implementaciones activas:

- Creación de roles.
- Asignación de permisos.

Roles creados:

| Rol | Alcance |
|---|---|
| `document_reader` | Lectura de las tablas del esquema. |
| `document_writer` | Lectura, inserción, actualización y eliminación. |
| `document_admin` | Privilegios administrativos sobre tablas y secuencias. |

Los roles son `NOLOGIN`, por lo que actúan como roles agrupadores. Un usuario de aplicación puede recibir uno de estos roles según sus responsabilidades.

También se configuran privilegios predeterminados para que las tablas futuras hereden los permisos esperados.

La carpeta de políticas existe, pero no contiene Row-Level Security ni políticas activas.

### 7.4 `04_tcl`

TCL significa **Transaction Control Language**.

La estructura está preparada para:

- Bloques transaccionales.
- Recuperaciones manuales.
- Tags o marcas de release.

Actualmente no hay scripts TCL activos en los subdirectorios incluidos.

### 7.5 `05_rollbacks`

Contiene la estructura espejo necesaria para revertir las migraciones.

Rollbacks implementados:

- Eliminar `pgcrypto`.
- Eliminar el esquema `document`.
- Eliminar cada tabla.
- Eliminar las claves foráneas.

Es importante que Liquibase ejecute los rollbacks en orden inverso al de creación. Por ejemplo, las relaciones deben eliminarse antes de intentar eliminar tablas relacionadas.

---

## 8. Punto de entrada de Liquibase

El archivo principal es:

```text
changelog/changelog-master.yaml
```

Este archivo incluye, en orden:

```yaml
01_ddl/changelog.yaml
02_dml/changelog.yaml
03_dcl/changelog.yaml
04_tcl/changelog.yaml
```

El orden es importante:

1. Primero se crea la estructura.
2. Luego podrían cargarse datos.
3. Después se crean roles y permisos.
4. Finalmente podrían aplicarse operaciones transaccionales o tags.

Dentro de `01_ddl/changelog.yaml`, el orden también es controlado:

```text
Extensiones
    ↓
Esquemas
    ↓
Tipos
    ↓
Tablas
    ↓
Alteraciones y claves foráneas
    ↓
Vistas, funciones, procedimientos, triggers e índices
```

Este diseño evita intentar crear un objeto antes de que existan sus dependencias.

---

## 9. Changesets implementados

Liquibase identifica cada cambio mediante un Changeset compuesto principalmente por:

- `id`.
- `author`.
- `labels`.
- Archivo SQL de ejecución.
- Definición de rollback.

Los Changesets activos principales son:

| ID | Función |
|---|---|
| `001-create-pgcrypto-extension` | Habilita generación de UUID con `gen_random_uuid()`. |
| `002-create-schema-document` | Crea el esquema propio `document`. |
| `003-create-table-document-template` | Crea la tabla de plantillas. |
| `004-create-table-document` | Crea la tabla principal de documentos. |
| `005-create-table-document-version` | Crea el historial de versiones. |
| `006-create-foreign-keys-tables` | Agrega las relaciones entre tablas. |
| `dcl-roles-document-001` | Crea roles de aplicación. |
| `dcl-grants-document-001` | Asigna permisos de mínimo privilegio. |

Cuando Liquibase ejecuta estos Changesets, también crea sus tablas internas:

- `DATABASECHANGELOG`: registra qué Changesets ya fueron ejecutados.
- `DATABASECHANGELOGLOCK`: evita ejecuciones concurrentes que puedan corromper el proceso.

Gracias a esto, ejecutar `update` varias veces no vuelve a ejecutar cambios ya registrados. Solo se aplican los Changesets nuevos.

---

## 10. Tablas implementadas

### 10.1 `document.document_template`

Representa una plantilla de generación.

Campos implementados:

| Campo | Función |
|---|---|
| `id` | UUID generado automáticamente. |
| `code` | Código identificador de la plantilla. |
| `name` | Nombre legible. |
| `template_body` | Contenido de la plantilla. |
| `output_type` | Tipo de salida esperado. |
| `version` | Versión de la plantilla. |
| `created_at` | Fecha de creación. |
| `created_by` | Actor que creó el registro. |
| `updated_at` | Fecha de actualización. |
| `updated_by` | Actor que actualizó el registro. |
| `state` | Estado general, con valor inicial `ACTIVE`. |

Esta tabla soportaría al componente `template-api`.

### 10.2 `document.document`

Es la tabla central del microservicio.

Campos implementados:

| Campo | Función |
|---|---|
| `id` | Identificador UUID. |
| `template_id` | Plantilla utilizada; puede ser nulo. |
| `title` | Título o nombre del documento. |
| `domain` | Dominio que originó el documento. |
| `owner_service` | Microservicio propietario o solicitante. |
| `owner_entity_id` | Entidad de negocio relacionada. |
| `storage_key` | Ruta del archivo en MinIO o S3. |
| `mime_type` | Tipo MIME. |
| `size_bytes` | Tamaño del archivo. |
| `status` | Estado funcional de generación o disponibilidad. |
| `row_version` | Control de concurrencia optimista. |
| `created_at`, `created_by` | Auditoría de creación. |
| `updated_at`, `updated_by` | Auditoría de actualización. |
| `state` | Estado técnico general. |

Esta tabla no almacena el PDF o Excel. Solo guarda su `storage_key`.

### 10.3 `document.document_version`

Almacena el historial de versiones.

Campos implementados:

| Campo | Función |
|---|---|
| `id` | Identificador UUID de la versión. |
| `document_id` | Documento padre. |
| `version_number` | Número de versión. |
| `storage_key` | Ruta del archivo correspondiente a esta versión. |
| `notes` | Observaciones. |
| Campos de auditoría | Creación y actualización. |
| `state` | Estado técnico general. |

La versión actual se mantiene también en `document.storage_key`, mientras que esta tabla conserva el historial.

---

## 11. Relaciones implementadas

Las tablas primero se crean sin claves foráneas y luego las relaciones se agregan desde `01_ddl/04_alter`.

### 11.1 Documento hacia plantilla

```text
document.template_id
        │
        ▼
document_template.id
```

Configuración:

- `ON UPDATE CASCADE`.
- `ON DELETE RESTRICT`.

Esto significa que una plantilla utilizada por un documento no puede eliminarse físicamente mientras exista la referencia.

### 11.2 Versión hacia documento

```text
document_version.document_id
        │
        ▼
document.id
```

Configuración:

- `ON UPDATE CASCADE`.
- `ON DELETE CASCADE`.

Si un documento se elimina físicamente, sus versiones también se eliminan.

---

## 12. Cómo funciona dentro de la carpeta de trabajo

El flujo de ejecución esperado es el siguiente:

```text
Desarrollador clona o abre la carpeta de trabajo
                        │
                        ▼
Docker Compose levanta PostgreSQL
                        │
                        ▼
El contenedor de Liquibase usa changelog-master.yaml
                        │
                        ▼
Liquibase revisa DATABASECHANGELOG
                        │
                        ▼
Ejecuta únicamente los Changesets pendientes
                        │
                        ▼
PostgreSQL queda con el esquema document, tablas, FKs, roles y grants
```

Desde la infraestructura Docker descrita por el repositorio, los comandos documentados son:

```bash
docker compose --env-file .env.develop up postgres -d

docker compose --env-file .env.develop \
  --profile tooling run --rm liquibase-document update
```

Para consultar el estado:

```bash
docker compose --profile tooling run --rm \
  liquibase-document status --verbose
```

Para revertir el último Changeset:

```bash
docker compose --profile tooling run --rm \
  liquibase-document rollbackCount 1
```

En una organización completa con dos carpetas hermanas, estos comandos se ejecutan normalmente desde `docker-infra`, que monta o referencia el changelog del repositorio de base de datos.

### Papel de `liquibase.properties`

Aunque no aparece en la copia comprimida analizada del repositorio DB, el contexto suministrado indica que se encuentra en `docker-infra` y define parámetros como:

- URL JDBC.
- Usuario y contraseña.
- Driver PostgreSQL.
- Ruta del changelog maestro.
- Esquema o configuración Liquibase.

### Papel de los archivos `.env`

Cada ambiente puede proporcionar valores diferentes para:

- Host.
- Puerto.
- Nombre de base de datos.
- Usuario.
- Contraseña.
- Nombre del contenedor.

La migración permanece igual. Solo cambia la configuración del ambiente.

---

## 13. Funcionamiento en Develop, QA, Staging y Main

La infraestructura fue preparada para que los cuatro ambientes ejecuten el mismo historial de migraciones.

El principio es:

```text
Mismo repositorio de Changesets
             +
Variables específicas del ambiente
             =
Misma estructura lógica de base de datos
```

Esto evita diferencias causadas por crear objetos manualmente desde pgAdmin o ejecutar scripts sueltos.

Un cambio de base de datos debe seguir este flujo:

1. Crear un archivo SQL nuevo.
2. Crear o actualizar el `changelog.yaml` correspondiente.
3. Asignar un ID de Changeset único.
4. Definir rollback.
5. Probar `update` desde cero.
6. Probar actualización incremental.
7. Probar rollback.
8. Hacer commit en una rama de historia de usuario.
9. Promover el cambio mediante Pull Request hacia los ambientes.

---

## 14. Reorganización realizada en HU-04

El historial Git confirma el commit principal:

```text
HU-04: reorganize repository structure, add docker-infra, and rename 04_constraints to 04_alter
```

El cambio más visible dentro del repositorio fue:

```text
Antes: 01_ddl/04_constraints/
Ahora: 01_ddl/04_alter/
```

El mismo cambio se reflejó en:

```text
05_rollbacks/01_ddl/04_alter/
```

También se actualizaron las rutas de los changelogs para apuntar a la nueva ubicación.

El nombre `04_alter` es más general que `04_constraints`, porque puede contener diferentes modificaciones posteriores a la creación de las tablas, no solamente constraints.

El historial también evidencia una corrección posterior de rutas para el changelog de `04_alter`, lo que demuestra la importancia de validar rutas relativas y sensibilidad a mayúsculas/minúsculas en Linux.

---

## 15. Diferencias entre el diseño y la implementación actual

La arquitectura de `07-document-service` describe un estado objetivo más completo que el SQL implementado.

### 15.1 Elementos implementados correctamente

- Esquema aislado `document`.
- UUID mediante `pgcrypto`.
- Tablas de plantilla, documento y versión.
- Relación documento-plantilla.
- Relación documento-versión.
- `storage_key` para object storage.
- Campos básicos de auditoría.
- `row_version` en documento.
- Roles reader, writer y admin.
- Rollbacks.
- Changelogs organizados.

### 15.2 Elementos descritos, pero todavía no implementados

- Tabla de firma digital.
- Restricción única para `document_template.code`.
- Restricción única para `(document_id, version_number)`.
- Índice para `(owner_service, owner_entity_id)`.
- Índice para `document.status`.
- Restricciones `CHECK` para `output_type`.
- Restricciones `CHECK` para `domain`.
- Restricciones `CHECK` para `status`.
- Validación `size_bytes >= 0`.
- Campo `failure_reason` mencionado en eventos y runbook.
- `deleted_at` y `deleted_by` para borrado lógico.
- Campo booleano `is_active` propuesto por la convención.
- Uso consistente de `TIMESTAMPTZ`.
- APIs y workers.
- Eventos Kafka.
- Integración real con MinIO/S3.
- Funciones, procedimientos, triggers y vistas.

### 15.3 Diferencias de tipos y nombres

| Diseño documentado | Implementación actual |
|---|---|
| `is_active BOOLEAN` | `state VARCHAR(20)` |
| `created_by UUID` | `created_by VARCHAR(100)` |
| `updated_by UUID` | `updated_by VARCHAR(100)` |
| `TIMESTAMPTZ` | `TIMESTAMP` |
| `owner_service VARCHAR(50)` | `owner_service VARCHAR(100)` |
| `status VARCHAR(20)` | `status VARCHAR(30)` |
| MIME obligatorio | `mime_type` permite nulo |
| `updated_at` obligatorio en el modelo | `updated_at` permite nulo |

Estas diferencias no significan que Liquibase esté fallando. Significan que la implementación todavía no está completamente alineada con el modelo arquitectónico más reciente.

---

## 16. Observaciones técnicas sobre los rollbacks

Los rollbacks permiten revertir Changesets, pero deben manejarse con cuidado.

Por ejemplo:

```sql
DROP SCHEMA IF EXISTS document;
```

Sin `CASCADE`, el rollback del esquema solo funcionará si previamente se eliminaron todos los objetos contenidos en él.

Asimismo, eliminar `pgcrypto` puede afectar otros esquemas si comparten la misma instancia PostgreSQL y también dependen de la extensión. En una base compartida por varios microservicios, la extensión debería administrarse con una política coordinada.

El rollback de grants actualmente ejecuta `SELECT 1`, es decir, no revoca los permisos concedidos. Liquibase considera que existe un bloque de rollback, pero funcionalmente ese rollback no deshace los grants. Esto puede mejorarse con sentencias `REVOKE` y restauración de privilegios predeterminados.

---

## 17. Qué debe explicarse durante una sustentación

Una explicación clara puede seguir este orden:

### 17.1 Responsabilidad

> El repositorio administra la base de datos del Document Service. No contiene el backend del microservicio.

### 17.2 Razón para usar Liquibase

> Liquibase permite versionar la estructura, registrar qué cambios fueron aplicados y reproducir la misma base de datos en Develop, QA, Staging y Main.

### 17.3 Modelo implementado

> Se crearon las tablas de plantillas, documentos y versiones. Los archivos reales se almacenarán en MinIO o S3, y PostgreSQL guarda únicamente sus metadatos y rutas.

### 17.4 Orden de ejecución

> El changelog maestro llama primero DDL, después DML, DCL y TCL. Dentro del DDL se crean extensión, esquema, tablas y finalmente relaciones.

### 17.5 Seguridad

> Se implementaron roles de lectura, escritura y administración siguiendo el principio de mínimo privilegio.

### 17.6 Reversibilidad

> Cada Changeset estructural contiene su rollback para poder revertir cambios controladamente.

### 17.7 Reorganización HU-04

> Se adoptó la estructura estándar del proyecto, se separó la infraestructura Docker y se renombró `04_constraints` a `04_alter`, actualizando los changelogs y rollbacks.

### 17.8 Alcance no desarrollado

> No se desarrollaron endpoints, controladores, servicios Java, workers, generación de archivos ni integración con Kafka o almacenamiento de objetos. Esos elementos están definidos en la arquitectura, pero corresponden a otros repositorios o futuras implementaciones.

---

## 18. Ejemplo del flujo completo cuando exista el backend

```text
1. scheduling-service publica que un horario fue aprobado.

2. Document Service recibe la solicitud.

3. Se inserta un registro en document.document:
   - status = GENERATING
   - owner_service = scheduling-service
   - owner_entity_id = UUID del horario
   - template_id = plantilla seleccionada

4. Se envía el trabajo a document-generation-queue.

5. pdf-renderer-worker genera el PDF.

6. El PDF se sube a MinIO o S3.

7. Se actualiza document.document:
   - storage_key = ruta del archivo
   - mime_type = application/pdf
   - size_bytes = tamaño real
   - status = AVAILABLE

8. Se crea una fila en document.document_version.

9. Se publica document.document.generated.

10. document-api puede generar una URL prefirmada de descarga.
```

La carpeta de trabajo implementa las estructuras de los pasos 3, 7 y 8. Los demás pasos requieren código de aplicación e infraestructura de mensajería y almacenamiento.

---

## 19. Conclusión

`07-document-service` define la arquitectura integral de un servicio de gestión documental, compuesto por APIs, workers, eventos, object storage y una base de datos de metadatos.

La carpeta de trabajo implementa específicamente el repositorio **`design-software-document-db`**, cuya función es administrar la estructura PostgreSQL mediante Liquibase.

Actualmente el repositorio permite:

- Crear la extensión necesaria para UUID.
- Crear el esquema aislado `document`.
- Crear las tablas principales.
- Crear sus relaciones.
- Crear roles y permisos.
- Registrar los cambios mediante Changesets.
- Ejecutar migraciones incrementales.
- Revertir los principales cambios estructurales.
- Reproducir la misma base de datos entre ambientes.

El resultado es una base técnica ordenada y reproducible para que, cuando se implemente el backend de Document Service, sus componentes puedan trabajar sobre una estructura controlada y consistente, sin depender de configuraciones manuales en PostgreSQL.
