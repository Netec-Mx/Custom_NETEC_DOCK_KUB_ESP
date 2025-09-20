---
layout: lab
title: "Práctica 12: Despliegue con almacenamiento persistente usando PVC"
permalink: /capitulo12/lab12/
images_base: /labs/capitulo12/img
duration: "60 minutos"
objective:
  - Desplegar la **Agenda de Contactos con interfaz web** (Node.js + HTML/JS) en **Kubernetes (Minikube)** con **almacenamiento persistente** mediante **PersistentVolumeClaim (PVC)**. Implementarás la API para almacenar contactos en `/data/contacts.json`, **montado desde un PVC**.
prerequisites:
  - Visual Studio Code
  - Docker Desktop en ejecución
  - Minikube y kubectl configurados 
  - Terminal **Git Bash** dentro de VS Code
  - Conocimientos básicos de Node.js, Docker y Kubernetes
introduction:
  Un **PersistentVolume (PV)** representa almacenamiento físico (disco local/nube). Un **PersistentVolumeClaim (PVC)** es la **solicitud** de almacenamiento por parte de una app. Cuando montas un PVC en un Pod, los datos sobreviven a reinicios, reprogramaciones y actualizaciones de imagen. Convertiremos la **agenda efímera** parecida a la de la Práctica 1 en una **agenda persistente**, más cercana a producción.
slug: lab12
lab_number: 12
final_result: >
  Una **Agenda de Contactos** con interfaz web desplegada en Kubernetes que **persiste** información en un **PVC**. Se aplicaron **buenas prácticas de seguridad** (no root, `fsGroup`), **liveness/readiness probes**, despliegue controlado y **actualización sin pérdida de datos**.
notes: 
  - Para **alta disponibilidad** con múltiples réplicas que escriben, usa una **BD** o un backend de archivos con bloqueo. Un JSON plano no es seguro para escritura concurrente.  
  - En producción, considera exponer con **Ingress + TLS** (`minikube addons enable ingress`) y manejar observabilidad (logs/metrics).  
  - Ajusta `requests/limits` según la capacidad del clúster.  
  - Algunas StorageClass soportan **expansión de volumen**; revisa documentación si necesitas crecer el PVC.
references:
  - text: Persistent Volumes & Claims
    url: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
  - text: Volumes en Pods
    url: https://kubernetes.io/docs/concepts/storage/volumes/
  - text: Probes (liveness/readiness)
    url: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
  - text: Security Context
    url: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/

prev: /capitulo11/lab11/          
next: /capitulo13/lab13/
---


---

### Tarea 1: Crear la estructura del proyecto

Organizar carpetas para separar el código de la app y los manifiestos de Kubernetes. Esto facilita el mantenimiento, la depuración y el control de versiones.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`** lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VSCode** da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/19.png)

- **Paso 5.** Asegurate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**:

  > **NOTA:** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica**.

  > **NOTA:** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab12-k8spvcagenda && cd lab12-k8spvcagenda
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Esta sera la estructura de los directorios de la aplicación:

  > **NOTA:**  
  - Separar `app/` (código) 
  - `k8s/` agrupa todos los manifiestos. 
  {: .lab-note .info .compact}
  
  ```text
  lab12-k8spvcagenda/
  ├── app/
  │   ├── package.json
  │   ├── server.js
  │   └── public/
  │       └── index.html
  ├── Dockerfile
  ├── k8s/
  │   ├── namespace.yaml
  │   ├── pvc.yaml
  │   ├── deployment.yaml
  │   └── service.yaml
  └── .dockerignore
  ```

- **Paso 9.** Ahora crea la carpeta **app/** y sus archivos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab12-k8spvcagenda**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p app app/public && touch app/package.json app/server.js app/public/index.html
  ```

- **Paso 10.** Muy bien continua la cración del directorio **k8s/** con los manifiestos vacios.

  > **NOTA:**  El comando se ejecuta desde la raíz de la carpeta **lab12-k8spvcagenda**. Estos comandos ya crean los directorios y archivos.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/namespace.yaml k8s/pvc.yaml k8s/deployment.yaml k8s/service.yaml
  ```

- **Paso 11.** Crea los ultimos dos archivos del proyecto **.dockerignore** y **Dockerfile**.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab12-k8spvcagenda**
  {: .lab-note .info .compact}

  ```bash
  touch .dockerignore Dockerfile
  ```

- **Paso 12.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias:

  > **NOTA:** Reduce el **contexto de build** y acelera las compilaciones evitando subir archivos innecesarios al daemon de Docker.
  {: .lab-note .info .compact}

  ```gitignore
  # evita copiar node_modules del host (causa del ELF inválido)
  app/node_modules
  **/node_modules

  # basura
  npm-debug.log
  Dockerfile
  docker-compose.yml
  .git
  .gitignore
  .DS_Store
  ```

- **Paso 13.** Valida la creacion de la estructura de tu proyecto, escribe el siguiente comando.

  > **NOTA:** También puedes validarlo en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Implementar la Agenda (API + UI) con persistencia a archivo

Construirás una API Express (endpoints CRUD mínimos) y una UI simple. Los datos se guardan en **`/data/contacts.json`** para luego montar un PVC.

#### Tarea 2.1

- **Paso 14.** Dentro del archivo `app/package.json` agrega las siguientes dependencias.

  > **NOTA:** 
  - `express` sirve la API/UI.
  - `cors` permite pruebas desde otros orígenes si fuese necesario.
  {: .lab-note .info .compact}

  ```json
  {
    "name": "agenda-pvc",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": { "start": "node server.js" },
    "dependencies": {
      "express": "^4.18.2",
      "cors": "^2.8.5"
    }
  }
  ```

- **Paso 15.** Abre el archivo `app/server.js` y agrega el siguiente bloque de codigo para la logica de la aplicación.

  > **NOTA:** El archivo **`/data/contacts.json`** es el punto de persistencia donde montaremos el PVC.
  {: .lab-note .info .compact}

  ```javascript
  const fs = require('fs');
  const path = require('path');
  const express = require('express');
  const cors = require('cors');

  const app = express();
  const PORT = process.env.PORT || 3000;
  const DATA_DIR = process.env.DATA_DIR || '/data';
  const DATA_FILE = process.env.DATA_FILE || 'contacts.json';
  const DATA_PATH = path.join(DATA_DIR, DATA_FILE);

  app.use(cors());
  app.use(express.json());
  app.use(express.static(path.join(__dirname, 'public')));

  function ensureStorage() {
    if (!fs.existsSync(DATA_DIR)) fs.mkdirSync(DATA_DIR, { recursive: true });
    if (!fs.existsSync(DATA_PATH)) fs.writeFileSync(DATA_PATH, '[]', 'utf8');
  }

  function readContacts() {
    try {
      const raw = fs.readFileSync(DATA_PATH, 'utf8');
      return JSON.parse(raw);
    } catch (e) {
      console.error('Error leyendo contactos:', e.message);
      return [];
    }
  }

  function writeContacts(contacts) {
    fs.writeFileSync(DATA_PATH, JSON.stringify(contacts, null, 2), 'utf8');
  }

  app.get('/api/contacts', (_req, res) => {
    ensureStorage();
    return res.json(readContacts());
  });

  app.post('/api/contacts', (req, res) => {
    const { nombre, email, telefono } = req.body || {};
    if (!nombre || !email) {
      return res.status(400).json({ error: 'nombre y email son requeridos' });
    }
    ensureStorage();
    const contacts = readContacts();
    const newContact = {
      id: Date.now().toString(),
      nombre,
      email,
      telefono: telefono || ''
    };
    contacts.push(newContact);
    writeContacts(contacts);
    return res.status(201).json(newContact);
  });

  app.delete('/api/contacts/:id', (req, res) => {
    ensureStorage();
    const contacts = readContacts();
    const filtered = contacts.filter(c => c.id !== req.params.id);
    writeContacts(filtered);
    return res.status(204).end();
  });

  app.get('/health', (_req, res) => res.json({ status: 'ok' }));

  app.listen(PORT, () => {
    ensureStorage();
    console.log(`Agenda escuchando en http://localhost:${PORT}`);
  });
  ```

- **Paso 16.** Dentro del archivo `app/public/index.html` agrega el codigo para la pagina web y estilos sencillos.

  > **NOTA:** UI mínima para **crear/listar/eliminar**. Perfecta para verificar persistencia fácilmente.
  {: .lab-note .info .compact}
  
  ```html
  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>Agenda de Contactos (PVC)</title>
    <style>
      body{font-family:system-ui,sans-serif;max-width:900px;margin:2rem auto;padding:0 1rem}
      h1{margin:0 0 1rem 0}
      .card{border:1px solid #e5e7eb;border-radius:12px;padding:1rem;box-shadow:0 1px 2px rgba(0,0,0,.05)}
      form{display:flex;gap:.5rem;flex-wrap:wrap;margin-bottom:1rem}
      input,button{padding:.6rem .8rem;border:1px solid #e5e7eb;border-radius:10px}
      table{width:100%;border-collapse:collapse}
      th,td{padding:.6rem;border-bottom:1px solid #f0f0f0;text-align:left}
      .muted{color:#6b7280;font-size:.9rem}
    </style>
  </head>
  <body>
    <h1>Agenda de Contactos (PVC)</h1>
    <div class="card">
      <form id="form">
        <input id="nombre" placeholder="Nombre *" required />
        <input id="email" placeholder="Email *" type="email" required />
        <input id="telefono" placeholder="Teléfono" />
        <button type="submit">Agregar</button>
      </form>
      <p class="muted">Los datos se guardan en un archivo JSON montado desde un PVC.</p>

      <table>
        <thead><tr><th>Nombre</th><th>Email</th><th>Teléfono</th><th></th></tr></thead>
        <tbody id="tbody"></tbody>
      </table>
    </div>

    <script>
      const api = (p) => fetch(p).then(r=>r.json());

      function render(rows){
        const tbody = document.getElementById('tbody');
        tbody.innerHTML = rows.map(c => `
          <tr>
            <td>${c.nombre}</td>
            <td>${c.email}</td>
            <td>${c.telefono||''}</td>
            <td><button data-id="${c.id}" class="del">Eliminar</button></td>
          </tr>`).join('');
        [...document.querySelectorAll('.del')].forEach(b=>{
          b.onclick = async () => {
            await fetch('/api/contacts/'+b.dataset.id,{method:'DELETE'});
            load();
          };
        });
      }

      async function load(){ render(await api('/api/contacts')); }
      document.getElementById('form').onsubmit = async (e) => {
        e.preventDefault();
        const body = {
          nombre: document.getElementById('nombre').value.trim(),
          email: document.getElementById('email').value.trim(),
          telefono: document.getElementById('telefono').value.trim()
        };
        if(!body.nombre || !body.email){ alert('Campos requeridos'); return; }
        const res = await fetch('/api/contacts',{
          method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)
        });
        if(!res.ok){ alert('Error: revisa los campos'); return; }
        e.target.reset(); load();
      };
      load();
    </script>
  </body>
  </html>
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Dockerfile build, tag y push.

Crear imagen Docker, compilarla y subirla al repositorio remoto para reusarla en el cluster de Minikube.

#### Tarea 3.1

- **Paso 17.** Abre el archivo `Dockerfile` que se encuentra en la raíz del proyecto y agrega el siguiente codigo.

  ```dockerfile
  FROM node:20-alpine

  WORKDIR /app
  COPY app/package*.json ./
  RUN npm install --production
  COPY app ./

  ENV PORT=3000
  ENV DATA_DIR=/data
  ENV DATA_FILE=contacts.json

  # preparar permisos y usuario no root
  RUN mkdir -p /data && chown -R node:node /data /app
  USER node

  EXPOSE 3000
  CMD ["node", "server.js"]
  ```

- **Paso 18.** Recuerda que debes estar autenticado a tu cuenta de **Docker Hub**.

  - Si ya tienes cuenda da clic <a href="https://login.docker.com/u/login/identifier?state=hKFo2SB1QXpUSzVVc3ZDYTAzQzlkWlFoYk9LWnlLZ1VOMzNnU6Fur3VuaXZlcnNhbC1sb2dpbqN0aWTZIGx0SFpVWDNQTzNPaFZlT1FxVDlWZUpzdWUya09FaWtjo2NpZNkgbHZlOUdHbDhKdFNVcm5lUTFFVnVDMGxiakhkaTluYjk" target="_blank" rel="noopener noreferrer"><strong>AQUÍ - Iniciar Sesión</strong></a>.
  - Si no tienes cuenta da clic <a href="https://app.docker.com/signup?_gl=1*1ugpfey*_gcl_au*OTcyNTkxNjkyLjE3NTc2MDY4MTU.*_ga*MTQxMjc1NjI4My4xNzU3NjA2ODE1*_ga_XJWPQMJYHQ*czE3NTc2MTE1NTQkbzIkZzEkdDE3NTc2MTE1NTQkajYwJGwwJGgw" target="_blank" rel="noopener noreferrer"><strong>AQUÍ - Crear cuenta</strong></a>.

- **Paso 19.** Ahora regresa a la terminal de **VSCode** y escribe el siguiente comando para autenticar la terminal a **Docker Hub**.

  > **NOTA:** Sigue los pasos de la terminal para autenticarte.
  {: .lab-note .info .compact}

  ```bash
  docker login
  ```

- **Paso 20.** Compila la imagen, escribe el siguiente comando:

  > **NOTA:**
  - El comando se ejecuta dentro del directorio raíz **lab12-k8spvcagenda**.
  - Ajusta las rutas si es necesario
  {: .lab-note .info .compact}

  ```bash
  docker build -t agendapvc .
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 21.** Ahora etiqueta la imagen y subela al repositorio remoto en **Docker Hub**.

  > **NOTA:**
  - Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, **puede ser el mismo que usaste en la practica anterior**
  - El comando no dara salida a menos que haya un error.
  {: .lab-note .info .compact}

  ```bash
  docker tag agendapvc TU_USUARIO/TU_REPOSITORIO:agendapvc
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

- **Paso 22.** Si todo sale bien, el siguiente comando subira la imagen al repositorio remoto:

  > **NOTA:** Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la practica anterior
  {: .lab-note .info .compact}

  ```bash
  docker push TU_USUARIO/TU_REPOSITORIO:agendapvc
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Namespace y PVC (almacenamiento persistente)

Crearás un namespace para aislar recursos y un PVC que solicitará **1Gi** en modo `ReadWriteOnce`.

#### Tarea 4.1

- **Paso 23.** Abre el archivo `k8s/namespace.yaml` y define la siguiente configuración para crear el namespace.

  > **NOTA:** Un namespace evita mezclar objetos con otras prácticas y permite reglas específicas.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: agenda-pvc
    labels:
      app: agenda
  ```

- **Paso 24.** Primero asegurate de encender siempre minikube, escribe el siguiente comando.

  > **NOTA:** Espera unos segundos en lo que termina de inicializar.
  {: .lab-note .info .compact}

  ```bash
  minikube start
  ```

- **Paso 25.** Ahora aplica el manifiesto de la configuración del namespace.

  ```bash
  kubectl apply -f k8s/namespace.yaml
  ```

- **Paso 26.** Verifica que se haya creado correctamente el **Namespace**

  ```bash
  kubectl get ns agenda-pvc
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 27.** Abre el archivo `k8s/pvc.yaml` y define la configuración para solicitar el espacio de almacenamiento.

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: contacts-pvc
    namespace: agenda-pvc
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 1Gi
    volumeMode: Filesystem
  ```

- **Paso 28.** Ahora aplica el manifiesto de la configuración del namespace.

  ```bash
  kubectl apply -f k8s/pvc.yaml
  ```

- **Paso 29.** Verifica que se haya creado correctamente el **Namespace**

  > **NOTA:** `Bound` indica que el PVC encontró un PV compatible (StorageClass por defecto).
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc get pvc
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Deployment con volumen persistente, probes y seguridad

Crear un Deployment (1 réplica) que **monte el PVC** en `/data`. Añadir **probes** y **securityContext** para buenas prácticas.

#### Tarea 5.1

- **Paso 30.** Bien!, abre el archivo `k8s/deployment.yaml` y agrega la siguiente definición del deployment.

  > **NOTA:** La propiedad `fsGroup: 1000` hace que el sistema de archivos montado pertenezca al **grupo** del contenedor (uid/gid = 1000), garantizando permisos de escritura para el usuario `node` (no root). Una sola réplica evita conflictos de escritura en el **mismo archivo JSON**.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la practica anterior.
  {: .lab-note .important .compact}

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: agenda-deploy
    namespace: agenda-pvc
    labels:
      app: agenda
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: agenda
    template:
      metadata:
        labels:
          app: agenda
      spec:
        securityContext:
          fsGroup: 1000
          runAsUser: 1000
          runAsNonRoot: true
        containers:
        - name: web
          image: TU_USUARIO/TU_REPOSITORIO:agendapvc # CAMBIA AQUI LA CUENTA/REPOSITORIO DE DOCKER HUB
          ports:
            - containerPort: 3000
          env:
            - name: DATA_DIR
              value: "/data"
            - name: DATA_FILE
              value: "contacts.json"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          volumeMounts:
            - name: data
              mountPath: /data
        volumes:
          - name: data
            persistentVolumeClaim:
              claimName: contacts-pvc
  ```

- **Paso 31.** Ejecuta la configuración del archivo deployment, escribe el siguiente comando.

   ```bash
   kubectl apply -f k8s/deployment.yaml
   ```

- **Paso 32.** Verifica que el pod este corriendo correctamente.

  > **NOTA:** Deberias observar `Running` y `READY 1/1`  
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc get pods -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 33.** Obten el nombre del pod creado.

  ```bash
  POD=$(kubectl -n agenda-pvc get pod -l app=agenda -o jsonpath='{.items[0].metadata.name}')
  echo "El nombre del pod es: "$POD
  ```

- **Paso 34.** Revisa los detalles del pod.

  > **NOTA:** Identifica la seccion **`Mounts`** y **`Volumes`**
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc describe pod $POD
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Service (NodePort), pruebas funcionales y persistencia

Exponer la app con un **Service NodePort** y demostrar que los datos **persisten** tras reiniciar el Pod.

#### Tarea 6.1

- **Paso 35.** Dentro del archivo `k8s/service.yaml` agrega la siguiente definición de exposición.

  > **NOTA:** La propiedad `NodePort` expone rápidamente el servicio sin requerir Ingress.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: agenda-svc
    namespace: agenda-pvc
  spec:
    type: NodePort
    selector:
      app: agenda
    ports:
      - port: 3000
        targetPort: 3000
        nodePort: 30083
  ```

- **Paso 36.** Aplica el manifiesto del servicio.

  ```bash
  kubectl apply -f k8s/service.yaml
  ```
 
- **Paso 37.** Ejecuta el siguiente comando para obtener la URL de exposición 

  ```bash
  minikube service agenda-svc -n agenda-pvc --url
  ``` 

- **Paso 38.** Abre la URL en el navegador.

  > **NOTA:**
  - Crea **2–3 contactos**.  
  - Actualiza y verifica que se listan. 
  - Con esto validamos que **API + UI** funcionan antes de probar persistencia.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 39.** En la teminal que ocupa el proceso **minikube service** ejecuta `CTRL + c` cuando termines de probar la aplicación.

- **Paso 40.** Reinicia el Pod para probar persistencia.

  > **NOTA:**
  - Espera a que el nuevo Pod esté `Running`/`READY 1/1`.
  - Rompe el proceso de verificación con **`CTRL + c`**
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc delete pod -l app=agenda
  kubectl -n agenda-pvc get pods -w
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 41.** Levanta el servicio, abre la aplicación y verifica que **los contactos siguen** ahí.

  > **NOTA:**
  - Si todo salio bien debes ver tus contactos.
  - El **minikube service** asignara un nuevo puerto
  {: .lab-note .info .compact}

  ```bash
  minikube service agenda-svc -n agenda-pvc --url
  ``` 

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 42.** En la teminal que ocupa el proceso **minikube service** ejecuta `CTRL + c` cuando termines de probar la aplicación.

- **Paso 43.** Tambien puedes inspeccionar los datos dentro del Pod por CLI:

  > **NOTA:** El contenedor se destruye/recrea, **pero el PVC permanece**. Por eso el archivo vuelve a estar con tus datos.
  {: .lab-note .info .compact}

  ```bash
  POD=$(kubectl -n agenda-pvc get pods -l app=agenda -o jsonpath='{.items[0].metadata.name}')
  kubectl -n agenda-pvc exec -it "$POD" -- sh -lc 'ls -l /data && cat /data/contacts.json'
  ```
  
  ![micint]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Rolling update de la imagen sin perder datos

Simular una actualización de la app (cambio visual menor) y realizar un **rolling update** para confirmar que los datos permanecen intactos en el PVC.

#### Tarea 7.1

- **Paso 44.** En el archivo `app/public/index.html` reemplaza todo el codigo existente por este nuevo.

  > **NOTA:**
  - Cambios de estilos en los titulos, botones (Simula una nueva versión)
  - Cambio en el titulo de la aplicación
  {: .lab-note .info .compact}

  ```html
  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>Agenda de Contactos v2</title>
    <style>
      body {font-family: system-ui, sans-serif;max-width: 900px;margin: 2rem auto;padding: 0 1rem;background: #f9fafb;}
      h1 {margin: 0 0 1rem 0;color: #1d4ed8; /* azul más fuerte para distinguir v2 */}
      .card {border: 1px solid #cbd5e1;border-radius: 12px;padding: 1rem;box-shadow: 0 2px 6px rgba(0,0,0,.1);background: #ffffff;}
      form {display: flex;gap: .5rem;flex-wrap: wrap;margin-bottom: 1rem;}
      input, button {padding: .6rem .8rem;border: 1px solid #cbd5e1;border-radius: 10px;}
      button {background-color: #1d4ed8;color: #fff;cursor: pointer;transition: background .2s;}
      button:hover {background-color: #2563eb;}
      table {width: 100%;border-collapse: collapse;}
      th, td {padding: .6rem;border-bottom: 1px solid #f0f0f0;text-align: left;}
      th {background-color: #f1f5f9;color: #334155;}
      .muted {color: #64748b;font-size: .9rem;}
    </style>
  </head>
  <body>
    <h1>Agenda de Contactos (PVC) — Versión 2</h1>
    <div class="card">
      <form id="form">
        <input id="nombre" placeholder="Nombre *" required />
        <input id="email" placeholder="Email *" type="email" required />
        <input id="telefono" placeholder="Teléfono" />
        <button type="submit">Agregar</button>
      </form>
      <p class="muted">Versión 2: Los datos siguen guardándose en un archivo JSON montado desde un PVC.</p>

      <table>
        <thead><tr><th>Nombre</th><th>Email</th><th>Teléfono</th><th></th></tr></thead>
        <tbody id="tbody"></tbody>
      </table>
    </div>

    <script>
      const api = (p) => fetch(p).then(r=>r.json());

      function render(rows){
        const tbody = document.getElementById('tbody');
        tbody.innerHTML = rows.map(c => `
          <tr>
            <td>${c.nombre}</td>
            <td>${c.email}</td>
            <td>${c.telefono||''}</td>
            <td><button data-id="${c.id}" class="del">Eliminar</button></td>
          </tr>`).join('');
        [...document.querySelectorAll('.del')].forEach(b=>{
          b.onclick = async () => {
            await fetch('/api/contacts/'+b.dataset.id,{method:'DELETE'});
            load();
          };
        });
      }

      async function load(){ render(await api('/api/contacts')); }
      document.getElementById('form').onsubmit = async (e) => {
        e.preventDefault();
        const body = {
          nombre: document.getElementById('nombre').value.trim(),
          email: document.getElementById('email').value.trim(),
          telefono: document.getElementById('telefono').value.trim()
        };
        if(!body.nombre || !body.email){ alert('Campos requeridos'); return; }
        const res = await fetch('/api/contacts',{
          method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)
        });
        if(!res.ok){ alert('Error: revisa los campos'); return; }
        e.target.reset(); load();
      };
      load();
    </script>
  </body>
  </html>
  ```

- **Paso 45.** Reconstruye la imagen con una nueva etiqueta.

  > **NOTA:** Versionar imágenes permite **rollback** rápido si algo falla.
  {: .lab-note .info .compact}

  ```bash
  docker build -t agendapvc:1.1 .
  ```

- **Paso 46.** Ahora etiqueta la nueva imagen creada.

  > **NOTA:**
  - Sustituye `TU_USUARIO/TU_REPOSITORIO` por el de tu cuenta, **puede ser el mismo que usaste en la practica anterior**
  - El comando no dara salida a menos que haya un error.
  {: .lab-note .info .compact}

  ```bash
  docker tag agendapvc:1.1 TU_USUARIO/TU_REPOSITORIO:agendapvc-v1.1
  ```

- **Paso 47.** Sube la imagen a tu repositorio remoto:

  > **NOTA:** Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la practica anterior
  {: .lab-note .info .compact}

  ```bash
  docker push TU_USUARIO/TU_REPOSITORIO:agendapvc-v1.1
  ```

- **Paso 48.** Abre el archivo `k8s/deployment.yaml` para ajustar a la nueva imagen.

  > **NOTA:**
  - En la **linea 24** del archivo solo agrega al final `-v1.1`.
  - Quizas la linea pueda variar, si es asi identificala.
  - Puedes apoyarte de la imagen, cuidado de no borrar nadamas.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 49.** Ejecuta el manifiesto con la actualización de la imagen.

   ```bash
   kubectl apply -f k8s/deployment.yaml
   ```

- **Paso 50.** Levanta el servicio, abre la aplicación y verifica que **los contactos siguen** ahí ahora con el **nuevo diseño**.

  > **NOTA:**
  - Recuerda abrir la URL que entrega **minikube service**
  - Si todo salio bien debes ver tus contactos.
  {: .lab-note .info .compact}

  ```bash
  minikube service agenda-svc -n agenda-pvc --url
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 51.** En la teminal que ocupa el proceso **minikube service** ejecuta `CTRL + c` cuando termines de probar la aplicación.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8: Limpieza de recursos

Siempre es importante detener y eliminar recursos creados que no se usaran.

#### Tarea 18.1

- **Paso 52.** Ejecuta el siguiente comando que limpiara todo lo creado en el cluster.

  > **NOTA:** Puede tardar unos segundos.
  {: .lab-note .info .compact}

  ```bash
  kubectl delete namespace agenda-pvc
  ```

- **Paso 53.** Verifica que este limpio.

  > **NOTA:**
  - `service/kubernetes` no se borra es parte del cluster.
  - Verifica que ya no aparezca **agenda-pvc** como namespace.
  {: .lab-note .info .compact}

  ```bash
  kubectl get ns
  kubectl get all
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}