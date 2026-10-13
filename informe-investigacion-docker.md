# Taller del segundo corte: fundamentos de Docker

**Asignatura:** Arquitectura de Software  
**Estudiante:** Carlos Antonio Ramírez Galvis  
**Tema:** Docker y su relación con la arquitectura de software  
**Fecha:** 10 de octubre de 2026

---

## Introducción

En un proyecto de software no basta con que el código esté bien escrito. También importa el entorno en el que se ejecuta: la versión del lenguaje, las librerías instaladas, las variables de configuración, los puertos y los servicios externos, como una base de datos. Cuando esas condiciones cambian entre el computador de un desarrollador, el equipo de pruebas y el servidor, aparece el conocido problema de “en mi máquina funciona”. Docker propone una forma práctica de reducir esas diferencias mediante contenedores.

En este informe explico qué es Docker, cuáles son sus componentes, qué operaciones permite y qué problema busca resolver. Finalmente, relaciono esta tecnología con los estilos arquitectónicos estudiados en el curso. Me apoyo principalmente en el material de la semana 9 y en la documentación oficial, pero las explicaciones y los ejemplos están planteados con mis propias palabras.

## 1. ¿Qué es Docker?

Docker es una plataforma que permite empacar una aplicación junto con los archivos, dependencias y configuraciones que necesita para ejecutarse. Ese paquete se usa para crear un **contenedor**, que se puede entender como un proceso aislado con su propio sistema de archivos, red y configuración. El propósito es que la aplicación se comporte de una forma muy parecida sin importar si se ejecuta en el equipo de un desarrollador, en un servidor de pruebas o en la nube.

A alto nivel, el contenedor no inicia un computador completo. El proceso que está dentro del contenedor sigue utilizando el kernel del sistema operativo anfitrión, pero Docker le asigna espacios aislados para procesos, red, usuarios y archivos. Por eso dos contenedores pueden usar versiones distintas de una librería sin interferirse, aunque estén corriendo en la misma máquina. La imagen aporta el sistema de archivos y las dependencias; el host aporta el kernel y los recursos físicos.

Esta es la diferencia principal frente a una máquina virtual. Una máquina virtual emula hardware y arranca un sistema operativo completo, con su propio kernel. Como consecuencia, normalmente requiere más memoria, más espacio y más tiempo de inicio. El contenedor comparte el kernel del host y arranca solamente los procesos necesarios para la aplicación, así que suele ser más liviano y rápido. Esto no significa que contenedor y máquina virtual sean enemigos: se pueden ejecutar contenedores dentro de máquinas virtuales, algo común en proveedores de nube.

| Aspecto | Contenedor | Máquina virtual |
|---|---|---|
| Qué virtualiza | El entorno de ejecución de procesos | Una máquina completa |
| Sistema operativo | Comparte el kernel del host | Incluye un sistema operativo invitado |
| Inicio | Generalmente en segundos | Generalmente tarda más por el arranque del SO |
| Consumo | Menor sobrecarga | Mayor consumo de disco y memoria |
| Uso frecuente | Empacar y desplegar servicios | Aislar sistemas operativos o cargas completas |

## 2. Componentes de Docker

Docker funciona con una arquitectura cliente-servidor. No todo lo hace el comando que se escribe en la terminal; ese comando solicita una operación al motor de Docker.

- **CLI o cliente `docker`:** es la interfaz con la que el usuario escribe instrucciones, por ejemplo `docker run` o `docker ps`. El cliente transforma la instrucción en una solicitud para la API del motor.
- **Daemon `dockerd`:** es el proceso que recibe esas solicitudes y hace el trabajo pesado. Construye imágenes, crea y detiene contenedores, administra redes y volúmenes y se comunica con registros.
- **Dockerfile:** es un archivo de texto que describe, paso por paso, cómo construir una imagen. Puede indicar una imagen base, la carpeta de trabajo, los archivos que se copian, las dependencias que se instalan y el comando inicial.
- **Imagen:** es una plantilla inmutable y organizada por capas. Contiene el sistema de archivos, los binarios, las librerías y la configuración necesaria para crear contenedores. Una imagen no es todavía un programa en ejecución.
- **Contenedor:** es una instancia ejecutable de una imagen. Desde la misma imagen se pueden iniciar varios contenedores, cada uno con su propio estado y configuración. Si el contenedor se elimina, la imagen puede seguir disponible para crear otro.
- **Registro:** es un servicio donde se almacenan y distribuyen imágenes. Docker Hub es el registro público que Docker consulta por defecto, aunque también existen registros privados y alternativas de otros proveedores.

La relación entre estos elementos se puede resumir así:

```text
Dockerfile ──docker build──> Imagen ──docker run──> Contenedor
                              ▲                         │
                              │                         │
                    docker pull / push            proceso activo
                              │
                       Registro de imágenes

CLI docker ──solicitud──> Daemon dockerd ──administra todo lo anterior
```

Por ejemplo, el `Dockerfile` de una API en Go puede partir de una imagen de Go, copiar el código y compilarlo. Al ejecutar `docker build`, el daemon produce una imagen. Con `docker run`, esa imagen se convierte en un contenedor que atiende solicitudes. Si la imagen se publica en un registro, otro integrante del equipo la puede descargar con `docker pull` sin repetir manualmente la instalación.

## 3. Principales funcionalidades

### Portabilidad

Docker permite trasladar la aplicación como una unidad definida. La imagen conserva las versiones y dependencias que fueron usadas al construirla. Por ejemplo, una API que necesita Go y unas librerías específicas puede correr en otro computador con Docker aunque allí Go no esté instalado directamente.

### Aislamiento

Cada contenedor tiene procesos, red y sistema de archivos aislados en cierto grado. Esto evita muchos choques entre aplicaciones. Un ejemplo sería ejecutar dos versiones de MySQL en contenedores separados y mapearlas a puertos distintos del host, sin reemplazar una instalación con la otra.

### Versionado de imágenes

Las etiquetas permiten distinguir versiones como `mysql:8.0`, `node:18` o `mi-api:1.2`. Esto sirve para actualizar de forma controlada y también para regresar a una versión conocida si aparece un error. La etiqueta no reemplaza una estrategia completa de control de versiones, pero ayuda a identificar el artefacto que se despliega.

### Reproducibilidad del entorno

El Dockerfile y el archivo de Compose dejan por escrito cómo se construye y se ejecuta el sistema. En lugar de que cada integrante instale dependencias siguiendo pasos ligeramente diferentes, el equipo comparte la misma definición. Por ejemplo, un nuevo integrante puede clonar un repositorio y levantar API y base de datos con `docker compose up`.

### Escalabilidad

Una imagen se puede usar para arrancar varias instancias del mismo servicio. Si una API recibe más tráfico, se pueden ejecutar varios contenedores iguales detrás de un balanceador. El contenedor no realiza por sí solo toda la orquestación, pero sí proporciona una unidad pequeña y repetible para escalar.

### Apoyo a CI/CD

En una integración continua se puede construir una imagen, ejecutar pruebas sobre ella y publicar exactamente ese artefacto en un registro. Después, el despliegue consume la imagen ya verificada. De esta forma disminuye el riesgo de probar una cosa y desplegar otra diferente.

## 4. Principales comandos

| Comando | ¿Qué hace? | Ejemplo |
|---|---|---|
| `docker pull` | Descarga una imagen desde un registro. | `docker pull mysql:8.0` |
| `docker run` | Crea y arranca un contenedor desde una imagen. | `docker run -d --name web -p 8080:80 nginx` |
| `docker ps` | Lista los contenedores que están corriendo. | `docker ps` |
| `docker ps -a` | Lista también los contenedores detenidos. | `docker ps -a` |
| `docker logs` | Muestra la salida estándar de un contenedor. | `docker logs biblioteca_api` |
| `docker exec` | Ejecuta un comando dentro de un contenedor activo. | `docker exec -it biblioteca_api sh` |
| `docker stop` | Detiene de forma controlada un contenedor. | `docker stop web` |
| `docker start` | Vuelve a iniciar un contenedor detenido. | `docker start web` |
| `docker rm` | Elimina un contenedor detenido. | `docker rm web` |
| `docker images` | Lista las imágenes disponibles localmente. | `docker images` |
| `docker rmi` | Elimina una imagen local. | `docker rmi nginx:latest` |
| `docker build` | Construye una imagen a partir de un Dockerfile. | `docker build -t biblioteca-api:1.0 .` |
| `docker compose up` | Crea e inicia los servicios definidos en Compose. | `docker compose up --build` |

### Tres comandos explicados con más detalle

**`docker run`:** crea un contenedor nuevo y lo inicia. La opción `-d` lo deja en segundo plano; `--name` asigna un nombre fácil de recordar; `-p 8080:80` publica el puerto 80 del contenedor en el 8080 del host; `-e CLAVE=valor` entrega una variable de entorno, y `-v origen:destino` monta almacenamiento. Lo usaría para probar rápidamente un solo servicio, por ejemplo Nginx: `docker run -d --name web -p 8080:80 nginx`.

**`docker build`:** procesa el Dockerfile y produce una imagen. Con `-t nombre:etiqueta` se asigna un nombre y una versión, mientras que el punto final indica que la carpeta actual es el contexto de construcción. En el proyecto de biblioteca lo usaría como `docker build -t biblioteca-api:local .` para obtener la imagen de la API a partir del código local.

**`docker compose up`:** lee el archivo `docker-compose.yml` o `compose.yml` y levanta el conjunto de servicios con sus redes, variables, puertos y volúmenes. `--build` obliga a construir las imágenes que se basan en código local y `-d` deja el conjunto en segundo plano. Lo usaría cuando una solución necesita varios contenedores relacionados, como la API de biblioteca y MySQL: `docker compose up --build -d`.

## 5. ¿Qué problema resuelve?

El problema de “en mi máquina funciona” aparece porque ejecutar una aplicación depende de más cosas que su código. Un desarrollador puede tener Node 18 y otro Node 16; uno puede usar una versión nueva de MySQL; en el servidor puede faltar una variable de entorno; también puede haber puertos ocupados o instrucciones manuales interpretadas de forma distinta. En esas condiciones, dos personas ejecutan “el mismo proyecto”, pero en realidad lo hacen sobre entornos diferentes.

Antes de Docker, imaginemos que para ejecutar una API de biblioteca había que instalar Go, instalar MySQL, crear manualmente la base de datos, cargar un script SQL y configurar ocho variables. Cada persona podía omitir un paso o usar otra versión. Con Docker, el repositorio puede incluir un Dockerfile para la API, un archivo Compose para describir API y MySQL, un archivo `.env.example` para la configuración y migraciones para crear la base. Entonces el proceso se vuelve repetible y auditable.

Docker no elimina todos los problemas. Una imagen mal construida puede seguir fallando, las variables deben ser coherentes y el host necesita recursos suficientes. Sin embargo, reduce una fuente grande de variación porque el entorno deja de depender solamente de instalaciones manuales y pasa a estar descrito como archivos versionables.

## 6. Importancia dentro de una arquitectura de software

Docker es importante porque ayuda a llevar una decisión arquitectónica desde el diagrama hasta una ejecución concreta. Si una solución tiene una API, una base de datos y un broker de eventos, ejecutar cada elemento en su propio contenedor permite configurar, actualizar, observar y reiniciar cada servicio de manera separada. También evita que una dependencia quede mezclada con las demás en el sistema anfitrión.

En el despliegue, la misma imagen puede avanzar por desarrollo, pruebas y producción. Lo que cambia suele ser la configuración externa, como las credenciales o la dirección de un servicio, pero no el artefacto que fue probado. Esto mejora la consistencia y facilita automatizar el proceso en una tubería de CI/CD.

Para escalar, varias instancias de la misma imagen pueden atender solicitudes en paralelo. En la nube, la portabilidad de las imágenes simplifica mover la carga a otra máquina compatible o entregarla a un orquestador. Aun así, una arquitectura escalable exige otras decisiones: servicios sin estado cuando sea posible, persistencia externa, balanceo, observabilidad y manejo de fallos.

La relación con los estilos vistos en clase es directa:

- En **microservicios**, cada servicio puede tener su propia imagen y ciclo de despliegue. Esto apoya la independencia, aunque no convierte automáticamente un sistema en microservicios.
- En **SOA**, los servicios se pueden distribuir y configurar de manera consistente, conservando contratos y responsabilidades separados.
- En una arquitectura **orientada a eventos**, productores, consumidores y un broker como RabbitMQ o Kafka pueden ejecutarse como servicios diferentes dentro de una red de contenedores.
- En un **monolito por capas**, como la API de biblioteca, Docker también es útil. La aplicación sigue siendo un solo servicio con capas `handler`, `service` y `repository`, pero se despliega de forma reproducible junto a su base de datos.

Por lo tanto, Docker no define el estilo arquitectónico. Es una tecnología de empaquetado y ejecución que facilita implementar, probar y desplegar varios estilos de una forma más controlada.

## Conclusión

Después de revisar sus componentes, considero que el aporte más valioso de Docker no es solamente “instalar menos cosas”, sino volver explícito el entorno de una aplicación. El Dockerfile funciona como una receta que se puede revisar, la imagen como un artefacto versionable y el contenedor como una instancia que se puede crear y reemplazar.

Yo usaría Docker en un proyecto web con API y base de datos, especialmente cuando participan varias personas. En ese tipo de proyecto es fácil perder tiempo comparando versiones o configurando MySQL de manera distinta. Con un archivo Compose, el equipo puede compartir una base común y concentrarse más rápido en la funcionalidad. También lo usaría en pruebas automáticas, porque permite crear un entorno temporal, ejecutar las pruebas y eliminarlo al terminar.

## Referencias

1. Carreño Ortiz, L. G. (2026). *Docker: tu app en una caja* [Diapositivas de la semana 9]. Asignatura Arquitectura de Software, Universidad Tecnológica de Bolívar.
2. Docker, Inc. (2026). *What is Docker?* Docker Docs. https://docs.docker.com/get-started/docker-overview/
3. Docker, Inc. (2026). *What is a container?* Docker Docs. https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/
4. Docker, Inc. (2026). *What is an image?* Docker Docs. https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/
5. Docker, Inc. (2026). *Docker Compose*. Docker Docs. https://docs.docker.com/compose/
6. Docker, Inc. (2026). *docker container run*. Docker Docs. https://docs.docker.com/reference/cli/docker/container/run/

