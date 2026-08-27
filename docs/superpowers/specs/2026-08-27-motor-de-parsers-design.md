# Motor de parsers — diseño técnico

**Fecha:** 27 de agosto de 2026
**Sub-proyecto:** 1 de 9 (ver descomposición de la v1)
**Estado:** diseño aprobado, sin implementar
**Decisiones relacionadas:** D4, D7, D10, D17, D21, D23

---

## 1. Contexto y alcance

Primer sub-proyecto de la v1. Se eligió como punto de partida porque **no depende de nada** —ni relay, ni cifrado, ni PWA, ni bitácora—, porque los datos de prueba ya existen (seis correos reales en `sample/`), y porque es donde queda más incertidumbre: el esquema del formato de parsers se movió con cada tanda de muestras.

Construirlo **es** cómo se cierra ese esquema con fundamento en vez de por decreto.

**Dentro de alcance:**

- Normalización de correo (MIME, `quoted-printable`, HTML → bloques).
- Formas de extracción `etiqueta_valor` y `prosa`.
- Coerción de tipos: monto, fecha, moneda, dirección.
- Prueba de invarianza (mecanismo que SP7 consume).
- Definiciones YAML para **BAC**, banco piloto: compra CRC, compra USD, SINPE.

**Fuera de alcance, deliberadamente:**

- La forma `tabla` — su interfaz se define, su implementación no. Ninguna muestra de BAC la usa.
- Parsers de BCR y Credix.
- Ejecución en Web Worker y validador de dialecto (D23) — pertenecen a quien llama, no al paquete.

**Consecuencia aceptada del alcance:** con solo BAC quedan sin validar la forma `tabla` (única en BCR) y el separador decimal con coma (único en Credix). La segunda se mitiga con pruebas de coerción; la primera queda visible como prueba pendiente en CI (§10).

## 2. Principios

**Predecible antes que inteligente.** Un clasificador que acierta el 95% de las veces de formas impredecibles es peor que uno simple cuyas reglas se pueden leer y anticipar. Quien escribe un parser —o el mapeador manual de SP7— necesita saber qué va a salir sin ejecutarlo.

**Ante ambigüedad, fallar.** Nunca adivinar un monto, una fecha ni una moneda. Cero transacciones es mejor que una equivocada: apoyado en D17, una transacción faltante se corrige sola al confirmar un saldo real, pero una inventada descuadra activamente y sin explicación visible.

**El camino de falla alimenta el camino de arreglo.** Cuando ningún parser calza, la salida son los bloques — exactamente la entrada que necesita el mapeador manual.

## 3. Estructura del paquete

El paquete es **puro**: sin I/O, sin Workers, sin red. Recibe bytes, devuelve transacciones. El aislamiento en Worker que exige D23 lo maneja quien llama, que es quien sabe si está en la PWA o en un test.

```
packages/parsers/
├── src/
│   ├── index.ts              API pública
│   ├── normalizar/
│   │   ├── mime.ts           quoted-printable, charset, selección de parte
│   │   ├── limpiar.ts        descarta head/style/script, entidades, espacios
│   │   └── clasificar.ts     DOM → Bloque[]      ← el riesgo está acá
│   ├── extraer/
│   │   ├── etiqueta-valor.ts
│   │   ├── prosa.ts
│   │   └── tabla.ts          interfaz definida, sin implementación
│   ├── tipos/
│   │   └── coercion.ts       monto, fecha, moneda, dirección
│   ├── invarianza/
│   │   ├── mutar.ts          genera la copia con valores sustituidos
│   │   └── verificar.ts      corre el parser sobre ambas y compara
│   ├── esquema.ts            validación del YAML del parser
│   └── definiciones/
│       ├── bac-compra.yaml
│       ├── bac-sinpe.yaml
│       └── bac-pago-servicios.yaml
└── test/
    └── fixtures/             correos sintéticos (§10)
```

`tabla.ts` existe desde el inicio con su interfaz vacía. Marca el contrato para que el clasificador no la olvide.

**Dependencias:** una sola no trivial, un parser de HTML que corra en navegador y en Node. `DOMParser` es nativo en el navegador; se propone `linkedom` para Node. MIME y `quoted-printable` se implementan a mano: son pocas líneas y las bibliotecas de correo arrastran mucho más de lo necesario.

## 4. API pública

```ts
export function interpretar(correo: Uint8Array, biblioteca: Parser[]): Resultado;

type Resultado =
  | { estado: "ok";         parser: string;
      transacciones: TransaccionExtraida[]; descartadas: number }
  | { estado: "sin-parser"; bloques: Bloque[] }
  | { estado: "rechazado";  parser: string; campo: string; motivo: string }
  | { estado: "invalido";   motivo: string };
```

**`transacciones` es un arreglo desde el día uno.** BCR emite varias por correo; descubrirlo después obligaría a tocar todo lo que consume esto.

**`descartadas` se cuenta aparte de los errores.** Una transacción con `Estado: Negada` filtrada por `descartar_si` no es una falla: es el parser funcionando. Confundirlas haría que un correo correcto se vea como roto.

**`sin-parser` devuelve los bloques.** El mapeador manual (SP7) no reimplementa normalización, y lo que la persona ve al mapear es literalmente lo que el motor interpretó.

## 5. Normalización

```
bytes → parte text/html → quoted-printable → charset → DOM
      → borrar head/style/script → clasificar
```

Ninguna de las seis muestras trae `text/plain`. Y la codificación **parte palabras a la mitad** — esto es real, del correo de BAC:

```
Tipo d=
e Transacci=C3=B3n:
```

Sobre los bytes crudos no funciona ni un regex ni una búsqueda literal. Decodificar no es una optimización, es un requisito.

**Espacios que no se ven.** Además de desescapar entidades HTML, `&nbsp;` (U+00A0) se normaliza a espacio común. Un espacio duro es idéntico a la vista y distinto para el motor: `despues_de: "Monto:"` falla sin que nadie entienda por qué. De los bugs más caros de diagnosticar y más baratos de prevenir.

## 6. Clasificación en bloques

### Los elementos en línea no cortan bloque

`<span>`, `<b>`, `<strong>`, `<em>`, `<a>` se aplanan dentro del bloque que los contiene. Solo cortan los de nivel de bloque: `p`, `div`, `td`, `tr`, `table`, `li`.

Evidencia — el proveedor del pago de servicios de BAC:

```html
<p>Se ha realizado un pago de servicio de <span style="font-weight:bold;">JASEC</span> desde <span>Banca Móvil</span>.</p>
```

Sin esta regla `JASEC` queda en un bloque suelto y `entre: ["pago de servicio de ", " desde"]` no encuentra nada. Una regla de una línea que decide si un parser entero funciona.

### Las tres reglas

| # | Condición | Produce |
|---|---|---|
| 1 | `<table>` con encabezados y filas de ancho consistente | `tabla` |
| 2 | Fila de exactamente 2 celdas, la primera terminada en `:` | `par` |
| 3 | Cualquier otra cosa | `texto` |

**La regla 1 se implementa aunque la forma `tabla` esté fuera de alcance**, y no es una contradicción: el *clasificador* reconoce tablas desde el día uno; lo que se difiere es la *forma de extracción* que las consume. La asimetría de costo lo justifica — agregar una forma de extracción después es barato, pero si la representación intermedia aplanara las tablas porque BAC no las usa, meter BCR obligaría a rehacer la normalización, que es la parte cara.

**`texto` es el refugio seguro.** Cuando la regla 2 no está segura, no adivina. Ahí no se pierde información —`prosa` siempre puede extraer con `entre` o `patron`— solo cuesta una regla más explícita en el YAML. Es el mismo criterio de lista blanca de D21: fallar hacia lo conservador antes que hacia lo listo.

### La forma del bloque

```ts
type Bloque = { id: number; origen: string } & (
  | { tipo: "par";   etiqueta: string; valor: string }
  | { tipo: "tabla"; encabezados: string[]; filas: string[][] }
  | { tipo: "texto"; contenido: string }
);
```

`id` lo consume el mapeador manual (*"tocaste el bloque 3"*). `origen` es la escotilla: la ruta al nodo original, para el caso que la clasificación no cubra.

### Determinismo

El mismo correo produce siempre los mismos bloques con los mismos `id`. No es una preferencia: la deduplicación de D10 deriva identificadores del contenido, y la prueba de invarianza compara dos corridas. Prohibido recorrer estructuras sin orden garantizado.

### Tolerancia a basura

El correo de Credix trae `*|MC:SUBJECT|*`, una etiqueta de Mailchimp sin renderizar. Cae en `texto` y no molesta. **Un defecto en una parte del correo no invalida el resto.**

## 7. Extracción y coerción

| Forma | Opera sobre | Notas |
|---|---|---|
| `etiqueta_valor` | bloques `par` | Acepta alternativas de etiqueta: `["MASTER:", "VISA:"]` |
| `prosa` | bloques `texto` | Con `entre` o con `patron` (escotilla de regex) |
| `tabla` | bloques `tabla` | Interfaz definida, implementación diferida |

La coerción concentra las trampas encontradas en las muestras:

```yaml
formato_numero: "1,234.56"     # miles = coma, decimal = punto — declarado, nunca inferido
moneda_al: final               # "13,333.00 CRC"  vs  "CRC 1,190.00" (mismo banco)
monedas: { CRC: CRC, Colones: CRC, USD: USD, Dólares: USD }
direccion: egreso              # un SINPE recibido es ingreso
```

Si `formato_numero` declara punto decimal y llega `10500,00`, **el campo se rechaza**. Podría interpretarse como diez mil quinientos, pero acertar a veces y errar por un factor de 1000 el resto es peor que no producir nada.

## 8. Prueba de invarianza

Mecanismo que SP7 consume, construido acá porque necesita saber qué es un valor — y eso lo declara el parser.

```
correo original ──> parser ──> transacciones A
       │
       └─> mutar valores ──> parser ──> transacciones B

  B trae los valores nuevos  → el parser es estructural  ✅
  B trae los viejos, o falla → memorizó datos            ❌
```

**La estructura de bloques hace esto tratable**, y es el beneficio río abajo que justificó el enfoque C: el bloque ya distingue lo que es rótulo de lo que es dato.

| Bloque | Se muta | Se conserva |
|---|---|---|
| `par` | `valor` | `etiqueta` |
| `tabla` | celdas | encabezados |
| `texto` | secuencias de dígitos y valores extraídos | el resto |

**Limitación honesta:** la garantía es fuerte en `par` y `tabla`, donde rótulo y dato están separados por construcción. En `prosa` es más débil, porque texto fijo y dato van entremezclados y un nombre propio embebido puede pasar sin mutarse. No la presentamos como absoluta.

La mutación usa un generador con semilla, para que las pruebas sean reproducibles.

## 9. Errores

| Situación | Resultado |
|---|---|
| Ningún parser calza | `sin-parser` + bloques → mapeador manual |
| Calza, falta campo obligatorio | `rechazado`, indicando parser y campo |
| Campo presente, coerción falla | `rechazado` |
| `descartar_si` se cumple | **No es error** — `descartadas` incrementa |
| MIME o HTML ilegible | `invalido` |

**Nunca se produce una transacción a medias.** Un movimiento sin monto es inútil; uno sin fecha queda mal archivado. En ambos casos se rechaza el correo entero y se reporta qué campo falló, que es lo accionable para arreglar el parser.

## 10. Pruebas

### El problema de los fixtures

Los `.eml` reales están en `.gitignore`: contienen nombre completo, IBAN, últimos dígitos de tarjeta y transacciones reales. **CI no puede usarlos.**

Se construyen **correos sintéticos commiteados**: la estructura exacta de BAC con datos inventados, **incluidos los cortes de `quoted-printable`**, que es justamente lo que hay que ejercitar. Simetría útil: es el mismo artefacto que D22 exige para proponer un parser anónimo, así que el generador se escribe una vez y sirve para las dos cosas.

### Capas

| Capa | Qué prueba |
|---|---|
| MIME | Palabras partidas por salto suave, `=C3=B3`, charset, selección de parte |
| **Clasificador** | HTML → bloques esperados. **El grueso va acá**: es la parte riesgosa |
| Extracción | Bloques → campos, sin HTML de por medio |
| Coerción | Tabla de casos con todos los formatos observados |
| Invarianza | Un parser con un valor incrustado **debe** fallar la prueba |
| Extremo a extremo | Fixture → transacciones, para los 3 formatos de BAC |
| Determinismo | Dos corridas del mismo correo → bloques e `id` idénticos |

### Dos decisiones sobre las pruebas

**La coerción se prueba con formatos de bancos fuera de alcance.** Aunque no se despache parser de Credix, `10500,00` entra como caso. Cuesta una línea y blinda la trampa más cara encontrada.

**El hueco de `tabla` queda visible en CI.** Una prueba marcada como pendiente con un fixture del BCR. Así la limitación aparece en cada corrida en lugar de olvidarse — el destino habitual de lo que se deja "para después".

## 11. Qué desbloquea y qué queda pendiente

**Desbloquea:** SP2 (bitácora) consume `TransaccionExtraida`. SP7 (mapeador manual) consume `Bloque[]` y la invarianza — incluirla acá baja el riesgo del sub-proyecto más peligroso del plan.

**Queda pendiente, con dueño:**

| Pendiente | Dónde se resuelve |
|---|---|
| Forma `tabla` | Cuando entre BCR |
| Worker y presupuesto de tiempo (D23) | En la app cliente, SP3 |
| Validador de dialecto restringido (D23) | Relay, SP8 |
| Congelar el esquema del formato | **Al terminar este sub-proyecto** |
