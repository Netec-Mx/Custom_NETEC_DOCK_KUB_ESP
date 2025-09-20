---
layout: lab
title: " Práctica 14: Despliegue de aplicación con Ingress en Kubernetes"
permalink: /capitulo14/lab14/
images_base: /labs/capitulo14/img
duration: "60 minutos"
objective:
  - Desplegar **dos servicios HTTP (app1 y app2)** detrás de un **único recurso Ingress** usando **NGINX Ingress Controller** en **Minikube**, enrutando por **rutas** (`/app1` y `/app2`). Construirás imágenes Docker locales, crearás Deployments y Services **ClusterIP**, habilitarás el **addon de Ingress** de Minikube, definirás un **Ingress** con *path-based routing* y validarás el acceso vía
prerequisites:
  - Visual Studio Code
  - Docker Desktop en ejecución
  - Minikube y kubectl configurados 
  - Terminal **Git Bash** dentro de VS Code
  - Conocimientos básicos de Node.js, Docker y Kubernetes
introduction:
 Un **Ingress** expone servicios HTTP/HTTPS externamente y permite **enrutamiento L7** (por host y/o por path). En Minikube, puedes usar `minikube addons enable ingress` para desplegar **NGINX Ingress Controller**. En esta práctica, publicarás **dos apps Node.js** detrás de **un solo Ingress**. todas las peticiones a `/app1` irán al **Service app1**, y las de `/app2` al **Service app2**. Así verás cómo consolidar múltiples servicios bajo un único punto de entrada.
slug: lab14
lab_number: 14
final_result: >
  Publicaste **dos servicios** detrás de **un único Ingress** usando **NGINX Ingress Controller** en Minikube, con **enrutamiento por rutas**. Aprendiste a construir imágenes locales, desplegar Deployments/Services, habilitar el controlador, definir reglas de Ingress, probar acceso por **navegador**, y realizar **rolling updates** sin afectar a otros backends.
notes: 
  - Si el Ingress no responde, verifica que los Pods en `ingress-nginx` estén `Running` y revisa `kubectl describe ingress` para errores de reglas.  
  - Puedes usar **cert-manager** para TLS en prácticas futuras.  
  - Si tu entorno **no** usa Minikube, instala NGINX Ingress Controller con Helm o manifiestos oficiales del proyecto.  
  - Asegúrate de que tus **selectors** de Service coinciden exactamente con las **labels** del Deployment.
references:
  - text: Ingress
    url: https://kubernetes.io/docs/concepts/services-networking/ingress/
  - text: Minikube addon
    url: https://minikube.sigs.k8s.io/docs/commands/addons/
  - text: Anotaciones de NGINX Ingress
    url: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
  - text: Services
    url: https://kubernetes.io/docs/concepts/services-networking/service/
  - text: Deployments
    url: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/ 
prev: /capitulo13/lab13/          
next: /capitulo15/lab15/
---


---

### Tarea 1: Estructura del proyecto

Prepararás carpetas y archivos de ambas aplicaciones, manifiestos de Kubernetes y el recurso Ingress.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos. 

- **Paso 2.** Abre el **`Visual Studio Code`** lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VSCode** da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/23.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/24.png)

- **Paso 5.** Asegurate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**:

  > **NOTA:** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica**.

  > **NOTA:** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab14-k8singress && cd lab14-k8singress
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Crea la estructura base de directorios y archivos vacíos.

  > **NOTA:** Separar `app1`/`app2` hace fácil evolucionar cada microservicio.
  {: .lab-note .info .compact}

  ```text
  lab14-k8singress/
  ├── app1/
  │   ├── package.json
  │   └── server.js
  ├── app2/
  │   ├── package.json
  │   └── server.js
  ├── Dockerfile.app1
  ├── Dockerfile.app2
  ├── k8s/
  │   ├── app1-deployment.yaml
  │   ├── app1-service.yaml
  │   ├── app2-deployment.yaml
  │   ├── app2-service.yaml
  │   └── ingress.yaml
  └── .dockerignore
  ```

- **Paso 9.** Ahora crea la carpeta **app1/** y sus archivos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab14-k8singress**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p app1 && touch app1/package.json app1/server.js
  ```

- **Paso 10.** Ahora crea la carpeta **app2/** y sus archivos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab14-k8singress**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p app2 && touch app2/package.json app2/server.js
  ```

- **Paso 11.** Muy bien continua la creación del directorio **k8s/** con los manifiestos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab14-k8singress**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/app1-deployment.yaml k8s/app1-service.yaml k8s/app2-deployment.yaml k8s/app2-service.yaml k8s/ingress.yaml
  ```

- **Paso 12.** Crea los ultimos archivos del proyecto.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab14-k8singress**
  {: .lab-note .info .compact}

  ```bash
  touch .dockerignore Dockerfile.app1 Dockerfile.app2
  ```

- **Paso 13.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias:

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

- **Paso 14.** Valida la creacion de la estructura de tu proyecto, escribe el siguiente comando.

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

### Tarea 2: Implementar app1 y app2 (Node.js)

Crearás dos APIs muy simples con rutas `/` (mensaje) y `/health` (probes). Se distinguirán por el contenido.

#### Tarea 2.1 (app1)

- **Paso 15.** Dentro del archivo `app1/package.json` agrega las siguientes dependencias:

  ```json
  {
    "name": "app1",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": { "start": "node server.js" },
    "dependencies": { "express": "^4.18.2" }
  }
  ```

- **Paso 16.** Ahora abre `app1/server.js` y agrega la logica de la API:

  > **NOTA:** 
  - **Servidor Express** en puerto configurable (`PORT` o 3000).  
  - **Ruta `/`**: responde con el texto `"Que la Fuerza te acompañe desde APP1"`.  
  - **Ruta `/health`**: devuelve JSON con estado de la app (`{ status: "ok", app: "app1" }`).  
  - **Inicio del servidor**: imprime en consola `APP1 en puerto ...`. 
  {: .lab-note .info .compact}

  ```javascript
  const express = require('express');
  const app = express();
  const PORT = process.env.PORT || 3000;
  app.get('/', (_req, res) => res.send('Que la Fuerza te acompañe desde APP1'));
  app.get('/health', (_req, res) => res.json({ status: 'ok', app: 'app1' }));
  app.listen(PORT, () => console.log(`APP1 en puerto ${PORT}`));
  ````

#### Tarea 2.2 (app2)

- **Paso 17.** Dentro del archivo `app2/package.json` agrega las siguientes dependencias:

  ```json
  {
    "name": "app2",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": { "start": "node server.js" },
    "dependencies": { "express": "^4.18.2" }
  }
  ```

- **Paso 18.** Ahora abre `app2/server.js` y agrega la logica de la API:

  > **NOTA:** 
  - **Servidor Express** en puerto configurable (`PORT` o 3000).  
  - **Ruta `/`**: responde con el texto `"Hasta el infinito y mas alla desde APP2"`.  
  - **Ruta `/health`**: devuelve JSON con estado de la app (`{ status: "ok", app: "app2" }`).  
  - **Inicio del servidor**: imprime en consola `APP2 en puerto ...`.  
  {: .lab-note .info .compact}

  ```javascript
  const express = require('express');
  const app = express();
  const PORT = process.env.PORT || 3000;
  app.get('/', (_req, res) => res.send('Hasta el infinito y mas alla desde APP2'));
  app.get('/health', (_req, res) => res.json({ status: 'ok', app: 'app2' }));
  app.listen(PORT, () => console.log(`APP2 en puerto ${PORT}`));
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Dockerfiles y build de imágenes en Minikube

Compilarás dos imágenes (una por app) usando el **daemon de Docker de Minikube**, evitando publicarlas.

#### Tarea 3.1

- **Paso 19.** Dentro del archivo `Dockerfile.app1` agrega el siguiente contenido:

  ```dockerfile
  FROM node:20-alpine
  WORKDIR /app
  COPY app1/package*.json ./
  RUN npm install --production
  COPY app1 ./
  EXPOSE 3000
  CMD ["node", "server.js"]
  ```

- **Paso 20.** Dentro del archivo `Dockerfile.app2` agrega el siguiente contenido:

  ```dockerfile
  FROM node:20-alpine
  WORKDIR /app
  COPY app2/package*.json ./
  RUN npm install --production
  COPY app2 ./
  EXPOSE 3000
  CMD ["node", "server.js"]
  ```
- **Paso 21.** Recuerda encender siempre **minikube**, escribe el siguiente comando.

  > **NOTA:** Espera unos segundos en lo que termina de inicializar.
  {: .lab-note .info .compact}

  ```bash
  minikube start
  ```

- **Paso 22.** Construir **app1** dentro de Minikube:

  > **NOTA:**
  - Al compilar en el daemon de Minikube, los nodos pueden "ver" las imágenes sin `push` a un registry.
  - El comando se ejecuta desde la raíz de la carpeta **lab14-k8singress**
  {: .lab-note .info .compact}

  ```bash
  eval $(minikube docker-env)
  docker build -t app1-demo:1.0 -f Dockerfile.app1 .
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 23.** Construir **app2** dentro de Minikube:

  > **NOTA:**
  - Al compilar en el daemon de Minikube, los nodos pueden "ver" las imágenes sin `push` a un registry.
  - El comando se ejecuta desde la raíz de la carpeta **lab14-k8singress**
  {: .lab-note .info .compact}

  ```bash
  eval $(minikube docker-env)
  docker build -t app2-demo:1.0 -f Dockerfile.app2 .
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

- **Paso 24.** Verifica que se hayan creado correctamente las imagenes, escribe el siguiente comando.

  > **NOTA:** Debes ver `app1-demo:1.0` y `app2-demo:1.0` en la lista.
  {: .lab-note .info .compact}

  ```bash
  docker images | egrep 'app[12]-demo'
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Deployment y Service (ClusterIP) para cada app

Crearás un Deployment **(réplicas=2)** y un Service **ClusterIP** por cada app. El Ingress los referenciará por nombre y puerto.

#### Tarea 4.1 (app1)

- **Paso 25.** Dentro del archivo `k8s/app1-deployment.yaml` agrega el siguiente codigo:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app1-deploy
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: app1
    template:
      metadata:
        labels:
          app: app1
      spec:
        containers:
        - name: web
          image: app1-demo:1.0
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet: { path: /health, port: 3000 }
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health, port: 3000 }
            initialDelaySeconds: 3
            periodSeconds: 5
  ```

- **Paso 26.** Ahora en el archivo `k8s/app1-service.yaml` define la exposicion de la aplicación:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: app1-svc
  spec:
    type: ClusterIP
    selector:
      app: app1
    ports:
      - port: 80
        targetPort: 3000
  ```

#### Tarea 4.2 (app2)

- **Paso 27.** En el archivo `k8s/app2-deployment.yaml` carga el codigo para desplegar los pods mediante el deployment:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app2-deploy
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: app2
    template:
      metadata:
        labels:
          app: app2
      spec:
        containers:
        - name: web
          image: app2-demo:1.0
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet: { path: /health, port: 3000 }
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health, port: 3000 }
            initialDelaySeconds: 3
            periodSeconds: 5
  ```

- **Paso 28.** Dentro del archivo `k8s/app2-service.yaml` agrega el codigo para el service:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: app2-svc
  spec:
    type: ClusterIP
    selector:
      app: app2
    ports:
      - port: 80
        targetPort: 3000
  ```

- **Paso 29.** Ahora aplica todos los objetos juntos **deployments** y **service**.

  > **NOTA:** Los **selectors** (`app: app1` / `app: app2`) conectan Services con sus Pods. El Ingress enviará tráfico a `app1-svc:80` y `app2-svc:80`.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply -f k8s/app1-deployment.yaml -f k8s/app1-service.yaml
  kubectl apply -f k8s/app2-deployment.yaml -f k8s/app2-service.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 30.** Ahora valida que todo se haya creado correctamente.

  ```bash
  kubectl get deploy,po,svc
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Habilitar NGINX Ingress Controller (Minikube)

Activarás el **addon** de Minikube que instala el **NGINX Ingress Controller** en el namespace `ingress-nginx`.

#### Tarea 5.1

- **Paso 31.** Habilitar addon e inspeccionar Pods del controlador:

  > **NOTA:** Debes esperar a que los Pods del controlador estén **Running** antes de crear el Ingress.
  {: .lab-note .info .compact}

  ```bash
  minikube addons enable ingress
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 32.** Ya que haya terminado de instalar el **addon** de ingrees, verifica que este listo para usarse.

  > **NOTA:**
  - Es normal ver los `ingress-nginx-admission...` 0/1, son Jobs de una sola vez que generan/parchean los certificados del admission webhook.
  - Lo importante es que el controller esté **Running 1/1**
  {: .lab-note .info .compact}

  ```bash
  kubectl -n ingress-nginx get pods
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 33.** Ahora obten la IP del nodo:

  > **NOTA:**
  - Usarás esta IP para probar el Ingress.
  - Alternativamente, puedes correr `minikube tunnel` y trabajar con LoadBalancer (no requerido aquí).
  - Guarda la **IP** en un Bloc de Notas.
  {: .lab-note .info .compact}

  ```bash
  minikube ip
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Crear el recurso Ingress con rutas `/app1` y `/app2`

Definirás un **Ingress** que enrute por **path** a los Services. Opcionalmente, asignarás un **host** de pruebas (`demo.local`).

#### Tarea 6.1

- **Paso 34.** Bien! ahora abre el archivo `k8s/ingress.yaml` y agrega el siguiente contenido.

  > **NOTA:** La anotación `rewrite-target: /` reescribe `/app1` a `/` en el backend (igual para `/app2`).
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: apps-ingress
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    ingressClassName: nginx
    rules:
      - host: demo.local
        http:
          paths:
            - path: /app1
              pathType: Prefix
              backend:
                service:
                  name: app1-svc
                  port:
                    number: 80
            - path: /app2
              pathType: Prefix
              backend:
                service:
                  name: app2-svc
                  port:
                    number: 80
  ```

- **Paso 35.** Aplicar el manifiesto del objeto ingress:

  ```bash
  kubectl apply -f k8s/ingress.yaml
  ```

- **Paso 36.** Obten los valores de la configuración para saber si esta correctamente implementado

  ```bash
  kubectl get ingress
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 37.** Obeten los detalles del ingress implementado.

  ```bash
  kubectl describe ingress apps-ingress | sed -n '1,120p'
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 38.** Primero redireccionaremos el ingress mediante minikube.

  ```bash
  kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8080:80
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 39.** Ahora abre otra terminal de **GitBash** para probar las rutas usando las siguientes URLs, ejecuta los 2 comandos al mismo tiempo en la terminal:

  ```bash
  curl -s -H "Host: demo.local" http://127.0.0.1:8080/app1
  ```

  ```bash
  curl -s -H "Host: demo.local" http://127.0.0.1:8080/app2
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

- **Paso 40.** Agregar una entrada local al archivo HOST del Sistema Operativo para usar navegador sin la necesidad de llamar Host header manual (CLI), escribe el siguiente comando:

  > **NOTA:**
  - `C:\Windows\System32\drivers\etc\hosts` (Windows) y agrega una línea para definir el host personalizado:
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Recuerda estar en la segunda terminal
  {: .lab-note .important .compact}

  ```
  echo "127.0.0.1 demo.local" | sed 's/\r//' | tee -a /c/Windows/System32/drivers/etc/hosts | tail -n 1
  ```

- **Paso 41.** Luego abre en el navegador y pega cada URL en una pestaña diferente:

  > **NOTA:**
  - El **Ingress** usa *host-based + path-based routing*. Si no quieres usar un host, puedes omitir el `host` y hacer `curl` al **Service NodePort** del controller (cuando aplique) o seguir usando `-H "Host: ..."`
  {: .lab-note .info .compact}

  - **Que la Fuerza te acompañe desde APP1**

  ```bash
  http://demo.local:8080/app1
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

  - **Hasta el infinito y mas alla desde APP2**

  ```bash
  http://demo.local:8080/app2
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Actualización de una app y verificación de enrutamiento

Harás un pequeño cambio en `app1`, reconstruirás la imagen y observarás un **rolling update** sin afectar `app2`.

#### Tarea 7.1

- **Paso 42.** Edita `app1/server.js` para cambiar el mensaje, solo ajusta el texto de la **linea 4** por el que esta abajo:

  > **NOTA:** Recuerda que seguimos en la segunda terminal GitBash que abriste.
  {: .lab-note .info .compact}

  ```txt
  Un mago(APP1) nunca llega tarde, ni pronto, llega exactamente cuando se lo propone
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

- **Paso 43.** Reconstruye la imagen.

  > **NOTA:** El comando se ejecuta desde el directorio **lab14-k8singress**
  {: .lab-note .info .compact}

  ```bash
  cd lab14-k8singress
  eval $(minikube docker-env)
  docker build -t app1-demo:1.1 -f Dockerfile.app1 .
  ```

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 44.** Actualiza el manifiesto Deployment:

  ```bash
  kubectl set image deployment/app1-deploy web=app1-demo:1.1
  kubectl rollout status deployment/app1-deploy
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png)

- **Paso 45.** Prueba las rutas nuevamente.

  > **NOTA:** Si dejaste abiertas las paginas en el navegador solo actualizalas
  {: .lab-note .info .compact}

  - **Un mago(APP1) nunca llega tarde, ni pronto, llega exactamente cuando se lo propone**

  ```bash
  http://demo.local:8080/app1
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png)

  - **Hasta el infinito y mas alla desde APP2**

  ```bash
  http://demo.local:8080/app2
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 46.** Regresa a la teminal que ocupa el proceso **minikube ingress** ejecuta `CTRL + c` cuando termines de probar la aplicación.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8: Limpieza

Dejarás el clúster sin objetos del laboratorio.

#### Tarea 8.1

- **Paso 47.** Eliminar Ingress, Services y Deployments:

  > **NOTA:** Puedes ejecutar todos los comandos al mismo tiempo
  {: .lab-note .info .compact}

  ```bash
  kubectl delete -f k8s/ingress.yaml
  kubectl delete -f k8s/app1-service.yaml -f k8s/app1-deployment.yaml
  kubectl delete -f k8s/app2-service.yaml -f k8s/app2-deployment.yaml
  kubectl get all,ingress
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 48.** Deshabilitar addon de ingress:

  ```bash
  minikube addons disable ingress
  ```

  ![micint]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}