---
layout: lab
title: "Práctica 12. Despliegue con almacenamiento persistente usando PVC"
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
  Un **PersistentVolume (PV)** representa almacenamiento físico (disco local o nube). Un **PersistentVolumeClaim (PVC)** es la **solicitud** de almacenamiento por parte de una app. Cuando montas un PVC en un pod, los datos sobreviven a reinicios, reprogramaciones y actualizaciones de imagen. Convertiremos la **agenda efímera**, parecida a la de la Práctica 1, en una **agenda persistente**, más cercana a producción.
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

### Tarea 1. Crear la estructura del proyecto

Organizar carpetas para separar el código de la app y los manifiestos de Kubernetes. Esto facilita el mantenimiento, la depuración y el control de versiones.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`**. Lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VS Code**, da clic en el icono de la imagen para abrir la terminal. Se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**. Da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/19.png)

- **Paso 5.** Asegúrate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VS Code**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica**.

  > **Nota.** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab10-k8spvcagenda && cd lab10-k8spvcagenda
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VS Code que se haya creado el directorio.

  > **Nota.** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Esta será la estructura de los directorios de la aplicación.

  > **Notas**  
  - Separar `app/` (código). 
  - `k8s/` agrupa todos los manifiestos. 
  {: .lab-note .info .compact}
  
  ```text
  lab10-k8spvcagenda/
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

- **Paso 9.** Ahora, crea la carpeta **app/** y sus archivos vacíos.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab10-k8spvcagenda**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p app app/public && touch app/package.json app/server.js app/public/index.html
  ```

- **Paso 10.** Muy bien. Continúa la creación del directorio **k8s/** con los manifiestos vacíos.

  > **Nota.**  El comando se ejecuta desde la raíz de la carpeta **lab10-k8spvcagenda**. Estos comandos ya crean los directorios y archivos.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/namespace.yaml k8s/pvc.yaml k8s/deployment.yaml k8s/service.yaml
  ```

- **Paso 11.** Crea los últimos dos archivos del proyecto **.dockerignore** y **Dockerfile**.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab10-k8spvcagenda**
  {: .lab-note .info .compact}

  ```bash
  touch .dockerignore Dockerfile
  ```

- **Paso 12.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias.

  > **Nota.** Reduce el **contexto de build** y acelera las compilaciones evitando subir archivos innecesarios al daemon de Docker.
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

- **Paso 13.** Valida la creación de la estructura de tu proyecto. Escribe el siguiente comando.

  > **Nota.** También puedes validarlo en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Implementar la agenda (API + UI) con persistencia a archivo

Construir una API Express (endpoints CRUD mínimos) y una UI simple. Los datos se guardan en **`/data/contacts.json`** para luego montar un PVC.

#### Tarea 2.1

- **Paso 14.** Dentro del archivo `app/package.json`, agrega las siguientes dependencias.

  > **Notas** 
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

- **Paso 15.** Abre el archivo `app/server.js` y agrega el siguiente bloque de código para la lógica de la aplicación.

  > **Nota.** El archivo **`/data/contacts.json`** es el punto de persistencia donde montaremos el PVC.
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

- **Paso 16.** Dentro del archivo `app/public/index.html`, agrega el código para la página web y estilos sencillos.

  > **Nota.** UI mínima para **crear, listar o eliminar**. Perfecta para verificar persistencia fácilmente.
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

### Tarea 3. Dockerfile build, tag y push

Crear imagen Docker, compilarla y subirla al repositorio remoto para reusarla en el clúster de Minikube.

#### Tarea 3.1

- **Paso 17.** Abre el archivo `Dockerfile` que se encuentra en la raíz del proyecto y agrega el siguiente código.

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

- **Paso 18.** Asegúrate de haber autenticado tu cuenta de **Docker Hub**.

  - Si ya tienes cuenta, da clic [**AQUÍ - Iniciar Sesión**](https://login.docker.com/u/login/identifier?state=hKFo2SB1QXpUSzVVc3ZDYTAzQzlkWlFoYk9LWnlLZ1VOMzNnU6Fur3VuaXZlcnNhbC1sb2dpbqN0aWTZIGx0SFpVWDNQTzNPaFZlT1FxVDlWZUpzdWUya09FaWtjo2NpZNkgbHZlOUdHbDhKdFNVcm5lUTFFVnVDMGxiakhkaTluYjk)
  - Si no tienes cuenta, da clic [**AQUÍ - Crear cuenta**](https://app.docker.com/signup?_gl=1*1ugpfey*_gcl_au*OTcyNTkxNjkyLjE3NTc2MDY4MTU.*_ga*MTQxMjc1NjI4My4xNzU3NjA2ODE1*_ga_XJWPQMJYHQ*czE3NTc2MTE1NTQkbzIkZzEkdDE3NTc2MTE1NTQkajYwJGwwJGgw)

- **Paso 19.** Ahora, regresa a la terminal de **VS Code** y escribe el siguiente comando para autenticar la terminal a **Docker Hub**.

  > **Nota.** Sigue los pasos de la terminal para autenticarte.
  {: .lab-note .info .compact}

  ```bash
  docker login
  ```

- **Paso 20.** Compila la imagen. Escribe el siguiente comando.

  > **Notas**
  - El comando se ejecuta dentro del directorio raíz **lab10-k8spvcagenda**.
  - Ajusta las rutas si es necesario.
  {: .lab-note .info .compact}

  ```bash
  docker build -t agendapvc .
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 21.** Ahora, etiqueta la imagen y súbela al repositorio remoto en **Docker Hub**.

  > **Notas**
  - Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, **puede ser el mismo que usaste en la práctica anterior**.
  - El comando no dará salida a menos que haya un error.
  {: .lab-note .info .compact}

  ```bash
  docker tag agendapvc TU_USUARIO/TU_REPOSITORIO:agendapvc
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

- **Paso 22.** Si todo sale bien, el siguiente comando subirá la imagen al repositorio remoto.

  > **Nota.** Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la práctica anterior.
  {: .lab-note .info .compact}

  ```bash
  docker push TU_USUARIO/TU_REPOSITORIO:agendapvc
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Namespace y PVC (almacenamiento persistente)

Crear un namespace para aislar recursos y un PVC que solicite **1Gi** en modo `ReadWriteOnce`.

#### Tarea 4.1

- **Paso 23.** Abre el archivo `k8s/namespace.yaml` y define la siguiente configuración para crear el namespace.

  > **Nota.** Un namespace evita mezclar objetos con otras prácticas y permite reglas específicas.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: agenda-pvc
    labels:
      app: agenda
  ```

- **Paso 24.** Primero, asegúrate de encender siempre Minikube. Escribe el siguiente comando.

  > **Nota.** Espera unos segundos mientras finaliza el proceso de inicialización.
  {: .lab-note .info .compact}

  ```bash
  minikube start
  ```

- **Paso 25.** Ahora, aplica el manifiesto de la configuración del namespace.

  ```bash
  kubectl apply -f k8s/namespace.yaml
  ```

- **Paso 26.** Verifica que se haya creado correctamente el **namespace**.

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

- **Paso 28.** Ahora, aplica el manifiesto de la configuración del namespace.

  ```bash
  kubectl apply -f k8s/pvc.yaml
  ```

- **Paso 29.** Verifica que se haya creado correctamente el **Namespace**

  > **Nota.** `Bound` indica que el PVC encontró un PV compatible (StorageClass por defecto).
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc get pvc
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Deployment con volumen persistente, probes y seguridad

Crear un Deployment (1 réplica) que **monte el PVC** en `/data`. Añadir **probes** y **securityContext** para buenas prácticas.

#### Tarea 5.1

- **Paso 30.** ¡Bien! Abre el archivo `k8s/deployment.yaml` y agrega la siguiente definición del deployment.

  > **Nota.** La propiedad `fsGroup: 1000` hace que el sistema de archivos montado pertenezca al **grupo** del contenedor (uid/gid = 1000), garantizando permisos de escritura para el usuario `node` (no root). Una sola réplica evita conflictos de escritura en el **mismo archivo JSON**.
  {: .lab-note .info .compact}

  > **Importante.** Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la práctica anterior.
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

- **Paso 31.** Ejecuta la configuración del archivo deployment. Escribe el siguiente comando.

   ```bash
   kubectl apply -f k8s/deployment.yaml
   ```

- **Paso 32.** Verifica que el pod esté corriendo correctamente.

  > **Nota.** Deberías observar `Running` y `READY 1/1`  
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc get pods -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 33.** Obtén el nombre del pod creado.

  ```bash
  POD=$(kubectl -n agenda-pvc get pod -l app=agenda -o jsonpath='{.items[0].metadata.name}')
  echo "El nombre del pod es: "$POD
  ```

- **Paso 34.** Revisa los detalles del pod.

  > **Nota.** Identifica la sección **`Mounts`** y **`Volumes`**.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc describe pod $POD
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Service (NodePort), pruebas funcionales y persistencia

Exponer la app con un **Service NodePort** y demostrar que los datos **persisten** tras reiniciar el pod.

#### Tarea 6.1

- **Paso 35.** Dentro del archivo `k8s/service.yaml`, agrega la siguiente definición de exposición.

  > **Nota.** La propiedad `NodePort` expone rápidamente el servicio sin requerir Ingress.
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
 
- **Paso 37.** Ejecuta el siguiente comando para obtener la URL de exposición. 

  ```bash
  minikube service agenda-svc -n agenda-pvc --url
  ``` 

- **Paso 38.** Abre la URL en el navegador.

  > **Notas**
  - Crea **2 o 3 contactos**.  
  - Actualiza y verifica que se listan. 
  - Con esto validamos que **API + UI** funcionan antes de probar persistencia.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 39.** En la terminal que ocupa el proceso **Minikube service** ejecuta `CTRL + c` cuando termines de probar la aplicación.

- **Paso 40.** Reinicia el pod para probar persistencia.

  > **Notas**
  - Espera a que el nuevo pod esté `Running`/`READY 1/1`.
  - Rompe el proceso de verificación con **`CTRL + c`**.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n agenda-pvc delete pod -l app=agenda
  kubectl -n agenda-pvc get pods -w
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 41.** Levanta el servicio, abre la aplicación y verifica que **los contactos siguen** ahí.

  > **Notas**
  - Si todo salió bien, debes ver tus contactos.
  - El **Minikube service** asignará un nuevo puerto.
  {: .lab-note .info .compact}

  ```bash
  minikube service agenda-svc -n agenda-pvc --url
  ``` 

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 42.** En la terminal que ocupa el proceso **Minikube service**, ejecuta `CTRL + c` cuando termines de probar la aplicación.

- **Paso 43.** También puedes inspeccionar los datos dentro del pod por CLI.

  > **Nota.** El contenedor se destruye o recrea, **pero el PVC permanece**. Por eso, el archivo vuelve a estar con tus datos.
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

### Tarea 7. Rolling update de la imagen sin perder datos

Simular una actualización de la app (cambio visual menor) y realizar un **rolling update** para confirmar que los datos permanecen intactos en el PVC.

#### Tarea 7.1

- **Paso 44.** En el archivo `app/public/index.html`, reemplaza todo el código existente por este nuevo.

  > **Notas**
  - Cambios de estilos en los títulos y botones (simula una nueva versión).
  - Cambio en el título de la aplicación.
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

  > **Nota.** Versionar imágenes permite **rollback** rápido si algo falla.
  {: .lab-note .info .compact}

  ```bash
  docker build -t agendapvc:1.1 .
  ```

- **Paso 46.** Ahora, etiqueta la nueva imagen creada.

  > **Notas**
  - Sustituye `TU_USUARIO/TU_REPOSITORIO` por el de tu cuenta, **puede ser el mismo que usaste en la práctica anterior**.
  - El comando no dará salida, a menos que haya un error.
  {: .lab-note .info .compact}

  ```bash
  docker tag agendapvc:1.1 TU_USUARIO/TU_REPOSITORIO:agendapvc-v1.1
  ```

- **Paso 47.** Sube la imagen a tu repositorio remoto.

  > **Nota.** Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la práctica anterior.
  {: .lab-note .info .compact}

  ```bash
  docker push TU_USUARIO/TU_REPOSITORIO:agendapvc-v1.1
  ```

- **Paso 48.** Abre el archivo `k8s/deployment.yaml` para ajustar a la nueva imagen.

  > **Notas**
  - En la **línea 24** del archivo, agrega al final `-v1.1`.
  - Quizás la línea pueda variar; si es así, identifícala.
  - Puedes apoyarte de la imagen. Asegúrate de no borrar nada más.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 49.** Ejecuta el manifiesto con la actualización de la imagen.

   ```bash
   kubectl apply -f k8s/deployment.yaml
   ```

- **Paso 50.** Levanta el servicio, abre la aplicación y verifica que **los contactos siguen** ahí, ahora con el **nuevo diseño**.

  > **Notas**
  - Recuerda abrir la URL que entrega **Minikube Service**.
  - Si todo salió bien, debes ver tus contactos.
  {: .lab-note .info .compact}

  ```bash
  minikube service agenda-svc -n agenda-pvc --url
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 51.** En la terminal que ocupa el proceso **Minikube Service**, ejecuta `CTRL + c` cuando termines de probar la aplicación.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza de recursos

Siempre es importante detener y eliminar recursos creados que no se usarán.

#### Tarea 8.1

- **Paso 52.** Ejecuta el siguiente comando que limpiará todo lo creado en el clúster.

  > **Nota.** Puede tardar unos segundos.
  {: .lab-note .info .compact}

  ```bash
  kubectl delete namespace agenda-pvc
  ```

- **Paso 53.** Verifica que esté limpio.

  > **Notas**
  - `service/kubernetes` no se borra, es parte del clúster.
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
