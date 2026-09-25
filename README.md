# Ecosistema de Automatización IA — Recepción y aprobación de facturas

**Entrega Final — Coderhouse AI Automation**
Pablo Redolfi · Septiembre 2026

Caso de uso: **Glowing Beauty S.R.L.**, distribuidora de cosmética. Automatización del circuito de recepción, validación y aprobación de facturas de proveedores nacionales en el Departamento de Compras.

---

## Qué hace el sistema

Una factura llega por correo. El sistema la valida contra las reglas del instructivo, extrae sus datos con un modelo de lenguaje, verifica que los importes cierren, calcula la fecha de pago según el plazo pactado con ese proveedor, y la aprueba, la rechaza o la deriva a una persona. En todos los casos el proveedor recibe una respuesta escrita y queda registro en la base.

El principio que ordena el diseño: **la IA lee, el código calcula, la persona decide lo dudoso.** El modelo tiene una sola tarea —devolver los números que ve en un JSON— y no interviene en ninguna decisión de aprobación.

---

## Stack

| Categoría | Tecnología |
|---|---|
| Orquestador | n8n Cloud |
| Base de datos | Airtable |
| Procesamiento IA | OpenRouter (pasarela) → `inclusionai/ling-3.0-flash-fin:free` |
| Canal de entrada y salida | Gmail |
| Validación humana | Telegram (Send and Wait for Response) |

---

## Entregables

### Documentación (`/docs`)

| Criterio | Documento |
|---|---|
| 1 — Mapa de arquitectura | [`mapa-arquitectura.pdf`](docs/mapa-arquitectura.pdf) |
| 2 — Estructuras de datos | [`manual-operativo-datos.pdf`](docs/manual-operativo-datos.pdf) |
| 3 — Optimización de costos | [`matriz-decision-modelos.pdf`](docs/matriz-decision-modelos.pdf) |
| 4 — Seguridad y resiliencia | [`seguridad-resiliencia.pdf`](docs/seguridad-resiliencia.pdf) |
| — Instructivo para proveedores | [`instructivo-facturacion.pdf`](docs/instructivo-facturacion.pdf) |

El instructivo no puntúa por sí mismo, pero es el documento que define las reglas de admisión y los códigos de rechazo que el flujo implementa. El resto de la documentación lo referencia.

### Criterio 5 — Dashboard de control

Vistas compartidas públicas de Airtable, accesibles sin cuenta:

- **Facturas procesadas** — https://airtable.com/app0STyKWmZxgxNZc/shrdNZJQ0YYbFTjR0/tblFXGI7nMiKeLoik
- **Registro de errores** — https://airtable.com/app0STyKWmZxgxNZc/shrQQrG0PNIqXeFIo/tblIY2xXR2yCVLMhX

La vista de facturas está agrupada por estado y por código de rechazo, con totales por grupo: cantidad de facturas por estado, monto aprobado, distribución de rechazos y confianza promedio de lectura.

### Flujo

[`workflow/entrega-final-ai-automation.json`](workflow/entrega-final-ai-automation.json) — exportación del workflow de n8n. No contiene credenciales: las referencias apuntan al gestor de n8n, no a los valores.

### Video demo

[`video/demo.mp4`](video/demo.mp4) — 2:09. Muestra el trigger, el procesamiento en el orquestador y el resultado final.

> **Nota.** La aprobación por Telegram se responde desde el celular, fuera de cámara. El formulario de validación humana se abre con una firma de autorización visible en la URL, y mostrarla en la grabación equivaldría a exponer una credencial.

### Capturas (`/capturas`)

| Archivo | Contenido |
|---|---|
| `flujo-canvas.png` | El workflow completo en el editor de n8n |
| `ejecuciones.png` | Lista de ejecuciones del test de estrés |
| `airtable-facturas.png` | Los cinco registros del test, agrupados por estado y código |
| `airtable-log-errores.png` | Registro de errores capturados por las rutas de error |

---

## Test de estrés

Cinco presentaciones, una correcta y cuatro del camino infeliz:

| Presentación | Caso | Resultado | Tiempo |
|---|---|---|---|
| `FACTURA - 30712345678 - A0000300000412` | Factura correcta | Aprobada automática | 3m 06s |
| `FACTURA - 30755544433 - A0000700001185` | Neto + IVA ≠ total | Rechazada **RC-09** | 6,7s |
| `FACTURA - 30719904557 - A0000100000077` | CUIT fuera del legajo | Rechazada **RC-01** | 4,9s |
| `FACTURA - 30766677788 - A0000200000903` | Monto sobre el tope | Derivada a supervisión humana → aprobada | 1m 22s |
| `FACTURA - 30712345678 - A0000300000999` | Correo sin adjunto | Rechazada **RC-03** | 3,3s |

Los tiempos son la evidencia de la validación en cascada: lo que se rechaza antes del modelo se resuelve en segundos; lo que pasa por la extracción con IA tarda minutos. Siete de los nueve controles ocurren antes de consumir un token.

**Sobre la lista de ejecuciones.** Incluye corridas fallidas, conservadas deliberadamente. Corresponden a caídas del proveedor de IA durante el desarrollo, y son la evidencia de que las rutas de error funcionan: el flujo no se interrumpió, registró el fallo en `Log_Errores` y continuó. El documento de seguridad y resiliencia las analiza una por una.

---

## Modelo de datos

Base **Glowing Beauty - Compras**, tres tablas vinculadas:

- **`Proveedores`** — legajo comercial. Define el plazo de pago y el tope de aprobación automática de cada proveedor. El flujo solo la lee.
- **`Facturas`** — una fila por presentación recibida, incluidas las rechazadas. Campos de estado (`Aprobada`, `Rechazada`, `Esperando_aprobacion`, `En_revision`) y de código de rechazo.
- **`Log_Errores`** — registro de fallos capturados por las rutas de error.

El esquema completo, con tipos y origen de cada campo, está en el manual operativo de datos.

---

## Decisiones de diseño

| Decisión | Alternativa descartada | Motivo |
|---|---|---|
| La IA extrae, el código calcula | Pedirle al modelo que verifique los importes | Un LLM se equivoca multiplicando y su razonamiento no se puede auditar |
| Validación en cascada antes de la IA | Extraer primero y validar después | La mayoría de los rechazos no consume un token |
| Solo PDF con texto seleccionable | Modelo con visión para leer escaneos | Más caro, y el rechazo pasaría a depender de la interpretación del modelo |
| Pasarela en vez de proveedor directo | Integración contra la API de un proveedor | El modelo es un parámetro: se sustituyó siete veces sin tocar el flujo |
| Rutas de error por nodo | Workflow de error global | Permite reaccionar según qué falló y deja el registro en la misma base |
| Avisos con plantillas | Redacción por IA | Texto con valor operativo: debe ser idéntico ante el mismo código |

---

## Hallazgo técnico

El dato más relevante de la implementación, desarrollado en el documento de optimización de costos:

**Los tokens de razonamiento interno de un modelo cuentan contra el límite de salida pero no aparecen en la respuesta.** Con el tope fijado en 600 tokens —generoso frente a los ~250 que ocupa el JSON— el nodo devolvió cadena vacía, sin error y con la ejecución marcada como exitosa. La metadata lo delató: `finish_reason: "length"` con el contador exactamente en el tope.

La consecuencia práctica corrige una intuición: **el límite de tokens no puede dimensionarse por el tamaño de la respuesta esperada**, y un tope demasiado ajustado no ahorra, rompe — deriva la factura a revisión humana, que cuesta más que los tokens que se intentaba economizar.

---

*Los datos corresponden a una empresa ficticia utilizada con fines académicos.*
