
# Android-CI
Guía para montar un entorno de integración continua para Android sobre [Docker](https://www.docker.com/).

## Introducción
En está guía se dará una explicación práctica para, desde cero, montar un entorno de integración continúa para Android con [Jenkins](https://jenkins.io/). 

La tarea a ejecutar realizará los siguientes pasos:

* Descargar el código fuente desde un repositorio de GitHub.
* Aumentar la versión del ejecutable y compilar la aplicación.
* Chequeo estático del código fuente con [SonarQube](https://www.sonarqube.org/).
* Firma del APK generado.
* Subida del APK a Google Play en el canal alpha (es necesario tener una cuenta de desarrollador en Google Play).
* Envío de emails al equipo de desarrollo notificando el estado de la tarea de Jenkins.
* Si todo lo anterior ha ido bien, push de nuevo a GitHub para actualizar la versión.

### AVISO
Está guía no pretende ser la mejor manera de montar un servidor de integración. Sólo es una explicación resultado de los problemas que me he ido encontrando al montarlo y cómo los he solventado. Estoy seguro de que hay mil formas mejores de hacerlo.

Usa el código o los comandos que aquí aparecen bajo tu responsabilidad. No me hago responsable de cualquier daño que pueda causar en tu aplicación o código fuente.

## Entorno
Trabajaremos sobre contenedores Docker, lo que nos permitirá tener un entorno funcionando rápidamente y con pocos pasos. No es objeto de esta guía explicar el manejo de Docker y sólo veremos los comandos básicos que nos permitirán tener los contenedores corriendo.

Nuestro entorno de trabajo contará con los siguientes contenedores:
* Jenkins: Servidor de integración continua.
* SonarQube: Encargado del análisis estático del código.
* [PostgreSQL](https://www.postgresql.org/) : Base de datos para almacenar los resultados de Sonarqube
* Servidor SMTP: Servidor de correo para enviar las notificaciones.
* [Nginx](http://nginx.org/): Proxy para redirigir las peticiones al resto de contenedores. // TODO pendiente

### Instalación
Utilizaremos la versión [Community de Docker](https://www.docker.com/community-edition). Únicamente, reseñar que debemos darle por lo menos 4GiB de RAM, ya que solamente con los [requisitos de Sonarqube](https://docs.sonarqube.org/display/SONAR/Requirements) necesitaremos 2GiB para funcionar más 1GiB libre. Podemos saber los recursos que se están consumiendo a través del comando:

`docker stats`

Podemos configurar la cantidad de memoria disponible para Docker en `Settings > Advanced > Memory`. 

Obviamente, cuantos más recursos podamos dejar disponibles para Docker, mejor.

También tenemos que compartir con Docker la unidad desde la que vayamos a levantar los contenedores para que se puedan montar correctamente los volúmenes de nuestras aplicaciones. Esto lo haremos desde `Settings > Shared Drives`.

### docker-compose
Levantaremos todos los contenedores docker con un único comando. Desde la línea de comandos, situándonos en la carpeta docker, ejecutaremos:

`docker-compose up -d`

Con esto, se descargarán las imágenes necesarias, se crearán los contenedores y se lanzarán las distintas aplicaciones. Además, se descargará el [SDK de android](https://developer.android.com/studio/) y las build-tools necesarias para compilar el proyecto, además de aceptar las distintas licencias de estas.
Todo el compose se basa en el fichero creado por [galexandre](https://github.com/galexandre) y que puedes encontrar en su [repositorio de GitHub](https://github.com/galexandre/docker-jenkins-sonarqube). Este monta los contenedores de Jenkins y de SonarQube. Se le ha agregado parte del Dockerfile de [AckeeDevOps](https://github.com/AckeeDevOps/jenkins-android-docker-slave/blob/master/Dockerfile) que instala el SDK de Android y las build-tools. También se utiliza el docker-compose del servidor SMTP de [namsi](https://github.com/namshi/docker-smtp).

Por último, se le asigna una ip fija a cada uno de los contenedores para luego poder hacer referencia a ellos fácilmente desde Jenkins. Podemos ver, entre otra información, la ip de cada contenedor con el comando:

`docker inspect [NOMBRE_DOCKER]`

#### Nginx (opcional)
Utilizaremos nginx como proxy inverso para redirigir nuestras peticiones a Jenkins o Sonarqube sin tener que exponer los puertos de estas aplicaciones. La idea es tener en nuestro dominio expuestos Únicamente los puertos 80 para las peticiones http y 443 para https y que las peticiones que entren por _/jenkins_ vayan al puerto 8080 y las que entren por _/sonarqube_ al 9000.

También montaremos un volumen que enlace con el fichero docker/nginx/nginx.conf con el fichero de configuración de nginx, con lo que no hará falta entrar al contenedor para modificar su configuración.

`volumes:  - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro`

#### SMTP (opcional)
Crearemos un servidor SMTP para el envío de correos electrónicos al equipo de desarrollo para notificarles el estado de la ejecución de la tarea de Jenkins. Simplemente configuraremos el servidor para que haga uso del puerto 25. 

Podemos ver más opciones de configuración en el repositorio original de [namsi](https://github.com/namshi/docker-smtp)
#### Sonarqube
SonarQube nos permitirá realizar un análisis del código en busca de posibles fuentes de bugs, duplicidades, código innecesario, etc. 

Configuraremos SonarQube en el puerto 9000 y levantará también un contenedor de PostgreSQL necesario para guardar los datos de los análisis realizados.

#### Jenkins
Por último, tendremos un contenedor Jenkins configurado en el puerto 8080 para realizar la tarea de descarga y compilación del código fuente y firma y publicación del APK generado.

En el fichero /docker/jenkins/Dockerfile indicamos el war de Jenkins que se va a descargar, indicado en la línea 37 en la varíable `JENKINS_VERSION`. Si queremos modificar la versión a descargar, podemos ir a la [página de descarga](https://jenkins.io/download/) oficial y seleccionar la que queramos. Si cambiamos la versión, debemos cambiar también el parámetro `JENKINS_SHA` definido en este mismo fichero. Tendremos que calcular el SHA del war, para lo cual podemos hacer uso de la herramienta [File Checksum Integrity Verifier](https://www.microsoft.com/en-us/download/details.aspx?id=11533) de Microsoft. Descargamos la herramienta y el war y ejecutamos el siguiente comando para obtener su hash:

`fciv -sha1 [FILE_NAME]`

También en el fichero Dockerfile podemos encontrar la descarga del SDK de Android y de las build-tools, siendo aquí donde podremos modificar la versión a descargar. 

### Pasos previos
Una vez lanzado el `docker-compose` tendremos nuestro entorno preparado, pero antes de crear la tarea Jenkins configuraremos el proyecto en SonarQube y en la consola de desarrolladores de Google y de Google Play para poder publicar el apk automáticamente.

#### Sonarqube
Nos conectaremos a SonarQube desde _/sonarqube_ e iniciaremos sesión para poder crear nuestro proyecto. Por defecto, [las credenciales de SonarQube](https://docs.sonarqube.org/display/SONAR/Authentication) son admin/admin. 

Una vez iniciada la sesión, nos pedirá que creemos un token para el acceso anónimo. Lo creamos, por ejemplo, dándole nuestro nombre de dominio y lo apuntamos, ya que no podremos volver a verlo. Si no, deberíamos eliminar este y crear otro desde `administration > security > tokens`.

Una vez tengamos el token, crearemos un proyecto en `administration > project > management > create`.

#### Google Developers Console
// TODO desarrollar
#### Google Play
// TODO desarrollar
## Jenkins
Accederemos a través de _/jenkins_. Lo primero que nos pedirá será una clave que se mostró en el log de Docker al crear el contenedor. Podemos volver a verla con el comando:

`docker logs [NOMBRE_DOCKER]` 

En este caso, si no modificamos el docker-compose, será jenkins. Si no, podemos ver los contenedores que tenemos levantados a través del comando:

`docker ps -a`

También podemos ver la contraseña en la ruta `/var/jenkins_home/secrets/initialAdminPassword` de nuestro contenedor.

Una vez con la contraseña nos preguntará por los plugins a instalar, pudiendo dejar que instale los plugins recomendados. Cuándo termine de descargarlos, nos pedirá que creemos una cuenta de administrador. Con todo esto, ya podemos empezar a utilizar Jenkins.

#### Plugins
Antes de crear nuestra tarea vamos a dejar preparado el servidor de Jenkins para todo lo que queremos que haga. Lo primero que vamos a hacer es instalar los siguientes plugins:

* SonarQube Scanner: Plugin necesario para hacer el análisis de SonarQube.
* Android Signing: Plugin necesario para la firma del APK.
* Google Play Android Publisher: Plugin necesario para subir el APK a Google Play.

Para esto, iremos a **Administrar Jenkins > Administrar Plugins** y en la pestaña **Todos los plugins** buscaremos los plugins a instalar. Podemos filtrar a través del buscador para que nos sea más fácil. Una vez que los encontremos, los seleccionamos y le pulsamos en **Instalar sin reiniciar**. Cuándo termine de instalarlos podemos seguir con nuestra configuración.

#### Configuración
El siguiente paso será completar la configuración del servidor. 

Iremos a **Administrar Jenkins > Configurar el Sistema**. Aquí tenemos que agregar un servidor de SonarQube, en la sección **SonarQube Servers > Add SonarQube**. Le daremos un nombre (SonarQube-Server, por ejemplo) y una URL, que será la ruta de nuestro servidor de SonarQube, es decir, _/sonarqube_. Por último, debemos darle también el token de autenticación que habíamos creado previamente en SonarQube.

En esta misma pantalla, en la sección **Jenkins Location**, configuraremos la dirección web de jenkins (_/jenkins_) y la cuenta de correo del adminstrador.

Si seguimos bajando en la configuración, en la sección **Git plugin**, podemos dar un nombre de usuario y un email para los commits de Git, por ejemplo jenkins y jenkins@email.com.

Sin salir todavía de esta página, configuraremos también el servidor SMTP para las notificaciones por email. Por un lado, en la sección **Extended E-mail configuration** rellenaremos los siguientes campos:
* SMTP server: La ip de nuestro servidor SMTP. Si no modificamos nada en el docker-compose será 10.5.0.2
* Avanzado > SMTP port: Si no lo cambiamos en el docker-compose, 25.
* Default Recipients: Lista de correos, separados por comas, a los que se enviarán por defecto los correos. Después, podremos usar la variable **$DEFAULT_RECIPIENTS** en nuestros proyectos para que se envíen las notificaciones a esta lista. Si queremos poner alguna dirección en cc o en bcc podemos hacerlo añadiendo cc: o bcc: delante de la dirección concreta.
* Reply To List: Lista de correos, separados por comas, a los que responder. Es la variable **$DEFAULT_REPLYTO**.
* Defaul Subject: Asunto por defecto del email. Podemos poner, a través de variables de la aplicación, un asunto genérico que nos servirá para todos los proyectos:

  `[JENKINS-${JOB_NAME}] Result of Build $BUILD_NUMBER ${JOB_NAME} Project: $BUILD_STATUS`

  Con esto, para una tarea que se llamase Awesome-CI, tendríamos un asunto así:

  `[JENKINS-Awesome-CI] Result of Build 132 Awesome-CI Project: Successful`

* Default Content: Al igual que en el caso anterior, el contenido por defecto del email. Escribiremos también un asunto genérico que nos sirva para todos los projectos. En él, enviaremos el nombre de la tarea, el resultado de la ejecución, un enlace al apk generado y la lista de cambios desde el último commit:

      <html>      
      <p>Hello. </p>      
      <p>This mail is auto-generated as part of the Jenkins execution of the project <b>${JOB_NAME}</b> </p>      
      <h2> BUILD DETAILS: </h2>      
      <p>
         <b>Project Name:</b> ${JOB_NAME} <br>
         <b>Build URL:</b> ${BUILD_URL} <br>
         <b>Build Number: </b> ${BUILD_NUMBER} <br>
         <b>Build Status: </b> ${BUILD_STATUS} <br>
         <b>Download APK: </b> ${BUILD_URL}lastSuccessfulBuild/artifact/apks/ <br>
         <b>Log: </b> The log file is attached into this e-mail. <br>
         <b>Log URL: </b> ${BUILD_URL}${JOB_NAME}/lastBuild/console <br>
         <b>Changes: </b> ${CHANGES, format="List of changes: 
         <li>
            <ul>[%a] %m</ul>
            <ul>[Date:] %d</ul>
            <ul>[Revision:] %r</ul>
         </li> 
         <br>"} <br>
      </p>      
      <p> Thank you & Regards. </p>      
      </html>
      

* Default Triggers: Always. Así, siempre que se complete la ejecución, se nos notificará por email.

Una vez completada esta parte de la configuración pulsamos en **Guardar** y pasamos a configurar las herramientas. Para ellos, vamos a **Administrar Jenkins > Global Tool Configuration**. En esta página únicamente necesitamos configurar el instalador de SonarQube. Para ello, en la sección **SonarQube Scanner** seleccionaremos **Añadir SonarQube Scanner** y le daremos un nombre (SonarQube-Scanner, por ejemplo) y dejaremos el resto de campos como están (instalar automáticamente, Install from Maven Central). 

Le damos a **Guardar** y ya tenemos la configuración del servidor Jenkins completada.

#### Credenciales
El último paso antes de crear la tarea será crear las distintas credenciales que necesitara Jenkins para completar todas las acciones. Para crearlas iremos a **Credentials > System > Global credentials (unrestricted) > Add Credentials**.

Necesitaremos 3 credenciales distintas:

* Usuario y contraseña de GitHub para hacer descargar el código fuente y poder hacer el push si todo va bien.

      Kind: Username with password
      Scope: Global
      Username: [nuestro usuario de GitHub]
      Password: [nuestra password de GitHub]
      ID: GitHub
      Description: Credenciales de GitHub


* Keystore de Android para poder firmar el APK. 

      Kind: Certificate
      Scope: Global
      Certificate: Upload PKCS#12 certificate
      Password: Password del certificado
      ID: [nombre-proyecto]-keystore.p12
      Description: P12 del Keystore para firmar [nombre-proyecto]

Para firmar nuestro APK, debemos proveer a Jenkins de nuestra keystore para que pueda hacerlo. Para ello, exportaremos la Keystore como un certificado p12 que podemos subir a Jenkins. Para la exportación haremos uso de la herramienta keytool que viene incluida en la instalación de la JDK. Desde la línea de comandos, en la carpeta bin de nuestra JDK, ejecutaremos la siguiente instrucción:

`keytool -importkeystore -srckeystore [PATH_KEYSTORE] -destkeystore [PATH_P12] -srcstoretype JKS -deststoretype PKCS12 -deststorepass [P12_PASSWORD] -srcalias [KEYSTORE_ALIAS] -destalias [P12_ALIAS]`

Donde PATH_KEYSTORE es el path del fichero keystore, PATH_P12 es el path del fichero que queremos que nos genere, KEYSTORE_PASSWORD la clave del p12 generado, y los alias de origen y de destino. Por ejemplo, la instrucción podría ser algo así:

`keytool -importkeystore -srckeystore C:\keys\awesome-keystore.jks -destkeystore C:\keys\awesome.p12 -srcstoretype JKS -deststoretype PKCS12 -deststorepass awesome_pass -srcalias awesome_alias -destalias awesome_alias`
   
* Credenciales de Google Service para poder subir el APK a Google Play.

      Kind: Google Service Account from private Key
      Project name: [nombre del proyecto en Google Developers Console]
      JSON Key: [json generado en Google Developers Console]

## Tarea Jenkins
Con todo esto, ya tendríamos todo listo para crear nuestra tarea. Para ello, pulsaremos en **Nueva Tarea > Crear un proyecto de estilo libre**. Le daremos un nombre y pulsaremos en **OK**

### General
* Proyecto nombre: Nombre para la tarea
* Desechar ejecuciones antiguas
   * Strategy: Log rotation
   * Número máximo de ejecuciones para guardar: 10. Podemos poner más, pero con cuidado no sobrecarguemos el servidor.

* GitHub project
   * Project url: URL del repositorio de GitHub

### Configurar el origen del código fuente
* Git
   * Repositories
      * Repository URL: URL del repositorio de GitHub
      * Credentials: Las credenciales de GitHub
   * Branches to build<br>
      * Branch specifier: */stable

### Disparadores de ejecuciones
* Ejecutar periódicamente
   * Programador @midnight (Para que se ejecute cada medianoche).

### Entorno de ejecución
* Delete workspace before build starts
* Add timestamps to the Console Output

### Ejecutar
* Invoke Gradle script
   * Use gradle wrapper
      * Task app:assembleRelease (en nuestro fichero _build.gradle_ del módulo app aumentamos el _versionCode_ automáticamente al construir la release a través del fichero _version.properties_).

* Execute SonarQube Scanner
   * Analysis properties

         # unique project identifier (required)
         sonar.projectKey=[nombre proyecto SonarQube]
         # project metadata (used to be required, optional since SonarQube 6.1)
         sonar.projectName=[nombre proyecto SonarQube]
         sonar.projectVersion=1.0      
         # path to source directories (required)
         sonar.sources=app/src/main/java, common/src/main/java, data/src/main/java, model/src/main/java      
         # path to test source directories (optional)
         sonar.tests=      
         # path to Java project compiled classes (required)      
         sonar.java.binaries=app/build/intermediates/classes/release, common/build/classes, data/build/classes, model/build/classes
         # comma-separated list of paths to libraries (optional)
         # sonar.java.libraries=
         # Additional parameters
         # eliminamos de codigo duplicado las clases del modelo
         sonar.cpd.exclusions=**/model/*,**/entity/*,**/dto/*

   * Ejecutar línea de comandos (shell)
      * Comando: git commit -am "Build from Jenkins"

   * Sign Android APKs
      * Key store: La que creamos al agregar las credenciales
      * Key alias: El alias que generamos
      * APKs to Sign: **/app-release-unsigned.apk
      * Archive Signed APKs
   
### Acciones para ejecutar después

   * Guardar los archivos generados
      * Ficheros para guardar: **/app-release.apk

   * Upload Android APK to Google Play
      * Google Play account: Specific credentials > Las que creamos al agregar las credenciales
      * APK files: **/app-release.apk
      * Release track: alpha
      * Recent changes: Add language
         * Language: es-ES
         * Text: Build automática desde Jenkins

   * Editable email notification
      * Project From: Nombre del proyecto
      * Project recipient list: $DEFAULT_RECIPIENTS (La configuramos antes en _Configurar el sistema_).
      * Project Reply-To List: $ DEFAULT_REPLYTO (La configuramos antes en _Configurar el sistema_).
      * Content type: Both HTML and Plain Text
      * Default Subject: $DEFAULT_SUBJECT (Lo configuramos antes en _Configurar el sistema_).
      * Default Content: $DEFAULT_CONTENT (Lo configuramos antes en _Configurar el sistema_).
      * Attach Buils Log: Compress and Attach Build Log

   * Git publisher
      * Push Only If Build Succeeds
      * Branches
         Branch to push: stable
         Target remote name: origin

Con esto ya tendrúamos nuestro proyecto completamente configurado. Guardamos los cambios y le damos a **Construir ahora** para que se ejecute nuestra tarea. Podemos ver todas las acciones que va realizando si pulsamos en el la ejecución de la tarea y después en **Console output**. La primera vez veremos que tarda mucho ya que tiene que descargar todo lo necesario y arrancar Gradle, pero el resto de ejecuciones debería ser más rápida.

Cuándo termine, si todo ha ido bien, deberíamos ver un nuevo commit en el repositorio con el nuevo _versionCode_, un nuevo apk en la fase alpha de Google Play y tendremos un email notificándonos del estado de la tarea.

## Enlaces de interés y bibliografía

* [https://www.docker.com/](https://www.docker.com/)
* [https://github.com/galexandre/docker-jenkins-sonarqube](https://github.com/galexandre/docker-jenkins-sonarqube)
* [https://github.com/AckeeDevOps/jenkins-android-docker-slave](https://github.com/AckeeDevOps/jenkins-android-docker-slave)
* [https://developer.android.com/studio/](https://developer.android.com/studio/)
* [https://nginx.org/en/](https://nginx.org/en/)
* [https://github.com/namshi/docker-smtp](https://github.com/namshi/docker-smtp)
* [https://vnextcoder.wordpress.com/2016/10/16/part-1-setting-up-android-build-on-jenkins/](https://vnextcoder.wordpress.com/2016/10/16/part-1-setting-up-android-build-on-jenkins/)
* [http://www.vogella.com/tutorials/JenkinsAndroid/article.html](http://www.vogella.com/tutorials/JenkinsAndroid/article.html)
* [https://android.jlelse.eu/setting-up-jenkins-for-android-how-i-dealt-with-the-challenges-i-faced-1bbdb6580b8d](https://android.jlelse.eu/setting-up-jenkins-for-android-how-i-dealt-with-the-challenges-i-faced-1bbdb6580b8d)
* [https://sdos.es/como-compilar-aplicaciones-android-con-jenkins/](https://sdos.es/como-compilar-aplicaciones-android-con-jenkins/)
* [https://www.sonarqube.org/](https://www.sonarqube.org/)
* [https://www.adictosaltrabajo.com/tutoriales/sonar-android/](https://www.adictosaltrabajo.com/tutoriales/sonar-android/)
* [https://www.tbs-certificates.co.uk/FAQ/en/627.html](https://www.tbs-certificates.co.uk/FAQ/en/627.html)
* [https://wiki.jenkins.io/display/JENKINS/Android+Signing+Plugin](https://wiki.jenkins.io/display/JENKINS/Android+Signing+Plugin)
* [https://wiki.jenkins.io/display/JENKINS/Google+Play+Android+Publisher+Plugin](https://wiki.jenkins.io/display/JENKINS/Google+Play+Android+Publisher+Plugin)
* [https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Jenkins](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Jenkins)
