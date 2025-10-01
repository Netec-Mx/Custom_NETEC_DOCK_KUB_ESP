---
layout: lab
title: "Práctica 3. Construcción de una imagen Docker multietapa"
permalink: /capitulo3/lab3/
images_base: /labs/capitulo3/img
duration: "60 minutos"
objective:
  - El objetivo de esta práctica es que el participante aprenda a construir imágenes Docker optimizadas utilizando la técnica de **Multi-Stage Builds**, comparando el tamaño de la imagen con y sin multietapa. Usaremos como ejemplo una aplicación Node.js con Express que sirva un frontend estático y una API REST.
prerequisites:
  - Visual Studio Code instalado
  - Docker Desktop instalado y en ejecución
  - Git Bash configurado como terminal por defecto en VS Code
  - Conocimientos básicos de Node.js y Docker
  - Haber completado la Práctica 1
introduction:
  - Las **imágenes multietapa** permiten separar el proceso de construcción (instalación de dependencias y compilación de frontend) del de ejecución. Con esto se reduce el tamaño de la imagen final y se evitan dependencias innecesarias en producción. En esta práctica construiremos una aplicación Node.js con un backend REST y un frontend estático en React simulado con archivos preconstruidos, para demostrar el impacto de los Multi-Stage Builds.
slug: lab3
lab_number: 3
final_result: >
  El estudiante habrá construido una aplicación Node.js contenerizada con un **Dockerfile multietapa**, obteniendo una imagen ligera y optimizada, validando que la aplicación funciona correctamente y que el tamaño de la imagen se reduce respecto a una construcción tradicional.
notes: 
  - Multi-Stage es aplicable a múltiples lenguajes (Java, Go, .NET).
  - Siempre validar la funcionalidad de la aplicación antes de optimizar.
  - Para builds grandes, se recomienda usar `--target` para depuración de etapas intermedias.
references:
  - text: Docke Multi-stage builds
    url: https://docs.docker.com/develop/develop-images/multistage-build/
  - text: Docker Hub - Node Images
    url: https://hub.docker.com/_/node
prev: /capitulo2/lab2          
next: /capitulo4/lab4/
---


---

### Tarea 1. Preparar la estructura del proyecto

Crear la estructura de carpetas con backend y frontend.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`**. Lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VS Code**, da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 5.** Asegúrate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VS Code**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para retornar a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Ahora crea el directorio para trabajar en la Práctica 3.

  ```bash
  mkdir lab3-dockermultistage && cd lab3-dockermultistage
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VS Code que se haya creado el directorio.

  ![micint]({{ page.images_base | relative_url }}/2.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Creación del backend

Implementar una API REST en Node.js. Se reutilizará el backend creado en la Práctica 1.

#### Tarea 2.1

- **Paso 8.** Copia la carpeta **backend** de la Práctica 1, escribe el siguiente comando.

  > **Importante**
    - El comando se ejecuta desde adentro de la carpeta **lab3-dockermultistage**.
    - En caso de que no estés dentro de la carpeta **lab3...**, ajusta las rutas.
  {: .lab-note .important .compact}


  ```bash
  cp -r ../lab1-acontactos/backend/ .
  ```

- **Paso 9.** Ahora, escribe el siguiente comando para validar que se haya copiado correctamente la carpeta **backend**.

  > **Nota.** Recuerda que también puedes visualizarlo en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls -la
  ```

  ![micint]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Creación del frontend

Reutilizar el directorio **frontend** que tiene el sitio web estático.

#### Tarea 3.1

- **Paso 10.** Copia la carpeta **frontend** de la Práctica 1, escribe el siguiente comando.

  > **Importante**
    - El comando se ejecuta desde adentro de la carpeta **lab3-dockermultistage**.
    - En caso de que no estés dentro de la carpeta **lab3...** ajusta las rutas.
  {: .lab-note .important .compact}

  ```bash
  cp -r ../lab1-acontactos/frontend/ .
  ```

- **Paso 11.** Ahora, escribe el siguiente comando para validar que se haya copiado correctamente la carpeta **frontend**.

  > **Nota.** Recuerda que también puedes visualizarlo en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Construir imagen sin Multi-Stage

Crear un Dockerfile simple y analizar el tamaño.

#### Tarea 4.1

- **Paso 12.** Crea el archivo **Dockerfile** dentro del directorio **lab3-dockermultistage.**

  > **Nota.** El comando se ejecuta desde la carpeta **lab3...**
  {: .lab-note .info .compact}

  ```bash
  touch Dockerfile
  code Dockerfile
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

- **Paso 13.** Agrega el siguiente código al archivo **Dockerfile**.

  > **Nota.** Este Dockerfile crea una imagen sin Multi-Stage.
  {: .lab-note .info .compact}

  ```dockerfile
  FROM node:20-alpine
  WORKDIR /app
  COPY backend/package*.json ./
  RUN npm install --production
  COPY backend ./backend
  COPY frontend ./frontend
  EXPOSE 3000
  CMD ["node", "backend/server.js"]
  ```

- **Paso 14.** Compila el proyecto de docker. Escribe el siguiente comando dentro de la terminal.

  > **Nota.** El comando se ejecuta desde la carpeta **lab3...**
  {: .lab-note .info .compact}

  ```bash
  docker build -t contactos-tradicional .
  ```

- **Paso 15.** Ahora, valida el tamaño de la imagen creada después de la compilación.

  > **Nota.** Aproximadamente, el tamaño será de `207MB`, aunque puede diferir.
  {: .lab-note .info .compact}

  ```bash
  docker images contactos-tradicional
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 16.** Si es necesario, anota el número del tamaño de la imagen.

  > **Nota.** Esta es la forma tradicional, pero genera imágenes más grandes dependiendo de cómo esté estructurada la aplicación.
  {: .lab-note .info .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Construir imagen con Multi-Stage

Optimizar la construcción usando **Multi-Stage**.

#### Tarea 5.1

- **Paso 17.** Crea otro archivo **Dockerfile** para que quede mejor organizado y escribe el siguiente comando.

  > **Nota**
    - El nombre **opt** hace referencia a **Optimización** para la compilación del Multi-Stage.
    - Se crea dentro del mismo directorio **lab3...**.
  {: .lab-note .info .compact}

  ```bash
  touch Dockerfile.opt
  code Dockerfile.opt
  ```

- **Paso 18.** Ahora, agrega el siguiente código a ese nuevo archivo **Dockerfile.opt**.

  > **Nota.** El contenido es muy parecido al ejemplo anterior, pero internamente se mejora el proceso reduciendo las capas.
  {: .lab-note .info .compact}

  ```dockerfile
  # Etapa 1: build
  FROM node:20-alpine AS builder
  WORKDIR /app
  COPY backend/package*.json ./
  RUN npm install --production
  COPY backend ./backend
  COPY frontend ./frontend

  # Etapa 2: ejecución
  FROM node:20-alpine
  WORKDIR /app
  COPY --from=builder /app ./ 
  EXPOSE 3000
  CMD ["node", "backend/server.js"]
  ```

- **Paso 19.** Ahora construye la imagen, escribiendo el siguiente comando.

  > **Nota.** Quizás, alcances a notar que compila un poco más rápido, todo depende de cuánta información tenga tu proyecto.
  {: .lab-note .info .compact}

  ```bash
  docker build -f Dockerfile.opt -t contactos-opt .
  ```

- **Paso 20.** Escribe el siguiente comando para validar el tamaño de la imagen.

  ```bash
  docker images contactos-opt
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

- **Paso 21.** Si es necesario, anota el número del tamaño de la imagen.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Comparar resultados

Verificar la diferencia de tamaños de las imágenes.

#### Tarea 6.1

- **Paso 22.** Usa el siguiente comando para realizar la comparación de las imágenes.

  > **Importante**
    - El comando se ejecuta dentro de la terminal **GitBash**.
    - Puede estar en cualquier directorio, pero lo ideal es que esté en **lab3...**
  {: .lab-note .important .compact}

  ```bash
  docker images | grep contactos-
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 23.** Puedes observar que la imagen **opt** es ligeramente más pequeña, recuerda que depende de cuánta información tenga tu proyecto.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Añadir .dockerignore (mejor caché y tamaño)

Crear un archivo .dockerignore para excluir archivos innecesarios y mejorar el tamaño y la caché de las imágenes.

#### Tarea 7.1

- **Paso 24.** Crea y abre un archivo **.dockerignore** en la raíz **lab3-dockermultistage**.

  ```bash
  touch .dockerignore
  code .dockerignore
  ```

- **Paso 25.** Ahora, agrega el siguiente contenido al archivo.

  ```gitignore
  backend/node_modules
  frontend/node_modules
  /node_modules
  .git
  .vscode
  .DS_Store
  npm-debug.log
  Dockerfile*
  Thumbs.db
  ```

- **Paso 26.** Reconstruye la imagen **tradicional**.

  ```bash
  docker build -t contactos-tradicional .
  ```

- **Paso 27.** Reconstruye la imagen **optimizada**.

  ```bash
  docker build -f Dockerfile.opt -t contactos-opt .
  ```

- **Paso 28.** Ahora, valida ambos tamaños y compara el resultado.

  > **Nota.** Como puedes observar, se hizo una reducción relativa a la cantidad de archivos de la aplicación.
  {: .lab-note .info .compact}

  ```bash
  docker images | grep contactos-
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Variante de runtime aún más ligera


#### Tarea 8.1

Crear un Dockerfile.slim con Multi-Stage minimalista para generar imágenes más ligeras y seguras.

- **Paso 29.** Crea **Dockerfile.slim** para copiar únicamente lo necesario.

  > **Nota.** El comando se ejecuta dentro del directorio **lab3...**
  {: .lab-note .info .compact}

  ```bash
  touch Dockerfile.slim
  ```

- **Paso 30.** Agrega el siguiente contenido a ese nuevo Dockerfile.

  ```dockerfile
  FROM node:20-alpine AS deps
  WORKDIR /app
  COPY backend/package*.json ./
  RUN npm install --omit=dev
  COPY backend ./backend

  FROM node:20-alpine
  WORKDIR /app
  COPY --from=deps /app/backend ./backend
  COPY --from=deps /app/node_modules ./node_modules
  RUN addgroup -S appgrp && adduser -S appusr -G appgrp
  USER appusr
  EXPOSE 3000
  CMD ["node", "backend/server.js"]
  ```

- **Paso 31.** Construye la nueva imagen basada en el **Dockerfile.slim**.

  ```bash
  docker build -f Dockerfile.slim -t contactos-slim .
  ```

- **Paso 32.** Escribe el siguiente comando para observar el resultado.

  > **Nota.** Tanto la versión **opt** como la **slim** usan **Multi-Stage**,pero **slim** es mucho más minimalista, ya que la propiedad **--from=deps** copia solo lo que se necesita.
  {: .lab-note .info .compact}

  ```bash
  docker images | grep contactos-
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Limpieza y buenas prácticas

Eliminar las imágenes creadas para mantener limpio el ambiente.

#### Tarea 9.1

- **Paso 33.** Escribe el siguiente comando para eliminar las imágenes.

  ```bash
  docker rmi contactos-tradicional contactos-opt contactos-slim
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 34.** Verifica que ya no aparezca ninguna imagen **contactos-...**

  > **Nota.** En caso de que todavía exista alguna, repite el paso anterior.
  {: .lab-note .info .compact}

  ```bash
  docker images
  ```
  ![micint]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
