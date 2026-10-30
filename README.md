# Taller de observabilidad de microservicios

Dos microservicios Spring Boot 3.5 con Java 21 y Maven, con Actuator y métricas HTTP.

- `microservicio-productos`: puerto 8081, `GET /productos/{id}`
- `microservicio-pedidos`: puerto 8082, `GET /pedidos/{id}`, consulta a productos

## Ejecución

```bash
cd microservicio-productos && mvn spring-boot:run
cd microservicio-pedidos && mvn spring-boot:run
```

Iniciar primero productos y luego pedidos.

## Endpoints

- `http://localhost:8082/pedidos/1`
- `http://localhost:8081/actuator/health`
- `http://localhost:8082/actuator/health`
- `http://localhost:8082/actuator/metrics/http.server.requests`
- `http://localhost:8081/actuator/metrics/http.server.requests`

El informe con evidencias está en `Taller_Observabilidad_Microservicios.docx` y los resultados en `resultados-experimentos.md`.
