---
layout: lab
title: "Práctica 3: Construcción de una imagen Docker multi-etapa"
permalink: /capitulo3/lab3/
images_base: /labs/capitulo3/img
duration: "60 minutos"
objective:
  - El objetivo de esta práctica es que el participante aprenda a construir imágenes Docker optimizadas utilizando la técnica de **multi-stage builds**, comparando el tamaño de la imagen con y sin multi-etapa. Usaremos como ejemplo una aplicación Node.js con Express que sirva un frontend estático y una API REST.
prerequisites:
  - Visual Studio Code instalado
  - Docker Desktop instalado y en ejecución
  - Git Bash configurado como terminal por defecto en VS Code
  - Conocimientos básicos de Node.js y Docker
  - Haber completado la practica 1
introduction:
  - Las **imágenes multi-etapa** permiten separar el proceso de construcción (instalación de dependencias y compilación de frontend) del de ejecución. Con esto se reduce el tamaño de la imagen final y se evitan dependencias innecesarias en producción. En esta práctica construiremos una aplicación Node.js con un backend REST y un frontend estático en React simulado con archivos preconstruidos, para demostrar el impacto de los multi-stage builds.
slug: lab3
lab_number: 3
final_result: >
  El estudiante habrá construido una aplicación Node.js contenerizada con un **Dockerfile multi-etapa**, obteniendo una imagen ligera y optimizada, validando que la aplicación funciona correctamente y que el tamaño de la imagen se reduce respecto a una construcción tradicional.
notes: 
  - Multi-stage es aplicable a múltiples lenguajes (Java, Go, .NET).
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

### Tarea 1: Preparar la estructura del proyecto

Crear la estructura de carpetas con backend y frontend.

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

  ```bash
  mkdir lab3-dockermultistage && cd lab3-dockermultistage
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VSCode que se haya creado el directorio:

  ![micint](img/2.png)

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Entorno de trabajo listo para comenzar la creacion de archivos.

---

### Tarea 2: Creación del backend

Implementar una API REST en Node.js. Se reutilizara el backend creado en la practica 1

#### Tarea 2.1

- **Paso 8.** Copia la carpeta **backend** de la practica 1, escribe el siguiente comando.

  - El comando se ejecuta desde adentro de la carpeta **lab3-dockermultistage**.
  - En caso de que no estes dentro de la carpeta **lab3...** ajusta las rutas.

  ```bash
  cp -r ../lab1-acontactos/backend/ .
  ```

- **Paso 9.** Ahora escribe el siguiente comando para validar que se haya copiado correctamente la carpeta **backend**

  - Recuerda que tambien puedes visualizarlo en el explorador de archivos de VSCode.

  ```bash
  ls
  ```

  ![micint](img/3.png)

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Carpeta de backend integrada correctamente en el nuevo directorio

---

### Tarea 3: Creación del frontend

Se reutilizara el directorio **frontend** que tiene el sitio web estatico.

#### Tarea 3.1

- **Paso 10.** Copia la carpeta **frontend** de la practica 1, escribe el siguiente comando.

  - El comando se ejecuta desde adentro de la carpeta **lab3-dockermultistage**.
  - En caso de que no estes dentro de la carpeta **lab3...** ajusta las rutas.

  ```bash
  cp -r ../lab1-acontactos/frontend/ .
  ```

- **Paso 11.** Ahora escribe el siguiente comando para validar que se haya copiado correctamente la carpeta **frontend**

  - Recuerda que tambien puedes visualizarlo en el explorador de archivos de VSCode.

  ```bash
  ls
  ```

  ![micint](img/4.png)

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Carpeta de frontend integrada correctamente en el nuevo directorio

---

### Tarea 4: Construir imagen sin multi-stage

Crear un Dockerfile simple y analizar el tamaño.

#### Tarea 4.1

- **Paso 12.** Crea el archivo **Dockerfile** dentro del directorio **lab3-dockermultistage**

  - El comando se ejecuta desde la carpeta **lab3...**

  ```bash
  touch Dockerfile
  code Dockerfile
  ```

  ![micint](img/5.png)

- **Paso 13.** Agrea el siguiente codigo al archivo **Dockerfile**

  - Crea una imagen sin multi-stage.

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

- **Paso 14.** Compila el proyecto de docker, escribe el siguiente comando dentro de la terminal.

  ```bash
  docker build -t contactos-tradicional .
  ```

- **Paso 15.** Ahora valida el tamaño de la imagen creada despues de la compilación.

  - Aproximadamente el tamaño quedara `207MB` puede ser diferente.

  ```bash
  docker images contactos-tradicional
  ```

  ![micint](img/6.png)

- **Paso 16.** Si es necesario anota el numero del tamaño de la imagen.

  - Esta es la forma tradicional, pero genera imágenes más grandes.

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Imagen funcional pero de mayor tamaño.

---

### Tarea 5: Construir imagen con multi-stage

Optimizar la construcción usando multi-stage.

#### Tarea 5.1

- **Paso 17.** Crea otro archivo **Dockerfile** para que quede mejor organizado tanto archivos como el ejemplo, escribe el siguiente comando.

  - El nombre **opt** hace referencia a **Optimización** para la compilación del Multi-Stage
  - Se crea dentro del mismo directorio **lab3...** 

  ```bash
  touch Dockerfile.opt
  code Dockerfile.opt
  ```

- **Paso 18.** Ahora agrega el siguiente codigo a ese nuevo archivo **Dockerfile.opt**

  - Es muy parecido en contenido al ejemplo anterior, pero internamente se mejora el proceso reduciendo las capas.

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

- **Paso 19.** Ahora construye la imagen, escribe el siguiente comando.

  - Quizas alcances a notar que compila un poco mas rapido, todo depende de que tanta información tenga tu proyecto.

  ```bash
  docker build -f Dockerfile.opt -t contactos-opt .
  ```

- **Paso 20.** Escribe el siguiente comando para validar el tamaño de la imagen.

  ```bash
  docker images contactos-opt
  ```

  ![micint](img/7.png)

- **Paso 21.** Si es necesario anota el numero del tamaño de la imagen.

> **¡TAREA FINALIZADA!**

**Resultado esperado:** La etapa de construcción prepara dependencias y archivos, y la final contiene solo lo necesario para ejecutar. Imagen más ligera que la versión simple.

---

### Tarea 6: Comparar resultados

Verificar la diferencia de tamaños.

#### Tarea 6.1

- **Paso 22.** Usa el siguiente comando para realizar la comparacion de las imagnes.

  - El comando se ejecuta dentro de la terminal **GitBash**
  - Puede ser en cualquier directorio, pero deseado en **lab3...**

  ```bash
  docker images | grep contactos-
  ```

  ![micint](img/8.png)

- **Paso 23.** Puedes observar que ligeramente la imagen **opt** es mas pequeña, recuerda que depende de que tanta informacion tenga tu proyecto.

> **¡TAREA FINALIZADA!**

**Resultado esperado:** Validación del beneficio del multi-stage build.