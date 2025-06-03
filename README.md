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
