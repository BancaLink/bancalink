# BancaLink

**Tus finanzas, sin que nadie más las vea.**

BancaLink es una aplicación de finanzas personales libre, gratuita y de conocimiento cero para Costa Rica. Lee las notificaciones que tu banco ya te envía por correo y las convierte automáticamente en un registro de gastos e ingresos — sin que BancaLink, ni nadie más, pueda ver jamás el contenido de esos correos o tus transacciones.

> **Estado del proyecto:** las decisiones de dirección están cerradas. Estamos arrancando el diseño de la arquitectura técnica — todavía no hay código de aplicación en este repositorio.

## Por qué existe BancaLink

En Costa Rica no existe un estándar de banca abierta (Open Banking) que te permita conectar tus cuentas bancarias con aplicaciones de finanzas de forma oficial y segura. El BCCR y la SUGEF apenas lo están explorando. Ante ese vacío, las apps de finanzas que existen recurren a un truco: leen los correos de notificación que te manda el banco, usando el email como una tubería improvisada que nadie diseñó para eso.

BancaLink hace exactamente lo mismo — pero con dos diferencias que nos importan:

1. **Nuestro código es 100% libre y gratuito**, para siempre.
2. **Usamos este proyecto para exponer, con evidencia pública, la falta de un estándar oficial** — cada vez que un banco cambia el formato de sus correos y rompe un parser, queda documentado.

Manejar tu plata es parte de tu salud integral. Tener buenas herramientas para hacerlo no debería ser un lujo reservado para quienes pueden pagar una suscripción.

Este repositorio cubre la construcción de la aplicación (objetivo #1). El objetivo #2 — impulsar un marco normativo oficial de banca abierta en Costa Rica — se trabaja en un frente separado, usando esta app y su comunidad como evidencia.

## Cómo funciona (a alto nivel)

1. Configurás en tu correo (Gmail, iCloud, Yahoo, corporativo — el que sea) una regla de reenvío automático hacia una dirección dedicada que te da BancaLink.
2. Un servidor "relay" mínimo recibe ese correo, lo **cifra inmediatamente con tu llave pública** y borra el texto original. El relay nunca puede leer tu correo ni tus gastos.
3. Tu app (una PWA instalable, local-first) descarga el paquete cifrado, lo descifra en tu dispositivo, extrae la transacción y la guarda localmente.
4. Tus datos se respaldan y sincronizan entre tus dispositivos a través de tu propia nube (Google Drive, OneDrive, Dropbox) o un archivo local — siempre cifrados con tu clave, nunca legibles por nosotros.
5. Cuando un correo tiene un formato que la app no reconoce, y solo si vos lo autorizás y ponés tu propia clave de API, la IA ayuda a generar una regla de lectura (parser) una única vez. De ahí en adelante, ese formato se procesa gratis, localmente y sin conexión.

## Principios que no negociamos

- **Conocimiento cero:** el servidor recibe, pero no entiende. Nunca podemos leer tus correos ni tus transacciones.
- **Local-first:** tus datos viven en tu dispositivo; la nube es solo respaldo cifrado, bajo tu control.
- **Gratis para siempre, auto-hospedable:** licencia [AGPLv3](https://www.gnu.org/licenses/agpl-3.0.html) — cualquiera puede usar, auto-hospedar y modificar la app sin pagar, y cualquiera que la ofrezca como servicio debe liberar sus cambios.
- **Cero telemetría:** no medimos clics, no medimos uso, no medimos nada.
- **Bring your own AI:** la IA es opcional, la activás vos, y solo se usa para generar parsers — no para procesar tus correos de forma rutinaria.
- **Parsers legibles por humanos:** las reglas de lectura de cada banco son archivos YAML/JSON simples, mantenidos por la comunidad en GitHub.
- **Apego a la Ley 8968:** consentimiento claro, borrado real de datos, eliminación inmediata de correos del servidor tras procesarlos.

El razonamiento completo detrás de cada una de estas decisiones —y de las alternativas que descartamos— está documentado en [`docs/BancaLink_Decisiones.md`](docs/BancaLink_Decisiones.md).

## Alcance de la primera versión

Construimos poco, pero modelamos todo. La v1 se enfoca en: ingesta de correos, categorización de gastos e ingresos, y reportes básicos. El modelo de datos, sin embargo, está preparado desde el día uno para tarjetas de crédito, préstamos, metas de ahorro y pagos divididos entre personas.

## Licencia

[AGPLv3](https://www.gnu.org/licenses/agpl-3.0.html). Elegimos esta licencia específicamente para que nadie pueda tomar el código, cerrarlo, y cobrar por un servicio propietario sin devolverle nada a la comunidad.

## Contribuir

Todavía estamos en la etapa de diseño de arquitectura — pronto habrá guías de contribución, empezando por el formato de los parsers bancarios. Los aportes de código se firman con [DCO](https://developercertificate.org/) (sign-off en el commit), no con un CLA.

## Cumplimiento legal

Nos tomamos en serio la Ley de Protección de la Persona frente al Tratamiento de sus Datos Personales (Ley 8968). Los detalles de nuestro enfoque, incluyendo las preguntas legales aún pendientes de resolver antes del lanzamiento, están en el [registro de decisiones](docs/BancaLink_Decisiones.md).
