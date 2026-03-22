# Manejo Incorrecto de Recursos: La Fuga que Nadie Ve

> Conexiones abiertas, memoria que nunca se libera, archivos que nadie cierra. El sistema funciona. Por ahora.

---

## 🗂️ Cabecera

**Título:** Manejo Incorrecto de Recursos: La Fuga que Nadie Ve  
**Subtítulo:** Conexiones abiertas, memoria que nunca se libera, archivos que nadie cierra. El sistema funciona. Por ahora.  
**Autor:** Roberto Pineda  
**Fecha:** 2026-03-31  
**Categoría:** Clean Code / Ingeniería de Software  
**Nivel:** Intermedio  
**Idioma:** Español 🇭🇳  
**Serie:** Errores Comunes en Programación — Artículo 3 de 5  
**Tags:** `java` `clean-code` `recursos` `memoria` `backend` `produccion` `jdbc`  
**Tiempo de lectura:** ~7 minutos

---

## 1. 🎣 Introducción (Hook)

Imagina un grifo que gotea. No mucho. Solo un poco.

Solo que ese grifo son conexiones de base de datos que tu aplicación abre y nunca cierra. Y cada petición al sistema abre una nueva. Y el pool de conexiones tiene un límite.

Un sistema de core bancario que procesa miles de transacciones por hora puede llegar a ese límite en cuestión de horas. Cuando lo hace, no lanza un error elegante. El sistema simplemente deja de responder. Los logs dicen `Connection pool exhausted`. El equipo de operaciones recibe la alerta. Y alguien, a las 2am, tiene que reiniciar el servidor para liberar lo que el código debió liberar solo.

He visto este escenario tres veces en producción. Siempre el mismo patrón. Siempre el mismo fix temporal. Siempre la misma causa raíz: nadie cerró el recurso.

---

## 2. 🧩 Contexto del problema

Un recurso, en términos de software, es cualquier cosa que el sistema operativo o la infraestructura te presta temporalmente: una conexión de base de datos, un archivo abierto, un socket de red, un hilo de ejecución, un bloque de memoria.

La clave es "temporalmente". Se supone que lo devuelves cuando terminas.

Cuando no lo devuelves, ocurre una **fuga de recursos** — resource leak. El recurso sigue ocupado aunque nadie lo esté usando. Con el tiempo, el sistema se queda sin recursos disponibles. Y cuando eso pasa en producción, las consecuencias van desde degradación de rendimiento hasta caída total del servicio.

El problema no es difícil de entender. Es fácil de olvidar. Y en sistemas con alta concurrencia y miles de peticiones por segundo, olvidarlo una sola vez es suficiente para tener un incidente.

---

## 3. 📖 Desarrollo del tema

### 3.1 Conexiones de base de datos: el recurso más olvidado

Una conexión a base de datos no es gratis. Toma tiempo crearla, consume memoria en el servidor de BD, y el pool tiene un límite máximo — usualmente entre 10 y 100 conexiones dependiendo de la configuración.

El error más frecuente: abrir la conexión dentro del método, pero cerrarla solo en el camino feliz.

```java
// ❌ Si ocurre una excepción antes del close(), la conexión queda abierta para siempre
public void procesarTransferencia(String cuentaId) throws SQLException {
    Connection conn = dataSource.getConnection();
    PreparedStatement stmt = conn.prepareStatement("UPDATE cuentas SET saldo = ?");
    stmt.executeUpdate(); // Si esto lanza excepción...
    stmt.close();
    conn.close();         // ...esto nunca se ejecuta
}
```

Cada excepción no manejada deja una conexión abierta. En un sistema con alta concurrencia, el pool se agota en minutos.

**Recomendación:** Usa `try-with-resources`. Java cierra automáticamente cualquier recurso que implemente `AutoCloseable`, sin importar si hubo excepción o no.

```java
// ✅ Se cierra siempre — con excepción o sin ella
public void procesarTransferencia(String cuentaId) throws SQLException {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement("UPDATE cuentas SET saldo = ?")) {
        stmt.executeUpdate();
    } // conn y stmt se cierran aquí automáticamente
}
```

---

### 3.2 Archivos: abiertos hasta que el proceso muere

El manejo de archivos tiene el mismo problema. Un archivo abierto ocupa un file descriptor del sistema operativo. Los sistemas Linux tienen un límite — típicamente 1024 por proceso, configurable pero finito.

```java
// ❌ Si el procesamiento lanza excepción, el archivo queda abierto
public void generarReporte(String ruta) throws IOException {
    BufferedWriter writer = new BufferedWriter(new FileWriter(ruta));
    writer.write(obtenerDatos()); // puede lanzar excepción
    writer.close(); // nunca llega aquí si hay error arriba
}

// ✅ Se cierra siempre
public void generarReporte(String ruta) throws IOException {
    try (BufferedWriter writer = new BufferedWriter(new FileWriter(ruta))) {
        writer.write(obtenerDatos());
    }
}
```

En sistemas que generan reportes masivos — extractos bancarios, estados de cuenta, archivos de conciliación — este error puede dejar cientos de file descriptors abiertos en minutos.

**Recomendación:** `try-with-resources` aplica para cualquier `InputStream`, `OutputStream`, `Reader`, `Writer`. No hay razón para no usarlo.

---

### 3.3 Memory leaks: cuando la memoria no regresa

Un memory leak ocurre cuando el código reserva memoria que nunca libera. En Java, el garbage collector maneja la mayoría de los casos — pero hay patrones donde la memoria se retiene sin querer.

El más común en sistemas empresariales: colecciones estáticas que crecen sin límite.

```java
// ❌ Cache estático que nunca se limpia — crece con cada petición
public class CacheTransacciones {
    private static final Map<String, Transaccion> cache = new HashMap<>();

    public void agregar(String id, Transaccion t) {
        cache.put(id, t); // nunca se elimina nada
    }
}
```

Cada petición agrega entradas. Ninguna las elimina. La memoria crece hasta que la JVM lanza `OutOfMemoryError` y el proceso muere.

```java
// ✅ Cache con tamaño máximo usando LinkedHashMap con política LRU
public class CacheTransacciones {
    private static final int MAX_ENTRADAS = 1000;
    private final Map<String, Transaccion> cache = new LinkedHashMap<>(MAX_ENTRADAS, 0.75f, true) {
        @Override
        protected boolean removeEldestEntry(Map.Entry<String, Transaccion> eldest) {
            return size() > MAX_ENTRADAS;
        }
    };
}
```

O mejor aún, usar una librería especializada como Caffeine o Ehcache que maneja políticas de expiración por tiempo y tamaño.

**Recomendación:** Si implementas un cache manual, siempre define un límite de tamaño y una política de expiración. Un cache sin límite no es un cache — es una fuga de memoria con nombre bonito.

---

### 3.4 Streams en APIs modernas: el error que trae Java 8

Con la llegada de Streams en Java 8 apareció un nuevo tipo de recurso que muchos desarrolladores no identifican como tal: los streams de I/O wrapeados en `Stream<T>`.

```java
// ❌ El stream de líneas queda abierto si no se cierra explícitamente
public long contarLineas(Path archivo) throws IOException {
    return Files.lines(archivo).count();
}

// ✅ Cerrar el stream con try-with-resources
public long contarLineas(Path archivo) throws IOException {
    try (Stream<String> lineas = Files.lines(archivo)) {
        return lineas.count();
    }
}
```

`Files.lines()` abre un file descriptor internamente. Si no se cierra, se acumula con cada llamada.

**Recomendación:** Cualquier método que retorne un `Stream` que internamente maneje I/O debe cerrarse con `try-with-resources`. La documentación de Java lo indica, pero es fácil pasarlo por alto.

---

## 4. 💻 Ejemplo práctico

Este ejemplo muestra cómo un servicio bancario maneja correctamente múltiples recursos en una sola operación:

```java
public class TransferenciaRepository {

    private final DataSource dataSource;

    // Procesa una transferencia y genera el comprobante en archivo
    public void ejecutar(Transferencia transferencia) throws Exception {

        // Todos los recursos se cierran automáticamente al salir del bloque
        try (Connection conn = dataSource.getConnection();
             PreparedStatement debitoStmt = conn.prepareStatement(
                 "UPDATE cuentas SET saldo = saldo - ? WHERE id = ?");
             PreparedStatement creditoStmt = conn.prepareStatement(
                 "UPDATE cuentas SET saldo = saldo + ? WHERE id = ?");
             BufferedWriter comprobante = new BufferedWriter(
                 new FileWriter("comprobantes/" + transferencia.getId() + ".txt"))) {

            conn.setAutoCommit(false);

            // Operaciones de BD
            debitoStmt.setBigDecimal(1, transferencia.getMonto());
            debitoStmt.setString(2, transferencia.getCuentaOrigen());
            debitoStmt.executeUpdate();

            creditoStmt.setBigDecimal(1, transferencia.getMonto());
            creditoStmt.setString(2, transferencia.getCuentaDestino());
            creditoStmt.executeUpdate();

            conn.commit();

            // Generación del archivo
            comprobante.write("Transferencia: " + transferencia.getId());
            comprobante.newLine();
            comprobante.write("Monto: L" + transferencia.getMonto());

        } catch (Exception e) {
            // Los recursos se cierran incluso antes de llegar aquí
            log.error("Error en transferencia | id={} | correlationId={}",
                transferencia.getId(), transferencia.getCorrelationId(), e);
            throw e;
        }
        // conn, debitoStmt, creditoStmt y comprobante: todos cerrados aquí
    }
}
```

**¿Qué está pasando en este ejemplo?**

`try-with-resources` con múltiples recursos los cierra todos en orden inverso al que fueron declarados — primero el `BufferedWriter`, luego los `PreparedStatement`, finalmente la `Connection`. Si alguno lanza excepción al cerrarse, Java la maneja sin ocultar la excepción original. Todo esto sin una sola línea de `finally`.

---

## 5. 🔗 Conexión con artículos anteriores

> Este es el **artículo 3** de la serie *"Errores Comunes en Programación"*.
>
> En el artículo anterior vimos cómo las **excepciones no manejadas** silencian los errores. Este artículo y el anterior están directamente relacionados: una excepción que interrumpe el flujo normal es exactamente la situación en la que los recursos quedan sin cerrar, si no usas `try-with-resources`.
>
> Son dos errores que se amplifican mutuamente. Corregir uno sin el otro deja el sistema a medias.
>
> → [← Artículo 2: Excepciones no manejadas](./manejo-excepciones.md)

---

## 6. 💡 Insight o reflexión técnica

> Un sistema con fugas de recursos funciona bien en desarrollo, bien en QA, bien la primera semana en producción.
>
> El problema aparece bajo carga real y sostenida. Para cuando lo detectas, el daño ya está hecho.
> Cerrar los recursos correctamente no es optimización — es la condición mínima para que el sistema sea confiable.

---

## 7. ✅ Conclusión

Las fugas de recursos son silenciosas, acumulativas y devastadoras bajo carga. Tres reglas concretas:

**Primero:** Usa `try-with-resources` para todo lo que implemente `AutoCloseable` — conexiones, streams, archivos, readers, writers. Sin excepciones.

**Segundo:** Si implementas algún cache o colección que crece con el tiempo, define desde el diseño su límite máximo y su política de expiración. Un cache sin límite es una fuga con alias.

**Tercero:** Prueba bajo carga sostenida antes de ir a producción. Las fugas de recursos no aparecen con 10 peticiones en QA. Aparecen con 10,000 peticiones en producción a las 3am.

El grifo que gotea no parece urgente hasta que el tanque está vacío.

---

## 8. 📣 Llamado a la acción

¿Has tenido un incidente en producción causado por agotamiento del pool de conexiones o un `OutOfMemoryError`?

¿Cuánto tiempo tardaron en identificar que era una fuga de recursos y no un problema de infraestructura?

Comparte tu experiencia — estos casos siempre tienen detalles que no aparecen en los libros.

---

## 📚 Referencias

- *Effective Java*, Item 9: Prefer try-with-resources to try-finally — Joshua Bloch
- *Clean Code*, Capítulo 7: Error Handling — Robert C. Martin
- [Java AutoCloseable docs](https://docs.oracle.com/en/java/docs/api/java.base/java/lang/AutoCloseable.html)
- [Caffeine Cache](https://github.com/ben-manes/caffeine)

---

## 🔗 Artículos relacionados en este repo

- [← Artículo 2: Excepciones no manejadas](./manejo-excepciones.md)
- [→ Artículo 4: Errores de concurrencia *(próximamente — semana del 7 de abril)*](./errores-concurrencia.md)