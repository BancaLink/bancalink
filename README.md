# BancaLink

[![Licencia: AGPLv3](https://img.shields.io/badge/Licencia-AGPLv3-blue.svg)](LICENSE)

**Finanzas personales libres, empezando por Costa Rica. Tus correos bancarios se cifran al llegar y se interpretan solo en tu dispositivo: el servidor nunca los guarda ni los entiende.**

BancaLink lee las notificaciones que tu banco ya te envía por correo y las convierte automáticamente en un registro de gastos e ingresos. Es **libre, gratuita, de conocimiento cero y auto-hospedable**: no te pedimos acceso a tu buzón, no te creamos una cuenta, y tus transacciones descifradas nunca salen de tus dispositivos.

> **Estado:** el diseño técnico está cerrado y documentado. Todavía **no hay código de aplicación** en este repositorio — lo siguiente es el plan de implementación.

---

## Qué buscamos

El proyecto persigue dos objetivos en plazos muy distintos.

### Objetivo 1 — Una herramienta que nadie tenga que pagar

En Costa Rica no existe un estándar de banca abierta (*Open Banking*) que te permita conectar tus cuentas con aplicaciones de finanzas de forma oficial y segura. El BCCR y la SUGEF apenas lo están explorando. Ante ese vacío, las apps que existen recurren a un truco: leen los correos de notificación que te manda el banco, convirtiendo el email en una tubería improvisada que nadie diseñó para eso.

BancaLink hace exactamente lo mismo, con dos diferencias:

1. **El código es libre y gratuito, para siempre.** Podés usarlo, auditarlo, modificarlo y hospedarlo vos mismo.
2. **No tenés que creernos.** El servidor cifra cada correo apenas lo recibe y jamás lo interpreta; tu historial financiero nunca sale de tus dispositivos sin cifrar. Y como el código es auditable y auto-hospedable, podés verificarlo vos mismo — o correr tu propio relay y sacarnos de la ecuación por completo.

Manejar tu plata es parte de tu salud integral. Tener buenas herramientas para hacerlo no debería ser un lujo reservado para quien puede pagar una suscripción.

### Objetivo 2 — Que este parche deje de ser necesario

Leer correos es un remiendo, y lo decimos abiertamente. La solución real es que Costa Rica tenga un estándar que permita a cualquier aplicación —esta u otra— conectarse a tu banco con tu consentimiento.

Por eso el repositorio público de *parsers* es parte de la estrategia: cada vez que un banco cambia el formato de sus correos y rompe la lectura, queda un registro fechado y verificable de que la ausencia de un estándar le impone costos reales a los costarricenses. Esa evidencia vale más ante el BCCR que cualquier documento de posición.

Este repositorio cubre el objetivo 1. El objetivo 2 se trabaja en un frente separado.

---

## Cómo funciona

1. Configurás en tu correo (Gmail, iCloud, Yahoo, corporativo — el que sea) una **regla de reenvío automático** hacia una dirección dedicada que te da BancaLink. Nunca te pedimos acceso a tu buzón, y podés cortarlo cuando querás borrando la regla.
2. Un servidor *relay* mínimo recibe ese correo, lo **cifra de inmediato con tu llave pública** y borra el texto original. El relay nunca lo interpreta ni lo guarda en claro: no sabe de qué banco es, ni de cuánto fue el movimiento. Ese instante de recepción es el único punto donde el texto existe sin cifrar — y es justamente por eso que el relay es mínimo, auditable y auto-hospedable.
3. La app —una PWA instalable— descarga el paquete cifrado, **lo descifra y lo interpreta en tu dispositivo**, y guarda la transacción localmente.
4. Tus datos se respaldan y sincronizan a través de **tu propia nube** (Google Drive, OneDrive, Dropbox) o un archivo local, siempre cifrados con tu llave.

No hay registro: no te pedimos correo, ni contraseña, ni nombre, ni cédula. Tu dirección dedicada es tu identidad.

---

## ¿Sirve fuera de Costa Rica?

Técnicamente sí, y por diseño más que por casualidad. Nada del núcleo —el relay, el cifrado, la bitácora de eventos, el motor de parsers, la PWA— sabe en qué país está. El modelo de datos guarda la **moneda original de cada transacción** desde el día uno ([D11](docs/BancaLink_Decisiones.md)), así que registrar dólares, euros o quetzales no requiere ningún cambio de esquema. Un banco nuevo, en cualquier país, es un archivo YAML nuevo.

Lo que sí está atado a Costa Rica hoy:

- **La fuente del tipo de cambio** viene configurada con el indicador oficial del BCCR, pero es intercambiable: un proveedor se declara como configuración, igual que un parser ([D18](docs/BancaLink_Decisiones.md)). Agregar el de otro país no requiere tocar la app.
- **La biblioteca de parsers** arranca con bancos costarricenses, simplemente porque es donde estamos.
- **El objetivo 2** —empujar un estándar de banca abierta— se dirige al BCCR y la SUGEF. Otro país necesitaría su propio frente.
- **El marco legal** que seguimos es la Ley 8968.

El enfoque en Costa Rica es una decisión de estrategia, no un límite de la arquitectura. Si querés levantar los parsers de tu país, la puerta está abierta.

---

## Principios que no negociamos

- **Conocimiento cero.** El servidor recibe, pero no entiende.
- **Local-first.** Tus datos viven en tu dispositivo. La nube es respaldo cifrado bajo tu control.
- **Gratis y auto-hospedable.** Licencia [AGPLv3](https://www.gnu.org/licenses/agpl-3.0.html): cualquiera puede usar y modificar la app sin pagar, y quien la ofrezca como servicio debe liberar sus cambios.
- **Cero telemetría.** No medimos clics, ni tiempo de uso, ni nada. Nada.
- **Parsers legibles por humanos.** Las reglas de lectura de cada banco son archivos YAML simples, mantenidos por la comunidad.
- **Apego a la Ley 8968.** Consentimiento claro, borrado real, y eliminación inmediata de los correos del servidor una vez procesados.

El razonamiento detrás de cada decisión —y de las alternativas que descartamos— está en [`docs/BancaLink_Decisiones.md`](docs/BancaLink_Decisiones.md).

---

## Stack tecnológico

Monorepo, todo en **TypeScript**.

| Paquete | Qué es | Tecnología |
|---|---|---|
| `apps/client` | La app: toda la lógica, el cifrado y los datos | React · Vite · PWA instalable |
| `apps/relay` | El servidor ciego que recibe el correo reenviado | Node.js · `smtp-server` · Docker |
| `packages/parsers` | Motor que interpreta las reglas YAML de cada banco | TypeScript, sin `eval` |
| `packages/shared` | Tipos, utilidades de cifrado, formato de parser | TypeScript |

**Decisiones técnicas de fondo:**

- **Almacenamiento:** bitácora de eventos *append-only* en IndexedDB, en vez de una base de datos que se sobrescribe. Esto hace que sincronizar entre tus dispositivos no pueda generar conflictos, y que el mismo cobro nunca se registre dos veces.
- **Cifrado:** WebCrypto estándar (AES-256-GCM), sin criptografía casera.
- **Sin base de datos en servidor.** El relay solo guarda paquetes cifrados que no puede abrir, y los borra apenas los entrega.
- **PWA, no App Store.** Instalable desde el navegador, sin intermediarios que aprueben o cobren.

El diseño técnico completo está en [`docs/superpowers/specs/2026-08-26-arquitectura-tecnica-design.md`](docs/superpowers/specs/2026-08-26-arquitectura-tecnica-design.md).

---

## Sobre el uso de Inteligencia Artificial

Un proyecto cuyo argumento central es *"no confiés en nosotros, revisá el código"* está obligado a ser transparente sobre quién escribe ese código y sobre qué toca la IA. Hay **dos usos de IA completamente distintos** en este proyecto, y es importante no confundirlos.

### 1. IA para construir la app

**El código de BancaLink lo escribe principalmente un modelo de IA** (Claude), dirigido por una persona mediante *spec-driven development*: primero se discute y se documenta el diseño, y solo después se escribe el código que lo implementa.

Lo decimos abiertamente porque es información relevante para cualquiera que evalúe confiar en esta herramienta:

- **Cada decisión de diseño queda escrita antes de existir el código.** Por eso este repositorio tiene documentos de decisiones y specs desde antes de tener una sola línea de aplicación. Podés leer *por qué* está hecho como está, no solo *qué* hace.
- **Una persona dirige y revisa el trabajo**, y todo queda en el historial público de Git: cada cambio es rastreable hasta la decisión que lo originó.
- **La criptografía se apoya en primitivas estándar y auditadas** (WebCrypto del navegador), nunca en algoritmos propios. Es la parte donde un error sería más grave, y donde menos margen de improvisación existe — para una IA o para cualquiera.
- **La licencia AGPLv3 te da el derecho de verificar todo esto por tu cuenta.** No pedimos que nos creas.

**Esta IA nunca ve datos de nadie.** Escribe código que después corre en tu dispositivo. No tiene, ni tendrá, acceso a tus correos ni a tus finanzas.

### 2. IA dentro de la app (opcional, y nunca obligatoria)

La app **no** manda tus gastos a ningún modelo de IA. Sería caro y pésimo para la privacidad.

Hay un solo caso donde la IA podría entrar: cuando tu banco cambia el formato de sus correos y la app deja de entenderlos. Pero **la forma normal de arreglar eso no usa IA** ([D21](docs/BancaLink_Decisiones.md)): abrís el correo en la app y marcás vos mismo dónde está el monto, el comercio y la fecha. La app deduce la regla. Todo local, sin llaves, sin descargas, sin que el correo salga de tu dispositivo.

La IA solo sirve para **adelantarte ese trabajo**: si la activás —con un modelo local, o con tu propia clave de API— te propone el mapeo ya hecho para que lo confirmes. Si no la tenés, no hay red, o simplemente no querés usarla, el camino manual funciona completo. **La app nunca se queda esperando una IA.**

Y cuando alguien arregla el parser de su banco, ese arreglo se propone a la biblioteca compartida y le llega a todos los demás ([D20](docs/BancaLink_Decisiones.md)). Antes de salir de tu dispositivo, la app comprueba automáticamente que el parser no se haya llevado datos tuyos adentro: lo prueba contra una copia de tu correo con todos los valores cambiados, y si no sobrevive esa prueba, no se propone.

Es decir: alguien usa IA un puñado de veces **por cada cambio de formato**, no cada persona cada vez. Y podés no usarla nunca.

---

## Alcance de la primera versión

Construimos poco, pero modelamos todo.

**La v1 incluye:** ingesta de correos bancarios · entrada manual (efectivo, SINPE) · categorías y etiquetas · saldos y detección de diferencias · importación de estados de cuenta (CSV y Excel) · reportes · exportación de tus datos · respaldo y sincronización cifrados.

**Modelado desde el día uno, sin construir todavía:** tarjetas de crédito · créditos de comercio (Gollo, Monge y similares) · préstamos · depósitos a plazo · presupuestos · metas de ahorro · gastos compartidos.

La razón de definir el modelo de datos completo desde el inicio es que en una app *local-first* cada dispositivo migra su propio esquema — equivocarse ahí sale caro de forma permanente.

---

## Licencia

[AGPLv3](https://www.gnu.org/licenses/agpl-3.0.html). La elegimos específicamente para que nadie pueda tomar este código, cerrarlo y cobrar por un servicio propietario sin devolverle nada a la comunidad.

## Contribuir

Todavía estamos en la etapa previa a la implementación — no hay código de aplicación que aportar, pero sí hay diseño para discutir.

**Por dónde empezar:**

| | Qué es |
|---|---|
| [**Diagramas de arquitectura**](docs/diagramas/) | La vista de conjunto, en cinco diagramas. El camino más rápido para entender cómo encaja todo |
| [Registro de decisiones](docs/BancaLink_Decisiones.md) | Qué decidimos y por qué. Es la fuente de verdad |
| [Diseños técnicos](docs/superpowers/specs/) | El detalle de cómo se construye |

Después contanos qué pensás en las [Discussions](../../discussions): dudas, objeciones al diseño, interés en un frente específico (backend, cifrado, parsers bancarios, legal, PWA/mobile) — todo sirve en esta etapa. Las objeciones sirven especialmente: varias decisiones cambiaron al ponerlas a prueba contra correos bancarios reales.

Cuando arranque la implementación, las guías de contribución van a empezar por el formato de los parsers bancarios — pensado para que alguien sin experiencia programando pueda arreglar el correo de su banco.

Los aportes de código se firman con [DCO](https://developercertificate.org/) (un *sign-off* en el commit), no con un CLA. Es deliberado: un CLA nos permitiría cambiar la licencia en el futuro, y **no queremos tener ese poder**.

## Cumplimiento legal

Nos tomamos en serio la Ley de Protección de la Persona frente al Tratamiento de sus Datos Personales (Ley 8968). Nuestro enfoque, y las preguntas legales que quedan por resolver antes de un lanzamiento, están en el [registro de decisiones](docs/BancaLink_Decisiones.md).
