---
layout: lab
title: "Práctica 15. Afinidad y tolerancia de pods"
permalink: /capitulo15/lab15/
images_base: /labs/capitulo15/img
duration: "60 minutos"
objective:
  - Aprenderás a aplicar **afinidad, antiafinidad y tolerancias** en pods para controlar dónde se ejecutan tus aplicaciones dentro de un clúster multinodo en Minikube. Configurarás un entorno con múltiples nodos, desplegarás pods con diferentes reglas de scheduling y finalmente harás limpieza del entorno.
prerequisites:
  - Visual Studio Code  
  - Minikube instalado (con soporte para multinodo)  
  - kubectl instalado y configurado  
  - Docker Desktop como backend para Minikube  
  - Terminal **Git Bash** dentro de VS Code  
  - Conocimientos previos de pods, Deployments y Namespaces  
introduction: |
  En Kubernetes puedes influir en el **scheduler** (programador de pods) usando **afinidad y antiafinidad**. Esto te permite decidir si los pods se colocan en el mismo nodo o en nodos distintos. Además, las **toleraciones (tolerations)** permiten que un pod pueda ejecutarse en nodos que tienen **taints** (marcas que restringen el acceso).  

  En esta práctica, crearás un **clúster multinodo con Minikube** (tres nodos) y, luego, desplegarás **pods con afinidad, antiafinidad y tolerancias** para observar cómo Kubernetes programa los pods en diferentes nodos.
slug: lab15
lab_number: 15
final_result: >
  Has aprendido a crear un clúster multinodo con Minikube y aplicar **afinidad, antiafinidad y toleraciones** en pods y Deployments. Además, validaste cómo afectan al scheduling y realizaste una limpieza completa del entorno.
notes:
  - Siempre valida etiquetas y taints antes de aplicar pods.  
  - Usa `kubectl describe pod` y `kubectl describe node` para analizar por qué un pod se programa en cierto nodo.  
  - En entornos productivos se recomienda combinar afinidad/tolerancia con **zonas y regiones** para alta disponibilidad.
references:
  - text: Afinidad y antiafinidad
    url: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity  
  - text: Taints y tolerations
    url: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/ 
  - text: Node affinity
    url: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/  
  - text: Minikube multinodo
    url: https://minikube.sigs.k8s.io/docs/tutorials/multi_node/  
prev: /capitulo14/lab14/          
next: /capitulo16/lab16/
---


---

### Tarea 1. Crear clúster Minikube multinodo

Configurar Minikube para ejecutar un clúster con tres nodos (un máster y dos workers).

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos. 

- **Paso 2.** Abre el **`Visual Studio Code`** lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VS Code** da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 5.** Asegúrate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VS Code**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/3.png)

- **Paso 6.** Primero, ejecuta el siguiente comando para iniciar un clúster multinodo de Minikube:

  > **Nota.** Tener **múltiples nodos** te permitirá comprobar cómo funcionan la afinidad y la antiafinidad en un escenario más realista.
  {: .lab-note .info .compact}

  > **Importante.** Puede tardar alrededor de **cinco minutos**. Espera la creación antes de continuar.
  {: .lab-note .important .compact}
  
  ```bash
  minikube start --nodes=3 -p multinodo
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 7.** Ya creado el cluster multinodo puedes verificar los nodos creados con el siguiente comando.:

  > **Nota.** Debes ver al menos 3 nodos: `multinodo`, `multinodo-m02`, `multinodo-m03`.
  {: .lab-note .info .compact}

  ```bash
  kubectl get nodes -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear la estructura del proyecto

Organizar todos los manifiestos de esta práctica en una carpeta propia.

#### Tarea 2.1

- **Paso 8.** Crea el directorio para trabajar en la **práctica**:

  > **Nota.** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab15-k8safitol && cd lab15-k8safitol
  ```

- **Paso 9.** Valida en el **Explorador** de archivos, dentro de VS Code, que se haya creado el directorio.

  > **Nota.** Mantener cada práctica en su carpeta evita confusiones y facilita la limpieza o el versionado.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Crea la estructura base de directorios y archivos vacíos.

  > **Nota.** Separar manifiestos hace que sea más fácil probar y entender cada caso por separado.
  {: .lab-note .info .compact}

  ```text
  lab15-k8safitol/
  ├── k8s/
  │   ├── pod-afinidad.yaml
  │   ├── pod-antiafinidad.yaml
  │   ├── pod-tolerancia.yaml 
  └── └── deployment-mix.yaml
  ```

- **Paso 11.** Ahora, crea la carpeta **k8s/** y sus archivos vacíos.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab15-k8safitol**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/pod-afinidad.yaml k8s/pod-antiafinidad.yaml k8s/pod-tolerancia.yaml k8s/deployment-mix.yaml
  ```

- **Paso 12.** Valida la creación de la estructura de tu proyecto. Escribe el siguiente comando.

  > **Nota.** También puedes validarlo en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}
  ```bash
  ls -l -R
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Aplicar afinidad de pods

Crear un pod que **solo se ejecute en nodos etiquetados**.

#### Tarea 3.1

- **Paso 13.** Etiqueta uno de los nodos workers. Escribe el siguiente comando.

  ```bash
  kubectl label node multinodo-m02 zona=primaria
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 14.** Verifica que la etiqueta haya sido aplicada correctamente.

  ```bash
  kubectl get nodes -l zona=primaria
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 15.** Define la configuración del pod dentro del archivo `k8s/pod-afinidad.yaml`:

  > **Nota.** **Revisa entre la línea 11 y 14 del manifiesto.** El pod se programará en el nodo `multinodo-m02` porque es el único con la etiqueta `zona=primaria`.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-afinidad
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: zona
              operator: In
              values:
              - primaria
    containers:
    - name: nginx
      image: nginx:1.25
  ```

- **Paso 16.** Aplica el manifiesto del pod. Escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/pod-afinidad.yaml
  ```

- **Paso 17.** Verifica que en efecto el pod se haya creado en ese nodo gracias a la afinidad configurada.

  > **Nota.** Si verificaste muy rapido espera unos segundos y vuelve a ejecutar el comando.
  {: .lab-note .info .compact}

  ```bash
  kubectl get pod pod-afinidad -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Aplicar antiafinidad de pods

Crear pods que **eviten ejecutarse en el mismo nodo**.

#### Tarea 4.1

- **Paso 18.** Dentro del archivo `k8s/pod-antiafinidad.yaml`, guarda la siguiente configuración.

  > **Nota.** Si despliegas múltiples pods con `app=demo`, se distribuirán en nodos distintos gracias al `topologyKey` (línea 17).
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-antiafinidad
    labels:
      app: demo
  spec:
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - demo
          topologyKey: "kubernetes.io/hostname"
    containers:
    - name: nginx
      image: nginx:1.25
  ```

- **Paso 19.** Aplica el manifiesto del pod. Escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/pod-antiafinidad.yaml
  ```

- **Paso 20.** Ahora, observa en que nodo aterrizo el pod. 

  > **Nota.** Si verificaste muy rapido espera unos segundos y vuelve a ejecutar el comando.
  {: .lab-note .info .compact}

  ```bash
  kubectl get pods -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 21.** Crea un segundo pod con la misma etiqueta `(app=demo)`.

  > **Nota.** En caso de que solo se tenga 1 nodo, el **segundo pod** quedará `Pending` porque no puede convivir con otro **app=demo** en el mismo hostname.
  {: .lab-note .info .compact}

  ```bash
  kubectl run pod-antiafinidad-2 --image=nginx:1.25 --labels app=demo
  ```

  ```bash
  kubectl get pods -o wide
  ```
  
  ```bash
  kubectl describe pod pod-antiafinidad-2 | sed -n '/Events/,$p'
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Usar toleraciones con taints

Marcar (taint) un nodo para que no acepte pods normales y, luego, crear un pod con una tolerancia que le permita ejecutarse ahí. Esto asegura que solo workloads autorizados usen nodos dedicados.

#### Tarea 5.1. Aplicar el taint

- **Paso 22.** Aplica un taint en el nodo `multinodo-m03`:

  > **Notas** 
    - `uso=dedicado:` clave/valor de taint.
    - `NoSchedule:` ningún pod sin tolerancia podrá programarse en este nodo.
  {: .lab-note .info .compact}

  ```bash
  kubectl taint nodes multinodo-m03 uso=dedicado:NoSchedule
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)  

- **Paso 23.** Verifica que el taint se aplicó correctamente.

  ```bash
  kubectl describe node multinodo-m03 | grep Taints:
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)  

#### Tarea 5.2. Crear pod con tolerancia

- **Paso 24.** Define la siguiente configuración en el archivo `k8s/pod-tolerancia.yaml`.

  > **Nota.** Sin la tolerancia, el pod no podría ejecutarse en `multinodo-m03`. Con ella, se programa correctamente en ese nodo.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-tolerancia
  spec:
    tolerations:
    - key: "uso"
      operator: "Equal"
      value: "dedicado"
      effect: "NoSchedule"
    containers:
    - name: nginx
      image: nginx:1.25
  ```

- **Paso 25.** Aplica el manifiesto.

  ```bash
  kubectl apply -f k8s/pod-tolerancia.yaml
  ```

- **Paso 26.** Verifica que el pod corra en el nodo taintado

  > **Nota.** La columna `NODE` debe mostrar `multinodo-m03`.
  {: .lab-note .info .compact}

  ```bash
  kubectl get pod pod-tolerancia -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)  

- **Paso 27.** Verifica que otro **pod** sin tolerancia se queda en **Pending**

  > **Nota.** En este caso, el pod sin toleración fue asignado a `multinodo-02`. Pero dependiendo de la configuración verás un evento tipo **failedScheduling** con mensaje indicando que el nodo tiene taints no tolerados.
  {: .lab-note .info .compact}

  ```bash
  kubectl run test-no-tolerancia --image=nginx --restart=Never
  ```

  ```bash
  kubectl describe pod test-no-tolerancia | sed -n '/Events/,$p'
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Deployment combinando afinidad y tolerancia

Hacer un deployment con réplicas que usen reglas de afinidad y tolerancia al mismo tiempo.

#### Tarea 6.1

- **Paso 28.** Aplica una etiquta más al nodo `multinode-m03` para la afinidad.

  ```bash
  kubectl label nodes multinodo-m03 rol=dedicado --overwrite
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)  

- **Paso 29.** Este es un resumen de las etiquetas (afinidad) y manchas(taints) que tienen los nodos actualmente.

  ```bash
  kubectl get nodes -l zona=primaria
  ```

  ```bash
  kubectl get nodes -l rol=dedicado
  ```

  ```bash
  kubectl describe node multinodo-m03 | grep Taints:
  ```

  ```bash
  kubectl describe node multinodo-m02 | grep Taints:
  ```

  ![micint]({{ page.images_base | relative_url }}/18.png)  

- **Paso 30.** Abre el archivo `deployment-mix.yaml` y agrega el siguiente código.

  > **Notas**
    - `tolerations` permite entrar al nodo taintado.
    - `nodeAffinity` (preferida) **asigna** los pods a nodos con `rol=dedicado` (tu **multinodo-m03**).
    - `podAntiAffinity` (preferida) intenta que las réplicas no se apilen en el mismo `hostname`.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: deploy-mix
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: mix
    template:
      metadata:
        labels:
          app: mix
      spec:
        # 1) Toleración para poder correr en el nodo taintado: uso=dedicado:NoSchedule
        tolerations:
        - key: "uso"
          operator: "Equal"
          value: "dedicado"
          effect: "NoSchedule"

        # 2) Afinidad a nivel de nodo (preferida) para "atraer" pods al nodo etiquetado.
        affinity:
          nodeAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                - key: rol
                  operator: In
                  values: ["dedicado"]

          # 3) Antiafinidad entre pods (preferida) para no amontonarse en el mismo nodo.
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                  - key: app
                    operator: In
                    values: ["mix"]
                topologyKey: "kubernetes.io/hostname"

        containers:
        - name: nginx
          image: nginx:1.25
          ports:
          - containerPort: 80
  ```

- **Paso 31.** Aplica el manifiesto.

  ```bash
  kubectl apply -f k8s/deployment-mix.yaml
  kubectl rollout status deploy/deploy-mix
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png) 

- **Paso 32.** Ahora, observa la distribución por nodo.

  > **Notas** 
    - Verás **tres pods** distribuidos (idealmente no todos en el mismo hostname gracias a **podAntiAffinity**).
    - Es probable que **al menos 1** caiga en `multinodo-m03` por la combinación de **toleration + nodeAffinity**.
  {: .lab-note .info .compact}

  ```bash
  kubectl get pods -l app=mix -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png) 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza de recursos

Eliminar todos los pods, deployments y taints creados para dejar el entorno limpio.

#### Tarea 7.1

- **Paso 33.** Borra los recursos aplicados.

  ```bash
  kubectl delete -f k8s/pod-afinidad.yaml
  kubectl delete -f k8s/pod-antiafinidad.yaml
  kubectl delete -f k8s/pod-tolerancia.yaml
  kubectl delete -f k8s/deployment-mix.yaml
  ```

- **Paso 34.** Elimina taints y etiquetas en nodos.

  ```bash
  kubectl taint nodes multinodo-m03 uso:NoSchedule-
  kubectl label node multinodo-m02 zona-
  ```
  
- **Paso 35.** (Opcional). Borra todo el clúster.

  ```bash
  minikube delete -p multinodo
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
