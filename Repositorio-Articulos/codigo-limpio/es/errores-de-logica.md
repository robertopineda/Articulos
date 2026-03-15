# Errores de Lógica: Cuando el Código Hace lo que le Dices, No lo que Quieres

> El error más peligroso no es el que rompe el sistema. Es el que lo deja funcionar incorrectamente.

---

**Título:** Errores de Lógica: Cuando el Código Hace lo que le Dices, No lo que Quieres  
**Subtítulo:** El error más peligroso no es el que rompe el sistema. Es el que lo deja funcionar incorrectamente.  
**Autor:** Roberto Pineda  
**Fecha:** 2026-03-15  
**Categoría:** Clean Code / Ingeniería de Software  
**Nivel:** Básico  
**Idioma:** Español 🇭🇳  
**Serie:** Errores Comunes en Programación — Artículo 1 de 5  
**Tags:** `java` `clean-code` `logica` `backend` `buenas-practicas` `produccion`  
**Tiempo de lectura:** ~7 minutos

---

## 1. Introducción

Un sistema bancario procesó pagos durante tres días con el monto equivocado.

No lanzó ninguna excepción. No generó ninguna alerta. Los logs estaban limpios. El sistema, desde cualquier monitor de infraestructura, se veía perfectamente saludable.

El problema no era técnico. Era lógico.

Una condición mal escrita estaba aprobando transacciones que debían rechazarse. El código hacía exactamente lo que le habían dicho. El error estaba en lo que le dijeron.

Eso es un error de lógica. Y es el tipo de error más difícil de detectar, porque el sistema no pide ayuda.

---

## 2. Contexto del problema

Un error de lógica ocurre cuando el código es sintácticamente correcto, compila sin problemas, se ejecuta sin excepciones, pero produce resultados incorrectos porque la intención del desarrollador no quedó bien expresada en el código.

El compilador no puede detectarlo. Las herramientas de análisis estático raramente lo encuentran. Los tests no lo capturan si los tests también tienen el error de lógica. Y el sistema en producción no se queja.

Aparece en cualquier tipo de sistema, pero en entornos bancarios y financieros sus consecuencias son directas: cálculos erróneos, validaciones que no validan, condiciones de negocio que no se cumplen.

Lo que hace peligroso a este error no es su complejidad. Es su invisibilidad.

---

## 3. Desarrollo del tema

### 3.1 El error de condición: cuando los operadores mienten

El error de lógica más frecuente vive en las condiciones booleanas. Una sola diferencia entre `>` y `>=`, entre `&&` y `||`, entre `!` puesto y `!` no puesto, cambia completamente el comportamiento del sistema.

```java
// ❌ Error: aprueba cuando debería rechazar
if (saldo > limitePermitido) {
    aprobarTransaccion();
}

// ✅ Correcto: rechaza cuando el saldo supera el límite
if (saldo <= limitePermitido) {
    aprobarTransaccion();
}
```

El código incorrecto se lee con fluidez. Compila sin advertencias. Pasa revisión visual sin levantar sospechas. Solo un test con valores en el límite exacto lo descubriría.

**Recomendación:** En condiciones de negocio críticas, escribe el caso en lenguaje natural primero. "Se aprueba cuando el saldo es menor o igual al límite permitido." Luego tradúcelo. Luego verifica que la traducción sea exacta.

---

### 3.2 El error de precedencia: cuando el orden de operaciones traiciona

Las reglas de precedencia de operadores en casi cualquier lenguaje son contraintuitivas para el cerebro humano. Lo que se lee de izquierda a derecha no siempre se evalúa de izquierda a derecha.

```java
// ❌ Error de precedencia: && evalúa antes que ||
// Esto se lee como: esAdmin || (esActivo && tieneSaldo)
if (esAdmin || esActivo && tieneSaldo) {
    permitirAcceso();
}

// ✅ Con paréntesis explícitos — la intención es visible
if (esAdmin || (esActivo && tieneSaldo)) {
    permitirAcceso();
}

// O si la intención era diferente:
if ((esAdmin || esActivo) && tieneSaldo) {
    permitirAcceso();
}
```

Sin los paréntesis, el código es ambiguo para cualquier lector — incluyendo el autor original seis meses después.

**Recomendación:** En condiciones con más de dos operadores lógicos, usa paréntesis siempre, aunque no sean estrictamente necesarios. El compilador los ignora. Tu equipo te lo agradece.

---

### 3.3 El error de negación: cuando el `!` invierte lo que no debía

La negación de una condición compuesta es uno de los lugares donde más errores de lógica se esconden. Las Leyes de De Morgan existen precisamente porque la negación no se distribuye de manera intuitiva.

```java
// ❌ Intención: rechazar si NO es activo Y NO tiene saldo
// Error: esto rechaza si NO es activo O NO tiene saldo (son cosas distintas)
if (!esActivo && !tieneSaldo) {
    rechazar();
}

// ✅ Correcto según las Leyes de De Morgan
// NOT (A OR B) = NOT A AND NOT B
// NOT (A AND B) = NOT A OR NOT B
if (!(esActivo || tieneSaldo)) {
    rechazar();
}
```

**Recomendación:** Cuando necesites negar una condición compuesta, escribe primero la condición positiva. Luego niégala completa con `!()`. No intentes negar cada parte por separado mentalmente.

---

### 3.4 El error de off-by-one: cuando los límites están un paso mal

El error de off-by-one es clásico. Afecta índices, rangos, contadores, límites de paginación, validaciones de fechas. Es sutil porque la diferencia es exactamente uno — y uno parece poco hasta que no lo es.

```java
// ❌ Error: el último elemento nunca se procesa
for (int i = 0; i < lista.size() - 1; i++) {
    procesar(lista.get(i));
}

// ✅ Correcto
for (int i = 0; i < lista.size(); i++) {
    procesar(lista.get(i));
}
```

En un sistema de pagos, saltarse el último elemento de una lista de transacciones por un error de off-by-one puede significar una transferencia no procesada por cada lote. Invisible. Silenciosa. Acumulada.

**Recomendación:** Siempre prueba los valores en los bordes: el primero, el último, el que está exactamente en el límite. Son los casos que los tests normales no cubren y los errores de off-by-one sí afectan.

---

## 4. Ejemplo práctico

El siguiente escenario muestra cómo un error de lógica puede pasar por múltiples capas sin ser detectado:

```java
public class ValidadorTransferencia {

    private static final BigDecimal LIMITE_DIARIO = new BigDecimal("50000.00");

    // ❌ Versión con error de lógica
    public boolean puedeTransferir(BigDecimal monto, BigDecimal totalTransferidoHoy) {
        // Intención: aprobar si el total del día MÁS el monto nuevo
        // no supera el límite diario
        // Error: la condición está invertida
        if (totalTransferidoHoy.add(monto).compareTo(LIMITE_DIARIO) > 0) {
            return true; // ← Aprueba cuando debería rechazar
        }
        return false;
    }

    // ✅ Versión corregida
    public boolean puedeTransferir_v2(BigDecimal monto, BigDecimal totalTransferidoHoy) {
        BigDecimal totalProyectado = totalTransferidoHoy.add(monto);
        boolean dentroDelLimite = totalProyectado.compareTo(LIMITE_DIARIO) <= 0;
        return dentroDelLimite;
    }
}
```

**¿Qué está pasando en este ejemplo?**

La versión con error aprueba exactamente los casos que debería rechazar: cuando el total supera el límite. La condición es correcta en sintaxis, en tipos, en compilación. Solo está invertida en lógica.

La versión corregida introduce una variable intermedia con nombre descriptivo (`totalProyectado`, `dentroDelLimite`). Esto hace que la intención sea legible sin necesidad de interpretar la condición.

Esa variable extra no tiene costo en rendimiento. Tiene un valor enorme en claridad.

---

## 5. Conexión con artículos anteriores

> Este es el **artículo 1** de la serie *"Errores Comunes en Programación"*.
>
> Es el punto de partida porque los errores de lógica son la base de todos los demás.  
> En el siguiente artículo de la serie analizamos qué pasa cuando el sistema sí detecta el error pero nadie lo está escuchando: las excepciones no manejadas.
>
> → [Artículo 2: Excepciones no manejadas — el error que nadie ve hasta que es tarde](./manejo-excepciones.md)

---

## 6. Insight o reflexión técnica

> El compilador verifica la sintaxis. Los tests verifican los casos que pensaste. Solo el criterio técnico verifica si lo que escribiste es realmente lo que quisiste decir.
>
> Un error de lógica es, en el fondo, una brecha entre la intención y la expresión. Y esa brecha no la cierra ninguna herramienta automática — la cierra quien escribe código pensando en lo que dice, no solo en lo que compila.

---

## 7. Conclusión

Los errores de lógica son los más silenciosos y, por eso, los más costosos.

No piden ayuda. No generan alertas. No aparecen en los dashboards de infraestructura. Viven en producción durante días, semanas o meses hasta que alguien revisa los datos con suficiente atención.

Tres hábitos que los reducen de forma significativa:

**Primero:** Escribe la condición en lenguaje natural antes de escribirla en código. Si no puedes leerla en voz alta y entenderla, el código tampoco va a entenderla correctamente.

**Segundo:** Usa variables intermedias con nombres que expresen intención. `dentroDelLimite` es mejor que `monto.compareTo(limite) <= 0` inline, aunque sean equivalentes.

**Tercero:** Prueba siempre los valores en los bordes. El primer elemento, el último, el exactamente en el límite, el cero, el negativo. Son los casos donde los errores de lógica viven.

El código que parece correcto y produce resultados incorrectos es el enemigo más difícil. Reconocerlo empieza por saber que existe.

---

## 8. Llamado a la acción

¿Has encontrado un error de lógica en producción que pasó desapercibido por más tiempo del que debería?

¿Qué práctica o herramienta te ha ayudado más a detectarlos antes de que lleguen a producción?

Comparte tu experiencia — estos casos son los que más se aprende leyendo.

---

## 📚 Referencias

- *Clean Code*, Capítulo 4 (Comentarios) y Capítulo 17 (Code Smells) — Robert C. Martin
- *The Pragmatic Programmer*, Capítulo "Debugging" — David Thomas & Andrew Hunt
- *Code Complete*, Capítulo 19 "General Control Issues" — Steve McConnell

---

## 🔗 Artículos relacionados en este repo

- [→ Artículo 2: Excepciones no manejadas](./manejo-excepciones.md)
- [→ Artículo 3: Manejo incorrecto de recursos](./manejo-recursos.md)

---




---

*¿Preguntas o comentarios? → [X/@RobertoPinedaHn](https://x.com/RobertoPinedaHn) · [LinkedIn](https://www.linkedin.com/in/roberto-pineda-hernandez/)*
