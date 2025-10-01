---
layout: lab
title: "Práctica 11. Creación de roles, usuarios y permisos con RBAC"
permalink: /capitulo11/lab11/
images_base: /labs/capitulo11/img
duration: "60 minutos"
objective:
  - Aprender a configurar **Roles**, **RoleBindings**, **ClusterRoles** y **ClusterRoleBindings** en Kubernetes para controlar el acceso a recursos, creando usuarios de servicio y limitando sus permisos mediante **RBAC (Role-Based Access Control)**.
prerequisites:
  - Visual Studio Code  
  - Docker Desktop o Minikube con Kubernetes habilitado  
  - kubectl instalado y configurado  
  - Terminal **Git Bash** dentro de VS Code  
  - Conocimientos básicos de Pods, Namespaces y Services
introduction: |
  Los **RBAC (Role-Based Access Control)** en Kubernetes permite administrar quién puede hacer qué en el clúster. Con RBAC defines **Roles** (permisos) y los asignas a **usuarios/ServiceAccounts** mediante **Bindings**.

  En esta práctica crearás un **Namespace**, un **Role** que solo permita listar Pods, un **ServiceAccount**, y un **RoleBinding** para asociar ambos. Además, experimentarás con otros permisos como acceso restringido a ConfigMaps y Deployments, y observarás cómo se aplican las limitaciones de seguridad.
slug: lab11
lab_number: 11
final_result: |
  Has configurado **RBAC en Kubernetes** con Roles, RoleBindings, ClusterRoles y ClusterRoleBindings.

  El ServiceAccount `lector-pods` tiene permisos limitados y ampliados según la configuración:  
  - Solo puede listar Pods en un Namespace.  
  - Puede listar ConfigMaps.  
  - No puede escalar Deployments.  
  - Puede listar recursos a nivel global con un ClusterRole.  
notes:
  - En producción, se recomienda usar **principio de menor privilegio**.  
  - Usa **ClusterRoles** solo cuando sea necesario compartir permisos globales.  
  - Los tokens de ServiceAccounts pueden rotar, en versiones recientes se recomienda usar **kubeconfig con certificados** para usuarios humanos.  
  - Siempre organiza tus manifiestos en carpetas claras para mayor trazabilidad.
references:
  - text: RBAC Kubernetes
    url: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
  - text: ServiceAccounts
    url: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
  - text: Roles y ClusterRoles
    url: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole 
  - text: Seguridad en Kubernetes
    url: https://kubernetes.io/docs/concepts/security/overview/  
prev: /capitulo10/lab10/          
next: /capitulo12/lab12/
---


---

### Tarea 1. Crear estructura del proyecto

Organizar los manifiestos YAML de RBAC en una carpeta dedicada.

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

- **Paso 6.** Crea el directorio para trabajar en la **práctica**.

  > **Notas**
  - Aislar cada práctica evita colisiones de archivos y facilita montar rutas con precisión.
  {: .lab-note .info .compact}

  ```bash
  mkdir lab11-k8srbac && cd lab11-k8srbac
  ```

- **Paso 7.** Valida en el **Explorador** de archivos dentro de VS Code que se haya creado el directorio.

  > **Nota.** Trabajar en VS Code permite editar y versionar cómodamente. **Git Bash** brinda compatibilidad con comandos POSIX.
  {: .lab-note .info .compact}

  ![micint]({{ page.images_base | relative_url }}/4.png)

- **Paso 8.** Esta será la estructura de los directorios de la aplicación.

  ```text
  lab11-k8srbac/
  ├── k8s/
  │   ├── namespace.yaml
  │   └── serviceaccount.yaml
  │   └── role.yaml
  │   └── rolebinding.yaml
  │   └── test-pod.yaml
  │   └── role-configmap.yaml
  │   └── role-deployment.yaml
  │___└── clusterrole-view.yaml
  ```

- **Paso 9.** Ahora, crea la carpeta **k8s/** y sus archivos vacíos.

  > **Nota.** El comando se ejecuta desde la raíz de la carpeta **lab11-k8srbac**.
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s && touch k8s/namespace.yaml k8s/serviceaccount.yaml k8s/role.yaml k8s/rolebinding.yaml k8s/test-pod.yaml k8s/role-configmap.yaml k8s/role-deployment.yaml k8s/clusterrole-view.yaml
  ```

- **Paso 10.** Valida la creación de la estructura de tu proyecto. Escribe el siguiente comando.

  > **Nota.** Recuerda que tambien puedes visualizarlos en el explorador de archivos de VS Code.
  {: .lab-note .info .compact}

  ```bash
  ls -la -R
  ```

  ![micint]({{ page.images_base | relative_url }}/5.png)
    
{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear Namespace y ServiceAccount

Definirás un nuevo Namespace y un ServiceAccount para aislar recursos y usuarios.

#### Tarea 2.1

- **Paso 11.** Primero asegúrate de tener encendido el servidor de **Minikube**. Ejecuta el siguiente comando.

  > **Nota.** Si ya está encendido, puedes avanzar al siguiente paso.
  {: .lab-note .info .compact}

  ```bash
  minikube start
  ```

- **Paso 12.** Abre el archivo `k8s/namespace.yaml` y agrega la configuración para crear el namespace.

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: demo-rbac
  ```

- **Paso 13.** Ahora, aplica el manifiesto y verifica que se haya creado correctamente.

  ```bash
  kubectl apply -f k8s/namespace.yaml
  kubectl get ns demo-rbac --show-labels
  ```

  ![micint]({{ page.images_base | relative_url }}/6.png)

- **Paso 14.** Ahora, agrega el siguiente contenido al archivo `k8s/serviceaccount.yaml`.

  > **Nota.** El ServiceAccount será la **`identidad`** dentro del Namespace que usarás para restringir permisos.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: lector-pods
    namespace: demo-rbac
  ```

- **Paso 15.** Ahora, aplica el manifiesto y verifica que se haya creado correctamente.

  ```bash
  kubectl apply -f k8s/serviceaccount.yaml
  kubectl get sa lector-pods -n demo-rbac -o yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/7.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Definir Role con permisos limitados

Crearás un Role que solo permita **listar y obtener Pods** dentro del Namespace.

#### Tarea 3.1

- **Paso 16.** Abre el archivo `k8s/role.yaml` y agrega la siguiente configuración para el rol.

  > **Nota.** El Role define **qué acciones (verbs)** se permiten sobre **qué recursos** en un Namespace específico.
  {: .lab-note .info .compact}
  
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: demo-rbac
    name: rol-lector-pods
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  ```

- **Paso 17.** Ahora, aplica el manifiesto y verifica que se haya creado correctamente.

  ```bash
  kubectl apply -f k8s/role.yaml
  kubectl get role -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/8.png)

- **Paso 18.** Verifica que los permisos estén aplicados a los pods. Escribe el comando **`describe`** para ver los detalles.

  ```bash
  kubectl describe role rol-lector-pods -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/9.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear RoleBinding

Vincularás el **Role** con el **ServiceAccount** para que tenga los permisos definidos.

#### Tarea 4.1

- **Paso 19.** Ahora, dentro del archivo `k8s/rolebinding.yaml`, agrega el siguiente código para crear la relación entre los objetos.

  > **Nota.** El RoleBinding es el **puente** que conecta Roles (`permisos`) con sujetos (`usuarios`, `grupos` o `ServiceAccounts`).
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: lector-pods-binding
    namespace: demo-rbac
  subjects:
  - kind: ServiceAccount
    name: lector-pods
    namespace: demo-rbac
  roleRef:
    kind: Role
    name: rol-lector-pods
    apiGroup: rbac.authorization.k8s.io
  ```

- **Paso 20.** Ahora, aplica el manifiesto y verifica que se haya creado correctamente.

  ```bash
  kubectl apply -f k8s/rolebinding.yaml
  kubectl get rolebinding lector-pods-binding -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/10.png)

- **Paso 21.** Verifica que exista la relación entre los dos objetos.

  ```bash
  kubectl describe rolebinding lector-pods-binding -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Probar acceso con el ServiceAccount

Validarás que el ServiceAccount puede **listar pods**, pero no puede crear ni eliminarlos.

#### Tarea 5.1

- **Paso 22.** Abre el archivo `k8s/test-pod.yaml` y agrega la siguiente configuración de un pod.

  > **Nota.** Este **pod** se usará para probar los permisos **RBAC**.
  {: .lab-note .info .compact}

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-test
    namespace: demo-rbac
  spec:
    containers:
    - name: nginx
      image: nginx:1.25
  ```

- **Paso 23.** Ahora, aplica el manifiesto y verifica que se haya creado correctamente.

  > **Nota.** Si el pod no está en **Running**, espera unos segundos y vuelve a probar.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply -f k8s/test-pod.yaml
  kubectl get pods -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/12.png)

#### Tarea 5.2 Impersonación

- **Paso 24.** Primero, comprueba los permisos **teóricamente**.

  > **Nota.** El **API server** evalúa la acción como si fuera el SA. Esperado: **list** = `yes`, **create** = `no`.
  {: .lab-note .info .compact}

  ```bash
  kubectl auth can-i --as=system:serviceaccount:demo-rbac:lector-pods list pods -n demo-rbac
  kubectl auth can-i --as=system:serviceaccount:demo-rbac:lector-pods create pods -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/13.png)

- **Paso 25.** Ahora, prueba la lista (OK) y creación (Forbidden) con impersonación.

  - Debe listar

  ```bash
  kubectl --as=system:serviceaccount:demo-rbac:lector-pods -n demo-rbac get pods
  ```

  - Debe fallar

  ```bash
  kubectl --as=system:serviceaccount:demo-rbac:lector-pods -n demo-rbac run test-imp --image=nginx
  ```

  ![micint]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Role extra para ConfigMaps
 
Configurarás un Role que permita **listar y obtener ConfigMaps**.

#### Tarea 6.1

- **Paso 26.** Ahora, abre el archivo `k8s/role-configmap.yaml` y agrega la siguiente configuración para el rol.

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: demo-rbac
    name: rol-lector-configmaps
  rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  ```

- **Paso 27.** Ahora, aplica el manifiesto y verifica que se haya creado correctamente.

  ```bash
  kubectl apply -f k8s/role-configmap.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/15.png)

- **Paso 28.** Crea un ConfigMap de ejemplo para después validar los permisos. Escribe el siguiente comando.

  ```bash
  kubectl create configmap demo-config --from-literal=key=value -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/31.png)

- **Paso 29.** Ahora, crea un nuevo Role Binding para asociar el SA al nuevo rol de ConfigMaps.

  ```bash
  touch k8s/rolebinding-cm.yaml
  code k8s/rolebinding-cm.yaml
  ```

- **Paso 30.** Abre el archivo `k8s/rolebinding-cm.yaml` y agrega la siguiente configuración de asociación.

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: lector-cm-binding
    namespace: demo-rbac
  subjects:
  - kind: ServiceAccount
    name: lector-pods
    namespace: demo-rbac
  roleRef:
    kind: Role
    name: rol-lector-configmaps
    apiGroup: rbac.authorization.k8s.io
  ```

- **Paso 31.** Aplica la configuración del manifiesto. Escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/rolebinding-cm.yaml
  ```

- **Paso 32.** Ahora, valida el resultado del acceso al ConfigMap, usando los permisos del SA.

  ```bash
  kubectl auth can-i get configmaps
  ```

  ![micint]({{ page.images_base | relative_url }}/16.png)

- **Paso 33.** Aplica el rol y prueba el acceso.

  > **Nota.** Controlar el acceso a ConfigMaps es útil cuando un equipo solo debe **leer configuraciones** sin modificarlas.
  {: .lab-note .info .compact}


  ```bash
  kubectl --as=system:serviceaccount:demo-rbac:lector-pods -n demo-rbac get ConfigMaps
  ```

  ![micint]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Role extra para Deployments

Crear un Role que permita **listar y escalar Deployments**.

#### Tarea 7.1

- **Paso 34.** Abre el archivo `k8s/role-deployment.yaml` que otorga el permiso para los deployments.

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: demo-rbac
    name: rol-gestion-deployments
  rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  ```

- **Paso 35.** Aplica la configuración del manifiesto. Escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/role-deployment.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/18.png)

- **Paso 36.** Ahora, crea un nuevo Role Binding para asociar el SA al nuevo rol de Deployments.

  ```bash
  touch k8s/rolebinding-deploy.yaml
  code k8s/rolebinding-deploy.yaml
  ```

- **Paso 37.** Abre el archivo `k8s/rolebinding-deploy.yaml` y agrega la siguiente configuración de asociación.

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: gestion-deploy-binding
    namespace: demo-rbac
  subjects:
  - kind: ServiceAccount
    name: lector-pods
    namespace: demo-rbac
  roleRef:
    kind: Role
    name: rol-gestion-deployments
    apiGroup: rbac.authorization.k8s.io
  ```

- **Paso 38.** Aplica la configuración del manifiesto. Escribe el siguiente comando.

  ```bash
  kubectl apply -f k8s/rolebinding-deploy.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/19.png)

- **Paso 39.** Ahora, haz la prueba. Crea el siguiente deployment para revisar si tienes permiso o no.

  ```bash
  kubectl create deployment nginx-deploy --image=nginx -n demo-rbac
  ```

  ![micint]({{ page.images_base | relative_url }}/20.png)

- **Paso 40.** Valida teóricamente que si pueda escalar los pods. Escribe el siguiente comando.

  > **Nota.** El resultado es correcto porque estás comprobando que **update** y **kubectl scale** hacen una solicitud `PATCH` al subrecurso deployments/scale.
  {: .lab-note .info .compact}

  ```bash
  kubectl auth can-i --as=system:serviceaccount:demo-rbac:lector-pods -n demo-rbac update deployments/scale
  ```

  ![micint]({{ page.images_base | relative_url }}/21.png)

- **Paso 41.** Ahora, verifica que realmente **no** podrás escalar el deployment.

  > **Nota.** Tu Role permite operar sobre el recurso `deployments`, pero **`kubectl scale`** usa el **subrecurso** `deployments/scale` y eso no está permitido.
  {: .lab-note .info .compact}

  ```bash
  kubectl --as=system:serviceaccount:demo-rbac:lector-pods -n demo-rbac scale deployment nginx-deploy --replicas=3
  ```

  ![micint]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. ClusterRole de solo lectura global

Probarás un **ClusterRole** con permisos de solo lectura para todos los Namespaces.

#### Tarea 8.1

- **Paso 42.** Abre el archivo `k8s/clusterrole-view.yaml` y agrega las siguientes configuraciones.

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: cluster-lector
  rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list"]
  ```

- **Paso 43.** Aplica la configuración para el Cluster Role.

  ```bash
  kubectl apply -f k8s/clusterrole-view.yaml
  ```

  ![micint]({{ page.images_base | relative_url }}/23.png)

- **Paso 44.** Ahora, crea el Cluster Role Binding directamente en kubernetes. Escribe el siguiente comando.

  ```bash
  kubectl create clusterrolebinding lector-global --clusterrole=cluster-lector --serviceaccount=demo-rbac:lector-pods
  ```

  ![micint]({{ page.images_base | relative_url }}/24.png)

- **Paso 45.** Verifica que, teóricamente, puedas tener acceso en todo el clúster.

  ```bash
  kubectl auth can-i --as=system:serviceaccount:demo-rbac:lector-pods list pods -A
  ```

  ![micint]({{ page.images_base | relative_url }}/25.png)

- **Paso 46.** Ahora, valida que correctamente puedas ver **todos** los pods de todo el clúster.

  > **Nota.** El ClusterRole amplía permisos a nivel de todo el clúster.
  {: .lab-note .info .compact}

  ```bash
  kubectl --as=system:serviceaccount:demo-rbac:lector-pods get pods -A
  ```

  ![micint]({{ page.images_base | relative_url }}/26.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Limpieza (rollback total)

Eliminar objetos globales y el namespace con todo su contenido, además de archivos temporales.

#### Tarea 9.1

- **Paso 47.** Inspecciona los recursos que borrarás.

  ```bash
  kubectl get ns demo-rbac || true
  kubectl get clusterrole cluster-lector || true
  kubectl get clusterrolebinding lector-global || true
  ```

  ![micint]({{ page.images_base | relative_url }}/27.png) 

- **Paso 48.** Primero, elimina los objetos **Globales**.

  ```bash
  kubectl delete clusterrolebinding lector-global --ignore-not-found
  kubectl delete clusterrole cluster-lector --ignore-not-found
  ```

  ![micint]({{ page.images_base | relative_url }}/28.png) 

- **Paso 49.** Ahora, elimina el namespace (borra SA, Roles, Bindings, Pods, entre otros objetos).

  > **Nota.** Puede tardar unos segundos en borrar.
  {: .lab-note .info .compact}

  ```bash
  kubectl delete ns demo-rbac --ignore-not-found
  ```

  ![micint]({{ page.images_base | relative_url }}/29.png)

- **Paso 50.** Elimina artefactos locales que se hayan quedado escondidos.

  > **Nota.** Aunque el comando no genera salida, aplícalo.
  {: .lab-note .info .compact}

  ```bash
  rm -f /tmp/kcfg-lector /tmp/rbac-ca.crt 2>/dev/null || true
  ```

  ![micint]({{ page.images_base | relative_url }}/30.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
