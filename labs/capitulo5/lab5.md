---
layout: lab
title: "Práctica 5: Redes y comunicación entre contenedores"
permalink: /capitulo5/lab5/
images_base: /labs/capitulo5/img
duration: "60 minutos"
objective:
  - Aprender a **diseñar y validar la comunicación entre contenedores** usando **redes personalizadas de Docker** (bridge). Construirás una app multi-servicio con **API Node.js (Express)**, **PostgreSQL** y **Nginx (reverse proxy)** en **dos redes**. `frontend-net` (expuesta) y `backend-net` (privada). Practicarás **DNS interno**, **aislamiento de redes**, **múltiples redes por contenedor**, variables de entorno y **comprobaciones de salud**.
prerequisites:
  - Visual Studio Code
  - Docker Desktop en ejecución
  - Terminal Git Bash en VS Code
  - Conocimientos básicos de Node.js, SQL y Docker
  - Haber realizado prácticas previas de imágenes y volúmenes
introduction: |
  En entornos de microservicios solemos separar el tráfico **externo** (hacia un proxy o API Gateway) del **interno** (bases de datos, colas, servicios internos). Las **redes bridge definidas por el usuario** en Docker agregan.
  - **DNS interno.** resolución de nombres de contenedores (p. ej., `db`, `api`).
  - **Aislamiento**. servicios sensibles (BD) en redes privadas sin puertos expuestos al host.
  - **Flexibilidad**. un contenedor (Nginx) puede participar en múltiples redes.
  
  En esta práctica.
  - **PostgreSQL.** vivirá en `backend-net` (no expone puertos al host).
  - **API Node.js.** también en `backend-net` y consumirá la BD por **nombre de host** `db`.
  - **Nginx.** hará de **proxy inverso**, conectado a **ambas redes**. Expondremos solo al proxy (puerto 8080).
slug: lab5
lab_number: 5
final_result: >
  Arquitectura de **tres contenedores** con **dos redes**: `frontend-net` (expuesta) y `backend-net` (privada). **Nginx** enruta al **API**, el **API** consulta **PostgreSQL** vía **DNS interno**. Se valida aislamiento, nombres de servicio y salud de la aplicación.
notes: 
  - En producción añade SSL (Nginx) y usuarios/roles con permisos mínimos en Postgres.
  - Para desarrollo, puedes mapear un volumen para la carpeta de la API y activar **nodemon** para recarga.
  - Considera usar **Docker Compose** para declarar estos servicios y redes de forma reproducible.
references:
  - text: Redes en Docker
    url: https://docs.docker.com/network/
  - text: Bridge networks
    url: https://docs.docker.com/network/bridge/
  - text: Nginx reverse proxy
    url: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
  - text: PostgreSQL imagen oficial
    url: https://hub.docker.com/_/postgres
prev: /capitulo4/lab4          
next: /capitulo6/lab6/
---


---

### Tarea 1: Crear la estructura del proyecto

Generar una estructura clara para mapear código, configuración y scripts de inicialización de la base de datos.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre **Visual Studio Code (VS Code)**; lo puedes encontrar en el **Escritorio** del ambiente o buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VS Code**, da clic en el ícono de la imagen para abrir la terminal, ubicado en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 4.** Selecciona la terminal **Git Bash**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 5.** Asegúrate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VS Code**:

  > **NOTA:** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica 5**:

  > **NOTA:** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab5-dockernetworks && cd lab5-dockernetworks
  ```

- **Paso 7.** Verifica en el **Explorador** de archivos dentro de VS Code que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Crearás la siguiente estructura inicial del proyecto de la aplicación:

  > **NOTA:**  
  > - `db/init.sql` inicializa el esquema al arrancar PostgreSQL.  
  > - `proxy/nginx.conf` define el enrutamiento hacia la API.  
  > - `Dockerfile.api` contiene la construcción de la API.  
  {: .lab-note .info .compact}

  ```text
  lab5-dockernetworks/
  ├── api/
  │   ├── package.json
  │   └── server.js
  ├── db/
  │   └── init.sql
  ├── proxy/
  │   └── nginx.conf
  ├── Dockerfile.api
  └── .dockerignore
  ```

- **Paso 9.** Crea la carpeta **api/** y sus archivos vacíos.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab5-dockernetworks**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p api && touch api/package.json api/server.js
  ```

- **Paso 10.** Crea la carpeta **db/** y su archivo vacío.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab5-dockernetworks**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p db && touch db/init.sql
  ```

- **Paso 11.** Crea la carpeta **proxy/** y su archivo vacío.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab5-dockernetworks**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p proxy && touch proxy/nginx.conf
  ```

- **Paso 12.** Crea los últimos archivos del proyecto: **.dockerignore** y **Dockerfile.api**.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab5-dockernetworks**.
  {: .lab-note .info .compact}

  ```bash
  touch .dockerignore Dockerfile.api
  ```

- **Paso 13.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes más limpias:

  > **NOTA:** Evita copiar artefactos innecesarios dentro de la imagen.
  {: .lab-note .info .compact}

  ```gitignore
  node_modules
  npm-debug.log
  .DS_Store
  ```

- **Paso 14.** Valida la creación de la estructura de tu proyecto; escribe el siguiente comando:

  > **NOTA:** También puedes validarlo en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Deben existir `api/`, `db/`, `proxy/`, `Dockerfile.api` y `.dockerignore`.
  {: .lab-note .important .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Base de datos PostgreSQL e inicialización

Definir un script SQL de arranque y variables de entorno para crear una base de datos `agenda` con la tabla `usuarios`.

#### Tarea 2.1

- **Paso 15.** Abre el archivo `db/init.sql` y agrega el siguiente código SQL:

  > **NOTA:** Este archivo se ejecuta automáticamente cuando el contenedor de PostgreSQL detecta scripts en `/docker-entrypoint-initdb.d`.
  {: .lab-note .info .compact}

  ```sql
  CREATE DATABASE agenda;
  \c agenda;

  CREATE TABLE IF NOT EXISTS usuarios (
    id SERIAL PRIMARY KEY,
    nombre TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
  );

  INSERT INTO usuarios (nombre, email) VALUES
  ('Ada Lovelace', 'ada@example.com'),
  ('Grace Hopper', 'grace@example.com')
  ON CONFLICT DO NOTHING;
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Implementar API Node.js (Express) conectada a PostgreSQL

Exponer endpoints `/health`, `/usuarios` (GET) y `/usuarios` (POST) utilizando el paquete `pg`.

#### Tarea 3.1

- **Paso 16.** Abre el archivo `api/package.json` y agrega el siguiente contenido para definir las dependencias:

  ```json
  {
    "name": "api-agenda",
    "version": "1.0.0",
    "main": "server.js",
    "type": "module",
    "scripts": {
      "start": "node server.js"
    },
    "dependencies": {
      "express": "^4.18.2",
      "pg": "^8.11.5"
    }
  }
  ```

- **Paso 17.** Abre el archivo `api/server.js`, copia y pega el siguiente código:

  > **NOTA:**  
  > - **Servidor Express** corriendo en un puerto configurable (`PORT` o 3000).  
  > - **Conexión a PostgreSQL** utilizando `pg.Pool` con variables de entorno (host, puerto, usuario, contraseña y base de datos).  
  > - **Middleware JSON** para procesar peticiones con cuerpo en formato JSON.  
  > - **Endpoint `/health`**: ejecuta `SELECT 1` para verificar el estado de la base de datos.  
  > - **Endpoint GET `/usuarios`**: devuelve la lista de usuarios (`id`, `nombre`, `email`) ordenados por ID descendente.  
  > - **Endpoint POST `/usuarios`**: valida los campos y crea un usuario nuevo en la tabla `usuarios`, devolviendo el registro creado.  
  {: .lab-note .info .compact}

  ```javascript
  import express from "express";
  import pkg from "pg";
  const { Pool } = pkg;

  const app = express();
  const PORT = process.env.PORT || 3000;

  // Variables de entorno para la conexión
  const DB_HOST = process.env.DB_HOST || "db";
  const DB_PORT = Number(process.env.DB_PORT || 5432);
  const DB_USER = process.env.DB_USER || "postgres";
  const DB_PASSWORD = process.env.DB_PASSWORD || "postgres";
  const DB_NAME = process.env.DB_NAME || "agenda";

  const pool = new Pool({
    host: DB_HOST,
    port: DB_PORT,
    user: DB_USER,
    password: DB_PASSWORD,
    database: DB_NAME
  });

  app.use(express.json());

  app.get("/health", async (_req, res) => {
    try {
      const { rows } = await pool.query("SELECT 1 as ok");
      return res.json({ status: "ok", db: rows[0].ok === 1 ? "up" : "unknown" });
    } catch (err) {
      return res.status(500).json({ status: "error", message: err.message });
    }
  });

  app.get("/usuarios", async (_req, res) => {
    try {
      const { rows } = await pool.query("SELECT id, nombre, email FROM usuarios ORDER BY id DESC");
      return res.json(rows);
    } catch (err) {
      return res.status(500).json({ error: err.message });
    }
  });

  app.post("/usuarios", async (req, res) => {
    const { nombre, email } = req.body || {};
    if (!nombre || !email) return res.status(400).json({ error: "nombre y email son requeridos" });
    try {
      const { rows } = await pool.query(
        "INSERT INTO usuarios (nombre, email) VALUES ($1, $2) RETURNING id, nombre, email",
        [nombre, email]
      );
      return res.status(201).json(rows[0]);
    } catch (err) {
      return res.status(500).json({ error: err.message });
    }
  });

  app.listen(PORT, () => console.log(`API escuchando en http://localhost:${PORT}`));
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Dockerfile de la API con HEALTHCHECK

Construir la imagen `api-agenda` y añadir comprobación de salud.

#### Tarea 4.1

- **Paso 18.** Abre el archivo `Dockerfile.api` que creaste en la tarea 1 y pega el siguiente contenido para compilar la imagen Docker:

  > **NOTA:** El `HEALTHCHECK` permite observar el estado `healthy/unhealthy` de la API.
  {: .lab-note .info .compact}

  ```dockerfile
  FROM node:20-alpine

  WORKDIR /app
  COPY api/package*.json ./
  RUN npm install --production
  COPY api ./

  ENV PORT=3000
  EXPOSE 3000

  RUN apk add --no-cache curl
  HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3     CMD curl -fsS http://localhost:3000/health || exit 1

  CMD ["node", "server.js"]
  ```

- **Paso 19.** En la terminal de VS Code, dentro del directorio **lab5-dockernetworks**, ejecuta el siguiente comando:

  ```bash
  docker build -f Dockerfile.api -t api-agenda .
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 20.** Verifica que se haya creado correctamente la imagen; escribe el siguiente comando:

  > **NOTA:** La imagen debe existir con su tamaño reportado.
  {: .lab-note .info .compact}

  ```bash
  docker images | grep api-agenda
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Crear redes y levantar PostgreSQL (solo backend)

Crear redes y ejecutar PostgreSQL conectado **solo a `backend-net`**. Montar el script de inicialización.

#### Tarea 5.1

- **Paso 21.** En la terminal de VS Code, ejecuta los siguientes comandos para crear las redes de Docker:

  > **NOTA:** Ejecuta los comandos uno por uno.
  {: .lab-note .info .compact}

  ```bash
  docker network create backend-net
  ```

  ```bash
  docker network create frontend-net
  ```

  ```bash
  docker network ls | grep -E "backend-net|frontend-net"
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 22.** Ejecuta el siguiente `docker run` para levantar la base de datos de ejemplo en PostgreSQL:

  > **NOTA:**
  > - No publicamos el puerto 5432; el servicio `db` será visible solo en `backend-net`.
  > - Si no tienes la imagen **postgres:16-alpine**, se descargará automáticamente.
  {: .lab-note .info .compact}

  ```bash
  MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" \
  docker run -d --name db \
  --network backend-net \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_USER=postgres \
  -v "$(pwd)"/db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro \ postgres:16-alpine
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 23.** Para validar que está correctamente en ejecución, escribe el siguiente comando en la terminal.

  {% raw %}
  ```bash
  docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"
  ```
  {% endraw %}

  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 24.** Revisa los **logs** para corroborar que no existan errores:

  > **NOTA:** La salida es extensa; la imagen muestra los logs finales.
  {: .lab-note .info .compact}

  ```bash
  docker logs --tail=50 db
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Levantar la API en backend y conectarla a la BD

Ejecutar la API en `backend-net` con variables de entorno para acceder a Postgres por **nombre** `db`.

#### Tarea 6.1

- **Paso 25.** Levanta el contenedor Docker para que la **API** se comunique con **db**:

  ```bash
  docker run -d --name api \
  --network backend-net \
  -e DB_HOST=db \
  -e DB_PORT=5432 \
  -e DB_USER=postgres \
  -e DB_PASSWORD=postgres \
  -e DB_NAME=agenda \
  api-agenda
  ```

- **Paso 26.** Verifica que los **2 contenedores** estén creados correctamente.

  ```bash
  docker ps
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)  

- **Paso 27.** Valida la salud desde un contenedor temporal en la misma red:

  > **NOTA:** La ruta `/health` debe devolver `{ "status": "ok", "db": "up" }`.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Si no tienes la imagen `curlimages/curl`, Docker la descargará automáticamente.
  {: .lab-note .important .compact}

  ```bash
  docker run --rm --network backend-net curlimages/curl \
  curl -s http://api:3000/health
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)  

- **Paso 28.** Valida que existan los datos iniciales:

  > **NOTA:** La ruta `/usuarios` debe listar al menos **2 registros** iniciales cargados por el script SQL de inicialización (`db/init.sql`).
  {: .lab-note .info .compact}

  ```bash
  docker run --rm --network backend-net curlimages/curl \
  curl -s http://api:3000/usuarios
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Configurar Nginx como reverse proxy (dos redes)

Exponer solo **Nginx** al host. Nginx estará en **frontend-net** y en **backend-net**, y reenviará tráfico a la API.

#### Tarea 7.1

- **Paso 29.** Abre el archivo `proxy/nginx.conf` creado anteriormente y agrega el siguiente código:

  > **NOTA:**
  > - **Bloque `events {}`**: configuración mínima requerida en Nginx.  
  > - **Servidor HTTP** en el puerto **80**.  
  > - **Ruta `/proxy-health`**: responde con `200 ok` en texto plano (para chequeos de salud del proxy).  
  > - **Ruta `/` (raíz)**:  
  >   - Redirige todas las peticiones hacia `http://api:3000`.  
  >   - Reenvía cabecera `Host` original.  
  >   - Añade cabecera `X-Forwarded-For` con la IP del cliente.  
  {: .lab-note .info .compact}

  ```nginx
  events {}
  http {
    server {
      listen 80;
      # Health del proxy
      location /proxy-health {
        return 200 'ok';
        add_header Content-Type text/plain;
      }
      # Rutas hacia la API
      location / {
        proxy_pass http://api:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
    }
  }
  ```

- **Paso 30.** Ejecuta Nginx en la red `frontend-net` y publica el puerto:

  > **NOTA:** Si no tienes la imagen de **Nginx**, Docker la descargará para crear el contenedor.
  {: .lab-note .info .compact}

  ```bash
  MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" \
  docker run -d --name proxy \
  --network frontend-net \
  -p 8080:80 \
  -v "$(pwd)"/proxy/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:1.27-alpine
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 31.** Conecta Nginx a `backend-net` para que resuelva el contenedor `api`:

  ```bash
  docker network connect backend-net proxy
  ```

- **Paso 32.** Reinicia el contenedor **proxy** para que tome efecto la conexión a la red `backend-net`:

  ```bash
  docker restart proxy
  ```

- **Paso 33.** Valida la creación y el estado del contenedor **proxy**:

  {% raw %}
  ```bash
  docker ps --format "table {{.Names}}\t{{.Networks}}\t{{.Ports}}"
  ```
  {% endraw %}

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 34.** Valida la salud del endpoint del proxy:

  > **NOTA:** Debe devolver `ok`.
  {: .lab-note .info .compact}

  ```bash
  curl -s http://localhost:8080/proxy-health
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

- **Paso 35.** Valida la funcionalidad de la API a través de Nginx:

  > **NOTA:** Debe devolver la lista de usuarios.
  {: .lab-note .info .compact}

  ```bash
  curl -s http://localhost:8080/usuarios
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8: Aislamiento y DNS interno (pruebas)

Validar que la BD no es accesible desde el host y que la resolución DNS funciona solo en redes donde conviven los contenedores.

#### Tarea 8.1

- **Paso 36.** Intenta acceder a la BD desde el host **(debe fallar)**:

  ```bash
  ncat -vz localhost 5432 || echo "No accesible desde host (correcto)"
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)  

- **Paso 37.** Prueba la resolución desde `frontend-net` (sin backend):

  > **NOTA:** Se descargará la imagen ya que no existe. Debe **fallar** la resolución de `db` **(no comparten red)**.
  {: .lab-note .info .compact}

  ```bash
  docker run --rm --network frontend-net alpine sh -lc \
  "apk add --no-cache bind-tools >/dev/null && nslookup db || echo 'Sin Resolución'"
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png) 

- **Paso 38.** Prueba la resolución y acceso desde `backend-net`:

  > **NOTA:**
  > - Confirmamos que `api` responde dentro de `backend-net`.  
  > - Y que `db` no resuelve desde `frontend-net`.  
  {: .lab-note .info .compact}

  ```bash
  docker run --rm --network backend-net alpine sh -lc \
  "apk add --no-cache bind-tools curl >/dev/null && nslookup api && curl -s http://api:3000/usuarios | head -n1"
  ```

  ![micint]({{ page.images_base | relative_url }}/18.png) 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9: Limpieza de recursos

Detener y eliminar recursos creados.

#### Tarea 9.1

- **Paso 39.** Detén y elimina los contenedores de práctica:

  > **NOTA:** Libera puertos y recursos; los datos persisten si usaste volumen o bind mount.
  {: .lab-note .info .compact}

  ```bash
  docker rm -f proxy api db 2>/dev/null || true
  ```

- **Paso 40.** Elimina las redes creadas:

  ```bash
  docker network rm backend-net frontend-net 2>/dev/null || true
  ```

- **Paso 41.** Elimina las imágenes creadas temporalmente:

  ```bash
  docker rmi curlimages/curl:latest alpine:latest nginx:1.27-alpine postgres:16-alpine
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}