# Tercer Entregable Virtualización de Sistemas.

En este repositorio, **fork del original**, concretamente en la rama main, creada a partir de la rama master mediante un **git checkout -b main**, realizaremos el entregable tercero de la asignatura Virtualización de Sistemas, que abarca las tecnologías de **Terraform**, **Sistema de Control de Versiones SCV** y **Jenkins**.

### Autores
  * Álvaro Pérez Vargas **EHDEPELUCHE**
  * Mª Elena Vázquez Rodríguez **elenavaz**

## Índice
  * [Creación de la imagen de Jenkins](#creación-de-la-imagen-de-jenkins)
  * [Despliegue de los contenedores mediante Terraform](#despliegue-de-los-contenedores-mediante-terraform)
  * [Configuración de Jenkins](#configuración-de-jenkins)
    * [Creación del Pipeline](#creación-del-pipeline)

## Creación de la imagen de Jenkins

**Jenkins** está de por sí disponible como imagen para **docker** pero no incluye el plug-in de **Blue Ocean** por lo que vamos a hacer una instalación personalizada de Jenkins en docker y, para ello, haremos uso de un **Dockerfile** desde el que crearemos nuestra imagen personalizada.

El fichero **Dockerfile** es el siguiente:

```Dockerfile
FROM jenkins/jenkins:2.479.2-jdk17

USER root

RUN apt-get update && apt-get install -y lsb-release

RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg

RUN echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list

RUN apt-get update && apt-get install -y docker-ce-cli

USER jenkins

RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
```

A continuación una explicación línea a línea del fichero:
  * <u>**FROM** jenkins/jenkins:2.479.2-jdk17</u> indica que queremos usar la imagen **jenkins** del usuario **jenkins**, concretamente la que tiene la etiqueta **2.479.2-jdk17**.
  * <u>**USER** root</u> nos indica que todas las demás líneas a continuación se ejecutarán como si fueran usuario root, es decir, con **privilegios elevados**.
  * <u>**RUN** apt-get update && apt-get install -y lsb-release</u> realiza dos acciones: primero **actualiza el repositorio de apt-get** y posteriormente instala el paquete de **lsb-release**.
  * <u>**RUN** curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc https://download.docker.com/linux/debian/gpg</u> descarga un **archivo de clave gpg** desde la URL indicada y lo guarda en el directorio **keyrings** con el nombre de **docker-archive-keyring.asc**.
  * <u>**RUN** echo "deb [arch=\$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list</u> con **echo** añadimos una línea en el **archivo de fuentes** con la url de docker para debian. Obtiene una versión o otra dependiendo de la arquitectura del ordenador que esté ejecutando la línea.
  * <u>**RUN** apt-get update && apt-get install -y docker-ce-cli</u> vuelve a actualizar las fuentes e **instala docker**.
  * <u>**USER** jenkins</u> nos indica que todas las demás líneas a continuación se ejecutarán como si fueran **usuario jenkins**, que es el que tiene acceso por defecto a la orden de la siguiente línea.
  * <u>**RUN** jenkins-plugin-cli --plugins "blueocean docker-workflow"</u> mediante el uso de **jenkins-plugin-cli** instalamos el plugin que necesitamos, es decir, **blueocean** y **docker-workflow**.

Tras realizar la **build** o construcción del Dockerfile arriba descrito mediante la orden: 

```bash
docker build -t myjenkins-blueocean .
```

tendremos una imagen llamada **myjenkins-blueocean** creada a partir de la imagen original de jenkins pero con las adaptaciones hechas en el Dockerfile. A continuación veremos el despliegue de dos contenedores, uno a partir de la imagen ahora creada, y otra a partir de la imagen de docker **dind**.

## Despliegue de los contenedores mediante Terraform

Para poder desplegar ambos contenedores, el de **dind** y el de **myjenkins-blueocean**, de forma más sencilla y automatizando la creación y configuración de redes y volúmenes haremos uso de un fichero de terraform llamado **Despliegue.tf**. 

Aquí el contenido del fichero:

```Dockerfile
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}

provider "docker" {
  host = "npipe:////./pipe/docker_engine"
}

resource "docker_network" "jenkins" {
  name = "jenkins"
}

resource "docker_volume" "jenkins_docker_certs" {
  name = "jenkins-docker-certs"
}

resource "docker_volume" "jenkins_data" {
  name = "jenkins-data"
}

resource "docker_container" "jenkins_docker" {
  name  = "jenkins-docker"
  image = "docker:dind"
  privileged = true
  restart = "no"
  networks_advanced {
    name = docker_network.jenkins.name
    aliases = ["docker"]
  }
  env = [
    "DOCKER_TLS_CERTDIR=/certs"
  ]
  volumes {
    volume_name    = docker_volume.jenkins_docker_certs.name
    container_path = "/certs/client"
  }
  volumes {
    volume_name    = docker_volume.jenkins_data.name
    container_path = "/var/jenkins_home"
  }
  ports {
    internal = 2376
    external = 2376
  }
  storage_opts = {
    "overlay2.override_kernel_check" = "1"
  }
}

resource "docker_container" "jenkins_blueocean" {
  name  = "jenkins-blueocean"
  image = "myjenkins-blueocean"
  restart = "on-failure"
  networks_advanced {
    name = docker_network.jenkins.name
  }
  env = [
    "DOCKER_HOST=tcp://docker:2376",
    "DOCKER_CERT_PATH=/certs/client",
    "DOCKER_TLS_VERIFY=1"
  ]
  volumes {
    volume_name    = docker_volume.jenkins_data.name
    container_path = "/var/jenkins_home"
  }
  volumes {
    volume_name    = docker_volume.jenkins_docker_certs.name
    container_path = "/certs/client"
    read_only      = true
  }
  ports {
    internal = 8080
    external = 8080
  }
}
```

A continuación una explicación por bloques del fichero:
  * Bloque **terraform {}**: Incluye el bloque **required_providers {}** que indica a terraform que queremos usar docker como su provider.
  * Bloque **provider "docker" {}**: Indica la dirección del host Docker al que terraform se va a conectar, en este caso **"npipe:////./pipe/docker_engine"**.
  * Bloques **resource ... {}**: Son los bloques de mayor importancia ya que indican los recursos que va a gestionar terraform. Nosotros utilizaremos cinco:
    * Bloque **resource "docker_network" "jenkins" {}**: declara la red **jenkins**, a la que da de nombre jenkins.
    * Bloque **resource "docker_volume" "jenkins_docker_certs" {}**: declara el volumen **jenkins_docker_certs** al que da el nombre jenkins_docker_certs.
    * Bloque **resource "docker_volume" "jenkins_data" {}**: declara el volumen **jenkins_data** al que da el nombre jenkins_data.
    * Bloque **resource "docker_container" "jenkins_docker" {}**: es el bloque que define el contenedor de docker que va a ejecutar las órdenes de jenkins. Definimos un nombre para el contenedor (**jenkins-docker**), la imagen que debe usar (**docker:dind**), se le asigna permisos elevados (**privileged = true**) y se indica que no se reinicie con **restart = "no"**. Además se le asigna la red anteriormente creada bajo el alias de **docker**, los volúmenes anteriormente creados, que se ligan con las correspondientes carpetas, se crea una variable llamada **env** que especifica en este caso el directorio donde va a almacenar docker los certificados TLS y se promociona el puerto **2376** para que el contenedor de jenkins pueda hablar con el de docker por ese puerto. Finalmente se define la variable **storage_opts** con la clave **"overlay2.override_kernel_check"** con el valor de **"1"** o true, habilitando una opción específica de overlay2, que es el controlador de almacenamiento.
    * Bloque **resource "docker_container" "jenkins_blueocean" {}**: es el bloque que define el contenedor de jenkins. Definimos un nombre para el contenedor (**jenkins-blueocean**), la imagen que debe usar (**myjenkins-blueocean**), que es la que creamos anteriormente mediante el Dockerfile y se le indica que solo se reinicie si falla con **restart = "on-failure"**. Al igual que con el contenedor de docker se le asigna la red común creada anteriormente, se declara una variable **env** en la que definimos dónde encontrar al contenedor de docker, dónde almacenar los certificados y que use la capa de verificación TLS al interactuar con docker mediante **"DOCKER_TLS_VERIFY=1"**, se asignan los volúmenes con su ruta y, en el caso del volumen de los certificados, indicamos que sea solo de lectura con **read_only = true** y para terminar promocionamos el puerto **8080** para poder acceder a jenkins desde **localhost:8080**.

Podemos hacer que los contenedores se pongan a correr ejecutando en una terminal en el directorio en el que se encuentra el fichero **Despliegue.tf** la siguiente orden:

```bash
terraform init
terraform apply
```

que inicializan un directorio de trabajo terraform y aplican los cambios descritos en **Despliegue.tf**, creando los recursos pertinentes tras recibir una confirmación desde teclado.

Ahora debemos hacer la instalación de jenkins. Para ello vamos a usar cualquier navegador y vamos a entrar a la dirección **localhost:8080** para empezar la instalación.

## Configuración de Jenkins

Cuando desplegamos los contenedores y accedemos a **localhost:8080** encontramos una pantalla que nos solicita una contraseña de administrador. Para ver dicha contraseña vamos a ver los logs del contenedor de jenkins. Para ello vamos a ejecutar en la terminal la siguiente orden:

```bash
docker logs jenkins-blueocean
```

cuya salida incluye algo similar a lo siguiente:

```bash
(...)

*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

8ccada3616454f5094f4c7efd9249d7f

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************

(...)
```

De ahí, podemos copiar la contraseña e introducirla en el navegador con jenkins.

Tras meter la contraseña en el navegador y pulsar en siguiente vamos a elegir el tipo de instalación. En nuestro caso hemos escogido los plugins sugeridos y, tras esperar a que se instalen, podremos pasar a introducir el usuario administrador, modificar la URL de jenkins, si queremos, y finalmente podemos empezar a usar jenkins.

A partir de ahora vamos a seguir las instrucciones del tutorial para poder generar nuestro propio fichero **Jenkinsfile**.

### Creación del Pipeline

Para poder crear nuestro pipeline, lo primero que debemos hacer es hacer click en **Nueva Tarea** tras lo cual vamos a introducir el nombre de nuestro proyecto, **Entregable3-VS** en nuestro caso e indicar que es un Pipeline. 

Además, y antes de guardar la tarea vamos a editar la **Definition** y vamos a decirle que queremos que haga uso de un **SCM**, concretamente de git y le pasamos la URL de nuestro repositorio: **https://github.com/EHDEPELUCHE/Entregable3-VS** con la rama **main** y, debido a cómo tenemos estructurado el proyecto, le indicamos que el fichero **Jenkinsfile** lo va a encontrar en la ruta **docs/Jenkinsfile**. Ahora podemos guardar la tarea.

Ya con la tarea creada en jenkins vamos a crear el fichero **Jenkinsfile** (en la ruta que hemos indicado en la tarea) y pasamos a incluir las instrucciones propuestas en el tutorial:

```groovy
pipeline {
    agent any 
    stages {
        stage('Build') { 
            steps {
                sh 'python -m py_compile https://github.com/EHDEPELUCHE/Entregable3-VS/tree/main/sources/add2vals.py https://github.com/EHDEPELUCHE/Entregable3-VS/tree/main/sources/calc.py' 
                stash(name: 'compiled-results', includes: 'https://github.com/EHDEPELUCHE/Entregable3-VS/tree/main/sources/*.py*') 
            }
        }
    }
}
```
Lo que hace el **Jenkinsfile** arriba descrito indica que le vale cualquier agente (agent any), declara una fase, build, e incluye dos pasos pasa esa fase:

```bash
sh 'python -m py_compile https://github.com/EHDEPELUCHE/Entregable3-VS/tree/main/sources/add2vals.py https://github.com/EHDEPELUCHE/Entregable3-VS/tree/main/sources/calc.py' 
stash(name: 'compiled-results', includes: 'https://github.com/EHDEPELUCHE/Entregable3-VS/tree/main/sources/*.py*') 
```

El primero de estos pasos ejecuta el comando que **compila** la aplicación y la librería calc en un único **bytecode** y, el segundo paso, guarda el código source y los ficheros bytecode en el directorio **sources** del repositorio.