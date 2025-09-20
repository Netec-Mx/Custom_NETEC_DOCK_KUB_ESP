---
layout: lab
title: "Práctica 6: Orquestación de servicios con Docker Compose"
permalink: /capitulo6/lab6/
images_base: /labs/capitulo6/img
duration: "60 minutos"
objective:
  - Aprender a **orquestar múltiples servicios** con **Docker Compose**, definiendo redes, volúmenes, dependencias y health checks en un único archivo declarativo para levantar, validar, escalar y apagar un stack completo (Nginx + API Node.js + PostgreSQL) con un solo comando.
prerequisites:
  - Visual Studio Code
  - Docker Desktop en ejecución (incluye Docker Compose v2)
  - Terminal Git Bash en VS Code
  - Conocimientos básicos de Node.js, SQL y Docker
introduction: |
  **Docker Compose** permite describir una aplicación multi-contenedor en un archivo `docker-compose.yml`. Allí se definen **servicios**, **redes**, **volúmenes**, **variables de entorno**, **healthchecks** y **dependencias**. En esta práctica migraremos la arquitectura del laboratorio anterior (Nginx + API + PostgreSQL) a Compose, agregaremos validaciones automáticas y probaremos **escalado** de la API.
slug: lab6
lab_number: 6
final_result: >
  Una aplicación multi-servicio orquestada con **Docker Compose**: **Nginx** expone el puerto 8080, **API** consulta **PostgreSQL** en una red privada, con **volumen persistente** y **dependencias**. Se validó orquestación y escalado.
notes: 
  - Para producción usar TLS y secretos en lugar de `.env` con contraseñas planas.
  - Usa `docker compose logs -f` para depurar servicios.
  - El escalado en Compose es limitado; en producción se usan orquestadores como Swarm o Kubernetes.
references:
  - text: Compose Spec
    url: https://compose-spec.io/
  - text: Docker Compose (docs)
    url: https://docs.docker.com/compose/
  - text: PostgreSQL imagen oficial
    url: https://hub.docker.com/_/postgres
  - text: Nginx reverse proxy
    url: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
prev: /capitulo5/lab5          
next: /capitulo7/lab7/
---


---

### Tarea 1: Crear la estructura del proyecto

Definir una estructura clara con carpetas para código, configuración del proxy e inicialización de la base de datos.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`** lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VSCode** da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 5.** Asegurate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**:

  > **NOTA:** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/3.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica 6**:

  > **NOTA:** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab6-dockercompose && cd lab6-dockercompose
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 8.** Crearás la siguiente estructura inicial del proyecto de la aplicación:

  > **NOTA:**  
  - `db/init.sql` inicializa el esquema al arrancar PostgreSQL.  
  - `proxy/nginx.conf` define el enrutamiento hacia la API.  
  - `Dockerfile.api` contiene la construcción de la API. 
  - `docker-compose.yml` orquesta todo.
  - `.env` centraliza variables.
  {: .lab-note .info .compact}

  ```text
  lab6-dockercompose/
  ├── api/
  │   ├── package.json
  │   └── server.js
  ├── db/
  │   └── init.sql
  ├── proxy/
  │   └── nginx.conf
  ├── Dockerfile.api
  ├── docker-compose.yml
  └── .env
  ```

- **Paso 9.** Ahora crea las carpetas **api/**, **db/**, **proxy/** y los archivos correspondientes.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab6-dockercompose**. Puedes ejecutar todos los comandos juntos.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p api && touch api/package.json api/server.js
  mkdir -p db && touch db/init.sql
  mkdir -p proxy && touch proxy/nginx.conf
  touch Dockerfile.api docker-compose.yml .env
  ```

- **Paso 10.** Valida la creación de la estructura de tu proyecto; escribe el siguiente comando:

  > **NOTA:** También puedes validarlo en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Implementar la API Node.js (Express)

Crear una API con endpoints `/health` y `/usuarios` (GET/POST) que se conecta a PostgreSQL mediante variables de entorno.

#### Tarea 2.1

- **Paso 11.** Abre el archivo `api/package.json` y agrega el siguiente codigo:

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

- **Paso 12.** Abre el archivo `api/server.js` y agrega la logica del backend para la aplicación:

  > **NOTA:**
  - **Servidor Express** en puerto configurable (`PORT` o 3000 por defecto).  
  - **Conexión a PostgreSQL** mediante `pg.Pool`, con parámetros leídos de variables de entorno.  
  - **Middleware JSON** para procesar cuerpos de peticiones.  
  - **Endpoint `/health`**: ejecuta `SELECT 1` para verificar la conexión a la base de datos.  
  - **Endpoint GET `/usuarios`**: obtiene todos los usuarios (`id`, `nombre`, `email`) ordenados por ID descendente.  
  - **Endpoint POST `/usuarios`**: valida campos requeridos y crea un usuario nuevo, devolviendo el registro insertado.  
  {: .lab-note .info .compact}

  ```javascript
  import express from "express";
  import pkg from "pg";
  const { Pool } = pkg;

  const app = express();
  const PORT = process.env.PORT || 3000;

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
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Configuración de PostgreSQL e inicialización

Definir el script SQL que crea la BD `agenda`, una tabla y carga datos de ejemplo.

#### Tarea 3.1

- **Paso 13.** Abre el archivo `db/init.sql` y define la siguiente estrucutra de SQL para la BD:

  > **NOTA:**
  - **CREATE DATABASE agenda**: crea la base de datos llamada `agenda`.  
  - **\c agenda**: cambia la conexión actual a la base de datos `agenda`.  
  - **CREATE TABLE usuarios**: define la tabla con:  
  - `id` → entero autoincremental (PRIMARY KEY).  
  - `nombre` → texto obligatorio.  
  - `email` → texto obligatorio y único.  
  - **INSERT INTO usuarios**: agrega registros iniciales con nombre y email.  
  - **ON CONFLICT DO NOTHING**: evita error si los registros ya existen.  
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
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Nginx como reverse proxy

Nginx recibirá tráfico externo y lo reenviará a la API por su nombre de servicio interno (`api`).

#### Tarea 4.1

- **Paso 14.** Abre el archivo `proxy/nginx.conf` y da de alta el servicio nginx para recibir las solicitudes:

  > **NOTA:**
  - **Bloque `events {}`**: configuración mínima obligatoria en Nginx.  
  - **Servidor HTTP** escuchando en el puerto **80**.  
  - **Ruta `/health-proxy`**: responde `200 ok` en texto plano (para chequeos de salud).  
  - **Ruta `/`**:  
    - Redirige todas las peticiones hacia `http://api:3000/`.  
    - Reenvía la cabecera original `Host`.  
    - Añade cabecera `X-Forwarded-For` con la IP del cliente.  
  {: .lab-note .info .compact}

  ```nginx
  events {}
  http {
    server {
      listen 80;

      location /health-proxy {
        return 200 'ok';
        add_header Content-Type text/plain;
      }

      location / {
        proxy_pass http://api:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
    }
  }
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Dockerfile de la API con healthcheck

Crear imagen mínima de la API con verificación de salud.

#### Tarea 5.1

- **Paso 15.** Ahora abre el `Dockerfile.api` y define las configuraciónes para compilar la imagen:

  ```dockerfile
  FROM node:20-alpine

  WORKDIR /app
  COPY api/package*.json ./
  RUN npm install --production
  COPY api ./

  ENV PORT=3000
  EXPOSE 3000

  RUN apk add --no-cache curl
  HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -fsS http://localhost:3000/health || exit 1

  CMD ["node", "server.js"]
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Definir `docker-compose.yml` y `.env`

Declarar servicios, redes, volúmenes, dependencias y variables en Compose.

#### Tarea 6.1

- **Paso 16.** Dentro del archivo `.env` que se eneucuentra en la raíz del directorio **lab6...** agrega las siguientes variables:

  ```env
  API_IMAGE=api-agenda:1.0
  API_PORT=8080

  POSTGRES_USER=postgres
  POSTGRES_PASSWORD=postgres
  POSTGRES_DB=postgres

  DB_HOST=db
  DB_PORT=5432
  DB_NAME=agenda
  ```

- **Paso 17.** Ahora si dentro del archivo `docker-compose.yml` define la siguiente configuración para orquestar toda la aplicación:

  > **NOTA:**
  - **db (Postgres 16)**: base de datos con persistencia (`db_data`) e init script (`init.sql`).  
  - **api (Node/Express)**: construida con `Dockerfile.api`, conecta a `db` usando variables de entorno.  
  - **proxy (Nginx)**: expone `${API_PORT}`, usa `nginx.conf`, conecta con `api`.  
  - **Redes**: `backend-net` (interna) y `frontend-net` (externa).  
  - **Volumen**: `db_data` para datos de PostgreSQL.  
  {: .lab-note .info .compact}

  ```yaml
  services:
    db:
      image: postgres:16-alpine
      environment:
        POSTGRES_USER: ${POSTGRES_USER}
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
        POSTGRES_DB: ${POSTGRES_DB}
      volumes:
        - db_data:/var/lib/postgresql/data
        - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      networks:
        - backend-net

    api:
      build:
        context: .
        dockerfile: Dockerfile.api
      image: ${API_IMAGE}
      environment:
        DB_HOST: ${DB_HOST}
        DB_PORT: ${DB_PORT}
        DB_USER: ${POSTGRES_USER}
        DB_PASSWORD: ${POSTGRES_PASSWORD}
        DB_NAME: ${DB_NAME}
      depends_on:
        - db
      networks:
        - backend-net

    proxy:
      image: nginx:1.27-alpine
      ports:
        - "${API_PORT}:80"
      volumes:
        - ./proxy/nginx.conf:/etc/nginx/nginx.conf:ro
      depends_on:
        - api
      networks:
        - frontend-net
        - backend-net

  networks:
    frontend-net:
      driver: bridge
    backend-net:
      driver: bridge

  volumes:
    db_data:
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Levantar y validar el stack

Ejecutar el stack con Compose y valida los endpoints.

#### Tarea 7.1

- **Paso 18.** Levantar el compose en modo **"detached"**, ejecuta el siguiente comando:

  > **NOTA:** Ejecutalo dentro del directorio **lab6-dockercompose**
  {: .lab-note .info .compact}

  ```bash
  docker compose up -d --build
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 19.** Al finalizar observaras el resultado de todo lo que el **Docker Compose** creo.

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 20.** Verifica que los contenedores esten creados correctamente, escribe el siguiente comando.

  ```bash
  docker compose ps
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 21.** Si todo salio bien, primero valida la salud del endpoint:

  ```bash
  API_PORT=8080
  curl -s http://localhost:${API_PORT}/health-proxy
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 22.** Muy bien, ahora valida que la aplicación este funcionando bien y devuelva los usuarios,:

  ```bash
  curl -s http://localhost:${API_PORT}/usuarios
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 23.** Puedes usar este ejemplo para inyectar 1 o mas usuarios.

  ```bash
  curl -s -X POST http://localhost:8080/usuarios \
    -H "Content-Type: application/json" \
    -d '{"nombre":"Ana","email":"ana@example.com"}'
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 24.** Vuelve a validar para saber si inyecto correctamente el usuario nuevo.

  ```bash
  curl -s http://localhost:${API_PORT}/usuarios
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

- **Paso 25.** Usa la URL para validar la información desde el navegador web (sin interfaz grafica)

  ```bash
  http://localhost:8080/usuarios
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8: Escalado mediante compose

En esta tarea probaras la escalabilidad que ofrece **Docker Compose**.

#### Tarea 8.1

- **Paso 26.** Con Docker compose puedes escalar la API a 2 réplicas muy facilmente, escribe los siguientes comandos:

  ```bash
  docker compose up -d --scale api=2
  ```

  ```bash
  docker compose ps
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9: Limpieza de recursos

Detener y eliminar recursos creados.

#### Tarea 9.1

- **Paso 27.** Asi tambien es facil detener todo recurso y realizar la limpieza:

  > **NOTA:** Docker compose se hace cargo de la orquestación de todos los componentes.
  {: .lab-note .info .compact}

  ```bash
  docker compose down -v
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 28.** Elimina las imágenes creadas temporalmente:

  ```bash
  docker rmi nginx:1.27-alpine postgres:16-alpine
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}