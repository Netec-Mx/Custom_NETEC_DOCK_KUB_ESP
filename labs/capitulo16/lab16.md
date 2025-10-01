---
layout: lab
title: "Práctica 16. Observabilidad de kubernetes"
permalink: /capitulo16/lab16/
images_base: /labs/capitulo16/img
duration: "60 minutos"
objective:
  - Aprenderás a instalar **Metrics Server** en un clúster multinodo de Kubernetes y a usarlo para visualizar métricas de consumo de CPU y memoria de pods y nodos. También, desplegarás pods de prueba para ver el consumo en diferentes nodos.
prerequisites:
  - Visual Studio Code  
  - Minikube en modo multinodo
  - kubectl instalado y configurado  
  - Docker Desktop como backend de Minikube  
  - Terminal **Git Bash** dentro de VS Code   
introduction: |
  El **Metrics Server** es un agregador de métricas liviano en Kubernetes. Recoge datos de uso de **CPU y memoria** de los nodos y pods a través de los kubelets y los pone a disposición de la API `metrics.k8s.io`. Es fundamental para que funcionen herramientas como el **Horizontal Pod Autoscaler (HPA)**. 

  En esta práctica lo instalarás, validarás su funcionamiento y desplegarás pods que consuman recursos en distintos nodos para visualizar sus métricas.
slug: lab16
lab_number: 16
final_result: >
  Instalaste el **Metrics Server**, confirmaste que recopila métricas de pods y nodos y desplegaste pods de prueba que consumen recursos para visualizar el impacto. Finalmente, limpiaste los recursos creados.
notes:
  - El Metrics Server no almacena métricas históricas, solo ofrece datos en tiempo real.  
  - Para gráficas y monitoreo a largo plazo, considera Prometheus y Grafana.  
  - En clústeres grandes, ajusta las opciones de recursos del Metrics Server para evitar cuellos de botella.  
references:
  - text: Metrics Server
    url: https://github.com/kubernetes-sigs/metrics-server  
  - text: kubectl top
    url: https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/ 
  - text: Imagen de stress
    url: https://hub.docker.com/r/polinux/stress
prev: /capitulo15/lab15/          
next: /capitulo1/lab1/
---


---

### Tarea 1. Crear clúster Minikube multinodo

Configurar Minikube para ejecutar un clúster con tres nodos (un máster y dos workers).

#### Tarea 1.1

- **Paso 1.** Inicia sesión en tu máquina de trabajo como usuario con permisos administrativos. 

- **Paso 2.** Abre el **`Visual Studio Code`**, lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- **Paso 3.** Una vez abierto **VS Code**, da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  ![micint]({{ page.images_base | relative_url }}/1.png)

- **Paso 4.** Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  ![micint]({{ page.images_base | relative_url }}/2.png)

- **Paso 5.** Asegúrate de estar dentro de la carpeta del curso llamada **dockerlabs** en la terminal de **VS Code**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para volver a la raíz de laboratorios.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/3.png)

- **Paso 6.** Primero, ejecuta el siguiente comando para iniciar un clúster multinodo de Minikube.

  > **Importante.** Puede tardar alrededor de **cinco minutos**, espera la creación antes de continuar.
  {: .lab-note .important .compact}
  
  ```bash
  minikube start --nodes=3 -p multinodo
  ```

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 7.** Ya creado el clúster multinodo, puedes verificar los nodos creados con el siguiente comando.

  > **Nota.** Debes ver al menos tres nodos: `multinodo`, `multinodo-m02`, `multinodo-m03`.
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
 
Organizar los manifiestos de la práctica en una carpeta.

#### Tarea 2.1

- **Paso 8.** Crea el directorio para trabajar en la **práctica**.

  > **Nota.** Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab16-k8smetrics && cd lab16-k8smetrics
  ```

- **Paso 9.** Valida en el **Explorador** de archivos dentro de VS Code que se haya creado el directorio.

  > **Nota.** Mantener cada práctica en su carpeta evita confusiones y facilita limpieza/versionado.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Crea la siguiente estructura base de directorios y archivos.

  > **Nota.** Así separas el manifiesto de la carga de prueba de la instalación del Metrics Server.
  {: .lab-note .info .compact}

  ```text
  lab16-k8smetrics/
  ├── k8s/
  └── └── deployment-stress.yaml
  ```

- **Paso 11.** Ahora, crea la carpeta **k8s/** y sus archivos vacíos.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab16-k8smetrics**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/deployment-stress.yaml
  ```

- **Paso 12.** Valida la creación de la estructura de tu proyecto. Escribe el siguiente comando.

  > **Nota.** Recuerda que también puedes visualizarlos en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls -l -R
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Instalar Metrics Server

Instalar el Metrics Server usando los manifiestos oficiales de Kubernetes.

#### Tarea 3.1

- **Paso 13.** Aplica los manifiestos oficiales para instalar Metrics Server.

  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 14.** Ajusta el manifiesto desplegado de Metrics Server.

  > **Nota.** El manifiesto que tienes corre en el contenedor y las sondas apuntan a `https` (por nombre), pero **no existe un puerto nombrado** `https`.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
    {"op":"replace","path":"/spec/template/spec/containers/0/args","value":[
      "--cert-dir=/tmp",
      "--secure-port=4443",
      "--kubelet-insecure-tls",
      "--kubelet-preferred-address-types=InternalIP,Hostname",
      "--metric-resolution=15s"
    ]},
    {"op":"add","path":"/spec/template/spec/containers/0/ports","value":[
      {"name":"https","containerPort":4443}
    ]},
    {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/port","value":"https"},
    {"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/httpGet/port","value":"https"}
  ]'
  ```

- **Paso 15.** Verifica que los pods estén corriendo.

  > **Nota.** Espera unos segundos y realiza la prueba, verifica que el pod esté **`1/1`**.
  {: .lab-note .info .compact}

  ```bash
  kubectl get pods -n kube-system | grep metrics-server
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

- **Paso 16.** El siguiente comando debe indicar `True` en `Available`.

  > **Nota.** El Metrics Server instala un Deployment en el namespace `kube-system` y expone la API `metrics.k8s.io`.
  {: .lab-note .info .compact}

  ```bash
  kubectl get apiservices | grep metrics
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Validar métricas de nodos y pods

Consultar métricas básicas para confirmar que el servidor está funcionando.

#### Tarea 4.1

- **Paso 17.** Obtener métricas de los nodos.

  > **Nota.** Los comandos `kubectl top` consumen datos desde `metrics.k8s.io` y muestran CPU/memoria.
  {: .lab-note .info .compact}

  ```bash
  kubectl top nodes
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

- **Paso 18.** Obtener métricas de pods.

  ```bash
  kubectl top pods -A
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Desplegar pods de prueba que consuman recursos
 
Crear un feployment que genere carga de CPU (y opcionalmente memoria) para observar cómo cambian las métricas del clúster. Usar límites y requests claros para que kubectl top sea más expresivo y el scheduler distribuya correctamente las réplicas.

#### Tarea 5.1

- **Paso 19.** Abre el archivo `k8s/deployment-stress.yaml` y agrega la siguiente configuración.

  > **Nota.** La imagen `polinux/stress` genera carga **determinística**. Con requests/limits, el scheduler asigna CPU/memoria y el **metrics-server** refleja consumo real.
  {: .lab-note .info .compact}


  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: stress-test
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: stress
    template:
      metadata:
        labels:
          app: stress
      spec:
        # Intenta repartir las réplicas entre nodos
        topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: stress
        containers:
        - name: stress
          image: polinux/stress
          command: ["stress"]  
          # Límites/requests para que el scheduler y metrics tengan datos claros
          resources:
            limits:
              cpu: "200m"
              memory: "128Mi"
            requests:
              cpu: "100m"
              memory: "64Mi"
          # 2 workers de CPU durante 10 min
          args: ["--cpu", "2", "--timeout", "600s"]
  ```

- **Paso 20.** Aplica el Deployment.

  ```bash
  kubectl apply -f k8s/deployment-stress.yaml
  kubectl rollout status deploy/stress-test
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 21.** Verifica los logs de un pod desplegado por el deployment.

  ```bash
  P=$(kubectl get pods -l app=stress -o jsonpath='{.items[0].metadata.name}')
  kubectl describe pod "$P" | sed -n '/Events/,$p'
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

- **Paso 22.** Ve la ubicación por nodo (útil en clúster multinodo).

  > **Nota.** Debes ver tres pods en Running. En multinodo, lo normal es que queden repartidos (gracias a `topologySpreadConstraints`).
  {: .lab-note .info .compact}

  ```bash
  kubectl get pods -l app=stress -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png) 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Consultar métricas de los pods de carga

**Medir** el consumo de CPU o memoria de los pods con `kubectl top` y observar el impacto por nodo.

#### Tarea 6.1

- **Paso 23.** Espera unos segundos para que el metrics-server recolecte (15–30 s) y ejecuta.

  ```bash
  kubectl top pods -l app=stress
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png) 

- **Paso 24.** Verifica métricas de nodos.

  > **Nota.** Verás cómo los recursos solicitados y consumidos aparecen en la salida de métricas, distribuidos entre nodos del clúster multinodo.
  {: .lab-note .info .compact}

  ```bash
  kubectl top nodes
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png) 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Saturación manual y escalado manual de réplicas

**Forzar carga extra** en el clúster (CPU) y luego **escalar manualmente** el deployment para repartir la presión entre más pods con el fin de observar, sin HPA, cómo cambia el consumo cuando aumentas réplicas.

#### Tarea 7.1. Inyectar más carga temporal

- **Paso 25.** Verifica las estadisticas de recursos de los nodos, antes de realizar la saturación.

  ```bash
  kubectl top nodes
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png) 

- **Paso 26.** Lanza un pod temporal que genere carga adicional (no forma parte del deployment).

  > **Nota.** Este pod aislado añade **picos de CPU**. No pertenece al deployment `stress-test`, así separas la **"carga fondo"** (deployment) de la **"carga pico"** (pod suelto).
  {: .lab-note .info .compact}

  ```bash
  kubectl run stress-burst --image=polinux/stress --restart=Never \
  --command -- stress --cpu 3 --timeout 600s
  ```

- **Paso 27.** Verifica que esté creado el pod, `1/1`.

  ```bash
  kubectl get pod stress-burst
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png) 

- **Paso 28.** Cuando esté **Running**, consulta métricas a los **~15–30 s**.

  ```bash
  kubectl top nodes
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)

#### Tarea 7.2. Escalar manualmente el Deployment

- **Paso 29.** Sube las réplicas de **stress-test** de `3 a 6`.

  > **Nota.** Puede tardar unos segundos la escalabilidad.
  {: .lab-note .info .compact}

  ```bash
  kubectl scale deploy/stress-test --replicas=6
  kubectl rollout status deploy/stress-test
  ```

  ![micint]({{ page.images_base | relative_url }}/22.png)

- **Paso 30.** Valida que los pods de réplica se hayan creado correctamente.

  ```bash
  kubectl get pods -l app=stress -o wide
  ```

  ![micint]({{ page.images_base | relative_url }}/23.png)

- **Paso 31.** Observa el CPU por pod.

  ```bash
  kubectl top pods -l app=stress
  ```

  ![micint]({{ page.images_base | relative_url }}/24.png)

- **Paso 32.** Ahora, observa el CPU por nodo.

  > **Nota.** Al **duplicar réplicas**, distribuyes el trabajo y el consumo por pod **tiende a bajar** (misma presión, más "trabajadores"). También, puedes ver más nodos involucrados si hay capacidad.
  {: .lab-note .info .compact}

  ```bash
  kubectl top nodes
  ```

  ![micint]({{ page.images_base | relative_url }}/25.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza de recursos

Eliminar los pods de carga para no afectar al clúster.

#### Tarea 8.1

- **Paso 33.** Borra el deployment.

  ```bash
  kubectl delete -f k8s/deployment-stress.yaml
  ```

- **Paso 34.** Borra el Metrics Server.

  ```bash
  kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```

- **Paso 35.** Borra todo el clúster multinodo.

  ```bash
  minikube delete -p multinodo
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}  
