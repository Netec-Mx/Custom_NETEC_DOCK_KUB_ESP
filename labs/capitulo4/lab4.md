---
layout: lab
title: "Práctica 4: Persistencia de datos con volúmenes y bind mounts"
permalink: /capitulo4/lab4/
images_base: /labs/capitulo4/img
duration: "60 minutos"
objective:
  - Aprender a **persistir datos** en contenedores Docker usando **volúmenes** (named volumes) y **bind mounts**. Extenderás la **Agenda de contactos con interfaz** para almacenar la información en **SQLite**, validando que los datos sobreviven a reinicios y recreaciones de contenedores, y habilitando un flujo de desarrollo con **bind mounts** desde Visual Studio Code y Git Bash.
prerequisites:
  - Visual Studio Code instalado
  - Docker Desktop instalado y en ejecución
  - Git Bash configurado como terminal por defecto en VS Code
  - Conocimientos básicos de Node.js, HTML y Docker
introduction:
  - Por defecto, los contenedores Docker son **efímeros**, si se eliminan, sus datos se pierden. Para separar el ciclo de vida **de la app** y **de los datos**, Docker ofrece **volúmenes** (administrados por Docker) y **bind mounts** (montajes de carpetas del host). En esta práctica integrarás **SQLite** y probarás ambos enfoques, **named volume** para persistencia y **bind mount** para desarrollo con cambios en caliente.
slug: lab4
lab_number: 4
final_result: >
  Una agenda de contactos **persistente**: los datos permanecen aunque se eliminen y recrean contenedores (usando **volúmenes**). Además, un flujo de **desarrollo ágil** con **bind mounts** para editar código desde VS Code sin reconstruir imágenes.
notes: |
  En ambientes **Productivos** se prefiere **named volumes** o servicios de BD externos (MySQL/PostgreSQL gestionados).
  - El siguiente comando es un ejemplo de **Respaldo de los volumenes**
    ```bash
    mkdir -p backup && \
    docker run --rm -v agenda_data:/data -v "$(pwd)"/backup:/backup alpine \
      sh -c "apk add --no-cache tar && tar czf /backup/agenda_data.tgz -C / data"
    ```
  - La **Seguridad** minimiza paquetes extra en la imagen final, instala herramientas de diagnóstico solo bajo demanda (vía `docker exec`).
references:
  - text: Documentación oficial de Volúmenes
    url: https://docs.docker.com/storage/volumes/
  - text: Documentación oficial de Bind mounts
    url: https://docs.docker.com/storage/bind-mounts/
  - text: Dockerfile HEALTHCHECK
    url: https://docs.docker.com/engine/reference/builder/#healthcheck
  - text: SQLite + Node.js
    url: https://www.sqlitetutorial.net/sqlite-nodejs/
prev: /capitulo3/lab3          
next: /capitulo5/lab5/
---


---

### Tarea 1: Preparar la estructura del proyecto

Crear la carpeta base y la estructura mínima del proyecto para mapear correctamente datos y código durante los montajes.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`** lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VSCode** da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 5.** Asegurate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**:

  > **NOTA:** Si te quedaste en el directorio de una práctica usa `cd ..` para retornar a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Ahora crea el directorio para trabajar en la practica 4:

  > **NOTA:** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab4-dockervolumes && cd lab4-dockervolumes
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. Git Bash brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Crearas la siguiente estructura inicial del proyecto de la aplicación:

  > **NOTA:**
  - La carpeta `data/` será el destino de la base **SQLite** en el host; `frontend/` se servirá como contenido estático desde el backend.
  - En el siguiente paso crearas esa estructura con comandos GitBash.
  {: .lab-note .info .compact}

  ```text
  lab4-dockervolumes/
  ├── backend/
  │   ├── package.json
  │   └── server.js
  ├── frontend/
  │   └── index.html
  ├── data/           
  ├── .dockerignore
  └── Dockerfile
  ```

- **Paso 9.** Ahora crea la carpeta **backend/** y sus archivos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab4-dockervolumes**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p backend && touch backend/package.json backend/server.js
  ```

- **Paso 10.** Muy bien continua con la creación del directorio **frontend/** y el archivo index vacio.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab4-dockervolumes**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p frontend && touch frontend/index.html
  ```

- **Paso 11.** Crea el ultimo directorio y archivos del proyecto **data/**, **.dockerignore** y **Dockerfile**

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab4-dockervolumes**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p data && touch .dockerignore Dockerfile
  ```

- **Paso 12.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias:

  > **IMPORTANTE:** Evita copiar artefactos innecesarios y la carpeta `data` hacia la imagen, manteniéndola ligera.
  {: .lab-note .important .compact}

  ```gitignore
  # evita copiar node_modules del host (causa del ELF inválido)
  backend/node_modules
  frontend/node_modules
  **/node_modules

  # basura
  npm-debug.log
  Dockerfile
  docker-compose.yml
  .git
  .gitignore
  .DS_Store
  ```

- **Paso 13.** Valida la creacion de tu estructura de proyecto, escribe el siguiente comando.

  > **NOTA:** Recuerda que tambien puedes visualizarlos en el explorador de archivos de VSCode.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Debes ver `backend/`, `frontend/`, `data/`, `Dockerfile` y `.dockerignore`.
  {: .lab-note .important .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Backend Node.js con SQLite y endpoint de salud

Implementar API REST con SQLite, inicialización de esquema y endpoint `/health` para diagnóstico.

#### Tarea 2.1

- **Paso 14.** Abre el archivo `backend/package.json` y agrega el siguiente contenido:

  > **NOTA:** Declaramos dependencias mínimas y el script `start` para facilitar ejecuciones locales y en contenedor.
  {: .lab-note .info .compact}
  
  ```json
  {
    "name": "agenda-persistente",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": {
      "start": "node server.js"
    },
    "dependencies": {
      "express": "^4.18.2",
      "sqlite3": "^5.1.6"
    }
  }
  ```

- **Paso 15.** Abre el archivo `backend/server.js` y agrega el siguiente contenido:

  > **NOTA:** `DB_DIR` configurable via env (`/app/data` en contenedor) permite montar un volumen para persistencia.
  - **Express + SQLite3** para gestionar contactos con persistencia.  
  - **Tabla `contactos`** creada automáticamente si no existe.  
  - **Endpoints principales**:  
    - `/health` → verifica estado de la app y la base de datos.  
    - `/api/contactos` → listar y crear contactos.  
    - `/api/contactos/:id` → eliminar por ID.  
  - **Archivos estáticos** servidos desde `frontend`.  
  - **Base de datos persistente** almacenada en `data/contactos.db`.  
  {: .lab-note .info .compact}

  ```javascript
  const express = require('express');
  const sqlite3 = require('sqlite3').verbose();
  const path = require('path');
  const fs = require('fs');

  const app = express();
  const PORT = process.env.PORT || 3000;
  const DB_DIR = process.env.DB_DIR || path.join(__dirname, '..', 'data');
  const DB_PATH = path.join(DB_DIR, 'contactos.db');

  // Asegurar carpeta de datos
  if (!fs.existsSync(DB_DIR)) fs.mkdirSync(DB_DIR, { recursive: true });

  app.use(express.json());
  app.use(express.static(path.join(__dirname, '..', 'frontend')));

  // Inicializar DB
  const db = new sqlite3.Database(DB_PATH);
  db.serialize(() => {
    db.run(`CREATE TABLE IF NOT EXISTS contactos (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      nombre TEXT NOT NULL,
      telefono TEXT NOT NULL
    )`);
  });

  // Salud de la app y BD
  app.get('/health', (req, res) => {
    db.get('SELECT 1 as ok', [], (err, row) => {
      if (err) return res.status(500).json({ status: 'error', error: err.message });
      res.json({ status: 'ok', db: 'up', path: DB_PATH });
    });
  });

  // REMOVER SOLO PARA PROBAR
  app.get('/debug', (req, res) => {
    res.json({
      platform: process.platform,    // debe ser "linux" dentro del contenedor
      cwd: process.cwd(),
      pid: process.pid,
      DB_DIR: process.env.DB_DIR,
      DB_PATH
    });
  });

  // API Contactos
  app.get('/api/contactos', (req, res) => {
    db.all('SELECT * FROM contactos ORDER BY id DESC', [], (err, rows) => {
      if (err) return res.status(500).json({ error: err.message });
      res.json(rows);
    });
  });

  app.post('/api/contactos', (req, res) => {
    const { nombre, telefono } = req.body || {};
    if (!nombre || !telefono) return res.status(400).json({ error: 'nombre y telefono son requeridos' });
    db.run('INSERT INTO contactos (nombre, telefono) VALUES (?, ?)', [nombre, telefono], function (err) {
      if (err) return res.status(500).json({ error: err.message });
      res.status(201).json({ id: this.lastID, nombre, telefono });
    });
  });

  app.delete('/api/contactos/:id', (req, res) => {
    db.run('DELETE FROM contactos WHERE id = ?', [req.params.id], function (err) {
      if (err) return res.status(500).json({ error: err.message });
      res.json({ deleted: this.changes });
    });
  });

  app.listen(PORT, () => {
    console.log(`Servidor en http://localhost:${PORT} (DB: ${DB_PATH})`);
  });
  ```

- **Paso 16.** Valida localmente la aplicación:

  > **NOTA:**
    - Probar localmente reduce el ciclo de depuración antes de contenerizar.
    - El comando se ejecuta desde el directorio **lab4-dockervolumes**
  {: .lab-note .info .compact}

  ```bash
  cd backend && npm install && node server.js
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 17.** Abre otra terminal **GitBash** dentro de VSCode y ejecuta los siguientes comandos para la prueba local:

  > **NOTA:**
    - `curl /health` debe responder `{"status":"ok","db":"up",...}`
    - `curl /api/contactos` debe responder `[]` inicialmente.
  {: .lab-note .info .compact}

  ```bash
  curl -s http://localhost:3000/health
  ```

  ```bash
  curl -s http://localhost:3000/api/contactos
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

- **Paso 18.** Regresa a la terminal donde esta ocupando el proceso **node server.js** y detenlo con `CTRL + c`

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Frontend con operaciones CRUD básicas

Implementar una interfaz simple con **alta** y **eliminación** de contactos, consumiendo la API.

#### Tarea 3.1

- **Paso 19.** Abre el archivo `frontend/index.html` y agrega el siguiente contenido:

  > **NOTA:** UI minimalista que llama a la API del backend servido en el mismo host/puerto.
  - **UI mínima** con estilos inline y layout centrado.
  - **Inputs + botón** para capturar `nombre` y `telefono`.
  - **Tabla** que renderiza contactos obtenidos del backend.
  - **Carga inicial**: `cargar()` hace `GET /api/contactos` y dibuja filas.
  - **Alta**: botón **Agregar** envía `POST /api/contactos` y recarga la lista.
  - **Borrado**: botón **Eliminar** por fila hace `DELETE /api/contactos/:id`.
  - **Sin framework**: todo con `fetch` y `innerHTML` en vanilla JS.
  - **Contexto**: pensado para backend con persistencia (SQLite/volúmenes).
  {: .lab-note .info .compact}

  ```html
  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Agenda Persistente</title>
    <style>
      body { font-family: system-ui, sans-serif; max-width: 720px; margin: 2rem auto; }
      table { width: 100%; border-collapse: collapse; margin-top: 1rem; }
      th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
      th { background: #f6f8fa; }
      .row { display: flex; gap: .5rem; }
      input { flex: 1; padding: .5rem; }
      button { padding: .5rem .75rem; cursor: pointer; }
    </style>
  </head>
  <body>
    <h1>Agenda Persistente (SQLite + Volúmenes)</h1>

    <div class="row">
      <input type="text" id="nombre" placeholder="Nombre" required />
      <input type="text" id="telefono" placeholder="Teléfono" required />
      <button id="agregar">Agregar</button>
    </div>

    <table>
      <thead><tr><th>ID</th><th>Nombre</th><th>Teléfono</th><th>Acciones</th></tr></thead>
      <tbody id="tbody"></tbody>
    </table>

    <script>
      const tbody = document.getElementById('tbody');
      const nombre = document.getElementById('nombre');
      const telefono = document.getElementById('telefono');
      const agregar = document.getElementById('agregar');

      async function cargar() {
        const res = await fetch('/api/contactos');
        const data = await res.json();
        tbody.innerHTML = data.map(c => `
          <tr>
            <td>${c.id}</td>
            <td>${c.nombre}</td>
            <td>${c.telefono}</td>
            <td><button onclick="eliminar(${c.id})">Eliminar</button></td>
          </tr>
        `).join('');
      }

      async function eliminar(id) {
        await fetch('/api/contactos/' + id, { method: 'DELETE' });
        await cargar();
      }

      agregar.addEventListener('click', async () => {
        if (!nombre.value || !telefono.value) return alert('Completa ambos campos');
        await fetch('/api/contactos', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ nombre: nombre.value, telefono: telefono.value })
        });
        nombre.value = ''; telefono.value='';
        await cargar();
      });

      cargar();
    </script>
  </body>
  </html>
  ```

- **Paso 20.** En la terminal dentro del directorio **backend** escribe el siguiente comando para inicializar la app localmente.

  ```bash
  node server.js
  ```

- **Paso 21.** Abre el navegador **Google Chrome** y en la barra de direcciones escribe:

  ```bash
  http://localhost:3000
  ```

- **Paso 22.** Debera abrir la pagina web donde podras **registrar** y **eliminar contactos**.

  > **NOTA:** Prueba registrar cualquier contacto que gustes.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/6.png) 

- **Paso 23.** Cuando hayas terminado de interactuar con la aplicación, rompe el proceso del **server local**, escribe `CTRL + c`

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Dockerfile con HEALTHCHECK

Construir la imagen, exponiendo puertos y agregando **HEALTHCHECK** para diagnósticos.

#### Tarea 4.1

- **Paso 24.** Abre el archivo `Dockerfile` y agrega el contenido para compilar la imagen docker:

  > **NOTA:** `HEALTHCHECK` reporta estado `healthy/unhealthy` útil para automatización y orquestadores.
  {: .lab-note .info .compact}

  ```dockerfile
  FROM node:20-bullseye-slim

  ENV NODE_ENV=production
  ENV PORT=3000
  ENV DB_DIR=/app/data

  WORKDIR /app

  # curl para healthcheck
  RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

  # Instalar deps del backend dentro de la imagen (binarios correctos de Linux)
  COPY backend/package*.json ./backend/
  RUN cd backend && npm ci --omit=dev

  # Copiar el resto del código (SIN node_modules gracias a .dockerignore)
  COPY backend ./backend
  COPY frontend ./frontend

  # Datos persistentes
  RUN mkdir -p ${DB_DIR}
  VOLUME ["/app/data"]

  EXPOSE 3000

  # Healthcheck (asegúrate de tener GET /health)
  HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -fsS http://localhost:${PORT}/health || exit 1

  CMD ["node", "backend/server.js"]

  ```

- **Paso 25.** Construye la imagen, escribe el siguiente comando en la terminal:

  > **IMPORTANTE:**
  - El comando se ejecuta desde la carpeta **backend**, para regresar un nivel al directorio **raíz**
  - Si es necesario ajusta las rutas en tu terminal.
  - La compilación puede tardar maximo hasta **200 segundos** pero puede terminar antes.
  {: .lab-note .important .compact}

  ```bash
  cd ..
  docker build --no-cache -t agenda-persistente .
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png) 

- **Paso 26.** Verificar que la imagen se haya creado correctamente, escribe el siguiente comando:

  > **NOTA:** La imagen `agenda-persistente` debe aparecer listada con su SIZE.
  {: .lab-note .info .compact}

  ```bash
  docker images | grep agenda-persistente
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png) 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Persistencia con **named volume**

Crear un **volumen** administrado por Docker y ejecutar la app montándolo en `/app/data` para que la BD persista.

#### Tarea 5.1

- **Paso 27.** Crea el siguiente volumen de datos persistente, escribe los siguientes comandos:

  > **NOTA:** Los **named volumes** son gestionados por Docker y no dependen de rutas del host.
  {: .lab-note .info .compact}

  ```bash
  docker volume create agenda_data
  ```

  ```bash
  docker volume ls | grep agenda_data
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png) 

- **Paso 28.** Ejecuta el contenedor usando el volumen que se acaba de crear:

  > **NOTA:** Montas el volumen en la ruta que el backend espera para guardar la BD.
  {: .lab-note .info .compact}

  ```bash
  MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" \
  docker run -d \
    --name agenda \
    -p 3000:3000 \
    -e PORT=3000 \
    -e DB_DIR=/app/data \
    -v agenda_data:/app/data \
    agenda-persistente
  ```

- **Paso 29.** Verifica que el contenedor este corriendo correctamente, escribe el siguiente comando.

  ```bash
  docker ps
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png) 


- **Paso 30.** Verifica los logs del contenedor, escribe el siguiente comando.

  ```bash
  docker logs -f agenda
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 31.** Rompe el proceso de los logs con `CTRL + c`.

- **Paso 32.** Verifica la salud de la aplicación agenda, escribe el siguiente comando.

  > **NOTA:** Verificas la conectividad de la app/BD
  {: .lab-note .info .compact}

  ```bash
  curl -s http://localhost:3000/health
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 33.** Valida que el endpoint de la api de contactos funcione correctamente, escribe el siguiente comando:

  > **NOTA:** Respuesta inicial (lista vacía).
  {: .lab-note .info .compact}

  ```bash
  curl -s http://localhost:3000/api/contactos
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 34.** Abre la URL del la aplicación e inserta contactos desde la interfaz grafica en el navegador **Google Chrome**.

  ```bash
  http://localhost:3000
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

- **Paso 35.** Regresa a la terminal de VSCode. Prueba registra un contacto desde la CLI, escribe el siguiente comando.

  > **NOTA:** Si es necesario registra mas contactos para que tanga mas información
  {: .lab-note .info .compact}

  ```bash
  curl -s -X POST http://localhost:3000/api/contactos \
    -H "Content-Type: application/json" \
    -d '{"nombre":"Ana","telefono":"555-111-2222"}'
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 36.** Elimina el contenedor activo y recrealo de nuevo, ejecuta los siguientes comandos:

  > **NOTA:** Los datos siguen en el volumen aunque el contenedor fue recreado.
  {: .lab-note .info .compact}

  ```bash
  docker rm -f agenda 2>/dev/null || true
  ```  

  ```bash
  MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" \
  docker run -d \
    --name agenda \
    -p 3000:3000 \
    -e PORT=3000 \
    -e DB_DIR=/app/data \
    -v agenda_data:/app/data \
    agenda-persistente
  ```

- **Paso 37.** Puedes verificar con las siguientes opciones:

  - **Opción 1 CLI**
  
  ```bash
  curl -s http://localhost:3000/api/contactos
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

  - **Opción 2 Web**

  ```bash
  http://localhost:3000
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Flujo de desarrollo con **bind mounts**

Montar carpetas del host (código y datos) para ver cambios sin reconstruir la imagen. Ideal para desarrollo en VS Code.

#### Tarea 6.1

- **Paso 38.** Primero eliminamos el contenedor actual :

  ```bash
  docker rm -f agenda 2>/dev/null || true
  ```

- **Paso 39.** Ejecuta con **bind mounts** en la terminal de **GitBash** la montura de los directorios:

  > **NOTA:** `$(pwd)` monta directorios del proyecto dentro del contenedor para edición en caliente.
  {: .lab-note .info .compact}

  ```bash
  MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" \
  docker run -d --name agenda \
    -p 3001:3000 \
    -e DB_DIR=/app/data \
    -v "$(pwd)"/frontend:/app/frontend \
    -v "$(pwd)"/data:/app/data \
    agenda-persistente
  ```

- **Paso 40.** Edita el archivo `frontend/index.html`, cambia el valor de la etiqueta `<h1>`.

  > **IMPORTANTE:** Puedes agregar un nombre al final o cambiar el titulo completamente.
  {: .lab-note .important .compact}

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 41.** Abre la pagina en **Google Chrome** ver el cambio reflejado.

  > **NOTA:** El frontend se sirve estático, al actualizar verás los cambios sin rebuild.
  {: .lab-note .info .compact}

  ```bash
  http://localhost:3001
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza y buenas prácticas

Detener y eliminar recursos creados; entender implicaciones de limpieza en volúmenes.

#### Tarea 7.1

- **Paso 42.** Detener y eliminar contenedores de práctica:

  > **NOTA:** Libera puertos/recursos; los datos persisten si usaste volumen o bind mount.
  {: .lab-note .info .compact}

  ```bash
  docker rm -f agenda 2>/dev/null || true
  ```

- **Paso 43.** Eliminar el volumen creado:

  ```bash
  docker volume rm agenda_data
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}