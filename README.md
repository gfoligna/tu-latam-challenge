# LATAM Airlines - Challenge DevSecOps/SRE

## Ideas iniciales
### Parte 1: Infraestructura e IaC
1. Ingestar, almacenar y exponer datos
   1. AWS SNS
   2. PostgreSQL
   3. ECS para servir la app containerizada y un ALB para escalar si es necesario
2. Escribir codigo TF

### Parte 2: Aplicaciones y flujo CI/CD
1. (pendiente) Busca solución en DockerHub para servir contenido de la DB, API HTTP. ¿Estructurar los GET para apuntar a una URI con: "/dbname/q="? Ver opciones disponibles de la API
2. Usar GitHub Actions para deployment?
3. Ver DockerHub o AWS para (Guardar mensajes de un topic a la DB)
4. Hacer diagrama en draw.io

### Parte 3: Pruebas de Integración y Puntos Críticos de Calidad
1. Test integración haciendo un healthcheck de la API confirmando que expone datos (usar un '/ping' o similar)
2. (literal) Proponer otras pruebas de integración que validen que el sistema está funcionando
correctamente y cómo se implementarían.
3. (literal) Identificar posibles puntos críticos del sistema (a nivel de fallo o performance)
diferentes al punto anterior y proponer formas de testearlos o medirlos (no
implementar)
4. (literal) Proponer cómo robustecer técnicamente el sistema para compensar o solucionar
dichos puntos críticos

### Parte 4: Métricas y Monitoreo - no requiere implementación
1. 3 Métricas:
   1. Que la API esté up and running (usar healthcheck)
   2. Checkear puerto DB estableciendo conexión, o usar pg_isready (command) o un 'SELECT current_timestamp - pg_postmaster_start_time();' para ver el uptime
   3. Confirmar que servicio pub/sub esté recibiendo/enviando mensajes 
2. Grafana? Mostraría las mencionadas + basic health of all services/resources
   1. PostgreSQL integration for Grafana Cloud
   2. Otras integraciones nativas de Grafana + Prometheus
3. (literal) Describe a grandes rasgos cómo sería la implementación de esta herramienta en la
nube y cómo esta recolectaría las métricas del sistema
4. (literal) Describe cómo cambiará la visualización si escalamos la solución a 50 sistemas
similares y qué otras métricas o formas de visualización nos permite desbloquear
este escalamiento
5. (literal) Comenta qué dificultades o limitaciones podrían surgir a nivel de observabilidad de
los sistemas de no abordarse correctamente el problema de escalabilidad

### Parte 5: Alertas y SRE (Opcional) - no requiere implementación
1. Basado en
   1. **4.1.1** Si responde, y menor a 200 ms: OK. +200 ms: WARN. +500 ms o down: ALERT/CRITICAL.  Agregar checks de integridad de la data que devuelve de la DB?
   2. **4.1.2** Same
   3. **4.1.3** Same? 
2. Usar 99.9 o 99.99% uptime y agregar response times al SLO