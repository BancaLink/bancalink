# Bitácora de eventos y proyecciones — diseño técnico

**Fecha:** 27 de agosto de 2026
**Sub-proyecto:** 2 de 9
**Estado:** diseño aprobado, sin implementar
**Depende de:** SP1 (tipos `TransaccionExtraida`, `Bloque`)
**Decisiones relacionadas:** D4, D5, D8, D10, D12, D17, D19, D24, D25, D26

---

## 1. Contexto y alcance

§4 de la arquitectura ya define **qué** se guarda: eventos, tipos de cuenta, deduplicación, etiquetas, importación. Este documento define **cómo funciona en ejecución**.

Cubre los cuatro huecos que §4 dejaba abiertos: el sobre que envuelve cada evento, cómo se ordenan eventos de dispositivos distintos, cómo se calculan las proyecciones, y el mecanismo de migración de esquema — que D5 justifica como lo más caro de equivocar pero nunca describe.

**Dentro de alcance:** almacenamiento en IndexedDB, sobre del evento, reductor conmutativo, ventana activa, proyecciones de saldo y discrepancia (D17), agrupación por ciclo (D24), fusión entre dispositivos, migración de esquema.

**Fuera de alcance:** reportes elaborados (SP9), cifrado y respaldo (SP4, SP6), interfaz (SP3).

## 2. Almacenamiento

| Store | Contenido | Índices |
|---|---|---|
| `eventos` | La bitácora. Único origen de verdad | `fecha`, `tipo` |
| `correos` | `CorreoRetenido` (D26): bloques normalizados con expiración | `estado`, `expiraEn` |
| `meta` | Versión de esquema, id de dispositivo, marca de sincronización | — |
| `secretos` | Fuera de la bitácora por D19 — lo define SP4 | — |

El índice por `fecha` sostiene la ventana activa: cargar tres meses no puede implicar leer todo y filtrar.

### El sobre

```ts
type Sobre<E> = {
  id: string;           // huella determinística (D10) o UUID
  tipo: string;         // "TransaccionRegistrada"
  version: number;      // versión del esquema DE ESE TIPO, no global
  ts: string;           // ISO, reloj del dispositivo que lo creó
  dispositivo: string;  // id local aleatorio, para desempate determinista
  datos: E;
};
```

`version` es por tipo de evento y no global: si fuera global, cambiar `CuentaCreada` obligaría a versionar transacciones que no cambiaron.

`dispositivo` es aleatorio y local, no ligado a identidad (D15), y nunca sale hacia el relay. Sí queda en el respaldo, o sea que el blob revela cuántos dispositivos usa la persona — dato propio, en su nube, cifrado.

### Append-only en el tipo, no por disciplina

El store es de escritura única. La API no expone `update` ni `delete`; la restricción se aplica en el tipo.

### Borrar de verdad vs. borrar una transacción

§9 promete borrado real (derechos ARCO, Ley 8968) y la bitácora nunca borra nada. Se resuelven por separado porque son cosas distintas:

- **Borrar todo** es real: se destruye la base local y el blob de la nube. Nada sobrevive.
- **Borrar una transacción** es una lápida: `TransaccionEliminada` la saca de toda proyección; el evento original permanece. `TransaccionRestaurada` la revive.

**La fusión obliga a que sea así.** Si un dispositivo borrara el evento de verdad y otro todavía lo tuviera, la siguiente sincronización lo resucitaría: la unión de conjuntos no distingue "esto lo borré" de "esto no lo habías visto". Sin coordinación entre dispositivos —que en local-first no existe— la lápida es la única opción correcta.

**Consecuencia para la interfaz:** si alguien borra un movimiento porque le incomoda, la app no puede insinuar que desapareció del dispositivo. Desaparece de la vista. Quien quiera que no quede rastro debe borrar todo, y eso tiene que decirse donde se toma la decisión.

## 3. El reductor es conmutativo

**El requisito menos obvio y el más importante de este sub-proyecto.**

Un reductor normal asume orden cronológico: se pliegan los eventos en secuencia y el último gana. Acá no se puede, porque **una sincronización trae eventos viejos después de haber aplicado los nuevos** — el otro dispositivo estuvo días sin conectarse. Un reductor ingenuo dejaría que ese evento viejo pise la edición más reciente.

Cada campo mutable de la proyección guarda con qué evento se escribió:

```ts
type Campo<T> = { valor: T; ts: string; dispositivo: string };
```

Al plegar, se sobrescribe **solo si el evento entrante es más nuevo** por `(ts, dispositivo)`. Un evento viejo que llega tarde se descarta solo, sin reordenar nada.

Con eso el orden deja de importar: plegar los mismos eventos en cualquier secuencia produce el mismo estado. **Es lo que hace viable la actualización incremental** — sin esta propiedad, cada sincronización obligaría a un replay completo.

**Ediciones por campo, no por evento completo.** Si un dispositivo cambia la categoría y otro el comercio, sobreviven ambos cambios; con "último evento gana" se perdería uno en silencio. `TransaccionEditada` ya guarda solo los campos modificados.

**Borrado y restauración** son el mismo mecanismo: gana el más reciente entre `Eliminada` y `Restaurada`. Converge sin coordinación porque ambos dispositivos ven los mismos eventos y eligen el mismo último.

## 4. Proyecciones y ventana activa

**Un replay al abrir, estado en memoria, actualización incremental.** La bitácora es la verdad; la memoria es derivada. Si el estado en memoria se corrompe, se descarta y se vuelve a derivar.

Se descartaron las alternativas: replay completo en cada consulta paga la lectura en cada pantalla; proyecciones materializadas en IndexedDB introducen invalidación de caché en almacenamiento persistente, que es donde viven los bugs que solo aparecen en el dispositivo de otra persona. Queda disponible si alguna vez se **mide** que el arranque en frío molesta.

### La ventana y el punto de control de D17

Proyectar solo tres meses rompería el saldo: para saber cuánto hay hoy harían falta todos los movimientos desde siempre. Lo resuelve `SaldoReportado`:

```
saldo = último SaldoReportado + movimientos posteriores
ventana efectiva = max(ventana elegida, desde el último SaldoReportado)
```

La ventana no puede ser más corta que la distancia al último saldo confirmado, o el saldo sale mal.

**Efecto secundario deseable:** confirmar el saldo seguido es lo que mantiene la app rápida. El mecanismo que D17 creó para corregir errores acumulados resulta ser también el que acota el trabajo — los incentivos apuntan al mismo lado.

**Sin punto de control** no se inventa nada: se muestra el acumulado desde que se empezó a usar la app, dicho con esas palabras, y se invita a confirmar el real. Es la postura de D17: la app admite que está equivocada en vez de disimularlo.

Lo anterior a la ventana queda **disponible bajo demanda**: un reporte de dos años dispara un replay más ancho, se paga esa espera una vez y no en cada pantalla.

### El ciclo configurable (D24)

```ts
cicloDe(fecha, config) → { inicio, fin }
```

Función pura. Dos reglas que si no se fijan ahora producen bugs que solo aparecen cuando alguien tiene dos dispositivos:

**Recorte de días inexistentes.** Un ciclo quincenal `[15, 30]` en febrero, o mensual con `diaInicio: 31`. La regla es explícita —**se usa el último día del mes**— porque si cada dispositivo improvisa, los totales difieren y nadie entiende por qué.

**La zona horaria se guarda en la configuración, no se toma del dispositivo.** Con el teléfono en Costa Rica y la tablet de viaje en España, "25 de agosto" no empieza al mismo tiempo, y una transacción del borde cae en ciclos distintos. Costa Rica no tiene horario de verano, lo que ayuda, pero un dispositivo que viaja rompe la suposición.

### Lo que produce SP2

| Proyección | Qué da |
|---|---|
| Transacciones | Lista con borrados y ediciones resueltos |
| Saldo por cuenta | Desde el punto de control, en moneda original (D11) |
| Discrepancia (D17) | Proyectado contra confirmado, con la diferencia |
| Agrupación por ciclo | Totales del período que la persona definió (D24) |

## 5. Fusión entre dispositivos

Unión de conjuntos por `id`, plegando los eventos nuevos en el estado en memoria. Como el reductor es conmutativo, no hace falta reordenar ni recalcular.

### Corrección a §4: la huella colisiona con correos de varias transacciones

§4 define la huella de una transacción de correo como *"hash del `Message-ID`"*. **Eso colisiona.** El correo del BCR es una tabla y produce N transacciones — todas del mismo `Message-ID`, todas con la misma huella. La segunda en adelante se descartarían como duplicados y **se perderían movimientos reales**.

La huella tiene que incluir la posición dentro del correo:

```
id = hash(Message-ID + índice de fila)
```

Se elige el índice y no el contenido de la fila a propósito. Un correo es inmutable una vez enviado, así que el orden de sus filas nunca cambia: el índice es estable para siempre. Si la huella dependiera del contenido extraído, **una corrección de parser cambiaría la huella y la transacción entraría de nuevo como duplicada** — exactamente el problema que la deduplicación existe para evitar.

Corresponde reflejarlo en §4 de la arquitectura.

### El caso que §5 de la arquitectura no cubría

El `id` de una transacción de correo es el hash del `Message-ID`, **no de su contenido**. Entonces:

> El teléfono tiene la versión 1 del parser de BAC. La tablet ya bajó la 2, que corrige un campo. El mismo correo produce **el mismo `id` con datos distintos.**

La unión ve un duplicado y descarta uno, pero acá sí hay conflicto real de contenido.

**Resolución: gana la `parserVersion` más alta.** `fuente` ya guarda `parserId` y `parserVersion`, así que el dato está en el modelo. Es determinista —ambos dispositivos eligen igual— y correcto: una versión más nueva existe porque arregla algo.

Sin esta regla, dos dispositivos con versiones distintas podrían quedarse cada uno con su dato y no converger nunca.

## 6. Migración de esquema

D5 justifica modelar todo desde el día uno porque *"cada dispositivo migra su propio esquema, algunos usuarios nunca actualizan"*. El mecanismo no estaba escrito.

**Los eventos son inmutables y nunca se reescriben.** La adaptación ocurre al leer: cada tipo tiene su `version` y el lector aplica una cadena de conversores `v1 → v2 → v3`.

Reescribir sería peor de lo que parece: contradice D10, y si un dispositivo migrara y otro no, la fusión vería los mismos `id` con contenidos distintos.

### El problema difícil es el inverso

Adaptar al leer resuelve que lo nuevo entienda lo viejo. Pero **un dispositivo que nunca se actualiza va a leer un respaldo escrito por uno que sí**, y se encontrará con tipos o versiones que no conoce.

**Regla: guardar sin entender.**

- El log **acepta y conserva** cualquier evento, aunque el proyector no sepa interpretarlo.
- La proyección lo **omite** y avisa: *"hay movimientos que esta versión de la app no entiende"*.
- Como se conservó, actualizar lo recupera. **Nada se pierde por no actualizar.**

Esto impone algo al almacenamiento: **la capa que guarda no puede validar contra el esquema que conoce.** Si lo hiciera, un dispositivo viejo descartaría datos ajenos en cada sincronización y los borraría del respaldo compartido.

**Contrasta a propósito con el manejo de parsers (D25)**, donde un `formatoMinimo` no soportado hace que el parser se **descarte**. La diferencia es qué se pierde: un evento es dato del usuario e irrecuperable; un parser se vuelve a bajar del índice.

## 7. Retención de correos (D26)

El store `correos` guarda la salida del normalizador de SP1, no el correo. Su ciclo de vida pertenece a SP2 porque comparte almacenamiento y expiración:

| Estado | Retención |
|---|---|
| `sin-parser` | Hasta resolverse — es trabajo pendiente |
| `interpretado` | ~30 días, configurable hasta 0 |
| `rechazado` en la confirmación de D25 | Hasta resolverse |

No entra al respaldo (D8). La expiración se evalúa al abrir la app; no hace falta un temporizador.

## 8. Pruebas

| Capa | Qué prueba |
|---|---|
| **Conmutatividad** | Propiedad: plegar los mismos eventos en **orden aleatorio** siempre da el mismo estado |
| Reductor | Borrar → restaurar; edición por campo desde dos dispositivos |
| Ciclo | Recorte en febrero, `diaInicio: 31`, zona horaria fija vs. dispositivo |
| Fusión | Dos logs convergen; mismo `id` con `parserVersion` distinta |
| Migración | Lector nuevo sobre evento viejo; **lector viejo sobre evento nuevo** |
| Ventana | Saldo con y sin punto de control; ventana menor que la distancia al control |
| Retención | Expiración por estado; que `sin-parser` **no** expire |

**La primera fila es la más valiosa.** La conmutatividad es un requisito arquitectónico, no un detalle: si se rompe, los síntomas aparecen solo cuando alguien tiene dos dispositivos y uno estuvo desconectado varios días — el peor escenario posible para diagnosticar. Una prueba de propiedad con órdenes aleatorios la ejercita en cada corrida.

**En migración, la fila que importa es la segunda mitad:** lector viejo sobre evento nuevo. Protege a quien nunca actualiza, y la única forma de saber que funciona es probarla a propósito.

## 9. Qué desbloquea

| Consume de SP2 | Qué |
|---|---|
| SP3 (UI) | Las proyecciones |
| SP6 (sincronización) | La fusión y el formato serializado del log |
| SP9 (reportes) | El replay bajo demanda para ventanas anchas |

**Pendiente con dueño:** el cifrado del log para el respaldo es de SP4/SP6; SP2 produce el log en claro y define su forma serializada.
