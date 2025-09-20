---
layout: lab
title: "Práctica 13: Exponer una aplicación con ClusterIP y DNS interno"
permalink: /capitulo13/lab13/
images_base: /labs/capitulo13/img
duration: "60 minutos"
objective:
  - Que domines cómo **exponer servicios solo dentro del clúster** usando **Service tipo ClusterIP** y cómo **resolverlos vía DNS interno**. Desplegarás una **API Node.js** y probarás su acceso desde **Pods clientes** usando el nombre DNS del Service (formas corta y FQDN), además de validar que **no es accesible externamente**. Todo con pasos detallados para que puedas seguirlos tal cual en tu entorno.
prerequisites:
  - Visual Studio Code
  - Docker Desktop en ejecución
  - Minikube y kubectl configurados
  - Terminal **Git Bash** dentro de VS Code
  - Conocimientos básicos de Node.js, Docker y Kubernetes
introduction:
  Un **Service** en Kubernetes agrupa Pods y les da una identidad **estable** (IP + DNS) aunque los Pods cambien. El tipo **ClusterIP** (por defecto) solo es accesible **dentro** del clúster. Kubernetes crea un **registro DNS** para cada Service. puedes consumirlo usando el **nombre corto** (mismo namespace) o el **FQDN** (`<service>.<namespace>.svc.cluster.local`). En esta práctica te enfocarás en ese **DNS interno** y la **comunicación inter-Pods**.
slug: lab13
lab_number: 13
final_result: >
  Dejaste una app corriendo detrás de un **ClusterIP** y probaste su **resolución DNS interna**, entendiendo cuándo usar **nombre corto** y **FQDN** y por qué **no se expone** hacia fuera. Con esto ya puedes diseñar **comunicación inter-servicios** segura dentro del clúster.
notes: 
  - Si quieres exponer al exterior, usa **NodePort**, **LoadBalancer** o **Ingress**.  
  - Para resolver DNS, el **CoreDNS** del clúster debe estar sano (`kubectl -n kube-system get pods -l k8s-app=kube-dns`).  
  - Si `nslookup` no está en tu imagen cliente, puedes usar `getent hosts` (en distros glibc) o utilizar imágenes como `ghcr.io/kubernetes-sigs/e2e-test-images/dnsutils`.  
  - Usa etiquetas coherentes en Deployments/Services (`app ...`) para no romper el selector del Service.
references:
  - text: Services
    url: https://kubernetes.io/docs/concepts/services-networking/service/
  - text: DNS en Kubernetes
    url: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
prev: /capitulo12/lab12/          
next: /capitulo14/lab14/
---


---

### Tarea 1: Preparar la estructura del proyecto

Vas a crear el esqueleto del proyecto y archivos base. Separarás código (`app/`) y manifiestos (`k8s/`) para trabajar ordenado.

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
  mkdir lab13-k8sclusteripdns && cd lab13-k8sclusteripdns
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  > **NOTA:** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Crea la estructura base de directorios y archivos vacíos.

  ```text
  lab13-k8sclusteripdns/
  ├── app/
  │   ├── package.json
  │   └── server.js
  ├── Dockerfile
  ├── k8s/
  │   ├── deployment.yaml
  │   └── service.yaml
  └── .dockerignore
  ```

- **Paso 9.** Ahora crea la carpeta **app/** y sus archivos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab13-k8sclusteripdns**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p app && touch app/package.json app/server.js
  ```

- **Paso 10.** Muy bien continua la creación del directorio **k8s/** con los manifiestos vacios.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab13-k8sclusteripdns**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/deployment.yaml k8s/service.yaml
  ```

- **Paso 11.** Crea los ultimos dos archivos del proyecto **.dockerignore** y **Dockerfile**.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab13-k8sclusteripdns**
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

### Tarea 2: Implementar una API Node.js muy simple

Crearás una API con Express con `/hello` y `/health`. Será el backend que expondrás con ClusterIP.

#### Tarea 2.1

- **Paso 14.** Abre y define las siguientes dependencias en el archivo `app/package.json`.

  ```json
  {
    "name": "clusterip-demo",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": { "start": "node server.js" },
    "dependencies": { "express": "^4.18.2" }
  }
  ```

- **Paso 15.** Copia y pega el siguiente codigo dentro del archivo `app/server.js`.

  > **NOTA:** La ruta `/health` la usarás como liveness/readiness; la ruta `/hello` será el objetivo de tus pruebas desde otros Pods.
  {: .lab-note .info .compact}

  ```javascript
  const express = require('express');
  const app = express();
  const PORT = process.env.PORT || 3000;

  app.get('/hello', (_req, res) => {
    res.json({ msg: 'Hola desde ClusterIP en Kubernetes!' });
  });

  app.get('/health', (_req, res) => res.json({ status: 'ok' }));

  app.listen(PORT, () => console.log(`Servidor en puerto ${PORT}`));
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Crear la imagen Docker dentro de Minikube

Construirás la imagen en el **daemon de Docker de Minikube** para usarla directamente desde el clúster sin subir a un registry externo.

#### Tarea 3.1

- **Paso 16.** Dentro del archivo `Dockerfile` en la raíz del proyecto agrega la siguiente definición de compilación.

  ```dockerfile
  FROM node:20-alpine
  WORKDIR /app
  COPY app/package*.json ./
  RUN npm install --production
  COPY app ./
  EXPOSE 3000
  CMD ["node", "server.js"]
  ```

- **Paso 17.** Recuerda encender siempre **minikube**, escribe el siguiente comando.

  > **NOTA:** Espera unos segundos en lo que termina de inicializar.
  {: .lab-note .info .compact}

  ```bash
  minikube start
  ```

- **Paso 18.** Apunta tu shell al daemon de Docker de Minikube y construye la imagen.

  > **NOTA:** El comando se ejecuta desde la raíz de la carpeta **lab13-k8sclusteripdns**
  {: .lab-note .info .compact}

  ```bash
  eval $(minikube docker-env)
  docker build -t clusterip-app:1.0 .
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 19.** Verifica que la imagen existe en el daemon de Minikube.

  > **NOTA:** Al usar el daemon de Minikube, los nodos del clúster ya "ven" la imagen sin necesidad de `docker push`.
  {: .lab-note .info .compact}

  ```bash
  docker images | grep clusterip-app
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Desplegar Deployment y Service ClusterIP
 
Crearás el **Deployment** (2 réplicas) y el **Service** de tipo **ClusterIP** para exponerlo **solo** dentro del clúster con DNS interno.

#### Tarea 4.1

- **Paso 20.** Abre el archivo `k8s/deployment.yaml` y agrega el siguiente codigo para el manifiesto.

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: clusterip-app-deploy
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: clusterip-app
    template:
      metadata:
        labels:
          app: clusterip-app
      spec:
        containers:
        - name: web
          image: clusterip-app:1.0
          ports:
            - containerPort: 3000
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
  ```

- **Paso 21.** Ejecuta la configuración del archivo deployment, escribe el siguiente comando.

   ```bash
   kubectl apply -f k8s/deployment.yaml
   ```

- **Paso 22.** Verifica que el deployment y los pods esten corriendo correctamente.

  > **NOTA:** Deberias observar `Running` y `READY 1/1`
  {: .lab-note .info .compact}

  ```bash
  kubectl get deploy,pod
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 23.** Define la siguiente configuración en el archivo `k8s/service.yaml`.

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: clusterip-app-svc
  spec:
    type: ClusterIP
    selector:
      app: clusterip-app
    ports:
      - port: 80         # Puerto lógico del Service dentro del clúster
        targetPort: 3000 # Puerto del contenedor
  ```

- **Paso 24.** Ejecuta la configuración del archivo service, escribe el siguiente comando.

   ```bash
   kubectl apply -f k8s/service.yaml
   ```
   
- **Paso 25.** Verifica que el service este corriendo correctamente.

  > **NOTA:** Debes ver Service tipo **ClusterIP** sin External-IP
  {: .lab-note .info .compact}

  ```bash
  kubectl get service
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 26.** Observa el detalle del Service.

  > **NOTA:**
  - Anota el valor de **ClusterIP** y que el puerto 80 se traducira a **3000 (TARGET PORT)**
  - Con **2 réplicas**, el Service balanceará tráfico entre Pods relacionados por la etiqueta `app: clusterip-app`.
  - El DNS interno te permitirá llamarlo por nombre.
  {: .lab-note .info .compact}

  ```bash
  kubectl get svc clusterip-app-svc -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Probar acceso interno por DNS (mismo namespace)

Lanzarás Pods **cliente** para consumir la API a través del **nombre DNS del Service**. Validarás **nombre corto** y **FQDN**.

#### Tarea 5.1

- **Paso 27.** Crea un Pod efímero para probar HTTP con `curl` (imagen liviana).

  ```bash
  kubectl run curl-client --rm -it --image=curlimages/curl:8.10.1 --restart=Never -- sh
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 28.** Dentro del Pod, ejecuta los siguientes comandos:

  > **NOTA:**
  - Forma corta (mismo namespace):
  - Si el Pod cliente está en el **mismo namespace** `default`, puede usar el **nombre corto**.
  {: .lab-note .info .compact}

  ```sh
  curl -s http://clusterip-app-svc/hello | sed 's/ /_/g'
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

  > **NOTA:**
  - Forma FQDN explícita:
  - El **FQDN** siempre funciona e incluye `namespace.svc.cluster.local`.
  {: .lab-note .info .compact}

  ```sh
  curl -s http://clusterip-app-svc.default.svc.cluster.local/hello | sed 's/ /_/g'
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

  > **NOTA:** Salir del pod
  {: .lab-note .info .compact}

  ```sh
  exit
  ```

- **Paso 29.** Ejecuta un Pod para pruebas de **DNS** con busybox.

  ```bash
  kubectl run dns-client --rm -it --image=busybox:1.36 --restart=Never -- sh
  ```
  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 30.** Dentro del Pod, ejecuta los siguientes comandos:

  > **NOTA:**
  - En el resultado la IP debe ser la misma que observaste en el detalle del service.
  - Es normal que veas los mensajes **server can't find...** ya que intenta buscar resolverlo de diferentes formas.
  {: .lab-note .info .compact}

  ```sh
  nslookup clusterip-app-svc
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

  > **NOTA:** Con el siguiente comando la traducción es mas limpia.
  {: .lab-note .info .compact}

  ```sh
  nslookup clusterip-app-svc.default.svc.cluster.local
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

  > **NOTA:** Salir del pod de prueba
  {: .lab-note .info .compact}

  ```sh
  exit
  ```
 
{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Probar acceso **desde otro namespace** (requiere FQDN)

Crearás un **segundo namespace** y un Pod cliente allí. Verás que el **nombre corto** ya no funciona y que debes usar el **FQDN**.

#### Tarea 6.1

- **Paso 31.** Crea un nuevo namespace para la prueba.

  ```bash
  kubectl create ns pruebas-dns
  kubectl get ns | grep pruebas-dns
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 32.** Lanza un Pod cliente en `pruebas-dns` y prueba las resoluciones.

  ```bash
  kubectl -n pruebas-dns run dns-cross --rm -it --image=busybox:1.36 --restart=Never -- sh
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 33.** Dentro del Pod, ejecuta el siguiente comando y observa el comportamiento:

  > **NOTA:** Esto puede fallar porque el nombre corto busca en el namespace actual:
  {: .lab-note .info .compact}

  ```sh
  nslookup clusterip-app-svc || echo "No resuelve por nombre corto"
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

- **Paso 34.** Prueba usando FQDN apuntando al namespace `default`

  ```sh
  nslookup clusterip-app-svc.default.svc.cluster.local
  ```

  ![micint]({{ page.images_base | relative_url }}/18.png)

  > **NOTA:** Salir del pod de prueba
  {: .lab-note .info .compact}

  ```sh
  exit
  ```

- **Paso 35.** Ahora prueba con HTTP y curl usando una imagen que soporta curl.

  > **NOTA:**
  - El pod se crea, resuelve y se elimina, pero debes ver el mensaje correctamente.
  - Fuera del namespace de origen, debes **definir** el nombre: `<service>.<namespace>.svc.cluster.local`.
  {: .lab-note .info .compact}

  ```sh
  kubectl -n pruebas-dns run curl-cross --rm -it --image=curlimages/curl:8.10.1 --restart=Never -- sh -lc \
    "curl -s http://clusterip-app-svc.default.svc.cluster.local/hello"
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Confirmar que **no** hay acceso externo (solo interno)

Probarás desde tu **host** que el Service **ClusterIP** no es accesible. Verás que no aparece en `minikube service list` y que **no tiene External-IP**.

#### Tarea 7.1

- **Paso 36.** Verifica el tipo y la IP del Service en el clúster.

  ```bash
  kubectl get svc clusterip-app-svc -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 37.** Lista servicios `exponibles` por Minikube.

  > **NOTA:** El `clusterip-app-svc` **NO** debe aparecer con URL, porque no es NodePort/LoadBalancer
  {: .lab-note .info .compact}

  ```bash
  minikube service list | sed -n '1,10p'
  ```
  
  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 38.** Intenta (fallará) usar la IP del nodo o la del Service desde tu host.

  > **NOTA:**
  - IP del Service (interna del clúster) — NO es accesible desde tu host
  - ClusterIP solo existe dentro de la red del clúster. Para exponer hacia fuera, usarías **NodePort**, **LoadBalancer** o **Ingress**.
  {: .lab-note .info .compact}

  ```bash
  SVC_IP=$(kubectl get svc clusterip-app-svc -o jsonpath='{.spec.clusterIP}')
  echo $SVC_IP
  curl -m 3 -s http://$SVC_IP/hello || echo "Sin acceso externo (esperado)"
  ```

  ![micint]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8: Limpieza de recursos

Eliminarás los objetos creados para dejar el clúster limpio.

#### Tarea 8.1

- **Paso 39.** Borra Pods efímeros (si alguno quedó) y namespace de pruebas.

  > **NOTA:** Estos pods fueron eliminados por la etiqueta `--rm`, pero por si quedaron, eliminalos:
  {: .lab-note .info .compact}

  ```bash
  kubectl delete pod curl-client dns-client --ignore-not-found=true
  kubectl -n pruebas-dns delete pod dns-cross curl-cross --ignore-not-found=true
  kubectl delete ns pruebas-dns --ignore-not-found=true
  ```

  ![micint]({{ page.images_base | relative_url }}/23.png)

- **Paso 40.** Elimina Deployment y Service de la practica.

  > **NOTA:** Es buena práctica limpiar tus recursos para evitar confusiones y consumo innecesario.
  {: .lab-note .info .compact}

  ```bash
  kubectl delete -f k8s/service.yaml -f k8s/deployment.yaml
  kubectl get all
  ```

  ![micint]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}