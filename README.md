# BancaLink

[![Licencia: AGPLv3](https://img.shields.io/badge/Licencia-AGPLv3-blue.svg)](LICENSE)

**Tus finanzas, sin que nadie más las vea.**

BancaLink es una aplicación de finanzas personales **libre, gratuita y de conocimiento cero** para Costa Rica. Lee las notificaciones que tu banco ya te envía por correo y las convierte automáticamente en un registro de gastos e ingresos — sin que BancaLink, ni nadie más, pueda ver jamás el contenido de esos correos ni tus transacciones.

> **Estado:** el diseño técnico está cerrado y documentado. Todavía **no hay código de aplicación** en este repositorio — lo siguiente es el plan de implementación.

---

## Qué buscamos

El proyecto persigue dos objetivos en plazos muy distintos.

### Objetivo 1 — Una herramienta que nadie tenga que pagar

En Costa Rica no existe un estándar de banca abierta (*Open Banking*) que te permita conectar tus cuentas con aplicaciones de finanzas de forma oficial y segura. El BCCR y la SUGEF apenas lo están explorando. Ante ese vacío, las apps que existen recurren a un truco: leen los correos de notificación que te manda el banco, convirtiendo el email en una tubería improvisada que nadie diseñó para eso.

BancaLink hace exactamente lo mismo, con dos diferencias:

1. **El código es libre y gratuito, para siempre.** Podés usarlo, auditarlo, modificarlo y hospedarlo vos mismo.
2. **No podemos leer tus datos aunque quisiéramos.** No es una promesa de política de privacidad; es una consecuencia de cómo está construido.

Manejar tu plata es parte de tu salud integral. Tener buenas herramientas para hacerlo no debería ser un lujo reservado para quien puede pagar una suscripción.

### Objetivo 2 — Que este parche deje de ser necesario

Leer correos es un remiendo, y lo decimos abiertamente. La solución real es que Costa Rica tenga un estándar que permita a cualquier aplicación —esta u otra— conectarse a tu banco con tu consentimiento.

Por eso el repositorio público de *parsers* es parte de la estrategia: cada vez que un banco cambia el formato de sus correos y rompe la lectura, queda un registro fechado y verificable de que la ausencia de un estándar le impone costos reales a los costarricenses. Esa evidencia vale más ante el BCCR que cualquier documento de posición.

Este repositorio cubre el objetivo 1. El objetivo 2 se trabaja en un frente separado.

---

## Cómo funciona

1. Configurás en tu correo (Gmail, iCloud, Yahoo, corporativo — el que sea) una **regla de reenvío automático** hacia una dirección dedicada que te da BancaLink. Nunca te pedimos acceso a tu buzón, y podés cortarlo cuando querás borrando la regla.
2. Un servidor *relay* mínimo recibe ese correo, lo **cifra de inmediato con tu llave pública** y borra el texto original. El relay nunca puede leer nada.
3. La app —una PWA instalable— descarga el paquete cifrado, **lo descifra y lo interpreta en tu dispositivo**, y guarda la transacción localmente.
4. Tus datos se respaldan y sincronizan a través de **tu propia nube** (Google Drive, OneDrive, Dropbox) o un archivo local, siempre cifrados con tu llave.

No hay registro: no te pedimos correo, ni contraseña, ni nombre, ni cédula. Tu dirección dedicada es tu identidad.

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

### 2. IA dentro de la app (opcional, con tu propia llave)

La app **no** manda tus gastos a ningún modelo de IA. Sería caro y pésimo para la privacidad.

La IA se usa en un solo caso puntual: cuando un banco cambia el formato de sus correos y la app deja de entenderlos. Ahí, **solo si vos lo activás y ponés tu propia clave de API**, se envía *un correo* al modelo para que ayude a generar la regla de lectura. A partir de ese momento, todos los correos con ese formato se procesan localmente, gratis y sin conexión.

Es decir: la IA se usa un puñado de veces en la vida de una instalación, no miles. Y podés no usarla nunca — la biblioteca de parsers que mantiene la comunidad cubre los formatos ya conocidos.

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

Todavía estamos en la etapa previa a la implementación; pronto habrá guías de contribución, empezando por el formato de los parsers bancarios — pensado para que alguien sin experiencia programando pueda arreglar el correo de su banco.

Los aportes de código se firman con [DCO](https://developercertificate.org/) (un *sign-off* en el commit), no con un CLA. Es deliberado: un CLA nos permitiría cambiar la licencia en el futuro, y **no queremos tener ese poder**.

## Cumplimiento legal

Nos tomamos en serio la Ley de Protección de la Persona frente al Tratamiento de sus Datos Personales (Ley 8968). Nuestro enfoque, y las preguntas legales que quedan por resolver antes de un lanzamiento, están en el [registro de decisiones](docs/BancaLink_Decisiones.md).
