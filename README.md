# AgroSynchro - Plataforma Cloud para Agricultura de Precisión

> **Arquitectura Serverless AWS** - Integración de datos IoT y análisis de imágenes de drones para optimización agrícola

## 🏗️ Arquitectura General

### Separación de Responsabilidades
- **API Gateway**: Recepción de datos externos (sensores IoT, drones)  
- **Fargate**: Motor de procesamiento containerizado con auto-scaling
- **RDS**: Base de datos PostgreSQL Multi-AZ para persistencia
- **Queue**: Cola de mansajes para procesar datos de los sensores e imágenes

## 📦 Módulos de Infraestructura

### Módulos Personalizados (6)
- **`lambda/`** - Funciones Lambda para API y procesamiento inicial
- **`s3/`** - Buckets optimizados: frontend público, imágenes raw/procesadas privadas
- **`sqs/`** - Colas de mensajes con dead letter queue para tolerancia a fallos
- **`api-gateway/`** - REST API con integración Lambda/SQS y autenticación Cognito
- **`cognito/`** - User pools y autenticación OAuth2/OpenID Connect
- **`fargate/`** - Workers containerizados para procesamiento de imágenes

### Módulos Externos Empresariales (2)
- **`terraform-aws-modules/vpc/aws`** - VPC enterprise-grade con Multi-AZ, NAT gateways
- **`terraform-aws-modules/rds/aws`** - PostgreSQL con backups automáticos, encriptación

## 🎛️ Características Terraform Avanzadas

### Meta-argumentos Implementados
- **`depends_on`**: Control explícito de dependencias entre módulos (5 recursos)
- **`for_each`**: Tracking dinámico de componentes de infraestructura
- **`count`**: Recursos condicionales para réplicas y permisos
- **`lifecycle`**: `create_before_destroy` para recursos críticos

### Funciones Terraform (11 implementadas)
`split()`, `tostring()`, `md5()`, `jsonencode()`, `length()`, `lookup()`, `filemd5()`, `sha1()`, `slice()`, `formatdate()`, `timestamp()`

## 🚀 Despliegue Automatizado

### Prerrequisitos
- **Terraform >= 1.0**
- **AWS CLI configurado** (preferiblemente con LabRole para AWS Academy)  
- **Docker Desktop** (para processing engine)
- **Node.js + npm** (para frontend)
- **jq, bc** (utilities Unix estándar)

### Deployment 

**Pasos de ejecución:**

1. **Clonar y preparar el proyecto**

```bash
git clone https://github.com/FelipeCupito/agrosynchro-platform
cd agrosynchro-platform
```

2. **Configurar AWS CLI**

```bash
aws configure
# Ir al AWS Academy Lab y obtener credentials
# AWS Access Key ID: ***********
# AWS Secret Access Key: ***********
# AWS Session Token: ***********
# Default region name: us-east-1
# Default output format: json
```

3. **Ejecutar deployment**

```bash
   # Paso 1: Inicializar terraform
    cd terraform
    terraform init
    cd ..

   # Paso 2: Hacer ejecutables los scripts
   chmod +x terraform/scripts/*.sh
   chmod +x terraform/send_sensor_data.sh
   
   # Paso 3: Ejecutar el siguiente script para comenzar con el deploy
   ./terraform/scripts/deploy.sh 
   
   # Paso 4: Subir la imagen de docker. Es necesario tener docker corriendo antes de ejecutarlo.
   ./terraform/scripts/update-docker-ecr.sh

   # Dado que los reportes de generan mediante Inteligenica Artificial, es necesario utilizar una APIKey.

   # Paso 5: 
   ./terraform/scripts/update-api-key.sh

   # debe pasarle como argumento a este script su APIKEY para poder generar los reportes.

   # De todas formas ya está en uso una APIKEY de uno de los integrantes del grupo, pero si se produce un error al generar los reortes (producto de que se alcanzó el rate limit de la APIKEY), entonces será necesario introducir una nueva.
```
 Aclaración: Entendemos que tendríamos que haber utilizado el Servicio de "Secrets Manager" para guardar la APIKey utilizada para generar los reportes y que no hacerlo genera un problema de seguridad al tener la APIKey expuesta. Pudimos integrarlo a la arquitectura, pero no logramos que funcione correctamente la generación de reportes y por ese motivo optamos por no incluirlo, de todas formas queríamos comentarlo.

```bash
   # Paso 6 (opcional): En caso de que cognito falle, por favor ejecutar los siguientes comandos. 
   
   #Es importante estar parado en el directorio correcto al momento de correr el script.

   cd terraform
   ./scripts/update-cognito-lambda.sh
```

4. **Enviar datos de prueba**

Para poder probar el correcto funcionamiento de la aplicación se deben correr los siguientes scripts que se encargan de simular el envío de datos de sensores y de imágenes

Primero está el script que envía datos de sensores. Se va a solicitar el ID del usuario en cuestión, para poder asociar a dicho ID los datos que se envian.

Importante: El ID se encuentra en la parte superior derecha (debajo del mail del usuario), una vez realizada la autenticación mediante cognito.

```bash
	cd terraform
	./send_sensor_data.sh
   cd ..
```
El programa le solicitará que ingrese el número de ID del usuario.

Luego el script para cargar imágenes. También le solicitará el ID del usuario para el que quiere asociar las imágenes y además la ruta donde están las imágenes que quiere cargar.

Hay que ejecutarlo parado /terraform/scripts

```bash
   cd terraform/scripts/
	./upload_directory_images.sh
```
El script recibe dos parámetros:
1. La ruta del directorio donde se encuentran las imágenes a subir.
2. El ID del usuario

ejemplo:
```bash
   ./upload_directory_images.sh ./images/ 1
```

## Uso de la aplicación

Es importante una vez autenticado, establecer los umbrales para poder recibir las alarmas. Una vez hecho esto se podrán visualizar el promedio de las mediciones de los sensores y las alarmas disparadas.

También hay una sección en donde se pueden ver las imágenes subidas con un procesamiento para detectar situaciones irregulares.

Finalmente hay un apartado para la generación de reportes (en donde se emplea la APIKey) , y se puede obtener un reporte basado en los resultados arrojados por los sensores.
## Elección de arquitectura

Agrosynchro Cloud Architecture

Red (VPC, subnets y NAT Gateways)

La infraestructura de red se compone de subnets publicas, privadas y para la base de datos para poder aislar los diferentes niveles de la aplicacion y restringir el acceso al backend. Multiples AZs garantizan continuidad ante fallas en una zona

⸻

Almacenamiento S3

Se definieron 3 buckets:

- Frontend: Aloja los archivos estáticos del sitio web (HTML, CSS, JS).
- Imágenes sin procesar: Recibe las imágenes enviadas por drones o sensores.
- Imágenes procesadas: Almacena los resultados generados tras el análisis.

Justificación:
S3 ofrece almacenamiento de alta durabilidad, disponibilidad y bajo costo.
El acceso desde los servicios internos se realiza a través de un VPC Endpoint, evitando tráfico por Internet.

⸻

API Gateway

El API Gateway actúa como punto de entrada único para los clientes y servicios externos.
Expone endpoints REST que permiten:

- La recepción de datos desde los sensores IoT.
- El envío de mensajes a SQS.
- La invocación de funciones Lambda para procesamiento o validaciones.
- La integración con Cognito para autenticación.

Justificación:
API Gateway desacopla completamente la capa de acceso del backend, permite control de tráfico, autenticación, logging, rate limiting y escalabilidad automática.

⸻

SQS

SQS funciona como una cola de mensajeria intermedia entre la ingesta de datos (API Gateway y Lambda) y el procesamiento posterior (Fargate).

Justificación:
Introduce un mecanismo de comunicación asíncrono que desacopla los componentes, mejora la tolerancia a fallos y permite procesar cargas variables sin pérdida de mensajes.
Tambien facilita la escalabilidad horizontal de los servicios de procesamiento.

⸻

Fargate

El servicio Fargate ejecuta contenedores que:

1. Consumen mensajes de SQS.

2. Descargan imágenes sin procesar desde S3.

3. Ejecutan algoritmos de procesamiento o análisis.

4. Guardan los resultados en el bucket de imágenes procesadas.

5. Registran metadatos y resultados en RDS.

Justificación:
Fargate permite ejecutar contenedores sin administrar servidores, ajustando automáticamente la capacidad a la carga de trabajo.
El acceso a S3 se realiza mediante el VPC Endpoint, lo que elimina la necesidad de exponer el tráfico a Internet y reduce costos.

⸻

Lambda

Lambda se utiliza para operaciones rápidas y eventos desencadenados por API Gateway o SQS, tales como:

- Validación o transformación de datos.

- Envío de mensajes a la cola SQS.

- Comunicación con Cognito o servicios externos.

Justificación:
Permite ejecutar código bajo demanda con costos proporcionales al uso y sin administración de infraestructura.
Las funciones se despliegan dentro de la VPC, con acceso directo y seguro a RDS y otros servicios internos.

⸻

RDS

La base de datos relacional RDS almacena información estructurada y metadatos asociados a los procesos.

- Instancia principal: maneja las operaciones de lectura y escritura.
- Réplica de solo lectura: proporciona redundancia y capacidad adicional para consultas no críticas.

Justificación:
RDS asegura persistencia, integridad y disponibilidad de datos.
El aislamiento dentro de subnets privadas y las reglas de seguridad limitan el acceso únicamente a los servicios internos (Lambda y Fargate).

⸻

Cognito

Cognito gestiona la autenticación y autorización de los usuarios.
Incluye dominios personalizados, redirecciones seguras (callback y logout) y soporte para OAuth2 y OpenID Connect.

Justificación:
Proporciona autenticación centralizada y escalable sin necesidad de desarrollar ni mantener un sistema propio de gestión de usuarios.
Se integra nativamente con API Gateway y Lambda.

⸻

VPC Endpoint

El VPC Endpoint permite que los servicios dentro de la VPC (como Fargate y Lambda) accedan a S3 sin pasar por Internet.

Justificación:
Mejora la seguridad y el rendimiento, reduce la latencia y elimina costos asociados al tráfico a través de NAT Gateways.

⸻
