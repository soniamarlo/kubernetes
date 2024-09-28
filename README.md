# **Práctica Final: Kubernetes**
<a name="resources"></a>
# Despliegue de Nginx y MariaDB en Kubernetes con Helm

Índice de contenidos:

- [Descripción](#descripcion)
- [Requisitos Previos](#requisitos-previos)
- [Configuración](#configuracion)
- [Estructura del Proyecto](#estructura-del-proyecto)
- [Despliegue de la Aplicación](#despliegue-de-la-aplicacion)
    - [Paso 1: Clonar el Repositorio](#paso-1-clonar-el-repositorio)
    - [Paso 2: Instalación del Chart de Helm](#paso-2-instalacion-del-chart-de-helm)
    - [Paso 3: Crear el Namespace](#paso-3-crear-el-namespace)
    - [Paso 4: Verificación de Recursos](#paso-4-verificacion-de-recursos)
    - [Paso 5: Acceso a la Aplicación](#paso-5-acceso-a-la-aplicacion)
    - [Paso 6: Escalabilidad Automática](#paso-6-escalabilidad-automatica)
- [Configuración de Persistencia](#configuracion-de-persistencia)
- [Documentación de Ingress](#documentacion-de-ingress)
- [Minikube Tunnel](#minikube-tunnel)
- [Detener y Eliminar el Despliegue](#detener-y-eliminar-el-despliegue)
- [Errores Conocidos](#errores-conocidos)

<a name="install-chart"></a>

## **Descripción**

Este proyecto despliega una **aplicación web** utilizando **Nginx** y una base de datos **MariaDB** en un clúster de **Kubernetes**. El despliegue se realiza a través de **Helm**, lo que facilita la configuración, actualización y mantenimiento del entorno de producción.

La aplicación está configurada para soportar **persistencia de datos, alta disponibilidad y escalabilidad automática**, lo que asegura que los servicios puedan manejar carga adicional y recuperarse de fallos. Además, el acceso externo se maneja mediante **Ingress**.
## **Requisitos Previos**

- **Kubernetes**
- **Helm**(para gestionar el despliegue de los charts)

## **Configuración**
El despliegue de Nginx y MariaDB se realiza utilizando variables configuradas en el archivo values.yaml, lo que permite ajustar el número de réplicas, el uso de persistencia y los detalles de la configuración del Ingress para exponer la aplicación externamente.

### **Variables de Entorno**
Las configuraciones críticas como las contraseñas de la base de datos se manejan a través de **Kubernetes Secrets y ConfigMap** y se refieren en los archivos de despliegue.

- **Nginx**: Configurable en el archivo values.yaml, incluyendo el número de réplicas y la configuración del Ingress para exponer la aplicación al exterior.
- **MariaDB**: Configurada con credenciales almacenadas en Secrets, que permiten asegurar la información sensible, y con soporte para almacenamiento persistente mediante Persistent Volume Claims.

## **Estructura del Proyecto**
La estructura del proyecto es la siguiente:

PRACTICA-NGINX-MARIADB/

├── templates/
│   ├── configmap.yaml          # Configuración de variables de entorno

│   ├── deployment.yaml         # Despliegue de Nginx

│   ├── hpa.yaml                # Configuración del autoescalado

│   ├── ingress-mariadb.yaml    # Ingress para MariaDB

│   ├── ingress-nginx.yaml      # Ingress para Nginx

│   ├── pvc.yaml                # Persistent Volume Claim para MariaDB

│   ├── secret.yaml             # Secretos para las credenciales de MariaDB

│   ├── service.yaml            # Servicios para Nginx y MariaDB

│   └── statefulset.yaml        # StatefulSet para MariaDB

├── Chart.yaml                  # Archivo principal del chart de Helm

└── values.yaml                 # Configuraciones generales para el despliegue


## **Despliegue de la aplicación**
### Paso 1: Clonar el Repositorio

Primero, clona el repositorio donde está el chart de Helm de la práctica:

```bash
git clone https://github.com/soniamarlo/practica-nginx-mariadb.git
cd practica-nginx-mariadb
```
### Paso 2: Instalación del Chart de Helm

Para desplegar la aplicación, primero clona el repositorio en tu máquina local y luego ejecuta el siguiente comando para instalar el chart de Helm:
```
helm install nginx-mariadb ./practica-nginx-mariadb --namespace nginx-mariadb --create-namespace
```
Esto instalará Nginx y MariaDB en el clúster de Kubernetes en el namespace nginx-mariadb

### Paso 3: Crear el Namespace

Antes de instalar el chart de Helm, crea el namespace donde se desplegarán los recursos de Nginx y MariaDB:
```
kubectl create namespace nginx-mariadb
```

### Paso 4: Verificación de recursos

Para asegurarte de que todos los recursos (pods, servicios, ingress, volúmenes persistentes, etc.) están correctamente desplegados, puedes ejecutar los siguientes comandos:
- **Verificar pods**
```
kubectl get pods -n nginx-mariadb
```
Esto debería mostrar los pods de Nginx y MariaDB en estado Running
- **Verificar servicios**
```
kubectl get svc -n nginx-mariadb
```
- **Verificar ingress**
```
kubectl get ingress -n nginx-mariadb
```
- **Verificar volúmenes persistentes(PVC)**
```
kubectl get pvc -n nginx-mariadb
```
- **Verificar el autoescalador horizontal(HPA)**
```
kubectl get hpa -n nginx-mariadb
```
### Paso 5: Acceso a la Aplicación
- **Verificación externa del funcionamiento**

Una vez desplegado, Nginx será accesible a través del host definido en el Ingress:

  - Nginx: http://nginx-app.local
  - MariaDB: (aunque MariaDB no tiene una interfaz web, se puede conectar desde el servicio interno o clientes MySQL).

- **Verificación interna del funcionamiento**
Para verificar si Nginx está sirviendo contenido correctamente dentro del clúster, puedes ejecutar un pod temporal y utilizar curl para realizar solicitudes:
```
kubectl run curl-pod --rm -i --tty --image=curlimages/curl --restart=Never --namespace nginx-mariadb -- sh
```
Una vez dentro del pod temporal, ejecuta:
```
curl nginx-service
```
Para verificar si MariaDB está funcionando correctamente, puedes conectarte a MariaDB desde un pod temporal utilizando un cliente MySQL:
```
kubectl run -it --rm --image=mysql:5.7 --namespace nginx-mariadb mysql-client -- bash
```
### Paso 6: Escalabilidad Automática

La configuración del autoescalado está habilitada mediante el archivo hpa.yaml. El autoescalado está configurado para escalar las réplicas de Nginx entre 1 y 5 según la utilización de CPU

Puedes verificar el estado del autoescalador con el siguiente comando:
```
kubectl get hpa -n nginx-mariadb
```
Una vez dentro del pod, conéctate a la base de datos MariaDB:
```
mysql -h mariadb-service -u root -p
```
Introduce la contraseña configurada para MariaDB y luego puedes ejecutar comandos como SHOW DATABASES; para verificar que la base de datos esté accesible.
## **Configuración de Persistencia**

Para asegurar que los datos de MariaDB sean persistentes incluso si los pods se reinician, se configura un Persistent Volume Claim (PVC). El tamaño del almacenamiento persistente se puede ajustar en el archivo values.yaml
```
mariadb:
  persistence:
    enabled: true
    size: 1Gi
```
## **Documentación de Ingress**
El acceso externo está gestionado por Ingress, lo que permite enrutar el tráfico desde fuera del clúster hacia los servicios internos:

  - Nginx Ingress: Exposición del servicio de Nginx.
  - MariaDB Ingress: Aunque no tiene una interfaz visual, permite la conexión desde clientes externos.

El host para cada servicio se configura en el archivo values.yaml:
```
nginx:
  ingress:
    host: nginx-app.local
mariadb:
  ingress:
    host: mariadb-app.local
```
Asegúrate de agregar estas entradas en tu archivo hosts en tu máquina local:
```
<minikube ip> nginx-app.local
<minikube ip> mariadb-app.local
```

Una vez configurado, puedes acceder a Nginx en el navegador utilizando http://nginx-app.local
## **Minikube tunnel**
Para poder acceder a los servicios externos desde tu máquina local, es necesario iniciar un túnel de Minikube. Esto creará una ruta para exponer los servicios de tipo LoadBalancer en Minikube.

Ejecuta el siguiente comando en una terminal con **permisos de administrador**:
```
minikube tunnel
```
Mantén abierta la terminal donde está corriendo el túnel, ya que este proceso debe estar activo para que el acceso externo funcione.
Una vez que el túnel esté activo, podrás acceder a los hosts configurados en el archivo /etc/hosts.
## **Detener y Eliminar el Despliegue**
Para detener y eliminar el despliegue y todos los recursos asociados, ejecuta:
```
helm uninstall nginx-mariadb --namespace nginx-mariadb
```
Esto eliminará el namespace nginx-mariadb y todos los recursos asociados.
## **Errores Conocidos**

A pesar de que la aplicación funciona correctamente de forma interna, con los servicios de **Nginx** y **MariaDB** desplegados en el clúster de Kubernetes, no he logrado acceder a la aplicación desde la URL externa. 

### **Detalles del Problema**:
- Internamente, tanto **Nginx** como **MariaDB** están funcionando correctamente. Esto ha sido verificado mediante la ejecución de pods temporales en el clúster que permiten acceder al servicio de **Nginx** usando `curl` y conectarse a **MariaDB** utilizando un cliente MySQL.
- El acceso externo, que debería estar gestionado por **Ingress** y el servicio **ClusterIP** en **Minikube**, no se ha podido resolver completamente. Las solicitudes a las URLs externas configuradas (como `nginx-app.local` y `mariadb-app.local`) no devuelven contenido, aunque la configuración de Ingress y el túnel de Minikube parecen estar correctos.

Recomiendo utilizar las verificaciones internas para comprobar el correcto funcionamiento de la aplicación.

