# Manejo de Excepciones: El Error que Todos Cometen

**Autor:** Roberto Pineda  
**Fecha:** 2026-03-15  
**Categoría:** Clean Code  
**Tiempo de lectura:** ~6 minutos  
**Tags:** `java` `clean-code` `excepciones` `backend` `buenas-practicas`

---

## ¿De qué trata este artículo?

El manejo de excepciones es una de las áreas donde más deuda técnica se acumula silenciosamente. Este artículo explica los errores más comunes y cómo diseñar un flujo de excepciones que comunique intención, facilite el debugging y no rompa la lógica de negocio.

Dirigido a desarrolladores Java con experiencia media o senior que trabajan en sistemas empresariales.

---

## El problema

Cuántas veces has visto algo así en producción:

```java
try {
    procesarPago(monto, cuenta);
} catch (Exception e) {
    e.printStackTrace();
}
```

O peor:

```java
} catch (Exception e) {
    // TODO: manejar esto después
}
```

Estos patrones son bombas de tiempo. En sistemas bancarios o financieros, una excepción silenciada puede significar una transacción perdida, un registro corrupto o una auditoría fallida.

El problema no es que los desarrolladores sean descuidados. El problema es que nadie les enseñó a diseñar el flujo de excepciones como parte de la arquitectura.

---

## Principios para un manejo correcto

### 1. No captures lo que no puedes manejar

Capturar una excepción solo tiene sentido si puedes hacer algo útil con ella: recuperarte, transformarla, registrarla con contexto, o relanzarla con más información.

```java
// ❌ Mal — captura sin propósito
try {
    cliente = repositorio.buscarPorId(id);
} catch (Exception e) {
    return null;
}

// ✅ Bien — transforma con contexto
try {
    cliente = repositorio.buscarPorId(id);
} catch (ClienteNoEncontradoException e) {
    throw new RecursoNoDisponibleException("Cliente ID: " + id + " no existe", e);
}
```

### 2. Usa excepciones específicas, no genéricas

Crea tu propia jerarquía que refleje el dominio de tu aplicación:

```
AppException (base)
├── NegocioException
│   ├── SaldoInsuficienteException
│   ├── CuentaBloqueadaException
│   └── LimiteTransaccionException
└── InfraestructuraException
    ├── ConexionBDException
    └── ServicioExternoException
```

### 3. Registra con contexto suficiente

```java
// ❌ Mal
log.error("Error al procesar", e);

// ✅ Bien
log.error("Error procesando transferencia | cuentaOrigen={} | monto={} | correlationId={}",
    cuentaOrigen, monto, correlationId, e);
```

### 4. Separa negocio de infraestructura

Las fallas de negocio (saldo insuficiente) son parte del flujo normal. Las de infraestructura (BD caída) son eventos inesperados. No las trates igual.

### 5. En APIs REST, usa un manejador centralizado

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NegocioException.class)
    public ResponseEntity<ErrorResponse> manejarNegocio(NegocioException ex) {
        return ResponseEntity
            .status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(new ErrorResponse(ex.getCodigo(), ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> manejarGenerico(Exception ex) {
        log.error("Error no controlado", ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("ERR_INTERNO", "Ocurrió un error inesperado"));
    }
}
```

---

## Conclusión

Tres reglas para llevarte:

1. **Captura solo lo que puedes manejar.** Si no sabes qué hacer con la excepción, déjala subir.
2. **Usa excepciones que hablen el idioma del dominio.** No `RuntimeException` — `SaldoInsuficienteException`.
3. **Registra con contexto suficiente** para debuggear en producción sin reproducir el error.

El sistema que más te va a agradecer esto eres tú mismo, seis meses después, a las 11pm con una alerta en producción.

---

## Referencias

- *Clean Code*, Capítulo 7 — Robert C. Martin
- *Effective Java*, Items 69-77 — Joshua Bloch

---

## Artículos relacionados

- [Principios SOLID en Java EE](./solid-java-ee.md)
- [Por qué el código limpio no es un lujo](./codigo-limpio-necesidad.md)

---

*¿Preguntas? → [X/@RobertoPinedaHn](https://x.com/RobertoPinedaHn) · [LinkedIn](https://www.linkedin.com/in/roberto-pineda-hernandez/)*
