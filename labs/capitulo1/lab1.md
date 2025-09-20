---
layout: lab
title: "Práctica 1: Despliegue de una aplicación Node.js simple con Docker"
permalink: /capitulo1/lab1/
images_base: /labs/capitulo1/img
duration: "60 minutos"
objective:
  - El objetivo de esta práctica es que el participante aprenda a crear, ejecutar y contenerizar una aplicación simple en **Node.js con Express** que actúe como una agenda de contactos, junto con un frontend HTML básico. Al finalizar, el estudiante comprenderá cómo usar Docker para construir imágenes, ejecutar contenedores y validar la aplicación de forma local.
prerequisites:
  - Visual Studio Code instalado
  - Extensión de **Docker** instalada en VS Code (opcional pero recomendable)
  - Git Bash configurado como terminal por defecto en VS Code
  - Docker Desktop instalado y en ejecución
  - Node.js (versión LTS recomendada. 20.x o superior)
  - Navegador web (Chrome, Edge o Firefox)
introduction:
  - En esta práctica construiremos una aplicación **Agenda de contactos**, únicamente memoria. El backend estará hecho con **Node.js + Express**, mientras que el frontend será una página HTML con un formulario y una tabla para mostrar los contactos. Finalmente, aprenderemos a contenerizarla con Docker y ejecutarla de manera local.
slug: lab1
lab_number: 1
final_result: >
  Al finalizar la práctica, habrás desplegado exitosamente una aplicación **Node.js + Express + HTML** dentro de un contenedor Docker, comprendiendo la relación entre backend, frontend y la contenerización de aplicaciones.
notes:
  - Esta práctica no usa base de datos; los contactos se pierden al reiniciar el servidor.
  - El frontend se sirve desde el backend (mismo origen → sin CORS).
  - Puedes extender la práctica para persistir datos en archivos o una base de datos.
references:
  - text: Documentación oficial Docker
    url: https://docs.docker.com/
  - text: Express.js - Getting Started
    url: https://expressjs.com/es/starter/installing.html
  - text: Node.js
    url: https://nodejs.org/es
  - text: Guía de imágenes oficiales Node.js en Docker Hub
    url: https://hub.docker.com/_/node
prev: /capitulo16/lab16          
next: /capitulo2/lab2/
---


---

### Tarea 1: Preparar el entorno de trabajo

Configurar la estructura de carpetas y archivos del proyecto para mantener una buena organización desde el inicio.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu equipo con un usuario que tenga permisos administrativos.  

- **Paso 2.** Abre **Visual Studio Code (VS Code)**. Puedes hacerlo desde el **Escritorio** o buscándolo en las aplicaciones de Windows.

- **Paso 3.** En **VS Code**, abre la **Terminal** (icono en la parte superior derecha o con el menú **View → Terminal**).  
  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 4.** Selecciona la terminal **Git Bash** como shell activo.  
  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 5.** Verifica que **Docker** esté instalado.
  
  > **NOTA:** Copia y pega el siguiente comando en la terminal. **La versión puede variar.**
  {: .lab-note .info .compact}
  
  ```bash
  docker --version
  ```
  ![micint]({{ page.images_base | relative_url }}/3.png)

- **Paso 6.** Verifica **Node.js** y **npm**:

  ```bash
  node -v
  ```
  ![micint]({{ page.images_base | relative_url }}/5.png)

  ```bash
  npm -v
  ```
  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 7.** Crea la estructura base del directorio del curso en el **Escritorio** del equipo asignado:

  > **IMPORTANTE:** Si es necesario, ajusta manualmente las rutas en la terminal para crear la estructura de directorios.
  {: .lab-note .important .compact}

  - Entra al directorio **Desktop**.
  - Crea un directorio llamado `dockerlabs`.
  - Crea el subdirectorio `lab1-acontactos`.
  - Dentro, crea los directorios `frontend` y `backend`.
  - Puedes ejecutar lo siguiente para crearlo todo:

  ```bash
  cd Desktop/
  mkdir dockerlabs
  mkdir dockerlabs/lab1-acontactos && cd dockerlabs/lab1-acontactos
  mkdir frontend backend
  ```

- **Paso 8.** Verifica la estructura con el siguiente comando:

  ```bash
  ls -la -R
  ```
  ![micint]({{ page.images_base | relative_url }}/7.png)

#### Tarea 1.2

- **Paso 9.** Abre el directorio del proyecto en **VS Code**.

  > **NOTA:** Da clic en el icono como se muestra en la imagen.
  {: .lab-note .info .compact}
  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 10.** Clic en **Open Folder**.  
  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 11.** Navega al directorio **dockerlabs** (en el Escritorio) y da clic en **Select Folder**.

  > **NOTA:** Si aparece la ventana emergente, selecciona **Yes, I trust the authors**.
  {: .lab-note .info .compact}
  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 12.** Verás cargado tu directorio para comenzar a trabajar.  
  ![micint]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Crear el backend en Node.js

Implementar un servidor **Express** que maneje rutas para agregar, listar, actualizar y borrar contactos, **y que sirva el frontend**.

#### Tarea 2.1

- **Paso 13.** Asegúrate de estar dentro de la carpeta **backend**.

  > **NOTA:** Abre la **Terminal** desde la esquina superior derecha y navega al directorio desde la terminal.
  {: .lab-note .info .compact}

  ```bash
  cd lab1-acontactos/backend
  ```
  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 14.** Inicializa un proyecto de **Node.js** en la carpeta `backend` e instala **Express**:

  > **NOTA:** Verifica que el directorio actual sea **backend**.
  {: .lab-note .info .compact} 

  ```bash
  npm init -y
  npm install express
  ```
  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 15.** Crea el archivo `server.js` dentro de **backend**:

  ```bash
  touch server.js
  code server.js
  ```

- **Paso 16.** Agrega el siguiente contenido al archivo **`server.js`**.

  > **NOTA:** Puedes copiar y pegar. **Express** permite crear APIs REST de forma rápida y sencilla, ideal para microservicios ligeros.
  {: .lab-note .info .compact}

  - **Servidor Express** ejecutándose en el puerto **3000**.  
  - **Frontend estático** servido desde la carpeta `../frontend`.  
  - **Array en memoria** que guarda contactos con campos `id`, `nombre`, `telefono`.  
  - **GET `/contactos`**: lista todos los contactos.  
  - **POST `/contactos`**: crea un nuevo contacto con validación de campos.  
  - **PUT `/contactos/:id`**: actualiza un contacto existente por ID.  
  - **DELETE `/contactos/:id`**: elimina un contacto por ID.  
  - **GET `/`**: devuelve el archivo `index.html` como página principal.  

  ```js
  const express = require('express');
  const path = require('path');
  const app = express();
  const PORT = 3000;

  app.use(express.json());
  app.use(express.urlencoded({ extended: true }));

  // Servir frontend
  const publicDir = path.join(__dirname, '../frontend');
  app.use(express.static(publicDir));

  let contactos = [];
  let nextId = 1;

  // Listar
  app.get('/contactos', (req, res) => {
    res.json(contactos);
  });

  // Crear
  app.post('/contactos', (req, res) => {
    const { nombre, telefono } = req.body;
    if (!nombre || !telefono) {
      return res.status(400).json({ mensaje: 'Faltan campos' });
    }
    const nuevo = { id: nextId++, nombre, telefono };
    contactos.push(nuevo);
    res.status(201).json(nuevo);
  });

  // Actualizar
  app.put('/contactos/:id', (req, res) => {
    const id = Number(req.params.id);
    const { nombre, telefono } = req.body;
    const idx = contactos.findIndex(c => c.id === id);
    if (idx === -1) return res.status(404).json({ mensaje: 'No encontrado' });
    if (!nombre || !telefono) {
      return res.status(400).json({ mensaje: 'Faltan campos' });
    }
    contactos[idx] = { id, nombre, telefono };
    res.json(contactos[idx]);
  });

  // Borrar
  app.delete('/contactos/:id', (req, res) => {
    const id = Number(req.params.id);
    const idx = contactos.findIndex(c => c.id === id);
    if (idx === -1) return res.status(404).json({ mensaje: 'No encontrado' });
    const eliminado = contactos.splice(idx, 1)[0];
    res.json(eliminado);
  });

  app.get('/', (req, res) => {
    res.sendFile(path.join(publicDir, 'index.html'));
  });

  app.listen(PORT, () => {
    console.log(`Servidor escuchando en http://localhost:${PORT}`);
  });
  ```

- **Paso 17.** Valida la ejecución del servidor. Ejecuta el siguiente comando en la terminal dentro del directorio **backend**

  > **NOTA:** **Acepta** los permisos si aparece una ventana emergente.
  {: .lab-note .info .compact}

  ```bash
  node server.js
  ```

- **Paso 18.** El resultado esperado es ver el mensaje: `Servidor escuchando en http://localhost:3000`.
  ![micint]({{ page.images_base | relative_url }}/14.png)

- **Paso 19.** Abre otra terminal en **VS Code** e inserta un contacto de prueba:

  ```bash
  curl -X POST http://localhost:3000/contactos \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Skywalker", "telefono": "123456789"}'
  ```
  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 20.** Abre el siguiente endpoint en una pestaña del navegador (Google Chrome):

  ```
  http://localhost:3000/contactos
  ```

- **Paso 21.** Como resultado deberas ver un JSON con el contacto registrado.

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 22.** Regresa a la terminal donde se ejecutó **node server.js** y detén el proceso con `CTRL + C`.

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Crear el frontend

Construir una página **HTML/CSS/JS** que interactúe con el backend para agregar y mostrar contactos.

#### Tarea 3.1

- **Paso 23.** En la carpeta `frontend`, crea los archivos base y abre `index.html`:

  > **NOTA:** Si estás en **backend**, sube un nivel antes de entrar a **frontend**.
  {: .lab-note .info .compact}

  ```bash
  cd ../frontend
  touch index.html style.css
  code index.html
  ```

- **Paso 24.** Copia y pega el siguiente código **HTML** dentro de `index.html`.

  > **NOTA:** El **frontend** se comunica con el **backend** usando **fetch**, simulando el consumo de una API REST.
  {: .lab-note .info .compact}

  - **Título**: "Agenda de Contactos".  
  - **Formulario** para agregar o editar contactos (nombre, teléfono, botones de acción).  
  - **Tabla** con columnas: Nombre, Teléfono y Acciones.  
  - **Script `app.js`** para la lógica del frontend.  

  ```html
  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Agenda de Contactos</title>
    <link rel="stylesheet" href="style.css">
  </head>
  <body>
    <h1>Agenda de Contactos</h1>

    <form id="form">
      <input type="hidden" id="id" />
      <input type="text" id="nombre" placeholder="Nombre" required />
      <input type="text" id="telefono" placeholder="Teléfono" required />
      <button type="submit" id="btn-guardar">Agregar</button>
      <button type="button" id="btn-cancelar" style="display:none;">Cancelar</button>
    </form>

    <table border="1">
      <thead>
        <tr>
          <th>Nombre</th>
          <th>Teléfono</th>
          <th>Acciones</th>
        </tr>
      </thead>
      <tbody id="tabla"></tbody>
    </table>

    <script src="app.js"></script>
  </body>
  </html>
  ```

- **Paso 25.** Abre el archivo **style.css** (en el directorio **frontend**):

  ```bash
  code style.css
  ```

- **Paso 26.** Copia y pega el siguiente código en **style.css**:

  ```css
  /* Estilos generales */
  body {
    font-family: Arial, Helvetica, sans-serif;
    background: #f5f7fa;
    margin: 0;
    padding: 20px;
    color: #333;
  }

  /* Título */
  h1 {
    text-align: center;
    color: #2c3e50;
    margin-bottom: 20px;
  }

  /* Formulario */
  form {
    display: flex;
    justify-content: center;
    gap: 10px;
    margin-bottom: 20px;
  }

  form input {
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 6px;
    outline: none;
    transition: border-color 0.2s;
  }

  form input:focus {
    border-color: #2980b9;
  }

  form button {
    padding: 10px 20px;
    background: #2980b9;
    color: #fff;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.2s;
  }

  form button:hover {
    background: #1f6391;
  }

  /* Tabla */
  table {
    width: 60%;
    margin: 0 auto;
    border-collapse: collapse;
    background: #fff;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    border-radius: 8px;
    overflow: hidden;
  }

  thead {
    background: #2980b9;
    color: #fff;
  }

  th, td {
    padding: 12px 15px;
    text-align: center;
    border-bottom: 1px solid #ddd;
  }

  tbody tr:hover {
    background: #f0f8ff;
  }
  ```

- **Paso 27.** Crea el archivo **app.js** dentro de **frontend**:

  ```bash
  touch app.js
  code app.js
  ```

- **Paso 28.** Copia y pega el siguiente código dentro de **app.js**:

  > **NOTA:**
  - **Carga inicial**: obtiene `/contactos` y pinta filas en la tabla.
  - **Render seguro**: usa `escapeHtml` para evitar inyección al mostrar nombre/teléfono.
  - **Delegación de eventos**: botones **Editar**/**Borrar** se enlazan tras renderizar.
  - **Crear/Actualizar**: en `submit`, valida campos y hace `POST` o `PUT` según haya `id`.
  - **Modo edición**: `activarEdicion()` llena el formulario y cambia el botón a **Actualizar**.
  - **Borrado**: `borrarContacto()` confirma y hace `DELETE`, luego recarga la lista.
  - **Reset UI**: `resetForm()` limpia inputs y restaura botones (**Agregar** / oculta **Cancelar**).
  - **Estado en memoria**: reutiliza `contactos` cargados para ubicar el registro a editar.
  {: .lab-note .info .compact}

  ```js
  const form = document.getElementById('form');
  const tabla = document.getElementById('tabla');
  const idEl = document.getElementById('id');
  const nombreEl = document.getElementById('nombre');
  const telEl = document.getElementById('telefono');
  const btnGuardar = document.getElementById('btn-guardar');
  const btnCancelar = document.getElementById('btn-cancelar');

  async function cargarContactos() {
    const res = await fetch('/contactos');
    const contactos = await res.json();

    tabla.innerHTML = contactos.map(c => `
      <tr>
        <td>${escapeHtml(c.nombre)}</td>
        <td>${escapeHtml(c.telefono)}</td>
        <td>
          <button data-id="${c.id}" class="btn-editar">Editar</button>
          <button data-id="${c.id}" class="btn-borrar">Borrar</button>
        </td>
      </tr>
    `).join('');

    // Delegación de eventos para Editar/Borrar
    tabla.querySelectorAll('.btn-editar').forEach(btn => {
      btn.addEventListener('click', () => activarEdicion(btn.dataset.id, contactos));
    });

    tabla.querySelectorAll('.btn-borrar').forEach(btn => {
      btn.addEventListener('click', () => borrarContacto(btn.dataset.id));
    });
  }

  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    const payload = {
      nombre: nombreEl.value.trim(),
      telefono: telEl.value.trim()
    };
    if (!payload.nombre || !payload.telefono) {
      alert('Nombre y Teléfono son obligatorios');
      return;
    }

    // Crear o actualizar
    if (!idEl.value) {
      // CREATE
      const res = await fetch('/contactos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      if (!res.ok) return alert('Error al crear');
    } else {
      // UPDATE
      const res = await fetch(`/contactos/${idEl.value}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      if (!res.ok) return alert('Error al actualizar');
    }

    resetForm();
    await cargarContactos();
  });

  btnCancelar.addEventListener('click', () => resetForm());

  function activarEdicion(id, contactosCache) {
    const c = contactosCache.find(x => String(x.id) === String(id));
    if (!c) return;

    idEl.value = c.id;
    nombreEl.value = c.nombre;
    telEl.value = c.telefono;

    btnGuardar.textContent = 'Actualizar';
    btnCancelar.style.display = 'inline-block';
  }

  async function borrarContacto(id) {
    if (!confirm('¿Eliminar este contacto?')) return;
    const res = await fetch(`/contactos/${id}`, { method: 'DELETE' });
    if (!res.ok) return alert('Error al borrar');
    await cargarContactos();
  }

  function resetForm() {
    idEl.value = '';
    nombreEl.value = '';
    telEl.value = '';
    btnGuardar.textContent = 'Agregar';
    btnCancelar.style.display = 'none';
  }

  // Sencilla sanitización para insertar en HTML
  function escapeHtml(str) {
    return String(str)
      .replaceAll('&', '&amp;')
      .replaceAll('<', '&lt;')
      .replaceAll('>', '&gt;')
      .replaceAll('"', '&quot;')
      .replaceAll("'", '&#039;');
  }

  // Inicial
  cargarContactos();
  ```

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Crear el Dockerfile 

Contenerizar la aplicación en una imagen **Docker**.

#### Tarea 4.1

- **Paso 29.** Crea el archivo **Dockerfile** en el directorio **lab1-acontactos**.

  > **NOTA:** Si te encuentras en **frontend**, sube un nivel antes de crear el archivo. Ajusta las rutas si es necesario.
  {: .lab-note .info .compact}

  ```bash
  cd ..
  touch Dockerfile
  code Dockerfile
  ```

- **Paso 30.** Copia y pega el siguiente contenido en el **Dockerfile**:

  - **Imagen base**: usa `node:20-alpine` (ligera y optimizada).  
  - **Directorio de trabajo**: establece `/app`.  
  - **Dependencias**: copia `package*.json` del backend y ejecuta `npm install --only=production`.  
  - **Código fuente**: copia carpetas `backend` y `frontend` dentro del contenedor.  
  - **Puerto expuesto**: abre el **3000** (para el servidor Express).  
  - **Comando final**: arranca la app con `node backend/server.js`.  


  ```dockerfile
  FROM node:20-alpine

  WORKDIR /app

  # Instalar dependencias del backend
  COPY backend/package*.json ./backend/
  RUN cd backend && npm install --only=production

  # Copiar el resto del código
  COPY backend ./backend
  COPY frontend ./frontend

  EXPOSE 3000

  CMD ["node", "backend/server.js"]
  ```

- **Paso 31.** Construye la imagen **Docker**:

  > **NOTA:** El comando se ejecuta en el directorio raíz **lab1-acontactos**. La salida puede ser extensa.
  {: .lab-note .info .compact}

  ```bash
  docker build -t agenda-contactos .
  ```
  ![micint]({{ page.images_base | relative_url }}/17.png)

- **Paso 32.** Verifica la imagen creada:

  > **NOTA:** Puedes ignorar otras imágenes, se usarán en laboratorios posteriores.
  {: .lab-note .info .compact}

  ```bash
  docker images
  ```
  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 33.** Ejecuta el contenedor a partir de la imagen:

  > **NOTA:** Si aparece una ventana emergente de permisos, **permite** la ejecución.
  {: .lab-note .info .compact}

  ```bash
  docker run --rm -d -p 3000:3000 --name agenda agenda-contactos
  ```
  ![micint]({{ page.images_base | relative_url }}/19.png)

- **Paso 34.** Verifica que el contenedor esté en ejecución:

  ```bash
  docker ps
  ```
  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 35.** Abre una ventana de Google Chrome y accede a la aplicación:

  ```
  http://localhost:3000
  ```
  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 36.** Prueba registrando uno o más contactos.

  <table class="lab-table">
    <thead>
      <tr>
        <th>Nombre</th>
        <th>Teléfono</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Optimus</td>
        <td>2837018302</td>
      </tr>
      <tr>
        <td>Firestorm</td>
        <td>2929201832</td>
      </tr>
    </tbody>
  </table>

  ![micint]({{ page.images_base | relative_url }}/22.png)

- **Paso 37.** Cuando termines de interactuar con la **Agenda de Contactos**, detén el contenedor:

  ```bash
  docker stop agenda
  ```

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
