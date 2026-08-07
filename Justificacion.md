## Justificación para implementar la carpeta `.github`

Se propone agregar la siguiente estructura en los nueve repositorios de base de datos:

```text
.github/
├── CODEOWNERS
├── pull_request_template.md
├── dependabot.yml
└── workflows/
    └── database-pipeline.yml
```

El objetivo es establecer controles técnicos y automáticos que garanticen que los cambios realizados sobre las migraciones de base de datos sean revisados, aprobados, validados y registrados antes de llegar a las ramas principales.

Actualmente, un cambio incorrecto en un archivo SQL, en un changelog de Liquibase, en un rol o en una política de acceso puede provocar errores de despliegue, pérdida de información, inconsistencias entre ambientes o accesos no autorizados. Esta estructura permite reducir esos riesgos mediante controles preventivos.

---

# 1. Justificación de `CODEOWNERS`

El archivo:

```text
.github/CODEOWNERS
```

define oficialmente quiénes son los responsables de revisar y aprobar los cambios realizados en el repositorio.

En este proyecto, se utilizará para establecer que el Instructor y los coordinadores sean los responsables administrativos de los cambios críticos.

Ejemplo:

```text
* @code-sena/design-software
```

## ¿Por qué se necesita?

Porque permite que GitHub solicite automáticamente la revisión del equipo responsable cuando se modifiquen archivos del repositorio.

Esto evita que un cambio pueda ser aprobado únicamente por miembros del equipo de desarrollo sin la revisión de las personas encargadas de la seguridad y el gobierno del proyecto.

## Beneficios

* Garantiza una aprobación administrativa.
* Asigna responsables claros.
* Evita cambios sin supervisión.
* Protege archivos críticos como pipelines, roles, permisos y changelogs.
* Mantiene trazabilidad sobre quién autorizó cada modificación.
* Reduce el riesgo de cambios accidentales o no autorizados.

## Riesgo que controla

Sin `CODEOWNERS`, cualquier persona con permisos suficientes podría revisar un Pull Request y GitHub no sabría que debe solicitar obligatoriamente la aprobación de un coordinador o del Instructor.

---

# 2. Justificación de `pull_request_template.md`

El archivo:

```text
.github/pull_request_template.md
```

crea una plantilla automática cada vez que un integrante abre un Pull Request.

La plantilla obliga al autor a documentar información como:

* Qué se modificó.
* Por qué se realizó el cambio.
* Qué tablas, columnas, roles o scripts fueron afectados.
* Qué pruebas se ejecutaron.
* Si existe rollback.
* Si hay riesgo de pérdida de datos.
* Si se modifican permisos.
* Si el cambio afecta otros servicios.

## ¿Por qué se necesita?

Los cambios de base de datos no deben revisarse únicamente viendo el código. El revisor necesita conocer el impacto funcional, técnico y de seguridad.

La plantilla evita que se creen Pull Requests con descripciones incompletas como:

```text
“Se corrigió la base de datos”
```

y obliga a entregar información suficiente para tomar una decisión de aprobación o rechazo.

## Beneficios

* Estandariza todos los Pull Requests.
* Facilita la revisión de líderes y coordinadores.
* Obliga a documentar pruebas y riesgos.
* Permite verificar si existe rollback.
* Mejora la comunicación entre equipos.
* Genera evidencia para auditorías.
* Reduce errores por falta de contexto.

## Riesgo que controla

Sin una plantilla, cada desarrollador documentaría los cambios de manera diferente y podría omitir información crítica, como pérdida de datos, bloqueos de tablas o cambios en permisos.

---

# 3. Justificación de `dependabot.yml`

El archivo:

```text
.github/dependabot.yml
```

configura Dependabot para revisar periódicamente las dependencias utilizadas por GitHub Actions.

Aunque estos repositorios sean principalmente de migraciones, utilizan acciones externas dentro de los pipelines, por ejemplo:

```yaml
actions/checkout
actions/upload-artifact
```

Estas acciones también deben actualizarse y mantenerse seguras.

## ¿Por qué se necesita?

Las dependencias y acciones utilizadas en los workflows pueden quedar desactualizadas o presentar vulnerabilidades.

Dependabot crea Pull Requests automáticos cuando existe una actualización disponible.

## Beneficios

* Mantiene actualizadas las acciones de GitHub.
* Reduce vulnerabilidades conocidas.
* Evita depender de versiones obsoletas.
* Automatiza la revisión periódica.
* Genera Pull Requests visibles y auditables.
* Permite que los coordinadores aprueben o rechacen cada actualización.

## Riesgo que controla

Sin Dependabot, las acciones del pipeline podrían permanecer durante meses o años usando versiones antiguas, incompatibles o vulnerables.

Dependabot no realiza cambios directos en `main`; únicamente propone un Pull Request que debe pasar por las mismas revisiones y controles del proyecto.

---

# 4. Justificación de `database-pipeline.yml`

El archivo:

```text
.github/workflows/database-pipeline.yml
```

implementa el pipeline automático de validación para los repositorios de base de datos.

Este pipeline se ejecutará cada vez que se cree o actualice un Pull Request hacia `develop` o `main`.

## ¿Qué validará?

Entre otros controles:

* Existencia de la estructura obligatoria del repositorio.
* Ausencia de archivos sensibles.
* Ausencia de claves privadas o archivos `.env`.
* Ausencia de conflictos Git sin resolver.
* Uso seguro de permisos y roles.
* Existencia de rollback.
* Sintaxis y estructura de Liquibase.
* Ejecución de `liquibase validate`.
* Ejecución de las migraciones sobre PostgreSQL 16.
* Generación previa del SQL.
* Verificación de changesets pendientes.
* Prueba de rollback.
* Reaplicación de la migración después del rollback.

## ¿Por qué se necesita?

Una revisión manual no siempre detecta:

* Errores de sintaxis SQL.
* Includes incorrectos.
* Changesets duplicados.
* Dependencias faltantes.
* Errores de conexión.
* Rollbacks inválidos.
* Scripts que fallan al ejecutarse.
* Permisos peligrosos.
* Archivos sensibles agregados accidentalmente.

El pipeline prueba automáticamente los cambios en una base de datos temporal antes de permitir el merge.

## Beneficios

* Detecta errores antes del despliegue.
* Evita migraciones defectuosas.
* Valida compatibilidad con PostgreSQL 16.
* Comprueba que Liquibase pueda ejecutar los cambios.
* Genera evidencia automática.
* Reduce pruebas manuales repetitivas.
* Mantiene el mismo estándar en los nueve repositorios.
* Bloquea el merge si el cambio no cumple la política.

## Riesgo que controla

Sin pipeline, una migración podría aprobarse por revisión visual y fallar posteriormente en integración o producción.

Eso podría generar:

* Interrupción del servicio.
* Pérdida o corrupción de datos.
* Inconsistencias entre ambientes.
* Despliegues incompletos.
* Rollbacks imposibles.
* Permisos excesivos.
* Retrasos en el proyecto.

---

# 5. Por qué se utiliza la carpeta `.github`

La carpeta:

```text
.github/
```

es la ubicación estándar de GitHub para almacenar configuraciones relacionadas con:

* Gobierno del repositorio.
* Pull Requests.
* Propietarios de código.
* Automatización.
* Dependencias.
* Pipelines.
* Seguridad.

No modifica directamente la lógica de la base de datos ni los scripts existentes.

Su función es agregar una capa de control alrededor del proceso de desarrollo.

```text
Scripts y migraciones
        ↓
Controles de .github
        ↓
Revisión
        ↓
Pipeline
        ↓
Aprobación
        ↓
Merge autorizado
```

---

# 6. Relación con la política de seguridad

Cada archivo implementa una parte específica de la política:

| Archivo                    | Control implementado                              |
| -------------------------- | ------------------------------------------------- |
| `CODEOWNERS`               | Define quién debe aprobar                         |
| `pull_request_template.md` | Exige documentación y evidencias                  |
| `dependabot.yml`           | Mantiene dependencias y Actions actualizadas      |
| `database-pipeline.yml`    | Valida automáticamente seguridad y funcionamiento |

En conjunto permiten cumplir cuatro principios:

### Prevención

Evitar que cambios inseguros lleguen a las ramas protegidas.

### Detección

Identificar errores, secretos o configuraciones inseguras.

### Aprobación

Garantizar revisión del líder y de los responsables administrativos.

### Trazabilidad

Registrar quién creó, revisó, aprobó y fusionó cada cambio.

---

# 7. Impacto sobre el trabajo del equipo

La implementación no impide que los desarrolladores trabajen.

Los desarrolladores podrán:

* Crear ramas.
* Modificar scripts.
* Crear changesets.
* Proponer migraciones.
* Abrir Pull Requests.
* Corregir observaciones.
* Ejecutar pruebas.

La diferencia es que no podrán fusionar cambios defectuosos o no autorizados.

El flujo quedará:

```text
Desarrollador realiza el cambio
            ↓
Crea Pull Request
            ↓
Completa la plantilla
            ↓
El pipeline valida automáticamente
            ↓
El líder técnico revisa
            ↓
Instructor o coordinador aprueba
            ↓
Coordinador realiza el merge
```

---

# 8. Costos o consideraciones

La implementación tiene algunos costos operativos menores:

* Los Pull Requests pueden tardar más porque requieren revisión.
* Los pipelines pueden tardar varios minutos.
* Inicialmente será necesario corregir migraciones antiguas sin rollback.
* Los coordinadores deberán revisar las alertas y actualizaciones de Dependabot.
* Se deberá capacitar brevemente al equipo.

Sin embargo, estos costos son menores comparados con el impacto de una migración defectuosa en un ambiente compartido o productivo.

---

# 9. Beneficios Generales

Se propone agregar la carpeta `.github` en los nueve repositorios de base de datos con el fin de implementar controles técnicos de seguridad, revisión y validación automática esto permitara:

1. Formalizar quiénes pueden aprobar cambios.
2. Estandarizar la documentación de los Pull Requests.
3. Automatizar pruebas de migraciones.
4. Detectar errores antes del merge.
5. Proteger los roles y permisos de base de datos.
6. Comprobar los rollbacks.
7. Evitar secretos y archivos sensibles.
8. Mantener actualizadas las dependencias del pipeline.
9. Crear evidencia para seguimiento y auditoría.
10. Aplicar la misma política en todos los repositorios.


y El archivo `CODEOWNERS` permitirá que los cambios requieran aprobación del Instructor o de los coordinadores. La plantilla de Pull Request obligará a documentar el impacto, las pruebas y el rollback de cada migración. Dependabot ayudará a mantener actualizadas las acciones utilizadas por GitHub, y el pipeline validará las migraciones sobre PostgreSQL 16 mediante Liquibase antes de permitir el merge.

Esta implementación no modifica la lógica funcional de las bases de datos. Agrega una capa de control para evitar cambios no autorizados, errores de migración, pérdida de datos, permisos inseguros y despliegues sin evidencia.

Los equipos desarrolladores conservarán la posibilidad de crear ramas y Pull Requests, pero los cambios solo podrán integrarse cuando hayan sido revisados, aprobados y validados automáticamente.


## Pruebas Pilotos

La propuesta se aprobara inicialmente en un repositorio aparte, después de comprobar que el flujo funciona correctamente, podrá replicarse en los otros ocho repositorios. Para garantizar una mayor seguridad.
