---
layout: lab
title: "Práctica 8: Despliegue de una aplicación Node.js. en Kubernetes"
permalink: /capitulo8/lab8/
images_base: /labs/capitulo8/img
duration: "60 minutos"
objective:
  - Desplegar en **Kubernetes (Minikube)** una aplicación **Node.js + Express** que incluye una **interfaz web en tiempo real** usando **Socket.IO** para visualizar un contador de clics. Aprenderás a construir la imagen, cargarla en Minikube, y crear los objetos Kubernetes necesarios (**ConfigMap**, **Deployment** con probes y variables de entorno, y **Service NodePort**). Validarás el acceso desde el navegador y comandos `kubectl`.
prerequisites:
  - Visual Studio Code
  - Docker Desktop en ejecución
  - Minikube instalado
  - Tener `kubectl` instalado.
  - Terminal Git Bash dentro de VS Code
  - Conocimientos básicos de Docker
introduction:
  Kubernetes permite definir **despliegues declarativos** (Deployment) y exponerlos con **Services**. En esta práctica, además de la API simple, añadimos una **UI** con websockets (Socket.IO) que refleja en tiempo real el número de clics de todos los usuarios conectados. Para hacer la configuración más flexible, usaremos un **ConfigMap** con el **título de la aplicación**.
slug: lab8
lab_number: 8
final_result: >
  Una aplicación **Node.js con UI en tiempo real** (Socket.IO) desplegada en **Minikube**, parametrizada con **ConfigMap**, con **Deployment** (probes y escalable) y **Service NodePort**. Acceso desde navegador, validación con `kubectl` y comprensión del efecto de múltiples réplicas sobre el estado en memoria.
notes: 
  - En producción usarías **Ingress** (con `minikube addons enable ingress`) y **TLS**.
  - Para estado compartido entre Pods, integra un almacén externo (Redis/BD) o un Pub/Sub.
  - Versiona los manifiestos en Git y automatiza `kubectl apply` con CI/CD.
  - Si tu Minikube no enruta al azar entre Pods, puedes simularlo refrescando o usando `kubectl port-forward` a distintos Pods.
references:
  - text: Deployments
    url: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
  - text: Services
    url: https://kubernetes.io/docs/concepts/services-networking/service/
  - text: Minikube
    url: https://minikube.sigs.k8s.io/docs/
  - text: Socket.IO
    url: https://socket.io/
prev: /capitulo7/lab7/          
next: /capitulo9/lab9/
---


---

### Tarea 1: Crear la estructura del proyecto

Organizar el código, la configuración de Kubernetes y el Dockerfile para empaquetar la app con su UI.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`** lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VSCode** da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/17.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 5.** Asegurate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**:

  > **NOTA:** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica**:

  > **NOTA:** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab8-k8snodeapp && cd lab8-k8snodeapp
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Crearás la siguiente estructura inicial del proyecto de la aplicación:

  > **NOTA:**  
  - `public/` contiene la UI estática que Express servirá.
  - `k8s/` aloja los manifiestos YAML.
  {: .lab-note .info .compact}

  ```text
  lab8-k8snodeapp/
  ├── api/
  │   ├── package.json
  │   ├── server.js
  │   └── public/
  │       └── index.html
  ├── Dockerfile
  ├── k8s/
  │   ├── configmap.yaml
  │   ├── deployment.yaml
  │   └── service.yaml
  └── .dockerignore
  ```

- **Paso 9.** Ahora crea la carpeta **api/** y sus archivos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab8-k8snodeapp**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p api api/public && touch api/package.json api/server.js api/public/index.html
  ```

- **Paso 10.** Muy bien continua la creación del directorio **k8s/** con los manifiestos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab8-k8snodeapp**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/configmap.yaml k8s/deployment.yaml k8s/service.yaml
  ```

- **Paso 11.** Crea los ultimos archivos del proyecto **.dockerignore** y **Dockerfile**

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab8-k8snodeapp**.
  {: .lab-note .info .compact}

  ```bash
  touch .dockerignore Dockerfile
  ```

- **Paso 12.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias:

  > **NOTA:** Evita copiar artefactos innecesarios hacia la imagen, manteniéndola ligera.
  {: .lab-note .info .compact}

  ```gitignore
  # evita copiar node_modules del host (causa del ELF inválido)
  api/node_modules
  **/node_modules

  # basura
  npm-debug.log
  Dockerfile
  docker-compose.yml
  .git
  .gitignore
  .DS_Store
  ```

- **Paso 13.** Valida la creación de la estructura de tu proyecto, escribe el siguiente comando.

  > **NOTA:** Recuerda que tambien puedes visualizarlos en el explorador de archivos de VSCode.
  {: .lab-note .info .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Implementar la aplicación con interfaz y Socket.IO

Crear una app Express que sirva una UI estática con Socket.IO. La UI muestra un **contador global** que se incrementa al hacer clic y se sincroniza en tiempo real en todos los clientes.

#### Tarea 2.1

- **Paso 14.** Abre el archivo `api/package.json` agrega las sieguientes dependencias para la aplicación:

  > **NOTA:** `socket.io` simplifica la comunicación bidireccional en tiempo real vía WebSocket/fallbacks.
  {: .lab-note .info .compact}

  ```json
  {
    "name": "k8s-app-ui",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": {
      "start": "node server.js"
    },
    "dependencies": {
      "express": "^4.18.2",
      "socket.io": "^4.7.5"
    }
  }
  ```

- **Paso 15.** Abre el archivo `api/server.js` y agrega la siguiente logica para la aplicacion:

  > **NOTA:** El contador es **por Pod** (memoria local). Con 2 réplicas, cada Pod tendrá su propio contador (verás valores distintos si tus requests llegan a diferentes Pods).
  {: .lab-note .info .compact}

  ```javascript
  const express = require('express');
  const http = require('http');
  const path = require('path');
  const { Server } = require('socket.io');

  const app = express();
  const server = http.createServer(app);
  const io = new Server(server);

  // Título configurable por ConfigMap (variable de entorno)
  const APP_TITLE = process.env.APP_TITLE || 'Mini Dashboard en Kubernetes';

  const PORT = process.env.PORT || 3000;

  // Servir contenido estático (UI)
  app.use(express.static(path.join(__dirname, 'public')));

  // Endpoint simple para health checks
  app.get('/health', (_req, res) => res.json({ status: 'ok' }));

  // Endpoint para leer el título (útil para validar la variable de entorno desde UI)
  app.get('/config', (_req, res) => res.json({ title: APP_TITLE }));

  // Estado en memoria (efímero por pod)
  let clicks = 0;

  io.on('connection', (socket) => {
    // Al conectar un cliente, enviar estado actual
    socket.emit('state', { clicks });

    // Escuchar evento 'add_click' desde UI
    socket.on('add_click', () => {
      clicks += 1;
      // Emitir a todos los conectados
      io.emit('state', { clicks });
    });

    // Diagnóstico opcional
    socket.on('disconnect', () => {
      // console.log('Cliente desconectado');
    });
  });

  server.listen(PORT, () => {
    console.log(`Servidor escuchando en http://localhost:${PORT} - Título: ${APP_TITLE}`);
  });
  ```

- **Paso 16.** Abre el archivo `api/public/index.html` agrega el siguiente codigo que sera la interfaz grafica de ejemplo:

  > **NOTA:** Cada clic emite un evento al servidor; el servidor actualiza el contador y lo **difunde** a todos los clientes conectados a **ese Pod**.
  {: .lab-note .info .compact}

  ```html
  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mini Dashboard</title>
    <style>
      body { font-family: system-ui, sans-serif; max-width: 720px; margin: 2rem auto; }
      .card { border: 1px solid #e5e7eb; border-radius: 12px; padding: 1rem; box-shadow: 0 1px 2px rgba(0,0,0,.05); }
      h1 { margin: 0 0 .5rem 0; }
      button { padding: .6rem 1rem; border-radius: 10px; border: 1px solid #e5e7eb; cursor: pointer; }
      .row { display: flex; align-items: center; gap: 1rem; }
      .muted { color: #6b7280; font-size: .9rem; }
    </style>
  </head>
  <body>
    <div class="card">
      <h1 id="title">Mini Dashboard</h1>
      <p class="muted">Este título viene de <code>APP_TITLE</code> en un ConfigMap.</p>

      <div class="row">
        <button id="btn">Sumar clic</button>
        <div>Clics (por Pod): <strong id="count">0</strong></div>
      </div>

      <p class="muted" id="pod-hint"></p>
    </div>

    <script src="/socket.io/socket.io.js"></script>
    <script>
      // Actualizar título desde el endpoint /config (inyectado por ConfigMap)
      fetch('/config').then(r => r.json()).then(cfg => {
        document.getElementById('title').textContent = cfg.title || 'Mini Dashboard';
      });

      const socket = io();
      const btn = document.getElementById('btn');
      const count = document.getElementById('count');
      const hint = document.getElementById('pod-hint');

      socket.on('connect', () => {
        hint.textContent = 'Conectado al servidor WebSocket.';
      });

      socket.on('state', ({ clicks }) => {
        count.textContent = clicks;
      });

      btn.addEventListener('click', () => {
        socket.emit('add_click');
      });
    </script>
  </body>
  </html>
  ```

- **Paso 17.** Es importante siempre probar en un entorno local para identificar cualquier problema a tiempo. Ejecuta el siguiente comando para la prueba.

  > **NOTA:** Este comando se ejecuta desde el directorio **lab8-k8snodeapp**
  {: .lab-note .info .compact}

  ```bash
  cd api && npm install && node server.js
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 18.** Abre el navegador **Google Chrome** y verifica cada una de las siguientes URLs

  > **NOTA:** En un navegador (Da clics): 
  {: .lab-note .info .compact}  
  
  ```bash
  http://localhost:3000
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

  - Endpoint de salud (Abre otra pestaña):

  ```bash
  http://localhost:3000/health
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)
  
  - Endpoint de título (Abre otra pestaña):
  
  ```bash
  http://localhost:3000/config
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 19.** En la terminal donde esta activa la aplicación de **NodeJs** rompe el proceso con `CTRL + c`, y escribe el siguiente comando.

  > **NOTA:**
  - Debes regresar al directorio **lab8-k8snodeapp**
  - Tambien limpia los archivos de la prueba local.
  - Tarda unos segundos en limpiar.
  {: .lab-note .info .compact} 


  ```bash
  cd ..
  clear
  rm -r api/node_modules
  rm api/package-lock.json
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Crear Dockerfile e imagen y cargarla en Minikube

Empaquetar la app y construir la imagen dentro del demonio Docker de Minikube para que el clúster pueda usarla.

#### Tarea 3.1

- **Paso 20.** Abre el archivo `Dockerfile` que esta en la raíz y agrega el siguiente codigo:

  > **NOTA:** Imagen minimal basada en Alpine, suficiente para nuestra practica.
  {: .lab-note .info .compact}

  ```dockerfile
  FROM node:20-alpine

  WORKDIR /app
  COPY api/package*.json ./
  RUN npm install --production

  COPY api ./

  ENV PORT=3000
  EXPOSE 3000

  CMD ["node", "server.js"]
  ```

- **Paso 21.** Antes de aplicar el manifiesto, primero necesitamos encender el nodo de **Minikube**, escribe el siguiente comando.

  > **NOTA:** Espera unos minutos en lo que se levanta.
  {: .lab-note .info .compact}

  ```bash
  minikube start
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 22.** Apuntar Docker al demonio de Minikube y construir imagen:

  > **NOTA:**
  - Construir dentro del daemon de Minikube evita tener que subir la imagen a un registry.
  - La imagen `k8s-node-ui:1.0` debe listarse en `docker images`.
  {: .lab-note .info .compact}


  ```bash
  eval $(minikube docker-env)
  docker build -t k8s-node-ui:1.0 .
  docker images | grep k8s-node-ui
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Crear ConfigMap para el título de la app

El título de la UI se inyectará como variable de entorno `APP_TITLE` mediante un ConfigMap.

#### Tarea 4.1

- **Paso 23.** Abre el archivo `k8s/configmap.yaml` y agrega las siguiente etiqueta:

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: node-ui-config
  data:
    APP_TITLE: "Dashboard de Clics en Tiempo Real"
  ```

- **Paso 24.** Verificamos que minikibe haya encendido bien, escribe el siguiente comando

  > **NOTA:** Verifica que los nodos esten funcionando.
  {: .lab-note .info .compact}

  ```bash
  kubectl get nodes
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 25.** Ahora si, aplica el manifiesto del archivo **configmap**.

  > **NOTA:**
  - El comando se ejecuta desde el directorio **lab8-k8snodeapp**
  - Aplica y valida que se haya configurado correctamente.
  - Separar configuración del código permite cambiar títulos/slogans sin reconstruir la imagen.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply -f k8s/configmap.yaml
  kubectl get configmap node-ui-config -o yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Deployment con 2 réplicas y variables desde ConfigMap

Crear un Deployment con **2 réplicas**, **liveness/readiness probes** y variable `APP_TITLE` inyectada desde el ConfigMap.

#### Tarea 5.1

- **Paso 26.** Abre el archivo `k8s/deployment.yaml` y define la configuracion que implementara los pods:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: node-ui-deployment
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: node-ui
    template:
      metadata:
        labels:
          app: node-ui
      spec:
        containers:
        - name: node-ui
          image: k8s-node-ui:1.0
          ports:
          - containerPort: 3000
          env:
          - name: APP_TITLE
            valueFrom:
              configMapKeyRef:
                name: node-ui-config
                key: APP_TITLE
  ```

- **Paso 27.** Aplica el manifiesto y verifica que este listo cuando los pods terminen de crearse.

  > **NOTA:**
  - En caso de que tengas un error **ImagePullBackOff** usa este comando para cargar la imagen a minikube `minikube image load k8s-node-ui:1.0`
  - Con 2 réplicas, cada Pod mantiene su propio contador en memoria; esto evidencia el concepto de **stateful vs stateless** y la necesidad de almacenes compartidos si quisiéramos un contador global.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply -f k8s/deployment.yaml
  kubectl rollout status deployment/node-ui-deployment
  kubectl get pods -l app=node-ui -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Service NodePort para exponer la UI

Crear un Service tipo **NodePort** que expone el puerto 3000 de los Pods en el puerto 30080 del nodo Minikube.

#### Tarea 6.1

- **Paso 28.** Abre el archivo `k8s/service.yaml` agrega el siguiente contenido para exponer la aplicacion:

  > **NOTA:** La propiedad `NodePort` hace accesible el servicio desde fuera del clúster usando la IP del nodo y un puerto alto.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: node-ui-service
  spec:
    type: NodePort
    selector:
      app: node-ui
    ports:
      - port: 3000
        targetPort: 3000
        nodePort: 30080
  ```

- **Paso 29.** Ejecuta los siguientes comandos sobre el archivo service:

  - Aplica

  ```bash
  kubectl apply -f k8s/service.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

  - Verifica
  
  ```bash
  kubectl get svc node-ui-service
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

  - Activa

  ```bash
  minikube service node-ui-service --url
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 30.** Abre tu navegador de **Google Chrome** para validar la aplicacion:

- **Paso 31.** Ahora el comando `minikube service node-ui-service --url` que se quedo activo en la terminal devolvio una **URL** parecida al siguiente comando.

  > **NOTA:**
  - Cambia las letas **`x`** por el numero de puerto que te asigno.
  - Copia y pega la URL en tu navegador.
  - Da clics, es normal que tarde unos segundos en lo que los pods reciben la información.
  {: .lab-note .info .compact}

  > **IMPORTANTE:**
  - Dale unos minutos si el contador de clics no muestra inmediatamente los numeros.
  - Actualiza la pagina y vuelve a dar clics.
  {: .lab-note .important .compact}

  ```bash
  http://127.0.0.1:xxxx
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 32.** Regresa a la terminal donde esta el proceso de **minikbe service** y ejecuta **`CTRL + c`** para poder usar la terminal.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza de recursos

Siempre es importante detener y eliminar recursos creados que no se usaran.

#### Tarea 7.1

- **Paso 33.** Ejecuta el siguiente comando que limpiara todo lo creado en el cluster.

  ```bash
  kubectl delete all --all
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 34.** Verifica que haya quedado limpio.

  > **NOTA:** La propiedad `service/kubernetes` no se borra, es parte del cluster.
  {: .lab-note .info .compact}

  ```bash
  kubectl get all
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)
 
{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}