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
  **Producción** Prefiere **named volumes** o servicios de BD externos (MySQL/PostgreSQL gestionados).
  - **Backups del volumen** (ejemplo rápido)
    ```bash
    mkdir -p backup && \
    docker run --rm -v agenda_data:/data -v "$(pwd)"/backup:/backup alpine \
      sh -c "apk add --no-cache tar && tar czf /backup/agenda_data.tgz -C / data"
    ```
  - **Seguridad** Minimiza paquetes extra en la imagen final; instala herramientas de diagnóstico solo bajo demanda (vía `docker exec`).
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

  ![micint](../Capítulo1/img/1.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint](../Capítulo1/img/2.png)

- **Paso 5.** Asegurate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**:

  ![micint](img/1.png)

- **Paso 6.** Crear el directorio y acceder a él:

  - Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.

  ```bash
  mkdir lab4-dockervolumes && cd lab4-dockervolumes
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  - Trabajar en VS Code permite editar y versionar cómodamente; Git Bash brinda compatibilidad con comandos POSIX.

  ![micint](img/2.png)

- **Paso 8.** Crear la estructura inicial del proyecto de la aplicación:

  - La carpeta `data/` será el destino de la base SQLite en el host; `frontend/` se servirá como contenido estático desde el backend.
  - En el siguiente paso crearas esa estructura con comandos GitBash.

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

  - El comando se ejecuta desde la raíz de la carpeta **lab4-dockervolumes**

  ```bash
  mkdir -p backend && touch backend/package.json backend/server.js
  ```

- **Paso 10.** Muy bien continua la cración del directorio **frontend/** con el archivo index vacio.

  - El comando se ejecuta desde la raíz de la carpeta **lab4-dockervolumes**

  ```bash
  mkdir -p frontend && touch frontend/index.html
  ```

- **Paso 11.** Crea los ultimos directorios y archivos del proyecto **data/**, **.dockerignore** y **Dockerfile**

  - El comando se ejecuta desde la raíz de la carpeta **lab4-dockervolumes**

  ```bash
  mkdir -p data && touch .dockerignore Dockerfile
  ```

- **Paso 12.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias:

  - Evita copiar artefactos innecesarios y la carpeta `data` hacia la imagen, manteniéndola ligera.

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

  - Recuerda que tambien puedes visualizarlos en el explorador de archivos de VSCode.
  - Debes ver `backend/`, `frontend/`, `data/`, `Dockerfile` y `.dockerignore`.

  ```bash
  ls -la
  ```

  ![micint](img/3.png)

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Proyecto base organizado con carpetas para **código** y **datos**.

---

### Tarea 2: Backend Node.js con SQLite y endpoint de salud

Implementar API REST con SQLite, inicialización de esquema y endpoint `/health` para diagnóstico.

#### Tarea 2.1

- **Paso 14.** Abre el archivo `backend/package.json` y agrega el siguiente contenido:

  - Declaramos dependencias mínimas y el script `start` para facilitar ejecuciones locales y en contenedor.

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

  - `DB_DIR` configurable via env (`/app/data` en contenedor) permite montar un volumen para persistencia.

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

- **Paso 16.** Validar localmente si tienes Node.js:

  - El comando se ejecuta desde el directorio **lab4-dockervolumes**
  - Probar localmente reduce el ciclo de depuración antes de contenerizar.

  ```bash
  cd backend && npm install && node server.js
  ```

  ![micint](img/4.png)

- **Paso 17.** Abre otra terminal **GitBash** dentro de VSCode y ejecuta los siguientes comandos pra la prueba local:

  - `curl /health` debe responder `{"status":"ok","db":"up",...}`
  - `curl /api/contactos` debe responder `[]` inicialmente.

  ```bash
  curl -s http://localhost:3000/health
  curl -s http://localhost:3000/api/contactos
  ```

  ![micint](img/5.png)

- **Paso 18.** Regresa a la terminal donde esta ocupando el proceso **node server.js** y detendo con `CTRL + c`

> **¡TAREA FINALIZADA!**

**Resultado esperado:** API funcional con SQLite lista para contenerizarse.

---

### Tarea 3: Frontend con operaciones CRUD básicas

Implementar una interfaz simple con **alta** y **eliminación** de contactos, consumiendo la API.

#### Tarea 3.1

- **Paso 18.** Abre el archivo `frontend/index.html` y agrega el siguiente contenido:

  - UI minimalista que llama a la API del backend servido en el mismo host/puerto.

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

- **Paso 19.** En la terminal dentro del directorio **backend** escribe el comando `node server.js` para inicializar la app localmente.

- **Paso 20.** Abre el navegador **Google Chrome** y en la barra de direcciones escribe:

  - `http://localhost:3000`

- **Paso 21.** Debera abrir la pagina web donde podras **registrar un contacto** y **eliminar el contacto**.

  - Prueba registrar cualquier contacto que gustes.

  ![micint](img/6.png) 

- **Paso 22.** Rompe el proceso del **server local**, escribe `CTRL + c`

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Interfaz funcional con altas y bajas contra la API.

---

### Tarea 4: Dockerfile con HEALTHCHECK

Construir la imagen, exponiendo puertos y agregando **HEALTHCHECK** para diagnósticos.

#### Tarea 4.1

- **Paso 22.** Abre el archivo `Dockerfile` y agrega el contenido para compilar la imagen docker:

  - `HEALTHCHECK` reporta estado `healthy/unhealthy` útil para automatización y orquestadores.

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

- **Paso 23.** Construir imagen, escribe el siguiente comando en la terminal:

  - El comando se ejecuta desde la carpeta **backend**.
  - Se etiqueta la imagen para identificar la variante con volúmenes.
  - Si es necesario ajusta las rutas en tu terminal.
  - La compilación puede tardar maximo hasta **200 segundos** puede terminar antes.

  ```bash
  cd ..
  docker build --no-cache -t agenda-persistente .
  ```

  ![micint](img/7.png) 

- **Paso 24.** Verificar que la imagen se haya creado correctamente, escribe el siguiente comando:

  - La imagen `agenda-persistente` debe aparecer listada con su SIZE.

  ```bash
  docker images | grep agenda-persistente
  ```

  ![micint](img/8.png) 

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Imagen disponible y lista para ejecutar.

---

### Tarea 5: Persistencia con **named volume**

Crear un **volumen** administrado por Docker y ejecutar la app montándolo en `/app/data` para que la BD persista.

#### Tarea 5.1

- **Paso 25.** Crea el siguiente volumen de datos persistente, escribe los siguientes comandos:

  - Los **named volumes** son gestionados por Docker y no dependen de rutas del host.

  ```bash
  docker volume create agenda_data
  docker volume ls | grep agenda_data
  ```

  ![micint](img/9.png) 

- **Paso 26.** Ejecutar contenedor usando el volumen:

  - Montas el volumen en la ruta que el backend espera para guardar la BD.

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

- **Paso 27.** Verifica que el contenedor este corriendo correctamente, escribe el siguiente comando.

  ```bash
  docker ps
  ```

  ![micint](img/10.png) 


- **Paso 28.** Verifica los logs del contenedor, escribe el siguiente comando.

  ```bash
  docker logs -f agenda
  ```

  ![micint](img/11.png)

- **Paso 29.** Rompe el proceso de los logs con `CTRL + c`.

- **Paso 30.** Verifica la salud de la aplicación agenda, escribe el siguiente comando.

  - Verificas la conectividad de la app/BD

  ```bash
  curl -s http://localhost:3000/health
  ```

  ![micint](img/12.png)

- **Paso 31.** Valida el endpoints de la api de contactos que funcione correctamente, escribe el siguiente comando:

  - Respuesta inicial (lista vacía).

  ```bash
  curl -s http://localhost:3000/api/contactos
  ```

  ![micint](img/13.png)

- **Paso 32.** Insertar contactos desde la interfaz grafica en el navegador **Google Chrome** abre el siguiente endpoint.

  - `http://localhost:3000`

  ![micint](img/14.png)

- **Paso 33.** Regresa a la terminal de VSCode y prueba registra otro contacto desde la CLI, escribe el siguiente comando

  ```bash
  curl -s -X POST http://localhost:3000/api/contactos \
    -H "Content-Type: application/json" \
    -d '{"nombre":"Ana","telefono":"555-111-2222"}'
  ```

  ![micint](img/15.png)

- **Paso 34.** Elimina el contenedor activo y recrealo de nuevo, ejecuta los siguientes comandos:

  - Los datos siguen en el volumen aunque el contenedor fue recreado.

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

- **Paso 35.** Puedes verificar con las siguientes opciones:

  - **Opción 1 CLI**
  
  ```bash
  curl -s http://localhost:3000/api/contactos
  ```

  - **Opción 2 Web**

  ```bash
  http://localhost:3000
  ```

  ![micint](img/16.png)
  ![micint](img/17.png)

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Los Datos persisten usando `agenda_data` aunque se remueva el contenedor.

---

### Tarea 6: Flujo de desarrollo con **bind mounts**

Montar carpetas del host (código y datos) para ver cambios sin reconstruir la imagen. Ideal para desarrollo en VS Code.

#### Tarea 6.1

- **Paso 36.** Primero eliminamos el contenedor actual :

  ```bash
  docker rm -f agenda 2>/dev/null || true
  ```

- **Paso 37.** Ejecuta con bind mounts en la terminal de **GitBash** la montura de los directorios:

  - `$(pwd)` monta directorios del proyecto dentro del contenedor para edición en caliente.

  ```bash
  MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" \
  docker run -d --name agenda \
    -p 3001:3000 \
    -e DB_DIR=/app/data \
    -v "$(pwd)"/frontend:/app/frontend \
    -v "$(pwd)"/data:/app/data \
    agenda-persistente
  ```

- **Paso 38.** Edita el archivo `frontend/index.html`, cambia el valor de la etiqueta `<h1>`.

  - Puedes agregar un nombre al final o cambiar el titulo completamente.
  - Guarda el archivo

  ![micint](img/18.png)

- **Paso 39.** Abre la pagina en **Google Chrome** ver el cambio reflejado.

  - El frontend se sirve estático; al refrescar verás los cambios sin rebuild.

  ```bash
  http://localhost:3001
  ```

  ![micint](img/19.png)

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Flujo de desarrollo ágil con cambios inmediatos y datos persistentes en el host.

---

### Tarea 8: Limpieza y buenas prácticas

Detener y eliminar recursos creados; entender implicaciones de limpieza en volúmenes.

#### Tarea 8.1

- **Paso 40.** Detener y eliminar contenedores de práctica:

  - Libera puertos/recursos; los datos persisten si usaste volumen o bind mount.

  ```bash
  docker rm -f agenda 2>/dev/null || true
  ```

- **Paso 41.** Eliminar el volumen creado:

  ```bash
  docker volume rm agenda_data
  ```

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Entorno limpio y listo para la siguiente práctica.