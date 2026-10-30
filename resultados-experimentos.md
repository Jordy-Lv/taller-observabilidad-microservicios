# Taller de observabilidad de microservicios con Spring Boot Actuator

## Estructura

- `microservicio-productos` (puerto 8081): expone `GET /productos/{id}`.
- `microservicio-pedidos` (puerto 8082): expone `GET /pedidos/{id}` y consulta a productos mediante `RestClient`.
- `microservicio-pedidos/pruebas-observabilidad.http`: solicitudes para el cliente HTTP de IntelliJ.

Ambos incluyen Spring Web y Actuator con `health`, `info` y `metrics` expuestos.

## Ejecución

```bash
cd microservicio-productos && mvn spring-boot:run
cd microservicio-pedidos && mvn spring-boot:run
```

Iniciar primero productos y luego pedidos.

## Verificación de comunicación

`GET /pedidos/1` en 8082:

```json
{"pedidoId":1,"estado":"PROCESADO","producto":{"id":1,"nombre":"Computador portatil","precio":2500000}}
```

`GET /actuator/health` en ambos servicios devuelve `{"status":"UP"}` con los componentes `diskSpace`, `ping` y `ssl`.

## Experimento 1: métricas de solicitudes

Métrica `http.server.requests`. Los valores incluyen las peticiones a Actuator realizadas durante la consulta. TOTAL_TIME y MAX en segundos.

| Servicio | Momento | COUNT | TOTAL_TIME | MAX |
|---|---|---|---|---|
| pedidos | antes | 5 | 0.9086 | 0.8214 |
| pedidos | después de 10 consultas | 16 | 1.0070 | 0.8214 |
| productos | antes | 4 | 0.3031 | 0.2359 |
| productos | después de 10 consultas | 15 | 0.3585 | 0.2359 |

Interpretación: COUNT subió en 11 en ambos servicios (las diez consultas adicionales más la propia consulta de métricas anterior), porque cada consulta a pedidos genera una llamada a productos. TOTAL_TIME creció poco porque las diez consultas posteriores fueron rápidas (alrededor de 10 ms cada una). MAX no cambió porque el valor más alto corresponde a la primera petición, que carga clases y conexiones en frío.

## Experimento 2: fallo de dependencia

Con productos detenido, `GET /pedidos/1` responde:

```json
{"timestamp":"2026-10-30T14:54:25.839+00:00","status":500,"error":"Internal Server Error","path":"/pedidos/1"}
```

La consola de pedidos registra `ResourceAccessException: I/O error on GET request for "http://localhost:8081/productos/1"` con causa raíz `ClosedChannelException`. `GET /actuator/health` de pedidos sigue devolviendo `UP`. Al reiniciar productos, pedidos vuelve a responder `PROCESADO`.

## Evidencias

Carpeta `evidencias/`, generadas desde IntelliJ IDEA y Postman:

| Entregable de la guía | Archivo |
|---|---|
| Ambos microservicios ejecutándose en IntelliJ | `01-intellij-microservicio-productos-ejecutandose.jpg`, `01-intellij-microservicio-pedidos-ejecutandose.jpg` |
| Respuesta JSON de `/pedidos/1` | `02-postman-pedidos-1-json.jpg` |
| `/actuator/health` de ambos servicios | `03-postman-health-productos.jpg`, `03-postman-health-pedidos.jpg` |
| Métricas HTTP antes | `04-postman-metricas-pedidos-antes.jpg`, `04-postman-metricas-productos-antes.jpg` |
| Métricas HTTP después | `05-postman-metricas-pedidos-despues.jpg`, `05-postman-metricas-productos-despues.jpg` |
| Productos detenido | `06-intellij-productos-detenido.jpg` |
| Error HTTP 500 en pedidos | `06-postman-pedidos-error-productos-detenido.jpg` |
| Logs y excepción en pedidos | `08-intellij-pedidos-log-error.jpg` |
| Health de pedidos en UP con productos detenido | `07-postman-health-pedidos-up-con-productos-detenido.jpg` |
| Pedidos responde de nuevo al reiniciar productos | `09-postman-pedidos-recuperado.jpg` |

## Preguntas de análisis

1. **COUNT y TOTAL_TIME:** COUNT es el número de peticiones registradas; TOTAL_TIME es la suma de sus duraciones. Dividir TOTAL_TIME entre COUNT da el tiempo promedio por petición.
2. **Pedidos en UP con productos detenido:** el health predeterminado solo comprueba recursos locales (disco, ping, SSL). No verifica servicios HTTP externos, por lo que el proceso está sano aunque su operación de negocio falle.
3. **Detección de degradación de rendimiento:** TOTAL_TIME promedio y MAX de `http.server.requests` a lo largo del tiempo, junto con el número de errores 5xx por URI.
4. **Aporte de una traza distribuida:** conecta una misma solicitud a través de varios servicios con un identificador común, mostrando cuánto tiempo se gastó en cada salto. El log de un servicio solo muestra lo que ocurrió dentro de él.
5. **Riesgos de exponer Actuator con detalles:** revela rutas del servidor, estado de componentes, versiones y comportamiento interno, lo que facilita el reconocimiento por parte de un atacante. En producción se debe restringir el acceso y limitar los detalles.

## Conclusión

Las métricas muestran cuánto y con qué rapidez responde cada servicio, los logs explican qué ocurrió en una petición concreta y las trazas permiten seguirla entre servicios. El health predeterminado no refleja la disponibilidad de dependencias remotas, por lo que se requiere un indicador de salud específico para ellas.
