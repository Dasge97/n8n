# n8n-infra

Infraestructura portable con Docker Compose para ejecutar `n8n` como parte del ecosistema `dedaniel`. La pila está pensada para vivir en un repositorio Git, poder clonarse en cualquier entorno y desplegarse más adelante detrás de un proxy inverso Traefik ya existente con Let's Encrypt y enrutamiento por subdominios.

La pila incluye:

- `n8n-app` para el servicio de automatización
- `n8n-postgres` para la base de datos PostgreSQL dedicada
- etiquetas de Traefik para `https://n8n.dedaniel.com`
- volúmenes locales para persistir los datos de la aplicación y de la base de datos
- configuración preparada para webhooks detrás de Traefik

PostgreSQL guarda sus datos reales dentro de `./postgres/data/pgdata` para evitar problemas al versionar la carpeta `data`.

## Estructura de carpetas

```text
n8n-infra/
├── .env.example
├── .gitignore
├── README.md
├── docker-compose.yml
├── n8n/
│   └── data/
│       └── .gitkeep
└── postgres/
    └── data/
        └── .gitkeep
```

## Configuración del entorno

1. Copia el archivo de ejemplo:

   ```bash
   cp .env.example .env
   ```

2. Edita `.env` y reemplaza las contraseñas de ejemplo:

   - `POSTGRES_PASSWORD`
   - `DB_POSTGRESDB_PASSWORD`

3. Mantén sincronizadas las contraseñas de PostgreSQL:

   - `POSTGRES_PASSWORD` y `DB_POSTGRESDB_PASSWORD` deben tener el mismo valor.

4. La validación de Compose es estricta a propósito:

   - La pila no arrancará hasta que las variables de contraseña obligatorias estén definidas en `.env`.

Los valores por defecto ya coinciden con el despliegue previsto:

- Dominio: `n8n.dedaniel.com`
- Zona horaria: `Europe/Madrid`
- Nombre de la base de datos: `n8n`
- Usuario de la base de datos: `n8n`
- Cert resolver de Traefik: `letsencrypt` (ajústalo si tu servidor usa otro nombre, por ejemplo `myresolver`)

## Instrucciones para desarrollo local

Este proyecto asume que Traefik ya está en ejecución y que existe una red externa compartida de Docker llamada `web`.

1. Crea la red externa una vez, si todavía no existe:

   ```bash
   docker network create web
   ```

2. Configura `.env`:

   ```bash
   cp .env.example .env
   ```

3. Arranca la pila:

   ```bash
   docker compose up -d
   ```

4. Revisa el estado:

   ```bash
   docker compose ps
   ```

5. Consulta los logs si lo necesitas:

   ```bash
   docker compose logs -f n8n-app
   ```

Notas:

- Sin una instancia de Traefik conectada a la red `web`, los contenedores arrancarán igualmente, pero el servicio no será accesible desde `https://n8n.dedaniel.com`.
- Para pruebas locales usando el hostname de producción, asegúrate de que tu DNS o tu archivo `hosts` resuelvan `n8n.dedaniel.com` hacia la máquina donde está corriendo Traefik.
- PostgreSQL inicializa la base de datos en `./postgres/data/pgdata`, mientras que `./postgres/data/.gitkeep` solo existe para conservar la carpeta en Git.
- Los webhooks externos usan `WEBHOOK_URL`, que debe apuntar a la URL pública HTTPS de n8n cuando está detrás de Traefik.

## Instrucciones de despliegue en el servidor

Ruta objetivo en el servidor:

```text
/srv/infra/n8n
```

Flujo de despliegue sugerido:

1. Clona el repositorio en el servidor:

   ```bash
   git clone <your-repository-url> /srv/infra/n8n
   ```

2. Entra en el directorio del proyecto:

   ```bash
   cd /srv/infra/n8n
   ```

3. Crea y edita el archivo de entorno:

   ```bash
   cp .env.example .env
   ```

   Si tu instancia de Traefik no usa un resolver llamado `letsencrypt`, cambia `TRAEFIK_CERT_RESOLVER` al nombre real configurado en ese servidor.

4. Confirma que la red compartida de Traefik existe:

   ```bash
   docker network ls
   ```

5. Descarga las imágenes:

   ```bash
   docker compose pull
   ```

6. Arranca la pila:

   ```bash
   docker compose up -d
   ```

Si un arranque anterior falló antes de este ajuste y ves errores de inicialización de PostgreSQL, elimina únicamente el directorio parcial `postgres/data/pgdata` y vuelve a levantar la pila.

7. Verifica el acceso:

   - Abre `https://n8n.dedaniel.com`
   - Inicia sesión con la cuenta interna de n8n

## Instrucciones de actualización

Cuando actualices la infraestructura o las imágenes de los contenedores:

1. Revisa los cambios entrantes:

   ```bash
   git pull
   ```

2. Comprueba si `.env.example` cambió y actualiza `.env` si hace falta.

3. Descarga las últimas imágenes:

   ```bash
   docker compose pull
   ```

4. Recrea los servicios con la nueva configuración:

   ```bash
   docker compose up -d
   ```

5. Confirma que todo está sano:

   ```bash
   docker compose ps
   ```

Los datos persistentes permanecen en:

- `./postgres/data` (PostgreSQL usa `./postgres/data/pgdata`)
- `./n8n/data`
