---
layout: lab
title: "Práctica 7. Despliegue de servicios en Docker Swarm"
permalink: /capitulo7/lab7/
images_base: /labs/capitulo7/img
duration: "60 minutos"
objective:
  - Poner en marcha un **clúster Docker Swarm (mononodo)** y desplegar un **servicio web Node.js** con **réplicas, balanceo de carga (ingress)**, **descubrimiento DNS interno** y **actualizaciones continuas (rolling updates)**. También usarás **secrets** de Swarm, **redes overlay**, **escalado** y harás validaciones exhaustivas de estado y conectividad.
prerequisites:
  - Visual Studio Code
  - Docker Desktop (o Docker Engine) en ejecución
  - Terminal **Git Bash** dentro de VS Code
  - Conocimientos básicos de Docker (imágenes, contenedores) y Node.js
introduction: |
  **Docker Swarm** es el orquestador nativo de Docker. Te permite crear un **clúster** (Swarm) de nodos y desplegar **servicios** con varias réplicas, red **overlay** y **balanceo de carga** mediante el **routing mesh**. A diferencia de `docker run`, los **servicios** de Swarm agregan capacidades de **escalado**, **actualizaciones controladas**, **healthchecks** y **secrets** de manera declarativa y sencilla.
slug: lab7
lab_number: 7
final_result: >
  Has desplegado un **servicio en Docker Swarm** con réplicas, **balanceo de carga**, **secrets**, **red overlay**, **rolling updates** y **escalado**, validando cada aspecto con comandos y salidas esperadas. Esto te da una base sólida para operar servicios de forma declarativa y confiable en Swarm.
notes: 
  - En Swarm **multinodo**, las imágenes deben estar disponibles para todos los nodos (usa un **registry**).  
  - Para producción, define **políticas de actualización** y **healthchecks** apropiados, y monitoreo de **logs/metrics**.  
  - Si tu host no resuelve `localhost` para el published port, usa la IP de Docker Desktop/VM.  
  - Usar `docker stack deploy` con un `compose v3` es otra vía para **definir varios servicios**; aquí usamos `docker service create/update/scale` para comprender las bases.
references:
  - text: Docker Swarm mode overview
    url: https://docs.docker.com/engine/swarm
  - text: Deploy services in Swarm
    url: https://docs.docker.com/engine/swarm/services/
  - text: Routing mesh & publish ports
    url: https://docs.docker.com/engine/swarm/ingress/
  - text: Secrets
    url: https://docs.docker.com/engine/swarm/secrets/
  - text: Healthcheck
    url: https://docs.docker.com/engine/reference/builder/#healthcheck
prev: /capitulo6/lab6          
next: /capitulo8/lab8/
---


---

### Tarea 1. Crear la estructura del proyecto

Organizar el proyecto en carpetas separando código de app, configuración y manifiestos o recursos auxiliares.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`**. Lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VS Code**, da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**. Da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 5.** Asegúrate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VS Code**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/3.png)

- **Paso 6.** Crea el directorio para trabajar en la **práctica**.

  > **Nota.** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab7-dockerswarm && cd lab7-dockerswarm
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VS Code que se haya creado el directorio.

  > **Nota.** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 8.** Esta será la estructura de los directorios de la aplicación.

  > **Notas**  
  - `app/` contiene la aplicación. 
  - `secrets/` guardará el contenido que Swarm inyectará como **secret** al contenedor. 
  {: .lab-note .info .compact}

  ```text
  lab7-dockerswarm/
  ├── app/
  │   ├── package.json
  │   └── server.js
  ├── Dockerfile
  ├── secrets/
  │   └── banner.txt
  ├── .dockerignore
  ```

- **Paso 9.** Ahora, crea la carpeta **app/** y sus archivos vacíos.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab7-dockerswarm**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p app && touch app/package.json app/server.js
  ```

- **Paso 10.** Muy bien. Continúa la creación del directorio **secrets/** y el archivo `banner.txt`.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab7-dockerswarm**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p secrets && touch secrets/banner.txt
  ```

- **Paso 11.** Crea los últimos dos archivos del proyecto **.dockerignore** y **Dockerfile**.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab7-dockerswarm**.
  {: .lab-note .info .compact}

  ```bash
  touch .dockerignore Dockerfile
  ```

- **Paso 12.** Agrega el siguiente contenido al archivo **.dockerignore** para construir imágenes limpias.

  > **Nota.** Reduce el **contexto de build** y acelera las compilaciones, evitando subir archivos innecesarios al daemon de Docker.
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

  > **Nota.** Recuerda que también puedes visualizarlos en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Implementar la aplicación Node.js con lectura de Secret

Harás una API mínima con Express que responde en `/` y `/health`. Incluirás un **secret** (`banner`) que se incrusta en la respuesta para demostrar el uso de **Docker Swarm Secrets**.

#### Tarea 2.1

- **Paso 14.** Dentro del archivo `app/package.json` agrega las siguientes dependencias.

  > **Nota.** Dependencias mínimas para un servidor **HTTP** con Express.
  {: .lab-note .info .compact}

  ```json
  {
    "name": "swarm-web",
    "version": "1.0.0",
    "main": "server.js",
    "scripts": { "start": "node server.js" },
    "dependencies": { "express": "^4.18.2" }
  }
  ```

- **Paso 15.** Ahora, en el archivo `app/server.js`, inserta la siguiente lógica de la aplicación.

  > **Nota.** El endpoint `/` te mostrará el **hostname** del contenedor (para visualizar el balanceo entre réplicas) y el **banner** (secret).
  {: .lab-note .info .compact}
  > **Importante.** Revisa que no haya errores de sintaxis en VS Code.
  {: .lab-note .important .compact}
  
  ```javascript
  const fs = require('fs');
  const os = require('os');
  const express = require('express');
  const app = express();
  const PORT = process.env.PORT || 3000;
  const VERSION = process.env.APP_VERSION || '1.0';
  const SECRET_PATH = '/run/secrets/banner'; // Ruta estándar donde Swarm monta el secret

  function readBanner() {
    try {
      return fs.readFileSync(SECRET_PATH, 'utf8').trim();
    } catch (e) {
      return process.env.SECRET_BANNER || 'Bienvenido (sin secret)';
    }
  }

  app.get('/health', (_req, res) => res.json({ status: 'ok', version: VERSION }));

  app.get('/', (_req, res) => {
    const banner = readBanner();
    res.json({
      msg: 'Hola desde Docker Swarm!',
      version: VERSION,
      host: os.hostname(),
      banner
    });
  });

  app.listen(PORT, () => {
    console.log(`Servidor escuchando en puerto ${PORT} (v${VERSION})`);
  });
  ```

- **Paso 16.** En el archivo `secrets/banner.txt`.

  > **Notas**
  - Puedes dejar cualquier mensaje de notificación. **Como ejemplo, el mensaje de abajo.**
  - Este archivo se cargará como **secret** en el servicio Swarm y montado en `/run/secrets/banner`.
  {: .lab-note .info .compact}

  ```text
  ✨ Banner Swarm: Modo Producción (APP) ✨
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}  

---

### Tarea 3. Dockerfile con Healthcheck y usuario no root

Definirás un Dockerfile optimizado con **usuario no root** y **HEALTHCHECK** para integrarlo con Swarm.

#### Tarea 3.1

- **Paso 17.** Ahora, dentro del archivo `Dockerfile` en la raíz del directorio **lab7-...**, agrega el siguiente contenido para compilar la imagen.

  > **Nota.** El `HEALTHCHECK` permite a Swarm detectar contenedores no saludables para reiniciarlos. `USER node` reduce superficie de riesgo.
  {: .lab-note .info .compact}

  ```dockerfile
  FROM node:20-alpine

  # Instala curl para HEALTHCHECK
  RUN apk add --no-cache curl

  WORKDIR /app
  COPY app/package*.json ./
  RUN npm install --production
  COPY app ./

  ENV PORT=3000
  ENV APP_VERSION=1.0

  # Usar el usuario "node" provisto por la imagen base
  RUN chown -R node:node /app
  USER node

  EXPOSE 3000

  HEALTHCHECK --interval=10s --timeout=2s --start-period=5s --retries=3 \
    CMD curl -fsS http://localhost:3000/health || exit 1

  CMD ["node", "server.js"]
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Construir imagen local

Compilarás la imagen del servicio web para usarla en Swarm.

#### Tarea 4.1

- **Paso 18.** Compila la imagen, escribe el siguiente comando.

  > **Nota.** Recuerda que el comando se ejecuta en raíz de la carpeta **lab7...**
  {: .lab-note .info .compact}  

  ```bash
  docker build -t swarm-web .
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 19.** Verifica que la imagen existe. Escribe el siguiente comando en la terminal.

  > **Nota.** Debes ver `swarm-web` con la etiqueta `latest`.
  {: .lab-note .info .compact}

  > **Importante.** En un entorno multinodo real, deberías **pushear** (subir) la imagen a un registry accesible por todos los nodos. En este laboratorio, usarás un **Swarm mononodo**, por lo que la imagen local es suficiente.
  {: .lab-note .important .compact}

  ```bash
  docker images | grep swarm-web
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Inicializar Docker Swarm y crear red overlay
  
Levantar el **Swarm** y crear una **red overlay** adjuntable que usarán tus servicios.

#### Tarea 5.1

- **Paso 20.** Inicializa el Swarm (mononodo).

  > **Nota.** Recuerda que el comando se ejecuta en la raíz de la carpeta **lab7...**
  {: .lab-note .info .compact}    

  ```bash
  docker swarm init
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 21.** Valida que **Swarm** se haya inicializado correctamente. Escribe el comando en la terminal.

  > **Nota.** Debe devolver: **active**.
  {: .lab-note .info .compact}

  {% raw %}
  ```bash
  docker info --format '{{.Swarm.LocalNodeState}}'
  ```
  {% endraw %}

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 22.** Ahora, verifica que el nodo esté listo. Escribe el siguiente comando.

  > **Nota.** Debes ver tu nodo con **Leader** y **Ready**.
  {: .lab-note .info .compact}

  > **Importante.** Swarm **mononodo** es suficiente para practicar. El nodo actual será **manager**.
  {: .lab-note .important .compact}

  ```bash
  docker node ls
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)  

- **Paso 23.** Crea la red overlay adjuntable.

  > **Nota.** Las **redes overlay** permiten la comunicación entre tareas de servicios en diferentes nodos (o en el mismo, como aquí) y el **DNS interno** de Swarm resuelve por nombre de servicio.
  {: .lab-note .info .compact}

  {% raw %}
  ```bash
  docker network create --driver overlay --attachable appnet
  ```
  {% endraw %}

  ![micint]({{ page.images_base | relative_url }}/11.png)  

- **Paso 24.** Verifica que la red se haya creado correctamente, escribe el comando en la terminal.

  ```bash
  docker network ls | grep appnet
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)  

- **Paso 25.** Muy bien. Con el siguiente comando verifica que la red haya sido asociada.

  > **Nota.** Debe ser overlay / `attachable=true`
  {: .lab-note .info .compact}

  {% raw %}
  ```bash
  docker network inspect appnet --format '{{.Driver}} / attachable={{.Attachable}}'
  ```
  {% endraw %}

  ![micint]({{ page.images_base | relative_url }}/13.png) 
  
{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Crear Secret y desplegar servicio con réplicas
 
Subirás un **secret** a Swarm y crearás el servicio `web` con **tres réplicas**, publicando el puerto **8080 a 3000** mediante **ingress**.

#### Tarea 6.1

- **Paso 26** Crear el **secret** en el clúster de Swarm que apuntará al archivo **banner.txt**

  ```bash
  docker secret create banner ./secrets/banner.txt
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png) 

- **Paso 27** Verifica que el secreto esté creado correctamente.

  > **Nota.** Debe listar `banner`
  {: .lab-note .info .compact}

  > **Importante.** Swarm guarda el secret cifrado a nivel de raft; los contenedores lo verán montado en `/run/secrets/banner`.
  {: .lab-note .important .compact}

  ```bash
  docker secret ls
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png) 

- **Paso 28.** Crea el servicio `web` con tres réplicas. Escribe el siguiente comando en la terminal.

  ```bash
  docker service create \
    --name web \
    --replicas 3 \
    --publish published=8080,target=3000,mode=ingress \
    --network appnet \
    --secret source=banner,target=banner \
    swarm-web
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 29.** Verifica que las réplicas se hayan creado correctamente. Escribe el siguiente.

  ```bash
  docker service ls
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

- **Paso 30.** Ahora, revisa los tres procesos individuales. Ejecuta el siguiente comando.

  ```bash
  docker service ps web
  ```

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 31.** Revisa también los detalles del **servicio swarm**.

  > **Notas**
  - `--publish ... mode=ingress` activa el **routing mesh**: cualquier nodo publicaría el puerto y distribuiría al backend.
  - `--secret` monta el archivo como **solo lectura** dentro del contenedor.
  {: .lab-note .info .compact}

  ```bash
  docker service inspect web --pretty
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png)

- **Paso 32.** Para probar el balanceo vía puerto publicado de host a servicio, escribe el siguiente comando.

  > **Nota.** El **hostname** cambia entre réplicas y evidencia de **balanceo de carga**.
  {: .lab-note .info .compact}

  > **Importante.** Observa el campo `host:`. Debe alternar entre réplicas.
  {: .lab-note .important .compact}

  ```bash
  for i in {1..5}; do curl -s http://localhost:8080/ | jq .; done
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 33.** Para probar la resolución **DNS interna** (cliente en la red overlay), escribe el siguiente comando.

  > **Nota.** Dentro de la red `appnet`, el **nombre del servicio** `web` resuelve las tareas mediante el DNS de Swarm.
  {: .lab-note .info .compact}

  ```bash
  docker run --rm -it --network appnet curlimages/curl:8.10.1 sh -lc \
    'for i in 1 2 3 4 5; do curl -s http://web:3000/; echo; done'
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Rolling update de la imagen (latest(1.0) a 1.1) con control de actualización

Simular una nueva versión de la app y realizar un **rolling update** con **paralelismo** y **delay** para minimizar riesgos.

#### Tarea 7.1

- **Paso 34.** Abre y cambia la versión en el archivo `Dockerfile`.

  > **Nota.** El cambio se encuentra aproximadamente en la **línea 12** del archivo **Dockerfile**. 
  {: .lab-note .info .compact}

  ```dockerfile
  ENV APP_VERSION=1.1
  ```

  ![micint]({{ page.images_base | relative_url }}/22.png)

- **Paso 35.** Ahora, cambia el mensaje de la respuesta en el archivo `server.js` (el texto de `msg`).

  > **Nota.** Puedes usar un mensaje personalizado o colocar el del ejemplo. **Línea 22** del archivo, aproximadamente.
  {: .lab-note .info .compact}

    > **Importante.** Verifica la indentación del archivo después de modificar la línea del mensaje.
  {: .lab-note .important .compact}

  ```dockerfile
      msg: '¡Disfruta de las pequeñas cosas!'
  ```

  ![micint]({{ page.images_base | relative_url }}/23.png)

- **Paso 36.** Ahora, reconstruye la imagen para reflejar los cambios de la aplicación.

  > **Nota.** El comando se ejecuta en la raíz del directorio **lab7-...**
  {: .lab-note .info .compact}

  ```bash
  docker build --no-cache -t swarm-web:1.1 .
  ```

- **Paso 37.** Valida que la imagen exista con la nueva versión.

  ```bash
  docker images | grep swarm-web
  ```

  ![micint]({{ page.images_base | relative_url }}/24.png)

- **Paso 38.** Actualiza el servicio con el comando **update** .

  > **Notas** 
  - `--update-parallelism 1`: actualiza **una réplica a la vez**.  
  - `--update-delay 5s`: espera cinco segundos entre réplicas.  
  - `--update-failure-action pause`: si algo falla, **pausa** la actualización.
  {: .lab-note .info .compact}

  ```bash
  docker service update \
    --image swarm-web:1.1 \
    --update-parallelism 1 \
    --update-delay 5s \
    --update-monitor 10s \
    --update-failure-action pause \
    web
  ```

  ![micint]({{ page.images_base | relative_url }}/25.png)

- **Paso 39.** Verifica que la versión nueva esté atendiendo el tráfico.

  > **Nota.** Debes ver `1.1` (una vez finalizada la actualización) y el nuevo `mensaje` que cambiaste.
  {: .lab-note .info .compact}

  ```bash
  for i in {1..6}; do curl -s http://localhost:8080/ | jq ; done
  ```

  ![micint]({{ page.images_base | relative_url }}/26.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Escalar el servicio y observar la distribución

Aumentar el número de réplicas para manejar mayor carga y verificar la distribución de tareas.

#### Tarea 8.1

- **Paso 40.** Escribe el siguiente comando para escalar fácilmente a cinco réplicas.

  > **Nota.** Recuerda que ya tenías **tres réplicas** solo se agregarán **dos réplicas** para el total de **cinco réplicas**.
  {: .lab-note .info .compact}

  ```bash
  docker service scale web=5
  ```

  ![micint]({{ page.images_base | relative_url }}/27.png)

- **Paso 41.** Revisa el estado y prueba el tráfico ahora con las réplicas escaladas.

  > **Nota.** En un Swarm multinodo, verías tareas distribuidas en varios nodos. Aquí, todas residirán en el mismo (mononodo), pero el **balanceo** sigue activo.
  {: .lab-note .info .compact}

  ```bash
  docker service ps web
  ```

  ```bash
  for i in {1..8}; do curl -s http://localhost:8080/ | jq .host; done
  ```

  ![micint]({{ page.images_base | relative_url }}/28.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Diagnóstico y logs
  
Revisar eventos del servicio y logs de tareas para entender el ciclo de vida y facilitar el troubleshooting.

#### Tarea 9.1

- **Paso 42.** Inspecciona el detalle del servicio aplicado, después de la actualización.

  ```bash
  docker service inspect web | jq '.[0].Spec.UpdateConfig'
  ```

  ![micint]({{ page.images_base | relative_url }}/29.png)

- **Paso 43.** Escribe los siguientes comandos para que puedas ver los logs de una tarea concreta de swarm asociada al Container ID.

  {% raw %}
  ```bash
  TASK_ID=$(docker service ps web --filter desired-state=running -q | head -n 1)
  CONTAINER_ID=$(docker inspect $TASK_ID --format '{{.Status.ContainerStatus.ContainerID}}')
  echo "Task: $TASK_ID"
  echo "Container: $CONTAINER_ID"
  docker logs -f $CONTAINER_ID
  ```
  {% endraw %}

  ![micint]({{ page.images_base | relative_url }}/30.png)

- **Paso 44.** La terminal se quedará ocupada con el comando de los logs. Ejecuta **`CTRL + c`** para romper el proceso.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 10. Limpieza de recursos

Eliminar servicios, secretos y redes; además, salir del Swarm para dejar el host limpio.

#### Tarea 10.1

- **Paso 45.** Elimina el servicio y el secret.

  ```bash
  docker service rm web
  docker secret rm banner
  ```

  ![micint]({{ page.images_base | relative_url }}/31.png)

- **Paso 46.** Elimina el red overlay.

  ```bash
  docker network rm appnet
  ```

- **Paso 47.** Sal del Swarm.

  ```bash
  docker swarm leave --force
  ```

  ![micint]({{ page.images_base | relative_url }}/32.png)

- **Paso 48.** Confirma la limpieza.

  > **Nota.** Cerrar y limpiar te deja listo para próximas prácticas sin interferencias.
  {: .lab-note .info .compact}

  {% raw %}
  ```bash
  docker service ls
  docker secret ls
  docker network ls | grep appnet || echo "appnet eliminada"
  docker info --format '{{.Swarm.LocalNodeState}}'
  ```
  {% endraw %}

  ![micint]({{ page.images_base | relative_url }}/33.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[9] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
