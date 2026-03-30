# Errores de Concurrencia: Cuando Dos Hilos Quieren lo Mismo al Mismo Tiempo

> El bug que no puedes reproducir en local, no aparece en QA, y destruye datos en producción.

---

## Cabecera

**Título:** Errores de Concurrencia: Cuando Dos Hilos Quieren lo Mismo al Mismo Tiempo  
**Subtítulo:** El bug que no puedes reproducir en local, no aparece en QA, y destruye datos en producción.  
**Autor:** Roberto Pineda  
**Fecha:** 2026-03-29 
**Categoría:** Clean Code / Ingeniería de Software  
**Nivel:** Avanzado  
**Idioma:** Español 🇭🇳  
**Serie:** Errores Comunes en Programación — Artículo 4 de 5  
**Tags:** `java` `concurrencia` `threads` `race-condition` `backend` `produccion` `sincronizacion`  
**Tiempo de lectura:** ~9 minutos

---

## 1.Introducción

Dos cajeros automáticos. Un cliente. Una cuenta con L5,000.

El cliente retira L5,000 desde un cajero en Tegucigalpa. Al mismo tiempo, alguien más retira L5,000 desde otro cajero usando una tarjeta duplicada — un fraude real que ocurre mientras la primera transacción aún está procesándose.

Ambos cajeros consultan el saldo. Ambos ven L5,000 disponibles. Ambos aprueban el retiro. La cuenta termina en L-5,000.

Eso es una race condition. Y no necesitas fraude para que ocurra — basta con dos peticiones legítimas llegando al mismo tiempo a un sistema que no fue diseñado para manejarlas juntas.

He revisado sistemas bancarios donde este escenario era posible. No en teoría. En producción.

---

## 2.Contexto del problema

Un error de concurrencia ocurre cuando dos o más hilos de ejecución acceden y modifican el mismo dato al mismo tiempo, sin coordinación entre ellos.

El resultado es impredecible. Puede ser un saldo negativo, un registro duplicado, un contador incorrecto, o datos corruptos que nadie detecta hasta que el daño ya está hecho.

Lo que hace a estos errores especialmente difíciles es su naturaleza no determinista. No ocurren siempre. Dependen del timing exacto entre hilos — cuál llega primero, cuánto tarda el scheduler del sistema operativo, qué tan cargado está el servidor. En local con una sola petición, nunca aparecen. Bajo carga real con cientos de peticiones simultáneas, aparecen exactamente cuando menos los necesitas.

Los sistemas financieros son el terreno más peligroso para estos errores. Cada transacción modifica estado compartido. Cada milisegundo de diferencia puede significar dinero real.

---

## 3.Desarrollo del tema

### 3.1 Race condition: el error de los dos cajeros

Una race condition ocurre cuando el resultado de una operación depende del orden no controlado en que los hilos se ejecutan.

El ejemplo clásico en sistemas de pagos: verificar saldo y debitar en dos pasos separados.

```java
// ❌ Race condition — dos hilos pueden pasar la verificación al mismo tiempo
public class CuentaService {

    public void debitar(String cuentaId, BigDecimal monto) {
        Cuenta cuenta = repositorio.buscar(cuentaId);

        // Hilo A verifica: saldo = 5000, monto = 5000 → OK, pasa
        // Hilo B verifica: saldo = 5000, monto = 5000 → OK, pasa (todavía no se debitó)
        if (cuenta.getSaldo().compareTo(monto) < 0) {
            throw new SaldoInsuficienteException(cuentaId, monto);
        }

        // Hilo A debita: saldo = 0
        // Hilo B debita: saldo = -5000
        cuenta.setSaldo(cuenta.getSaldo().subtract(monto));
        repositorio.guardar(cuenta);
    }
}
```

Entre la verificación y el débito hay una ventana de tiempo. Bajo carga, dos hilos pueden pasar esa ventana simultáneamente.

**Recomendación:** La verificación y la modificación deben ser atómicas — una sola operación indivisible. En bases de datos, esto se logra con operaciones condicionales a nivel de SQL, no en código Java.

```java
// ✅ Operación atómica en BD — verifica y debita en un solo paso
public void debitar(String cuentaId, BigDecimal monto) {
    int filasAfectadas = jdbcTemplate.update(
        "UPDATE cuentas SET saldo = saldo - ? WHERE id = ? AND saldo >= ?",
        monto, cuentaId, monto
    );

    if (filasAfectadas == 0) {
        throw new SaldoInsuficienteException(cuentaId, monto);
    }
}
```

Si la fila no se actualiza, significa que el saldo era insuficiente o que otro hilo ya realizó el débito. Un solo resultado, sin ventana de tiempo.

---

### 3.2 Datos compartidos sin sincronización: el contador roto

Imagina un contador de transacciones procesadas en el día. Simple, ¿no? Un entero que se incrementa con cada transacción.

```java
// ❌ No thread-safe — varios hilos incrementando el mismo entero
public class EstadisticasService {
    private int transaccionesProcesadas = 0;

    public void registrar() {
        transaccionesProcesadas++; // Esto no es una operación atómica
    }
}
```

`transaccionesProcesadas++` parece una línea, pero son tres instrucciones: leer el valor, sumar uno, escribir el resultado. Si dos hilos leen el mismo valor antes de que alguno escriba, uno de los incrementos se pierde.

Con 1,000 peticiones simultáneas, el contador puede terminar en 950 en lugar de 1,000. Nadie nota 50 transacciones "perdidas" en las métricas hasta que alguien hace una auditoría.

```java
// ✅ Atómico por diseño — sin synchronized, sin locks
import java.util.concurrent.atomic.AtomicInteger;

public class EstadisticasService {
    private final AtomicInteger transaccionesProcesadas = new AtomicInteger(0);

    public void registrar() {
        transaccionesProcesadas.incrementAndGet(); // Atómico garantizado
    }

    public int obtener() {
        return transaccionesProcesadas.get();
    }
}
```

`AtomicInteger` usa instrucciones de hardware que garantizan atomicidad sin necesidad de bloqueos. Es más rápido que `synchronized` y elimina el riesgo de deadlock.

**Recomendación:** Para contadores y acumuladores compartidos entre hilos, usa `AtomicInteger`, `AtomicLong` o `AtomicReference`. Para estructuras más complejas, considera `ConcurrentHashMap` en lugar de `HashMap`.

---

### 3.3 Deadlock: cuando dos hilos se bloquean mutuamente

Un deadlock ocurre cuando dos hilos esperan indefinidamente el uno al otro.

Hilo A tiene el lock del recurso 1 y espera el lock del recurso 2.  
Hilo B tiene el lock del recurso 2 y espera el lock del recurso 1.  
Ninguno puede avanzar. El sistema se congela.

```java
// ❌ Potencial deadlock — el orden de los locks no está garantizado
public void transferir(Cuenta origen, Cuenta destino, BigDecimal monto) {
    synchronized (origen) {          // Hilo A bloquea cuenta A
        synchronized (destino) {     // Hilo A espera cuenta B
            origen.debitar(monto);   // Hilo B bloqueó cuenta B primero
            destino.acreditar(monto);// Hilo B espera cuenta A → DEADLOCK
        }
    }
}
```

Si dos transferencias van en direcciones opuestas al mismo tiempo — A→B y B→A — cada hilo tiene el lock que el otro necesita.

```java
// ✅ Orden consistente de locks — siempre por ID, elimina el deadlock
public void transferir(Cuenta origen, Cuenta destino, BigDecimal monto) {
    Cuenta primera = origen.getId().compareTo(destino.getId()) < 0 ? origen : destino;
    Cuenta segunda = primera == origen ? destino : origen;

    synchronized (primera) {
        synchronized (segunda) {
            origen.debitar(monto);
            destino.acreditar(monto);
        }
    }
}
```

Al ordenar los locks siempre de la misma manera, ambos hilos compiten por el mismo lock primero. Uno espera, el otro avanza. Sin deadlock.

**Recomendación:** Si necesitas múltiples locks, define siempre el mismo orden de adquisición. Sin excepciones. Y considera usar `java.util.concurrent.locks.ReentrantLock` con timeout para detectar deadlocks en lugar de bloquearte indefinidamente.

---

### 3.4 Visibilidad: cuando un hilo no ve los cambios del otro

Este es el error de concurrencia más invisible de todos. No hay excepción, no hay corrupción visible, no hay deadlock. Solo valores desactualizados que un hilo lee sin saber que otro ya los modificó.

```java
// ❌ Sin volatile — el hilo B puede leer un valor cacheado en su CPU
public class CircuitBreaker {
    private boolean abierto = false; // cada CPU puede tener su propia copia en cache

    public void abrir()  { abierto = true;  }
    public boolean estaAbierto() { return abierto; } // puede retornar false aunque sea true
}

// ✅ Con volatile — fuerza lectura/escritura directo a memoria principal
public class CircuitBreaker {
    private volatile boolean abierto = false;

    public void abrir()  { abierto = true;  }
    public boolean estaAbierto() { return abierto; }
}
```

Sin `volatile`, la JVM puede cachear el valor de `abierto` en el registro de la CPU. Un hilo abre el circuito, pero otro hilo sigue leyendo el valor viejo del cache. El circuit breaker no funciona.

**Recomendación:** Usa `volatile` para flags de control compartidos entre hilos cuando la operación sea solo lectura/escritura simple. Para operaciones compuestas (leer-modificar-escribir), `volatile` no es suficiente — necesitas `AtomicReference` o `synchronized`.

---

## 4.Ejemplo práctico

Un servicio de transferencia bancaria que combina todos los principios anteriores:

```java
public class TransferenciaService {

    private final JdbcTemplate jdbc;
    private final AtomicLong transferenciasHoy = new AtomicLong(0);
    private volatile boolean servicioActivo = true;

    // Circuit breaker simple
    public void desactivar() { servicioActivo = false; }

    public TransferenciaResult ejecutar(String origenId, String destinoId,
                                         BigDecimal monto, String correlationId) {
        // Verificar estado del servicio con visibilidad garantizada
        if (!servicioActivo) {
            throw new ServicioNoDisponibleException("Transferencias desactivadas temporalmente");
        }

        // Operación atómica en BD — verifica y debita en un solo UPDATE
        int debitoExitoso = jdbc.update(
            "UPDATE cuentas SET saldo = saldo - ? WHERE id = ? AND saldo >= ? AND activa = true",
            monto, origenId, monto
        );

        if (debitoExitoso == 0) {
            log.warn("Débito fallido | origen={} | monto={} | correlationId={}",
                origenId, monto, correlationId);
            throw new SaldoInsuficienteException(origenId, monto);
        }

        // Crédito — si falla, necesita compensación (ver Saga Pattern, artículo #27)
        jdbc.update(
            "UPDATE cuentas SET saldo = saldo + ? WHERE id = ? AND activa = true",
            monto, destinoId
        );

        // Contador atómico — sin race condition
        long totalHoy = transferenciasHoy.incrementAndGet();

        log.info("Transferencia exitosa | origen={} | destino={} | monto={} | totalHoy={} | correlationId={}",
            origenId, destinoId, monto, totalHoy, correlationId);

        return new TransferenciaResult(true, totalHoy);
    }
}
```

**¿Qué está pasando en este ejemplo?**

Tres capas de protección. El flag `servicioActivo` usa `volatile` para garantizar visibilidad entre hilos. El débito usa un UPDATE condicional atómico en BD — sin ventana de tiempo entre verificar y modificar. El contador usa `AtomicLong` para incrementos sin race condition. Ningún `synchronized`, ningún lock explícito — y sin posibilidad de deadlock.

---

## 5.Conexión con artículos anteriores

> Este es el **artículo 4** de la serie *"Errores Comunes en Programación"*.
>
> En el artículo anterior vimos cómo los recursos no cerrados crean fugas silenciosas. Los errores de concurrencia son la siguiente escala del mismo problema: ahora no es un recurso que se fuga — es el estado compartido entre múltiples hilos que se corrompe.
>
> Y cuando ese estado es dinero, la consecuencia no es un servidor lento. Es saldo negativo.
>
> → [← Artículo 3: Manejo incorrecto de recursos](./manejo-recursos.md)

---

## 6.Reflexión técnica

> Un sistema single-threaded tiene errores de lógica. Un sistema multi-threaded tiene errores de lógica que además dependen del tiempo.
>
> La diferencia es que los segundos no puedes reproducirlos a voluntad. Solo aparecen cuando el sistema está bajo carga real. Para cuando los detectas, el daño ya está hecho.
>
> Diseñar para concurrencia no es una optimización — es una responsabilidad del arquitecto desde el primer día.

---

## 7.Conclusión

Los errores de concurrencia son los más difíciles de la serie, no porque el concepto sea complicado, sino porque no se manifiestan en condiciones normales. Tres reglas para llevar:

**Primero:** Mueve las operaciones compuestas (verificar + modificar) a la base de datos como un solo UPDATE condicional. Es la forma más confiable de garantizar atomicidad en sistemas con múltiples hilos.

**Segundo:** Para contadores y flags compartidos, usa las clases del paquete `java.util.concurrent.atomic`. Son más rápidas que `synchronized` y no generan deadlocks.

**Tercero:** Si necesitas múltiples locks, define un orden de adquisición consistente y nunca lo cambies. Un lock fuera de orden es un deadlock esperando el momento equivocado.

Y si no estás seguro de si tu código tiene problemas de concurrencia: pruébalo bajo carga. Con 500 peticiones simultáneas al mismo endpoint. Si el resultado es siempre el esperado, probablemente está bien. Si a veces falla, tienes una race condition.

---

## 8.Llamado a la acción

¿Has encontrado una race condition en producción? ¿Cómo la detectaron — un reporte de auditoría, una alerta de monitoreo, o un cliente que llamó?

¿Qué herramientas usas para probar concurrencia antes de ir a producción?

Comparte tu caso — los errores de concurrencia son los que más varían entre sistemas y siempre hay algo nuevo que aprender de cada uno.

---

## 📚 Referencias

- *Java Concurrency in Practice* — Brian Goetz (el libro de referencia para este tema)
- *Effective Java*, Items 78–84: Concurrency — Joshua Bloch
- *Clean Code*, Capítulo 13: Concurrency — Robert C. Martin
- [java.util.concurrent docs](https://docs.oracle.com/en/java/docs/api/java.base/java/util/concurrent/package-summary.html)

---

## Artículos relacionados en este repo

- [← Artículo 3: Manejo incorrecto de recursos](./manejo-recursos.md)
- [→ Artículo 5: Malas prácticas de logging *(próximamente — semana del 14 de abril)*](./malas-practicas-logging.md)

---
