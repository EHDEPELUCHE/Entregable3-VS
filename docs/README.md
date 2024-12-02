# Entregable 3 Virtualización de Sistemas.

En este repositorio, **fork del original**, concretamente en la rama main creada a partir de la rama master mediante un **git checkout -b main** realizaremos el entregable 3 de la asignatura Virtualización de Sistemas, que abarca las tecnologías de **Terraform**, **Sistema de Control de Versiones SCV** y **Jenkins**.

### Autores
  * Álvaro Pérez Vargas **EHDEPELUCHE**
  * Mª Elena Vázquez Rodríguez **elenavaz**

## Índice
  * [Creación de la imagen de Jenkins](#creación-de-la-imagen-de-jenkins)
  * [Despliegue de los contenedores mediante Terraform](#despliegue-de-los-contenedores-mediante-terraform)
  * [Configuración de Jenkins](#configuración-de-jenkins)

## Creación de la imagen de Jenkins

**Jenkins** está de por sí disponible como imagen para **docker** pero no incluye el plug-in de **Blue Ocean** por lo que vamos a hacer una instalación personaliazada de Jenkins en docker y, para ello, haremos uso de un **Dockerfile** desde el que crearemos nuestra imagen personalizada.

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
  * <u>**USER** jenkins</u> nos indica que todas las demás líneas a continuación se ejecutarán como si fueran usuario jenkins, que es el que tiene acceso por defecto a la orden de la siguiente línea.
  * <u>**RUN** jenkins-plugin-cli --plugins "blueocean docker-workflow"</u> mediante el uso de **jenkins-plugin-cli** instalamos el plugin que necesitamos, es decir, **blueocean** y **docker-workflow**.

Tras la ejecución del Dockerfile arriba descrito mediante la orden: 

```bash
docker build -t myjenkins-blueocean .
```

tendremos una imagen llamada **myjenkins-blueocean** creada a partir de la imagen original de jenkins pero con las adaptaciones hechas en el Dockerfile. A continuación veremos el despliegue de dos contenedores, uno a partir de la imagen ahora creada, y otra a partir de la imagen de docker **dind**.

## Despliegue de los contenedores mediante Terraform

## Configuración de Jenkins