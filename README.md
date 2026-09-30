# Virtualization Workshop

## 1. Descripción

Este laboratorio documenta el ciclo de virtualización de una aplicación web Java: desarrollo con Spring Boot, empaquetado con Maven, creación de una imagen Docker, ejecución de múltiples contenedores, orquestación con Docker Compose y MongoDB, publicación en Docker Hub y despliegue en AWS EC2.

La aplicación expone un servicio REST sencillo que permite verificar cada etapa del workshop de forma incremental.

## 2. Tecnologías utilizadas

- Java 21
- Spring Boot 4.1.1
- Maven
- Docker
- Docker Compose
- MongoDB
- Docker Hub
- AWS EC2
- Amazon Linux 2023

## 3. Arquitectura

```text
                         Cliente HTTP
                              |
                              v
                    Spring Boot /greeting
                              |
                              v
                    Imagen Docker
                     /          \
                    v            v
            Docker Compose    Docker Hub
              /       \            |
             v         v            v
           web         db         AWS EC2
            |       MongoDB          |
            |                        v
            +----------------> Contenedor
                               8080:9000
```

En Docker Compose, el servicio `web` y el servicio `db` se ejecutan dentro de la red creada automáticamente por Compose. La aplicación se configura con `SPRING_DATA_MONGODB_URI=mongodb://db:27017/workshop`, donde `db` corresponde al nombre del servicio de MongoDB.

MongoDB se incorpora en esta etapa para demostrar la ejecución de múltiples servicios, networking, volúmenes y administración mediante Docker Compose. El endpoint `/greeting` no depende de MongoDB para responder.

## 4. Requisitos

- JDK 21.
- Maven 3.9 o superior.
- Docker Desktop o un motor Docker compatible.
- Docker Compose v2.
- Acceso a Docker Hub para descargar o publicar imágenes cuando corresponda.
- Una cuenta de AWS para la parte de despliegue en EC2.

> **Nota de compatibilidad:** el `compose.yaml` oficial utiliza MongoDB 8. MongoDB 5.0 y posteriores requieren soporte AVX en la CPU. En equipos sin AVX debe utilizarse el override local descrito en la sección 9.

## 5. Ejecución local

### Información del proyecto

| Propiedad | Valor |
|---|---|
| Group ID | `co.edu.escuelaing` |
| Artifact ID | `virtualization-lab` |
| Version | `1.0.0` |
| Java | `21` |
| Spring Boot | `4.1.1` |
| Dependencia principal | Spring Web |

### Construcción del JAR

En PowerShell:

```powershell
mvn clean package
```

### Ejecución con Maven

```powershell
mvn spring-boot:run
```

La clase de entrada es `RestServiceApplication` y `HelloRestController` implementa `GET /greeting`.

Pruebas:

```powershell
curl.exe "http://localhost:9000/greeting?name=Pedro"
curl.exe "http://localhost:9000/greeting"
```

Respuestas esperadas:

```text
Hello, Pedro!
Hello, World!
```

### Configuración del puerto

La aplicación utiliza la variable de entorno `PORT`. El valor predeterminado es `9000`.

En PowerShell se puede probar otro puerto:

```powershell
$env:PORT = 9010
mvn spring-boot:run
```

## 6. Docker

El `Dockerfile` utiliza `amazoncorretto:21`, establece `/app` como directorio de trabajo, copia el JAR como `app.jar`, define `PORT=9000`, expone el puerto `9000` y arranca con `java -jar app.jar`.

Antes de construir la imagen debe existir el JAR en `target/`:

```powershell
mvn clean package
```

Construcción:

```powershell
docker build -t sebastianvillarraga/virtualization-lab:1.0 .
```

Ejemplo de ejecución:

```powershell
docker run -d --name virtualization-lab -e PORT=9000 -p 9000:9000 sebastianvillarraga/virtualization-lab:1.0
```

La imagen utilizada para el laboratorio es:

```text
sebastianvillarraga/virtualization-lab
```

## 7. Múltiples contenedores

Se ejecutaron tres instancias independientes de la misma imagen:

| Contenedor | Mapeo de puertos |
|---|---|
| `virtualization-lab-1` | `34000:9000` |
| `virtualization-lab-2` | `34001:9000` |
| `virtualization-lab-3` | `34002:9000` |

Comandos utilizados:

```powershell
docker run -d --name virtualization-lab-1 -e PORT=9000 -p 34000:9000 sebastianvillarraga/virtualization-lab:1.0
docker run -d --name virtualization-lab-2 -e PORT=9000 -p 34001:9000 sebastianvillarraga/virtualization-lab:1.0
docker run -d --name virtualization-lab-3 -e PORT=9000 -p 34002:9000 sebastianvillarraga/virtualization-lab:1.0
```

Cada contenedor escucha internamente en el puerto `9000`, mientras Docker asigna un puerto diferente del host. Los tres endpoints fueron probados correctamente.

## 8. Docker Compose

El archivo oficial `compose.yaml` define:

- `web`: construye la aplicación con `Dockerfile`, utiliza `PORT=9000`, se expone como `8087:9000` y depende de `db`.
- `db`: utiliza `mongo:8`, se expone como `27017:27017` y utiliza los volúmenes `mongodb` y `mongodb_config`.
- `SPRING_DATA_MONGODB_URI`: `mongodb://db:27017/workshop`.

Compose crea automáticamente una red para los servicios.

En un equipo compatible con AVX, el entorno oficial se inicia con:

```powershell
docker compose up -d --build
```

Verificación:

```powershell
docker compose ps
docker compose logs web
docker compose logs db
```

La aplicación queda disponible en:

```text
http://localhost:8087/greeting?name=Compose
```

Respuesta esperada:

```text
Hello, Compose!
```

## 9. Compatibilidad local de MongoDB

La configuración oficial se conserva en `compose.yaml` con:

```yaml
image: mongo:8
```

No se modificó para ocultar una limitación del entorno local.

El equipo utilizado para el laboratorio tiene un:

```text
Intel Pentium CPU 6405U @ 2.40GHz
```

MongoDB 8 no pudo ejecutarse localmente porque MongoDB 5.0 o posterior requiere soporte AVX. Los logs mostraron:

```text
MongoDB 5.0+ requires a CPU with AVX support
```

Por esta razón existe `compose.local.yaml`, únicamente como un override para el entorno local. Mantiene el nombre del contenedor, puertos, comando y volúmenes del servicio `db`, pero utiliza:

```yaml
image: mongo:4.4
```

**MongoDB 4.4 no es una sustitución oficial del workshop.** El requisito oficial continúa siendo MongoDB 8 en `compose.yaml`.

Para ejecutar localmente el entorno adaptado:

```powershell
docker compose -f compose.yaml -f compose.local.yaml up -d
```

Para comprobar la configuración efectiva:

```powershell
docker compose -f compose.yaml -f compose.local.yaml config
```

Para verificar los servicios:

```powershell
docker compose -f compose.yaml -f compose.local.yaml ps
```

Para detener el entorno sin eliminar los volúmenes:

```powershell
docker compose -f compose.yaml -f compose.local.yaml stop
```

## 10. Prueba de MongoDB

Debido a que la adaptación local utiliza MongoDB 4.4, el cliente disponible es `mongo` en lugar de `mongosh`.

Entrada al cliente:

```powershell
docker compose -f compose.yaml -f compose.local.yaml exec db mongo
```

Dentro de MongoDB:

```javascript
show dbs
use workshop
db.messages.insertOne({ message: "Hello from Docker Compose" })
db.messages.find()
```

El documento fue insertado y consultado correctamente.

Para salir:

```text
exit
```

También se validó la aplicación local en:

```text
http://localhost:8087/greeting?name=Compose
```

con la respuesta:

```text
Hello, Compose!
```

## 11. Docker Hub

Repositorio:

```text
sebastianvillarraga/virtualization-lab
```

Los tags publicados son:

- `1.0`
- `latest`

Repositorio web:

https://hub.docker.com/r/sebastianvillarraga/virtualization-lab

Crear el tag `latest`:

```powershell
docker tag sebastianvillarraga/virtualization-lab:1.0 sebastianvillarraga/virtualization-lab:latest
```

Publicar ambas versiones:

```powershell
docker push sebastianvillarraga/virtualization-lab:1.0
docker push sebastianvillarraga/virtualization-lab:latest
```

La autenticación en Docker Hub debe realizarse previamente en el equipo del usuario. Este repositorio no contiene credenciales ni tokens.

## 12. AWS EC2

Se creó una instancia llamada:

```text
virtualization-workshop
```

Características:

| Propiedad | Valor |
|---|---|
| Sistema operativo | Amazon Linux 2023 |
| Arquitectura | `x86_64` |
| Tipo de instancia | `t3.micro` |
| vCPU | 2 |
| Memoria | 1 GiB |
| Almacenamiento | 8 GiB `gp3` |

El Security Group se configuró con:

| Regla | Puerto | Origen |
|---|---:|---|
| SSH | 22 | IP del usuario |
| TCP personalizado | 8080 | `0.0.0.0/0` |

En la instancia se instaló Docker y se descargó la imagen publicada:

```bash
docker pull sebastianvillarraga/virtualization-lab:1.0
```

El contenedor se ejecutó con:

```bash
docker run -d \
  --name virtualization-lab \
  --restart unless-stopped \
  -e PORT=9000 \
  -p 8080:9000 \
  sebastianvillarraga/virtualization-lab:1.0
```

Verificación:

```bash
docker ps
```

El contenedor mostró el mapeo:

```text
0.0.0.0:8080->9000/tcp
```

Los logs confirmaron el arranque de Spring Boot y Tomcat escuchando en el puerto `9000` dentro del contenedor.

La opción:

```text
--restart unless-stopped
```

permite que Docker vuelva a iniciar el contenedor después de reinicios de Docker o de la instancia, siempre que el contenedor no haya sido detenido manualmente.

## 13. Prueba del deployment

Durante las pruebas se utilizó la IP pública:

```text
54.242.15.125
```

Endpoint:

```text
http://54.242.15.125:8080/greeting?name=AWS
```

Respuesta:

```text
Hello, AWS!
```

La IP pública de una instancia EC2 puede cambiar si la instancia se detiene y se inicia nuevamente. Por lo tanto, esta dirección corresponde a la configuración utilizada durante las pruebas y no debe considerarse permanente.

## 14. Verificación

| Componente | Prueba | Resultado |
|---|---|---|
| Spring Boot | `GET /greeting` | OK |
| Docker | `docker images` | OK |
| Contenedores independientes | Puertos `34000`, `34001` y `34002` | OK |
| Docker Compose | Servicios `web` y `db` | OK |
| MongoDB local | `insertOne` y `find` | OK |
| Docker Hub | Tags `1.0` y `latest` | OK |
| AWS EC2 | Endpoint publicado en `8080` | OK |

## 15. Problemas encontrados y soluciones

### 15.1 MongoDB 8 sin AVX

MongoDB 8 no pudo ejecutarse en el equipo local porque el Intel Pentium CPU 6405U no proporciona el soporte AVX requerido por MongoDB 5.0 y posteriores.

**Solución:** mantener `mongo:8` en el `compose.yaml` oficial y utilizar `mongo:4.4` mediante `compose.local.yaml` exclusivamente para las pruebas locales.

### 15.2 Necesidad de `compose.local.yaml`

El workshop exige `mongo:8`, por lo que `compose.yaml` no se modificó.

El entorno local se ejecuta combinando:

```powershell
docker compose -f compose.yaml -f compose.local.yaml up -d
```

### 15.3 Cliente de MongoDB 4.4

La imagen local `mongo:4.4` utiliza el cliente:

```text
mongo
```

en lugar de:

```text
mongosh
```

### 15.4 Permisos iniciales del archivo `.pem`

Al conectarse desde Windows a AWS EC2, SSH rechazó inicialmente la clave privada porque sus permisos eran demasiado amplios.

**Solución:** se ajustaron los permisos del archivo `.pem` para que únicamente el usuario autorizado pudiera leerlo.

No se incluyen archivos `.pem`, claves privadas, contraseñas ni tokens en este repositorio.

### 15.5 Regla SSH del Security Group

La conexión SSH inicialmente presentó un timeout porque la regla no correspondía a la IP pública actual del usuario.

**Solución:** se configuró SSH TCP `22` restringido a la IP actual del usuario mediante la opción `Mi IP`.

## 16. Análisis de costos AWS

El análisis siguiente utiliza como referencia una instancia Linux/Unix `t3.micro` On-Demand en **US East (N. Virginia, `us-east-1`)**, porque esa es la región/precio para el que se dispone de una tarifa pública verificable en la documentación consultada. La tarifa publicada por AWS para `t3.micro` es de **US$0.0104 por hora**. citeturn0search1

> **Importante:** el costo real depende de la región, uso efectivo, Free Tier/créditos, transferencia de datos y otros servicios habilitados en la cuenta. AWS factura las instancias On-Demand por el tiempo en estado `running`, con un mínimo de 60 segundos. citeturn0search11

### 16.1 EC2

Suposición:

- `t3.micro`
- Linux
- On-Demand
- 24 horas/día
- 365 días/año
- 8,760 horas/año
- Sin aplicar descuentos, Savings Plans, Reserved Instances ni créditos.

Cálculo:

```text
US$0.0104/hora × 8,760 horas
= US$91.104/año
```

Aproximación:

```text
EC2 ≈ US$91.10/año
```

La tarifa publicada por AWS para `t3.micro` en US East (N. Virginia) es US$0.0104/hora. citeturn0search1

### 16.2 EBS

La instancia utiliza un volumen raíz de:

```text
8 GiB gp3
```

Amazon EBS cobra por la cantidad de almacenamiento provisionado mientras el volumen exista. Los volúmenes gp3 incluyen una línea base de 3,000 IOPS y 125 MB/s de throughput; cualquier IOPS o throughput adicional se factura aparte. citeturn0search2

El precio exacto del almacenamiento gp3 depende de la región. Por lo tanto, para una estimación final se debe utilizar la tarifa gp3 correspondiente a la región real de la instancia.

Como referencia, AWS muestra ejemplos de gp3 con una tarifa de US$0.08 por GB-mes en determinadas regiones. citeturn0search2

Si se utilizara esa tarifa únicamente como referencia:

```text
8 GiB × US$0.08/GB-mes × 12 meses
= US$7.68/año
```

**Este valor es una referencia y no debe interpretarse como el precio confirmado de la región real sin verificarlo en AWS Pricing Calculator.**

### 16.3 Transferencia de datos

El costo de transferencia depende del tráfico generado por la aplicación. Para este laboratorio no se realizó una medición suficiente del tráfico anual como para proyectar un consumo realista de 12 meses.

Por ello:

```text
Transferencia de datos = variable según tráfico
```

AWS documenta que ciertas transferencias hacia/desde direcciones IPv4 públicas pueden generar cargos y que las reglas dependen del tipo y dirección de la transferencia. citeturn0search3

### 16.4 Dirección IPv4 pública

AWS cobra por las direcciones IPv4 públicas, incluidas las asociadas a instancias en ejecución. citeturn0search7

Por tanto, una estimación completa debe considerar también el costo aplicable a la dirección IPv4 pública utilizada por la instancia.

### 16.5 Estimación base

Con las hipótesis anteriores:

| Componente | Estimación anual |
|---|---:|
| EC2 `t3.micro` | US$91.10 |
| EBS `8 GiB gp3` | US$7.68* |
| Transferencia | Variable |
| IPv4 pública | Depende de la tarifa aplicable |
| **Base calculada** | **US$98.78 + cargos variables** |

\* El valor de EBS es únicamente una referencia basada en el ejemplo de precio de US$0.08/GB-mes mostrado por AWS; debe sustituirse por la tarifa correspondiente a la región real de la instancia para obtener un total definitivo. citeturn0search2

Para una estimación precisa y actualizada, AWS proporciona Pricing Calculator, que permite calcular los costos según región, configuración, descuentos y compromisos aplicables. citeturn0search10

### 16.6 Consideración sobre Free Tier y créditos

El costo efectivo de una cuenta concreta puede ser inferior si dispone de Free Tier o créditos promocionales. AWS indica que las condiciones del Free Tier dependen de la elegibilidad de la cuenta y que existen créditos para nuevos clientes bajo las condiciones vigentes. citeturn0search2

Por este motivo, los valores anteriores representan un escenario de referencia y no una factura garantizada.

## 17. Conclusiones

El workshop permitió desarrollar y empaquetar una aplicación Spring Boot con Java 21, ejecutarla en múltiples contenedores Docker, orquestar servicios mediante Docker Compose, trabajar con MongoDB, publicar la imagen en Docker Hub y desplegarla posteriormente en AWS EC2.

El repositorio conserva `compose.yaml` con MongoDB 8 como configuración oficial del workshop y utiliza `compose.local.yaml` únicamente como adaptación para el entorno local, debido a la ausencia de soporte AVX en el procesador utilizado.

Finalmente, la aplicación fue ejecutada en AWS EC2 mediante la imagen publicada en Docker Hub y se verificó el endpoint público:

```text
/greeting?name=AWS
```

con la respuesta:

```text
Hello, AWS!
```

## 18. Evidencias

No se incluyen capturas de pantalla ni archivos de evidencia visual en este repositorio, de acuerdo con la entrega solicitada.

Las pruebas realizadas durante el laboratorio incluyeron:

- Ejecución local de `/greeting`.
- Ejecución simultánea de tres contenedores.
- Ejecución de Docker Compose.
- Inserción y consulta de un documento en MongoDB.
- Publicación de las imágenes `1.0` y `latest` en Docker Hub.
- Descarga de la imagen desde Docker Hub en AWS EC2.
- Ejecución del contenedor en EC2.
- Verificación del endpoint público en el puerto `8080`.
