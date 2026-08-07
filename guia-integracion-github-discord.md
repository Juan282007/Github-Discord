# Integración de GitHub con Discord

## Objetivo

Crear un canal en Discord donde se publiquen automáticamente las notificaciones de actividad de los 11 repositorios del grupo `design-software-develop`.

Las notificaciones mostrarán principalmente:

- Repositorio modificado.
- Usuario que realizó el cambio.
- Rama afectada.
- Mensaje del commit.
- Pull Requests.
- Releases.

La integración se realizará mediante **webhooks**, sin necesidad de crear un bot ni una API propia.

```text
Repositorios de GitHub
        ↓
Webhooks
        ↓
Canal de Discord
```

---

## Requisitos en Discord

Se necesita:

```text
Permiso: Administrar webhooks
```

También se debe crear un canal de texto, por ejemplo:

```text
#github-actividad
```

### Crear el webhook de Discord

Ir a:

```text
Configuración del servidor
→ Integraciones
→ Webhooks
→ Nuevo webhook
```

Configurar:

```text
Nombre: GitHub code-sena
Canal: #github-actividad
```

Luego copiar la URL del webhook.

Ejemplo:

```text
https://discord.com/api/webhooks/ID/TOKEN
```

Agregar al final:

```text
/github
```

La URL final debe quedar así:

```text
https://discord.com/api/webhooks/ID/TOKEN/github
```

> La URL debe mantenerse privada porque funciona como una credencial.

---

## Requisitos en GitHub

Ya se cuenta con:

```text
Rol Admin en los 11 repositorios
```

Eso permite acceder en cada repositorio a:

```text
Repositorio
→ Settings
→ Webhooks
→ Add webhook
```

No es necesario ser creador ni `Owner` de la organización, porque la integración se configurará individualmente en cada repositorio.

---

## Configuración en cada repositorio

En cada uno de los 11 repositorios:

1. Ir a:

```text
Settings
→ Webhooks
→ Add webhook
```

2. Configurar:

```text
Payload URL:
URL de Discord terminada en /github

Content type:
application/json

Secret:
vacío

SSL verification:
activado

Active:
activado
```

3. Seleccionar:

```text
Let me select individual events
```

4. Activar inicialmente:

```text
Pushes
Pull requests
Releases
```

5. Guardar con:

```text
Add webhook
```

Este proceso debe repetirse en los 11 repositorios usando la misma URL de Discord.

---

## Prueba inicial

Se recomienda configurar primero un solo repositorio.

Después realizar una prueba:

```bash
git add .
git commit -m "Prueba integración GitHub con Discord"
git push
```

En Discord debe aparecer:

- Nombre del repositorio.
- Rama.
- Usuario.
- Commit.
- Descripción del commit.
- Enlace al cambio.

Si la prueba funciona, se replica la misma configuración en los otros 10 repositorios.

---

## Resultado final

La arquitectura será:

```text
Repositorio 1  ─┐
Repositorio 2  ─┤
Repositorio 3  ─┤
...             ├──→ Webhook de Discord → #github-actividad
Repositorio 11 ─┘
```

---

## Resumen de requisitos

### En Discord

```text
Permiso Administrar webhooks
Canal #github-actividad
Un webhook de Discord
```

### En GitHub

```text
Rol Admin en los 11 repositorios
Un webhook configurado en cada repositorio
Eventos Pushes, Pull requests y Releases
```

---

## Conclusión

No se necesita:

- Bot de Discord.
- Servidor propio.
- Base de datos.
- API personalizada.
- Código adicional.

La implementación se hace creando un webhook en Discord y configurando esa misma URL en cada uno de los 11 repositorios de GitHub.
