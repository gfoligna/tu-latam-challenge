# LATAM Airlines - Challenge DevSecOps/SRE
## Implementación
### Parte 1 (Infra)
1. Crear VPC nueva con rango IPv4 a elección (ejemplo: `10.0.0.0/16`)
   1. Tres subnets con rango `/18`, o incluso más chicas (`/19` o `/20`) con posiblidad de acceso a internet
   2. Una route table con:
      1. Internet Gateway para el rango `0.0.0.0/0`
      2. Obviamente la ruta local para `10.0.0.0/16` (o para el rango que hayamos elegido al principio)
2. Crear AWS SNS topic llamado `latam-challenge`
   1. TO-DO: agregar SQS luego para no perder mensajes
3. Crear RDS PostgreSQL v17 (instancia tipo: `db.t4g.micro` por ahora) llamada 'db01', en una sola AZ (para instancias de prueba y para mantener el costo bajo no es necesario que la DB sea Multi-AZ)
   1. Sólo acceso privado (no asignar IP pública)
   2. Agregar Security Group para limitar el acceso sólo desde IPs privadas dentro del rango elegido al principio
   3. Sin backups (snapshots) en esta etapa de prueba
4. Para la API HTTP (lectura y escritura) usaremos [DB2Rest](https://db2rest.com/) en ECS Fargate
   1. Localmente levantarlo como prueba`docker run -p 8080:8080 --name db2rest -e DB_URL='jdbc:postgresql://127.0.0.1:5432/latam?currentSchema=public' -e DB_USER=postgres -e DB_PASSWORD=postgres -d kdhrubo/db2rest:latest`
   2. En la task definition del servicio vamos a utilizar las variables de entorno (`DB_URL`, `DB_USER`, `DB_PASSWORD`, etc.) para los inputs usados manualmente (arriba), y usar SSM Parameter Store (o Secrets Manager) para acceder a la contraseña de forma segura a la hora de hacer el deployment. Para pruebas podemos hardcodearla.
   3. Esto nos va a permitir: 
      1. Leer data (`SELECT`) utilizando el método HTTP GET
      2. Escribir data (`INSERT`) utilizando método POST
      3. Todo esto apuntando a una URL, como por ejemplo: 
         1. Para leer: `curl -XGET http://api.latamchallenge.com:8080/v1/rdbms/db/<nombre de tabla>`
         2. Escribir, por ejemplo: `curl -XPOST http://api.latamchallenge.com:8080/v1/rdbms/db/<nombre de tabla> --header 'Content-Type: application/json' --data '{"flight_number" : "LA407", "from_airport" : "MVD", "to_airport" : "SCL"}'`
      4. Documentación oficial de la herramienta: https://db2rest.com/docs/intro
      5. TO-DO: en mis pruebas locales noté que la respuesta cuando se hace un GET incluye los caracteres totales del tipo de columna, es decir que si la columna `flight_number` es `character(10)` la repsuesta va a ser `LA407-----` (caracter `-` reemplaza a los espacios por un tema del formato Markdown que se utiliza en este README)
5. Terraform
   1. WIP


### Parte 2 (App)
1. Ver Parte 1 punto 4
2. Podríamos utilizar GitHub Actions para:
   1. Registrar la Task Definition en ECS
   2. Hacer el deployment seleccionando esta última TD creada en el punto anterior. ECS va a utilizar el método [rolling deployment](https://docs.aws.amazon.com/whitepapers/latest/overview-deployment-options/rolling-deployments.html)
   3. La suscripción al topic de mensajes se haría con una Lambda que parsee la data en JSON y corra el request POST en la API HTTP
   4. Ver [diagrama.svg](diagrama.svg)
 
Usar Docker (local). DB2Rest para la API 
Crear network para correr ambos containers
GET: `curl -s -XGET http://localhost:8080/v1/rdbms/db/flight`
Fix: DB2Rest responses where value contains max column characters ('XXX      ') 

### Parte 3 (Pruebas de Integración y Puntos Críticos de Calidad)
1. Aquí escribiría un test que localmente instale dos contenedores: DB2Rest y PostgreSQL, donde utilize la misma configuración. Inicializar la DB con data falsa conservando la estructura, hacer una query utilizando la API, y por último también correr la misma query directamente a la DB. 
2. Podríamos implementar también un checkeo para tratar de insertar data con formato y/o contenido incorrecto para corroborar que la data en la DB sea fidedigna
3. A nivel de performance creo que habría varios puntos a tener en cuenta:
   4. Recursos de la DB para poder responder a la demanda requerida. Vigilar recursos y escalar la DB, o directamente utilizar [RDS Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html#aurora-rds-comparison) para no tener downtime en caso de necesitar escalar
   5. Que la API pueda ser escalada con alta disponibilidad (HA), tal vez levantando un balanceador de cargas (ALB) con múltiples instancias manejadas por AutoScaling dependiendo del uso de CPU y/o RAM
   6. Limitaciones de los servicios [SNS](https://docs.aws.amazon.com/general/latest/gr/sns.html), [SQS](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html) y [Lambda](https://docs.aws.amazon.com/general/latest/gr/lambda-service.html).
4. Respondido a grandez razgos en el punto anterior

### Parte 4: Métricas y Monitoreo
1. 3 Métricas:
   1. Que la API esté up and running (usar healthcheck `/actuator/health` [ver documentación](https://db2rest.com/docs/run-db2rest-on-docker#verify-db2rest-installation))
   2. Checkear puerto DB estableciendo conexión, o usar pg_isready (command) o un 'SELECT current_timestamp - pg_postmaster_start_time();' para ver el uptime
   3. Confirmar que servicio pub/sub esté recibiendo/enviando mensajes
2. Grafana podría ser una de las soluciones. Mostraría las mencionadas + basic health of all services/resources
   1. PostgreSQL integration for Grafana Cloud
   2. Otras integraciones nativas de Grafana + Prometheus (como [AWS Data Sources](https://grafana.com/grafana/plugins/aws-datasource-provisioner-app/?tab=overview))
3. Se podría utilizar Grafana Cloud como un SaaS, o utilizar el servicio [Amazon Managed Grafana](https://aws.amazon.com/grafana/). También se podría deployar en una instancia de EC2, pero teniendo dos grandes servicios disponibles (mencionados anteriormente) desalentaría a utilizar una instalación custom is no hay muy buena razón o requerimiento
4. Podemos medir cantidad de requests a la API, queries a la DB, hacer un aggregate de métricas de los distintos sistemas para ver un promedio y también ver si hay alguna instancia problemática en un caso particular
5. Utilizar correctamente una convención de nombres de los recursos para poder identificar correctamente cada sistema, sumado a la utilización de etiquetas (`Tags`) para poder hacer uso de ellas no sólo en el monitoreo de salud del entorno sino también para la parte de costos de AWS

### Parte 5: Alertas y SRE (Opcional)
1. Ideas:
   1. API HTTP: Si responde, y menor a 200 ms: OK. +200 ms: WARN. +500 ms o down (no responde): ALERT/CRITICAL 
   2. API y DB: Agregar checks de integridad de la data para comparar que API y DB devuelvan data idéntica
   3. Establecer pruebas de escritura: enviar mensaje de prueba al servicio Pub/Sub y confirmar que este fue escrito en la DB, así también corroborar que pueda ser leído por la API 
2. Usar 99.9 o 99.99% uptime y agregar response times al SLO. WIP.
