# Manejo de Errores en Spring Boot con DTOs de Respuesta

Este proyecto muestra cómo manejar errores de manera efectiva en una API RESTful utilizando Spring Boot. La idea es retornar un objeto estandarizado con información sobre el error o éxito de la operación.

## 1. DTO de Respuesta (`ApiResponse`)

Este es un objeto genérico que encapsula el resultado de una operación, ya sea exitosa o fallida. Se incluye un campo para el mensaje, los datos y un código de error técnico.

```java
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private String errorCode;

    public ApiResponse() {}

    public ApiResponse(boolean success, String message, T data, String errorCode) {
        this.success = success;
        this.message = message;
        this.data = data;
        this.errorCode = errorCode;
    }

    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, "Operación exitosa", data, null);
    }

    public static <T> ApiResponse<T> error(String message, String errorCode) {
        return new ApiResponse<>(false, message, null, errorCode);
    }

    // Getters y setters...
}
```












## 2. Ejemplo de Uso en un Controlador
El siguiente ejemplo muestra cómo se usa ApiResponse en un controlador para responder a las solicitudes de la API.

```java
@GetMapping("/usuario/{id}")
public ResponseEntity<ApiResponse<UsuarioDTO>> getUsuario(@PathVariable Long id) {
    UsuarioDTO dto = usuarioService.obtenerUsuario(id);
    return ResponseEntity.ok(ApiResponse.ok(dto));
}
```
## 3. Manejo de Errores con @ControllerAdvice
Utilizamos @ControllerAdvice para manejar excepciones globalmente, asegurándonos de devolver una respuesta coherente y apropiada cuando ocurra un error en cualquier parte de la aplicación.

``` java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ApiResponse<Object>> handleNotFound(EntityNotFoundException ex) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ex.getMessage(), "RECURSO_NO_ENCONTRADO"));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ApiResponse<Object>> handleInsertFail(DataIntegrityViolationException ex) {
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error("Error al insertar los datos", "ERROR_INSERTAR"));
    }

    @ExceptionHandler(SQLException.class)
    public ResponseEntity<ApiResponse<Object>> handleDBError(SQLException ex) {
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("Error de conexión a la base de datos", "DB_ERROR"));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Object>> handleGeneral(Exception ex) {
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("Error interno del servidor", "INTERNAL_ERROR"));
    }
}
```

## 4. Ejemplo de Respuesta en el Frontend
Cuando ocurre un error o éxito, la respuesta tiene la siguiente estructura:

Respuesta Exitosa:
```java 
{
  "success": true,
  "message": "Operación exitosa",
  "data": {
    "id": 1,
    "nombre": "Juan Pérez",
    "email": "juan@example.com"
  },
  "errorCode": null
}
```

Respuesta de Error:
``` java
{
  "success": false,
  "message": "Recurso con ID 5 no encontrado",
  "data": null,
  "errorCode": "RECURSO_NO_ENCONTRADO"
}
```
## Conclusión
Esta técnica es útil porque:

Estandariza las respuestas, lo que facilita el manejo de la información en el frontend.

Se centralizan los errores mediante el uso de un @ControllerAdvice, evitando la duplicación de código de manejo de excepciones.

Los códigos HTTP son adecuados, permitiendo una depuración más eficiente.

Recomendaciones:
Extiende ApiResponse con más campos si es necesario (por ejemplo, timestamps, errores detallados).

Mantén el manejo de errores simple pero efectivo, utilizando códigos de error claros y mensajes amigables para el usuario.















1. Qué deberías loguear (mínimo útil)

Cuando ocurre un error:

endpoint (/api/v1/...)

método HTTP (GET, POST)

IP del cliente

timestamp

usuario (si tienes auth)

query params

body (opcional, cuidado con datos sensibles)

stacktrace

requestId (clave 🔥 para debug)





Mejora tu GlobalExceptionHandler

Inyecta HttpServletRequest para capturar contexto:

```
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ApiResponse<Object>> handleNotFound(
            EntityNotFoundException ex,
            HttpServletRequest request) {

        log.warn(buildLog("NOT_FOUND", ex, request));

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ex.getMessage(), "RECURSO_NO_ENCONTRADO"));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ApiResponse<Object>> handleInsertFail(
            DataIntegrityViolationException ex,
            HttpServletRequest request) {

        log.error(buildLog("DB_ERROR", ex, request), ex);

        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error("Error al insertar los datos", "ERROR_INSERTAR"));
    }

    private String buildLog(String type, Exception ex, HttpServletRequest request) {
        return String.format(
                "[%s] %s %s | IP: %s | Query: %s | Error: %s",
                type,
                request.getMethod(),
                request.getRequestURI(),
                getClientIp(request),
                request.getQueryString(),
                ex.getMessage()
        );
    }

    private String getClientIp(HttpServletRequest request) {
        String xfHeader = request.getHeader("X-Forwarded-For");
        return (xfHeader == null) ? request.getRemoteAddr() : xfHeader.split(",")[0];
    }
}
```

# 3. Agrega un requestId (nivel PRO)

Esto es CLAVE para debug en producción.

```
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import java.io.IOException;
import java.util.UUID;

@Component
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        try {
            String requestId = UUID.randomUUID().toString();
            MDC.put("requestId", requestId);

            HttpServletRequest req = (HttpServletRequest) request;

            MDC.put("path", req.getRequestURI());
            MDC.put("method", req.getMethod());

            chain.doFilter(request, response);

        } finally {
            MDC.clear();
        }
    }
}


````
# 4. Configura logs bonitos (application.yml)


```
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%X{requestId}] %-5level %logger - %msg%n"

```
Resultado:
 ```
2026-03-20 10:15:22 [a1b2c3] ERROR GlobalExceptionHandler - [DB_ERROR] POST /api/v1/tasks | IP: 192.168.1.1 | Error: duplicate key
```






