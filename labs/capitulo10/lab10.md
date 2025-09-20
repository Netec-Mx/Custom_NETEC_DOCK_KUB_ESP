---
layout: lab
title: "Práctica 10: Uso de ConfigMaps, Secrets y controladores con despliegue de una app"
permalink: /capitulo10/lab10/
images_base: /labs/capitulo10/img
duration: "60 minutos"
objective:
  - Aplicar **ConfigMaps**, **Secrets** y **controladores de Kubernetes** (Deployment, Service, Job y CronJob) para desplegar una aplicación **Node.js** en **Minikube**, administrando configuración no sensible, credenciales sensibles, tareas *one-shot* y tareas programadas. Validarás que la app consume valores desde un **ConfigMap montado como archivo** y variables de entorno provenientes de un **Secret**, y que los controladores se comportan según lo esperado.
prerequisites:
  - Visual Studio Code
  - Docker Desktop en ejecución
  - Minikube y kubectl configurados
  - Terminal **Git Bash** dentro de VS Code
  - Conocimientos básicos de Node.js, Docker y Kubernetes
  - Cuenta en Docker Hub
introduction:
  En Kubernetes, separar **código** de **configuración** y **credenciales** es una práctica esencial. Los **ConfigMaps** almacenan configuración **no sensible** (por ejemplo *feature flags*, títulos, endpoints), mientras que los **Secrets** guardan **datos sensibles** (tokens, claves, contraseñas) codificados en **Base64**. Los **controladores** como **Deployment**, **Job** y **CronJob** permiten mantener réplicas estables, ejecutar tareas únicas o periódicas. En este laboratorio montarás una app que lee un archivo `config.json` desde un ConfigMap y un `SECRET_TOKEN` desde un Secret, y además crearás un Job y un CronJob de validación.
slug: lab10
lab_number: 10
final_result: >
  Un despliegue completo en **Kubernetes con Minikube** que utiliza **ConfigMaps** (montados como archivo), **Secrets** (como variables), y **controladores** (Deployment, Service, Job, CronJob). Validaste que la app consume configuración y secretos, y automatizaste chequeos *one-shot* y periódicos.
notes: 
  - Es importante que **Nunca** imprimas secretos en logs ni los devuelvas en endpoints en producción. Aquí solo se muestra la **longitud** como guía didáctica.
  - ConfigMaps montados como volumen pueden tardar en reflejar cambios; reiniciar el Deployment es práctico y claro para prácticas.
  - Considera **RBAC**, **ResourceQuota** y **LimitRange** (vistos en la práctica 8) para endurecer entornos.
  - Para producción, expón la app con **Ingress + TLS** en lugar de NodePort.
references:
  - text: ConfigMaps
    url: https://kubernetes.io/docs/concepts/configuration/configmap/
  - text: Secrets
    url: https://kubernetes.io/docs/concepts/configuration/secret/
  - text: Deployments
    url: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
  - text: Jobs y CronJobs
    url: https://kubernetes.io/docs/concepts/workloads/controllers/job/
  - text: Services
    url: https://kubernetes.io/docs/concepts/services-networking/service/
  - text: Minikube
    url: https://minikube.sigs.k8s.io/docs/
prev: /capitulo9/lab9/          
next: /capitulo11/lab11/
---


---

### Tarea 1: Estructura del proyecto

Crear el esqueleto del proyecto, separando el código de la app, los manifiestos de Kubernetes y los archivos de configuración.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`** lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VSCode** da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/25.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/26.png)

- **Paso 5.** Asegurate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**:

  > **NOTA:** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica**.

  > **NOTA:** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab10-k8sobjetos && cd lab10-k8sobjetos
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Esta sera la estructura de los directorios de la aplicación:

  > **NOTA:**  
  - `k8s/` agrupa todos los manifiestos.  
  - `app/` contiene el código e imagen. 
  {: .lab-note .info .compact}

  ```text
  lab10-k8sobjetos/
  ├── app/
  │   ├── package.json
  │   ├── server.js
  │   └── Dockerfile
  ├── k8s/
  │   ├── configmap.yaml
  │   ├── secret.yaml
  │   ├── deployment.yaml
  │   ├── service.yaml
  │   ├── job-init.yaml
  │   └── cronjob-health.yaml
  └── .dockerignore
  ```

- **Paso 9.** Ahora crea la carpeta **app/** y sus archivos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab10-k8sobjetos**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p app && touch app/package.json app/server.js app/Dockerfile
  ```

- **Paso 10.** Muy bien continua la creación del directorio **k8s/** con los manifiestos vacios.

  > **NOTA:**  El comando se ejecuta desde la raíz de la carpeta **lab10-k8sobjetos**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/configmap.yaml k8s/secret.yaml k8s/deployment.yaml k8s/service.yaml k8s/job-init.yaml k8s/cronjob-health.yaml
  ```

- **Paso 11.** Crea el ultimo archivo del proyecto **.dockerignore**

  > **NOTA:**  El comando se ejecuta desde la raíz de la carpeta **lab10-k8sobjetos**
  {: .lab-note .info .compact}

  ```bash
  touch .dockerignore
  ```

- **Paso 12.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias:

  > **NOTA:** Evita copiar artefactos innecesarios hacia la imagen, manteniéndola ligera.
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

### Tarea 2: Implementar la aplicación Node.js

Crear una app Express con tres endpoints: `/` (mensaje con configuración), `/config` (devuelve config y secret) y `/health` (para probes). La app leerá `config.json` (montado desde ConfigMap) y `SECRET_TOKEN` (desde Secret).

#### Tarea 2.1

- **Paso 14.** Abre el archivo `app/package.json` y agrega las siguientes dependencias:

  > **NOTA:** Dependencia mínima para servir HTTP.
  {: .lab-note .info .compact}

  ```json
  {
    "name": "cfg-secrets-demo",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": { "start": "node server.js" },
    "dependencies": {
      "express": "^4.18.2"
    }
  }
  ```

- **Paso 15.** Dentro de `app/server.js` agrega el codigo de ejemplo:

  > **NOTA:** La app lee un **archivo** de configuración para mostrar cómo montar ConfigMaps como volumen.
  {: .lab-note .info .compact}

  ```javascript
  const fs = require('fs');
  const path = require('path');
  const express = require('express');
  const app = express();

  const PORT = process.env.PORT || 3000;
  const CONFIG_PATH = process.env.CONFIG_PATH || '/config/config.json';
  const SECRET_TOKEN = process.env.SECRET_TOKEN || 'no-token';

  function readConfig() {
    try {
      const raw = fs.readFileSync(path.resolve(CONFIG_PATH), 'utf8');
      return JSON.parse(raw);
    } catch (err) {
      return { error: 'No se pudo leer config.json', detail: err.message };
    }
  }

  app.get('/', (_req, res) => {
    const cfg = readConfig();
    const name = (cfg && cfg.appName) || 'App';
    res.send(`Bienvenido a ${name} - FeatureFlag: ${cfg?.featureFlag ?? 'N/A'}`);
  });

  app.get('/config', (_req, res) => {
    const cfg = readConfig();
    // Nunca regreses secretos reales en producción; esto es solo para fines didácticos
    res.json({ config: cfg, secretTokenLength: SECRET_TOKEN.length });
  });

  app.get('/health', (_req, res) => res.json({ status: 'ok' }));

  app.listen(PORT, () => {
    console.log(`Servidor escuchando en puerto ${PORT}`);
  });
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Dockerfile, compilación y carga de la imagen

Construir la imagen usando el daemon Docker de Minikube para no requerir Docker Hub.

#### Tarea 3.1

- **Paso 16.** Abre el archivo `app/Dockerfile` para definir las instrucciones que compilaran la imagen:

  > **NOTA:** Multi-stage pequeño para evitar reinstalar dependencias dos veces.
  {: .lab-note .info .compact}

  ```dockerfile
  FROM node:20-alpine AS deps
  WORKDIR /app
  COPY package*.json ./
  RUN npm install --production

  FROM node:20-alpine
  WORKDIR /app
  COPY --from=deps /app/node_modules ./node_modules
  COPY . .
  ENV PORT=3000
  EXPOSE 3000
  CMD ["node", "server.js"]
  ```

- **Paso 17.** Recuerda que debes estar autenticado a tu cuenta de **Docker Hub**.

  - Si ya tienes cuenda da clic <a href="https://login.docker.com/u/login/identifier?state=hKFo2SB1QXpUSzVVc3ZDYTAzQzlkWlFoYk9LWnlLZ1VOMzNnU6Fur3VuaXZlcnNhbC1sb2dpbqN0aWTZIGx0SFpVWDNQTzNPaFZlT1FxVDlWZUpzdWUya09FaWtjo2NpZNkgbHZlOUdHbDhKdFNVcm5lUTFFVnVDMGxiakhkaTluYjk" target="_blank" rel="noopener noreferrer"><strong>AQUÍ - Iniciar Sesión</strong></a>.
  - Si no tienes cuenta da clic <a href="https://app.docker.com/signup?_gl=1*1ugpfey*_gcl_au*OTcyNTkxNjkyLjE3NTc2MDY4MTU.*_ga*MTQxMjc1NjI4My4xNzU3NjA2ODE1*_ga_XJWPQMJYHQ*czE3NTc2MTE1NTQkbzIkZzEkdDE3NTc2MTE1NTQkajYwJGwwJGgw" target="_blank" rel="noopener noreferrer"><strong>AQUÍ - Crear cuenta</strong></a>.

- **Paso 18.** Ahora regresa a la terminal de **VSCode** y escribe el siguiente comando para autenticar la terminal a **Docker Hub**.

  > **NOTA:** Sigue los pasos de la terminal para autenticarte.
  {: .lab-note .info .compact}

  ```bash
  docker login
  ```

- **Paso 19.** Compila la imagen, escribe el siguiente comando:

  > **NOTA:**
  - El comando se ejecuta dentro del directorio **lab10-k8sobjetos/**.
  - El **docker build...** se ejecuta desde **app/** 
  - Ajusta las rutas si es necesario
  {: .lab-note .info .compact}

  ```bash
  cd app
  docker build -t cfg-secrets-demo .
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 20.** Ahora etiqueta la imagen y subela al repositorio remoto en **Docker Hub**

  > **NOTA:**
  - Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, **puede ser el mismo que usaste en la practica anterior**
  - El comando no dara salida a menos que haya un error.
  {: .lab-note .info .compact}

  ```bash
  docker tag cfg-secrets-demo TU_USUARIO/TU_REPOSITORIO:cfg-secrets-demo
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

- **Paso 21.** Si todo sale bien, el siguiente comando subira la imagen al repositorio remoto:

  > **NOTA:** Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la practica anterior
  {: .lab-note .info .compact}

  ```bash
  docker push TU_USUARIO/TU_REPOSITORIO:cfg-secrets-demo
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Definir ConfigMap (archivo) y Secret (env)

Crear un **ConfigMap** con `config.json` (montado como archivo) y un **Secret** con `SECRET_TOKEN` consumido como variable de entorno.

#### Tarea 4.1 Configmap

- **Paso 22.** Dentro del archivo `k8s/configmap.yaml` agrega la siguiente referencia del archivo **config.json**:

  > **NOTA:** El `|` permite incluir contenido multilínea.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-config
  data:
    config.json: |
      {
        "appName": "Demo ConfigMaps & Secrets",
        "featureFlag": true
      }
  ```

#### Tarea 4.2 Secret

- **Paso 23.** Abre el archivo `k8s/secret.yaml` (valores en texto plano y K8s los codifica en base64) y agrega el siguiente codigo:

  > **NOTA:** La propiedad `stringData` simplifica escribir sin base64; el API server lo convierte a `data` (base64).
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: app-secrets
  type: Opaque
  stringData:
    SECRET_TOKEN: "super-secreto-123"
  ```

- **Paso 24.** Ahora aplica los manifiestos **Configmap y Secret**.

  > **IMPORTANTE:** El comando **`cd ..`** te regresara un nivel de directorio, para que puedas aplicar los manifiestos.
  {: .lab-note .important .compact}

  ```bash
  cd ..
  kubectl apply -f k8s/configmap.yaml
  kubectl apply -f k8s/secret.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 25.** Verifica la creación correspondiente de cada objeto

  - Verifica **Configmap**

  ```bash
  kubectl get configmap app-config -o yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

  - Verifica **Secrets**

  ```bash
  kubectl get secret app-secrets -o yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Deployment (controlador) que consume ConfigMap y Secret

Crear un **Deployment** con 2 réplicas que monte el ConfigMap como **archivo** y exponga el Secret como **variable de entorno**. Añadir **probes** y **recursos**.

#### Tarea 5.1

- **Paso 26.** Abre el archivo `k8s/deployment.yaml` y agrega la configuración del deployment:

  > **NOTA:**
  - Sustituye **`TU_USUARIO/TU_REPOSITORIO`** por el de tu cuenta, puede ser el mismo que usaste en la practica anterior.
  - El cambio que debes realizar esta en la **linea 19** aproximadamente.
  - Montamos **solo** `config.json` en `/config`; usamos Secret vía env.
  {: .lab-note .info .compact}


  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: cfg-secrets-deploy
    labels:
      app: cfg-secrets
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: cfg-secrets
    template:
      metadata:
        labels:
          app: cfg-secrets
      spec:
        containers:
        - name: web
          image: TU_USUARIO/TU_REPOSITORIO:cfg-secrets-demo # CAMBIA AQUI LA CUENTA/REPOSITORIO DE DOCKER HUB
          ports:
          - containerPort: 3000
          env:
          - name: SECRET_TOKEN
            valueFrom:
              secretKeyRef:
                name: app-secrets
                key: SECRET_TOKEN
          - name: CONFIG_PATH
            value: "/config/config.json"
          volumeMounts:
          - name: cfg-volume
            mountPath: /config
            readOnly: true
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
        volumes:
        - name: cfg-volume
          configMap:
            name: app-config
            items:
            - key: config.json
              path: config.json
  ```

- **Paso 27.** Ahora aplica el manifiesto, escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/deployment.yaml
  kubectl rollout status deployment/cfg-secrets-deploy
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 28.** Verifica que esten implementados correctamente.

  ```bash
  kubectl get pods -l app=cfg-secrets -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 29.** Inspeccionar un Pod con los detalles del configmap y secrets:

  > **NOTA:**
  - Verifica que el archivo exista y la variable de entorno esté presente.
  - Guarda el ID de un pod en la variable **POD**.
  {: .lab-note .info .compact}

  ```bash
  POD=$(kubectl get pods -l app=cfg-secrets -o jsonpath='{.items[0].metadata.name}')
  ```

- **Paso 30.** Manda un comando `exec` para ver el config map dentro del Pod.

  ```bash
  kubectl exec -it "$POD" -- sh -lc 'ls -l /config && cat /config/config.json'
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 31.** Manda otro comando `exec` para validar la longitud de los caracteres del secreto.

  ```bash
  kubectl exec -it "$POD" -- sh -lc 'echo "TOKEN_LEN=${#SECRET_TOKEN}"'
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Service (controlador) para exponer la app

Crear un **Service NodePort** para exponer la app desde el host hacia el clúster.

#### Tarea 6.1

- **Paso 32.** Abre el archivo `k8s/service.yaml` y agrega el codigo para exponer la aplicación mediante este service:

  > **NOTA:** **NodePort** asigna un puerto alto del nodo (Minikube) para acceso externo.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: cfg-secrets-svc
  spec:
    type: NodePort
    selector:
      app: cfg-secrets
    ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30082
  ```

- **Paso 33.** Ejecuta el siguiente comando para aplicar el objeto service.

  ```bash
  kubectl apply -f k8s/service.yaml
  ```

- **Paso 34.** Escribe el comando que levantara el service mediante minikube.

  ```bash
  minikube service cfg-secrets-svc --url
  ```

- **Paso 35.** Usa la URL localhost con el puerto dinamico y pruebala en el navegador Google Chrome.

  > **NOTA:** Existosamente obtenemos el configmap con las etiquetas.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/14.png)

- **Paso 36.** En la teminal que ocupa el proceso **minikube service** ejecuta `CTRL + c` cuando termines de probar la aplicación.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Job (controlador) para validación *one-shot*

Crear un **Job** que ejecute una vez una verificación de salud llamando a `/health` del servicio. Útil para **smoke tests** post-despliegue.

#### Tarea 7.1

- **Paso 37.** Abre el archivo `k8s/job-init.yaml` y agrega el siguiente ejemplo de tarea unica de ejecución:

  > **NOTA:** Dentro del clúster, el **DNS** del servicio es `cfg-secrets-svc` (ClusterIP).
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: job-smoke-health
  spec:
    template:
      spec:
        restartPolicy: Never
        containers:
        - name: curl
          image: curlimages/curl:8.10.1
          args: ["sh", "-lc", "curl -fsS http://cfg-secrets-svc:3000/health && echo OK"]
    backoffLimit: 1
  ```

- **Paso 38.** Aplica el manifiesto de tipo **Job**, escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/job-init.yaml
  ```

- **Paso 39.** Verifica que se haya creado correctamente.

  > **NOTA:** Si la primera ejecución marca **COMPLETITIONS 0/0** espera unos segundos y vuelve a probar debe salir **1/1**
  {: .lab-note .info .compact}

  ```bash
  kubectl get jobs
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 40.** Finalmente verifica que se haya ejecutado el job correctamente observando los logs del job.

  ```bash
  POD=$(kubectl get pods --selector=job-name=job-smoke-health -o jsonpath='{.items[0].metadata.name}')
  kubectl logs "$POD"
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8: CronJob (controlador) para chequeos periódicos

Crear un **CronJob** que cada minuto consulte `/config` y registre el tamaño del token (sin imprimir el secreto) para monitoreo simple.

#### Tarea 8.1

- **Paso 41.** Abre el archivo `k8s/cronjob-health.yaml` y agrega la configuración del cronjob:

  > **NOTA:** El CronJob usa la **DNS interna** para alcanzar el Service.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: batch/v1
  kind: CronJob
  metadata:
    name: cj-config-check
  spec:
    schedule: "*/1 * * * *"
    successfulJobsHistoryLimit: 2
    failedJobsHistoryLimit: 2
    jobTemplate:
      spec:
        template:
          spec:
            restartPolicy: OnFailure
            containers:
            - name: curl
              image: curlimages/curl:8.10.1
              command: ["/bin/sh","-lc"]
              args:
                - |
                  echo "[Cron] $(date) consultando /config";
                  curl -fsS http://cfg-secrets-svc:3000/config
  ```

- **Paso 42.** Aplica el objeto tipo CronJob, escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/cronjob-health.yaml
  ```

- **Paso 43.** Verifica la creación correcta del objeto.

  > **NOTA:** Espera **~1 min** y consulta los jobs/pods generados:
  {: .lab-note .info .compact}

  ```bash
  kubectl get cronjobs
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

- **Paso 44.** Tambien puedes usar el siguiente comando que mantiene la verificación activa, sin la necesidad de re-ejecutar el comando.

  > **NOTA:** Para salir del **watch** ejecuta `CTRL + c`
  {: .lab-note .info .compact}

  ```bash
  kubectl get jobs --watch
  ```

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 45.** Usa el siguiente comando para verificar el contenido del unltimo job creaado.

  ```bash
  LAST_JOB=$(kubectl get jobs --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
  echo "Ultimo Job" = $LAST_JOB
  kubectl logs -l job-name=$LAST_JOB --tail=-1
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9: Actualización de ConfigMap y *rolling restart*

Modificar el ConfigMap y forzar un **rolling restart** del Deployment para que los Pods lean el contenido actualizado (por diseño, los env variables no cambian; los volúmenes de ConfigMap pueden reflejar cambios con retraso, pero se recomienda reiniciar).

#### Tarea 9.1

- **Paso 46.** Edita el archivo `k8s/configmap.yaml` y sustituye el siguiente código:

  > **NOTA:** Sustituye todo el bloque de la **linea 8 a la linea 10** cuida mucho la identación, YAML es muy sensible a los espacios.
  {: .lab-note .info .compact}

  ```yaml
      {
        "appName": "Demo CM/Secrets - v2",
        "featureFlag": false
      }
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 47.** Aplica la nueva configuración en el manifiesto y reinicia el deployment (***rolling***):

  > **NOTA:** Sin ***restart***, los Pods pueden no leer cambios inmediatamente.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply -f k8s/configmap.yaml
  kubectl rollout restart deployment/cfg-secrets-deploy
  kubectl rollout status deployment/cfg-secrets-deploy
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 48.** Escribe el comando que levantara el service mediante minikube.

  ```bash
  minikube service cfg-secrets-svc --url
  ```

- **Paso 49.** Actualiza la URL en el navegador Google Chrome o abre la URL localhost con el puerto dinamico y pruebala .

  > **NOTA:**
  - Existosamente obtenemos el configmap con las etiquetas.
  - Ahora observa que viene el cambio **V2** y **false**.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/22.png)

- **Paso 50.** En la teminal que ocupa el proceso **minikube service** ejecuta `CTRL + c` cuando termines de probar la aplicación.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 10: Limpieza de recursos

Siempre es importante detener y eliminar recursos creados que no se usaran.

#### Tarea 10.1

- **Paso 51.** Ejecuta el siguiente comando que limpiara todo lo creado en el cluster.

  ```bash
  kubectl delete all --all
  ```

  ![micint]({{ page.images_base | relative_url }}/23.png)

- **Paso 52.** Verifica que este limpio.

  > **NOTA:** La propiedad `service/kubernetes` no se borra es parte del cluster.
  {: .lab-note .info .compact}

  ```bash
  kubectl get all
  ```

  ![micint]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[9] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}