---
layout: lab
title: "Práctica 2. Operaciones esenciales con contenedores Docker"
permalink: /capitulo2/lab2/
images_base: /labs/capitulo2/img
duration: "60 minutos"
objective:
  - El participante aprenderá a administrar contenedores en Docker, dominando operaciones básicas como creación, ejecución, inspección, gestión y eliminación desde la terminal Git Bash en Visual Studio Code.
prerequisites:
  - Visual Studio Code instalado
  - Docker Desktop instalado y en ejecución
  - Git Bash configurado como terminal por defecto en VS Code
  - Conocimientos básicos de terminal (cd, ls, entre otros comandos)
introduction:
  - Docker permite ejecutar aplicaciones de forma aislada mediante contenedores. En esta práctica aprenderemos a realizar las operaciones esenciales, crear y correr contenedores, listar y detenerlos, inspeccionar su estado, ver logs y finalmente eliminarlos. Estas operaciones constituyen el núcleo de la administración diaria en entornos de microservicios.
slug: lab2
lab_number: 2
final_result: >
  Al terminar esta práctica, el estudiante dominará las operaciones básicas de administración de contenedores en Docker: ejecutar, listar, inspeccionar, detener, reiniciar y eliminar. Esto constituye la base para escenarios más complejos en microservicios y DevOps.
notes: 
  - Si un contenedor no se elimina porque está en ejecución, usar.
    ```bash
    docker rm -f <ID_CONTENEDOR>
    ``` 
  - Es buena práctica limpiar imágenes y contenedores no usados periódicamente.
    ```bash
    docker system prune
    ```
references:
  - text: Documentación oficial Docker - Comandos
    url: https://docs.docker.com/engine/reference/commandline/docker/
  - text: Docker Labs
    url: https://github.com/docker/labs
  - text: Play with Docker
    url: https://labs.play-with-docker.com/
prev: /capitulo1/lab1          
next: /capitulo3/lab3/
---


---

### Tarea 1. Preparar la carpeta de la práctica

Crear una carpeta dedicada para esta práctica que servirá para organizar notas y ejemplos.

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos.  

- **Paso 2.** Abre el **`Visual Studio Code`**. Lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VSCode**, da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/19.png)

- **Paso 5.** Asegúrate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VSCode**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para retornar a la raíz de los laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 6.** Ahora crea el directorio para trabajar en la Práctica 2.

  ```bash
  mkdir lab2-dockerops && cd lab2-dockerops
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de **VSCode** que se haya creado el directorio.

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 8.** Crear un archivo de notas `README.md` dentro del directorio **lab2-dockerops** para ir documentando los comandos ejecutados.

  ```bash
  touch README.md
  ```

- **Paso 9.** Confirma la estructura del archivo y directorio creados; luego, ejecuta el siguiente comando.

  > **Nota.** Organizar cada práctica en carpetas separadas facilita la gestión de ejemplos y evita confusiones.
  {: .lab-note .info .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Descargar y ejecutar un contenedor básico

Ejecutar un contenedor simple (hello-world) para validar la instalación de Docker.

#### Tarea 2.1

- **Paso 10.** Ejecuta el siguiente comando para ejecutar el contenedor.

  > **Importante.** Aunque no importa la ruta donde lo ejecutes, deberías de estar en el directorio **lab2-dockerops**.
  {: .lab-note .important .compact}

  > **Nota.** La salida debe mostrar un mensaje de bienvenida indicando que la instalación de Docker funciona correctamente.
  {: .lab-note .info .compact}

  > **Nota.** El contenedor `hello-world` es la forma más simple de validar que Docker puede ejecutar imágenes.
  {: .lab-note .info .compact}

  ```bash
  docker run hello-world
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 11.** Verifica que el docker se encuentre en el estado detenido; luego, escribe el siguiente comando.

  ```bash
  docker ps -a
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Ejecutar un contenedor interactivo

Aprender a correr un contenedor de Linux en modo interactivo.

#### Tarea 3.1

- **Paso 12.** Ejecuta el siguiente comando para descargar y ejecutar un contenedor de Ubuntu:

  - El parámetro `-it` crea una sesión interactiva para acceder al container.

  ```bash
  docker run -it ubuntu bash
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

- **Paso 13.** Dentro del contenedor ejecuta el siguiente comando.

  - Debe mostrarse información de Ubuntu.
  - Al salir, el contenedor se detiene.
  - Este tipo de contenedor es útil para pruebas rápidas o ambientes aislados de desarrollo.

  ```bash
  cat /etc/os-release
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 14.** Sal del contenedor y escribe el siguiente comando.

  ```bash
  exit
  ```

- **Paso 15.** Verifica el contenedor detenido y escribe el siguiente comando.

  - Ejecutar los contenedores interactivos permite probar entornos sin afectar la máquina anfitriona.

  ```bash
  docker ps -a
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Listar, inspeccionar y administrar contenedores

Manejar los comandos más importantes para trabajar con contenedores en ejecución o detenidos.

#### Tarea 4.1

- **Paso 16.** Ejecuta el contenedor de Ubuntu en forma desasociada y escribe el siguiente comando.

  ```bash
  docker run -d -it ubuntu bash
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 17.** Lista los contenedores activos.

  ```bash
  docker ps
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 18.** Lista todos los contenedores, incluidos los detenidos.

  ```bash
  docker ps -a
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 19.** Ve los detalles de un contenedor, **copia el ID del contenedor Ubuntu en ejecución, el valor de la primera columna del comando anterior y sustitúyelo en la etiqueta `<ID_CONTENEDOR>` de este comando.**

  > **Nota.** El comando tiene una salida muy extensa, la imagen es solo representativa.
  {: .lab-note .info .compact}

  ```bash
  docker inspect <ID_CONTENEDOR>
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

- **Paso 20.** Ve los logs de un contenedor, **sustituye el ID del contenedor Ubuntu en ejecución y sustitúyelo en la etiqueta `<ID_CONTENEDOR>` de este comando.**

  > **Nota.** El contenedor no mostrará logs, ya que eso depende de si se generan o de que las aplicaciones los muestren.
  {: .lab-note .info .compact}

  > **Importante.** En este caso, es un contenedor sencillo.
  {: .lab-note .important .compact}

  ```bash
  docker logs <ID_CONTENEDOR>
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 21.** Ve las estadísticas de uso en tiempo real.

  > **Nota.** Toma unos minutos para analizar la información de las estadísticas.
  {: .lab-note .info .compact}

  > **Importante.** Cuando hayas terminado, ejecuta `CTRL + c` para salir de las estadísticas.
  {: .lab-note .important .compact}

  ```bash
  docker stats
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Detener, reiniciar y eliminar contenedores

Aprender a controlar el ciclo de vida de los contenedores.

#### Tarea 5.1

- **Paso 22.** Para detener el contenedor, escribe el siguiente comando: primero obtén el valor del ID y sustitúyelo en la etiqueta **`<ID_CONTAINER>`** del contenedor que deseas parar. En este caso, el ID del contenedor **Ubuntu**.

  ```bash
  docker ps
  ```

  ```bash
  docker stop <ID_CONTENEDOR>
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 23.** Si deseas iniciar nuevamente el contenedor detenido, ejecuta el siguiente comando, pero sustituye el ID del contenedor Ubuntu.

  ```bash
  docker start <ID_CONTENEDOR>
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 24.** Para eliminar un contenedor primero se debe de detener.

  ```bash
  docker stop <ID_CONTENEDOR>
  ```

- **Paso 25.** Ya detenido, escribe los siguientes comandos. **Recuerda sustituir el ID del contenedor Ubuntu.**

  > **Nota.** Primero, **visualiza** los contenedores detenidos. Luego, **selecciona** los ID del contenedor a eliminar.
  {: .lab-note .info .compact}

  ```bash
  docker ps -a
  ```

  ```bash
  docker rm <ID_CONTENEDOR>
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

- **Paso 26.** Para validar que ya no exista el contenedor eliminado, escribe el siguiente comando.

  > **Nota.** Si el contenedor sigue apareciendo, repite el comando del paso anterior. **Solo debe de quedar la imagen de `Minikube`**
  {: .lab-note .info .compact}

  ```bash
  docker ps -a
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## Tarea 6. Manejo de imágenes Docker

Listar, buscar, descargar y eliminar imágenes Docker para gestionar eficientemente el repositorio local.

### Tarea 6.1

- **Paso 27.** Lista todas las imágenes descargadas.  

  ```bash
  docker images
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 28.** Busca imágenes en Docker Hub.  

  > **Nota.** La búsqueda de imágenes puede ser muy larga, aparecerán todos los que han contribuido o repositorios públicos.
  {: .lab-note .info .compact}

  ```bash
  docker search alpine
  ```

  ![micint]({{ page.images_base | relative_url }}/22.png)

- **Paso 29.** Descarga una imagen específica.  

  > **Nota.** La imagen siempre se acompaña de la etiqueta `:latest`, pero también puede ser una etiqueta personalizada.
  {: .lab-note .info .compact}

  ```bash
  docker pull alpine:latest
  ```

  ![micint]({{ page.images_base | relative_url }}/23.png)

- **Paso 30.** Elimina una imagen.  

  ```bash
  docker rmi alpine:latest
  ```

  ![micint]({{ page.images_base | relative_url }}/24.png)

- **Paso 31.** Ahora elimina el resto de las imágenes creadas en este laboratorio.  

  ```bash
  docker rmi ubuntu:latest hello-world:latest
  ```

  ![micint]({{ page.images_base | relative_url }}/25.png)

- **Paso 32.** Verifica que solo te quede la imagen **agenda-contactos** y la de **minikube**.

  ```bash
  docker images
  ```

  ![micint]({{ page.images_base | relative_url }}/26.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## Tarea 7. Comandos adicionales de administración

Crear un contenedor con nombre, revisar procesos y limpiar recursos para administrar mejor Docker.

### Tarea 7.1

- **Paso 33.** Crea un contenedor con nombre personalizado.  

  > **Nota.** La etiqueta **`--name`** define el nombre necesario. `-p` define la relación puerto **host:container**.
  {: .lab-note .info .compact}

  ```bash
  docker run -d -p 8080:80 --name mi_nginx nginx
  ```

  ![micint]({{ page.images_base | relative_url }}/27.png)

- **Paso 34.** Verifica la relación de puertos expuestos.  

  ```bash
  docker port mi_nginx
  ```

  ![micint]({{ page.images_base | relative_url }}/28.png)

- **Paso 35.** Inspecciona los recursos del contenedor.  

  > **Nota.** Este comando se utiliza para mostrar los procesos en ejecución de un contenedor específico.
  {: .lab-note .info .compact}

  ```bash
  docker top mi_nginx
  ```

  ![micint]({{ page.images_base | relative_url }}/29.png)

- **Paso 36.** Elimina todos los contenedores detenidos. Cuando aparezca la pregunta, escribe **N**. El contenedor se eliminará manualmente.

  > **Nota.** Este comando es útil para liberar espacio en disco ocupado por contenedores que ya no se están ejecutando y no están destinados a reiniciarse.
  {: .lab-note .info .compact}

  > **Importante.** Se debe tener mucho cuidado al usar este comando.
  {: .lab-note .important .compact}

  ```bash
  docker stop mi_nginx
  docker ps -a
  ```

  ```bash
  docker container prune
  ```

  ![micint]({{ page.images_base | relative_url }}/30.png)

- **Paso 37.** Eliminar imágenes, redes y volúmenes no usados.  

  > **Nota.** Este comando se utiliza para limpiar objetos Docker no utilizados, lo que ayuda a liberar espacio en disco. Elimina varios componentes.
  - Todos los contenedores detenidos.
  - Todas las redes no utilizadas por al menos un contenedor.
  - Todas las imágenes colgantes (capas de imágenes que ya no están etiquetadas ni asociadas a ningún contenedor).
  - La caché de compilación.
  {: .lab-note .info .compact}

  > **Importante.** Se debe tener mucho cuidado al usar este comando.
  {: .lab-note .important .compact}

  ```bash
  docker system prune
  ```

  ![micint]({{ page.images_base | relative_url }}/31.png)

- **Paso 38.** Finalmente, elimina el contenedor ejemplo y escribe los siguientes comandos.

  ```bash
  docker rm mi_nginx
  docker rmi nginx:latest
  ```

  ![micint]({{ page.images_base | relative_url }}/32.png) 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## Tarea 8. Documentación final en README.md

Documentar los comandos esenciales de Docker en un archivo **README.md** para consolidar la práctica.

### Tarea 8.1

- **Paso 39.** Ahora abre el archivo **README.md**.

  > **Nota.** El comando `code` se ejecuta en la raíz del directorio **lab2-dockerops**
  {: .lab-note .info .compact}

  ```bash
  code README.md
  ```

- **Paso 40.** Agrega esta lista de comandos como parte de la documentación de la práctica.

  ```markdown
  # Práctica 2. Lista de comandos esenciales en Docker

  | Comando | Descripción |
  |---------|-------------|
  | `docker run hello-world` | Ejecuta un contenedor de prueba para validar Docker. |
  | `docker ps -a` | Lista todos los contenedores, incluidos los detenidos. |
  | `docker run -it ubuntu bash` | Ejecuta Ubuntu en modo interactivo. |
  | `cat /etc/os-release` | Muestra información del sistema operativo. |
  | `docker run -d -it ubuntu bash` | Ejecuta Ubuntu en segundo plano. |
  | `docker inspect <ID>` | Muestra detalles del contenedor. |
  | `docker logs <ID>` | Muestra logs del contenedor. |
  | `docker stats` | Muestra estadísticas en tiempo real. |
  | `docker stop <ID>` | Detiene un contenedor. |
  | `docker start <ID>` | Reinicia un contenedor detenido. |
  | `docker rm <ID>` | Elimina un contenedor detenido. |
  | `docker images` | Lista las imágenes disponibles. |
  | `docker pull alpine` | Descarga la imagen Alpine desde Docker Hub. |
  | `docker rmi alpine` | Elimina la imagen Alpine local. |
  | `docker run -d --name mi_nginx nginx` | Ejecuta un contenedor Nginx con nombre personalizado. |
  | `docker port mi_nginx` | Muestra los puertos expuestos. |
  | `docker top mi_nginx` | Lista los procesos dentro del contenedor. |
  | `docker container prune` | Elimina los contenedores detenidos. |
  | `docker system prune` | Limpia los recursos no utilizados. |
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
