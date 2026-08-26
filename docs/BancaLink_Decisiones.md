# **El camino de BancaLink: Registro de Decisiones**

Este es el diario de navegación de **BancaLink**. Aquí documentamos por qué tomamos las decisiones de dirección que definen el proyecto. Queremos que cualquier persona que se una al equipo más adelante entienda nuestro razonamiento sin tener que reabrir debates ya cerrados.

*(Nota: Ya hicimos la verificación defensiva del nombre en el Registro de Propiedad Industrial conforme a nuestra directriz D13).*

**Estado actual:** Decisiones de dirección cerradas. El diseño de la arquitectura técnica también quedó cerrado y vive en [`docs/superpowers/specs/2026-08-26-arquitectura-tecnica-design.md`](superpowers/specs/2026-08-26-arquitectura-tecnica-design.md). Lo siguiente es el plan de implementación. **Última actualización:** 26 Agosto 2026\.

## **Nuestra razón de ser (El Contexto)**

En Costa Rica no tenemos un estándar de "banca abierta" (Open Banking) que le permita a una persona conectar sus cuentas con aplicaciones de finanzas de manera oficial y segura. Mientras en otros países hay marcos regulatorios, aquí el Banco Central (BCCR) y la SUGEF apenas lo están explorando.

Ante este vacío, las apps que existen recurren a un truco: leen los correos de notificación que te envía el banco. Usan el email como una tubería improvisada. Nosotros vamos a hacer exactamente lo mismo, pero con dos grandes diferencias: nuestro código será 100% libre y gratuito, y vamos a usar este proyecto para exponer públicamente la falta de un estándar oficial.

**Nuestra premisa innegociable:** Manejar tu plata es parte de tu salud integral. Tener buenas herramientas para hacerlo no debería ser un lujo reservado para quienes pueden pagar una suscripción.

*(Nota: Este documento cubre la construcción de la aplicación. Nuestro segundo objetivo —impulsar un marco normativo oficial— lo llevamos en un frente separado).*

## **Índice rápido de decisiones**

* **Legal y Comunidad:** \[D1\] Licencia AGPLv3, \[D13\] Marca registrada, \[D14\] Acuerdos de contribución, \[Extra\] Figura legal.  
* **Arquitectura y Privacidad:** \[D2\] Relay ciego \+ Local-first, \[D3\] Ingesta por reenvío, \[D6\] El rol de la IA, \[D7\] Parsers compartidos, \[D8\] Respaldos en tu propia nube, \[D9\] Manejo de claves, \[D15\] Sin cuentas ni registro, \[D16\] Cómo recuperás tu historial.  
* **Producto y Datos:** \[D4\] Para quién diseñamos, \[D5\] Alcance inicial, \[D12\] Cero telemetría, \[D10\] Sincronización sin conflictos, \[D11\] Manejo de monedas, \[D17\] Saldos y discrepancias.

## **1\. Reglas de juego (Legal y Comunidad)**

### **D1 — Elegimos la Licencia AGPLv3**

Queríamos una licencia que obligará a cualquiera que copie el proyecto a mantenerlo abierto. Licencias como MIT o Apache 2.0 son muy permisivas y permiten que alguien tome nuestro código, lo cierre y empiece a cobrar por un servicio propietario sin devolverle nada a la comunidad.

La licencia GPL normal tampoco nos servía porque solo te obliga a compartir el código si "distribuyes" la app (y una app web no se distribuye, se visita). Por eso elegimos **AGPLv3**, que cierra ese portillo: si alguien monta nuestro código en un servidor para que otros lo usen, tiene que liberar sus modificaciones. Así garantizamos que la herramienta principal siempre sea libre y accesible para todos.

### **D13 — El nombre queda libre (por ahora)**

En Costa Rica, los derechos de marca se ganan registrando, no solo usando. Sin embargo, registrar una marca formalmente toma tiempo y recursos. Por ahora, hicimos una verificación gratuita en línea para asegurarnos de no pisar el nombre de nadie más. Ya apartamos el dominio y la organización en GitHub, pero dejaremos el papeleo oficial de la marca para cuando el proyecto tenga más peso.

### **D14 — Preferimos un DCO en lugar de un CLA**

Cuando alguien aporte código, usaremos un "Developer Certificate of Origin" (una simple firma en sus commits) en lugar de hacerles firmar un contrato legal extenso (CLA). Un CLA nos permitiría cambiar la licencia del proyecto a futuro, pero *no queremos* tener ese poder. El hecho de que sea casi imposible cambiar la licencia sin contactar a cada contribuyente es una protección estructural para que BancaLink nunca deje de ser libre.

### **Sobre crear una fundación o asociación**

Originalmente pensamos en crear una asociación sin fines de lucro para tener peso político ante la Asamblea Legislativa y separar los gastos. Sin embargo, **hemos decidido posponerlo**.

¿Por qué? Porque el costo, la contabilidad y la burocracia nos iban a frenar en seco justo cuando necesitamos programar. Además, nuestra estrategia política cambió: en lugar de pelear una ley en la Asamblea, vamos a ir directamente al BCCR con argumentos técnicos. Les mostraremos la app funcionando y nuestro repositorio de *parsers* como prueba de que el sistema actual es un parche.

Por ahora, la infraestructura (dominio y un servidor pequeño) la asumiremos de forma personal.

## **2\. Cómo está construida (Arquitectura y Privacidad)**

### **D2 — El servidor recibe, pero no entiende (Relay ciego)**

Teníamos un dilema: el reenvío automático de correos requiere un servidor 24/7, pero queríamos que la app fuera "local-first" (que tus datos vivan solo en tu teléfono).

Nuestra solución es tener un servidor "relay" ultra básico. Este servidor recibe tu correo bancario, lo encripta inmediatamente con tu llave pública, lo guarda como un paquete cerrado y borra el texto original. Cuando abres la app, esta descarga el paquete, lo desencripta en tu celular y extrae el gasto. **Nosotros nunca podemos leer tus correos ni tus gastos.**

### **D3 — Ingesta por reenvío**

Descartamos conectarnos directamente a tu Gmail. Hacerlo requiere que Google nos haga auditorías de seguridad carísimas (miles de dólares al año) que no podemos pagar. En su lugar, te pediremos que entres a tu correo y crees una regla automática para reenviar las notificaciones del banco a nuestro relay. Sabemos que da un poco de pereza configurarlo la primera vez, así que crearemos guías visuales súper sencillas. A cambio, ganas total privacidad: nosotros nunca tocamos tu buzón y puedes cortarnos el acceso cuando quieras.

### **D6 — La Inteligencia Artificial no lee tus correos siempre**

Mandar cada uno de tus gastos a ChatGPT sería carísimo y pésimo para la privacidad. Solo usamos la IA cuando el banco cambia el formato del correo y nuestra app no lo entiende. En ese caso (y solo si tú nos das permiso y usas tu propia clave de API), enviamos *un solo correo* a la IA para que nos ayude a crear una regla de lectura (un parser). A partir de ahí, todos los correos futuros con ese formato se leen localmente y gratis.

### **D7 — Parsers legibles por humanos**

Esas reglas de lectura (parsers) no serán código complejo, sino archivos de texto simples (YAML o JSON). Esto permite que cualquier persona, incluso sin saber programar mucho, pueda entrar a GitHub y arreglar una regla si su banco cambió el formato.

### **D8 — Tus respaldos van a TU nube**

Si pierdes el teléfono y la app es local, pierdes tus datos. Para evitarlo, la app se sincronizará automáticamente con tu propio Google Drive, OneDrive o Dropbox. Los datos irán cifrados, por supuesto. Nosotros no guardamos nada y tú mantienes el control total.

### **D9 — Sin contraseñas que recordar (Passkeys)**

Usaremos biometría (huella o rostro de tu teléfono) como método principal para encriptar y abrir la app, respaldado por una contraseña normal. Si llegas a perder tus clave, siempre puedes volver a reenviar tus correos viejos del banco y la app reconstruirá tu historial.

### **D15 — No te vamos a pedir que te registres (26 Agosto 2026\)**

*Decisión nueva, surgida al diseñar la arquitectura técnica.*

No hay registro: no te pedimos correo, ni contraseña, ni nombre, ni cédula. La primera vez que abres la app, esta genera un par de llaves en tu propio dispositivo y le pide al relay una dirección de correo dedicada. Lo único que el relay guarda es "esta dirección corresponde a esta llave pública". **Tu dirección dedicada es tu identidad.**

Ese mismo par de llaves también te autentica: para descargar tus correos pendientes, la app firma un desafío que el relay verifica. Esto resuelve un hueco que teníamos: como el relay entrega cada paquete una sola vez y luego lo borra (D2), alguien que descubriera tu dirección podría hacerte perder transacciones aunque nunca pudiera leerlas.

Una consecuencia de no tener cuentas centrales: nada impide que una misma persona termine con varias identidades separadas. Si es a propósito (por ejemplo, separar tus finanzas personales de las de tu negocio), nos parece bien. Si es por accidente —instalar la app en un segundo teléfono sin saber que hay que vincularlo— es un problema, y lo resolvemos preguntando siempre al inicio *"¿es tu primera vez, o ya usas BancaLink en otro dispositivo?"* antes de ofrecerte empezar de cero.

### **D16 — Cómo recuperás tu historial si perdés el dispositivo (26 Agosto 2026\)**

*Decisión nueva, surgida al diseñar la arquitectura técnica. Amplía y precisa lo que D9 decía de forma general.*

Hay tres escenarios, de mejor a peor:

1. **Tenés otro dispositivo con la app** (aunque no lo tengas encima en este momento): lo vinculás escaneando un código QR, o escribiendo un código corto que te muestra el dispositivo viejo. No perdés nada.  
2. **No tenés otro dispositivo, pero sí tu contraseña y tu respaldo en la nube** (D8): instalás, elegís "restaurar", conectás tu nube y desbloqueás con tu contraseña. No perdés nada. Si no te acordás en cuál de tus cuentas de nube quedó el respaldo, la app te deja probar una por una hasta encontrarlo — la carpeta está oculta y no la podrías buscar a mano.  
3. **No tenés ni lo uno ni lo otro:** no hay forma de recuperar tu historial. Ni nosotros ni nadie: esa es la consecuencia directa de que nunca tengamos una copia de tu llave. Lo que sí podés hacer es empezar de nuevo y reenviar los correos viejos del banco que sigan en tu bandeja de entrada, y la app reconstruye lo que pueda leer automáticamente. **Se pierden las cosas que hiciste a mano** (categorías que cambiaste, gastos en efectivo que anotaste).

Nos comprometemos a decir esto con claridad desde el principio, no en letra chica. Y a insistirte durante la configuración inicial para que dejés al menos un método de recuperación listo — si no, el escenario 3 deja de ser la excepción y se vuelve lo normal.

## **3\. Producto y Manejo de Datos**

### **D4 — Diseñado para Ticos, no para informáticos**

Nuestra prioridad es que cualquier persona pueda usarla. Si una decisión técnica requiere que el usuario sepa qué es un contenedor o la terminal, la descartamos. Por eso es una aplicación web instalable (PWA) fácil de usar.

### **D5 — Construiremos poco, pero modelaremos todo**

En la primera versión solo haremos lo básico: leer ingresos, gastos, categorizar y mostrar reportes. Sin embargo, la base de datos interna desde el día uno estará preparada para tarjetas de crédito, préstamos y pagos divididos.

**Ampliación (26 Agosto 2026):** al diseñar la arquitectura técnica nos dimos cuenta de que la primera versión necesitaba tres cosas más para ser realmente útil, y las agregamos:

* **Efectivo y entrada manual.** El correo nunca va a ver la plata en efectivo, ni buena parte de los SINPE. Una app que solo lee correos deja por fuera a mucha gente, así que ingresar movimientos a mano es parte de la v1, no un agregado posterior.  
* **Saldos y diferencias** (ver D17).  
* **Etiquetas** para marcar transacciones de forma transversal ("vacaciones", "reembolsable"), e **importar tu estado de cuenta** en CSV o Excel para arrancar con el último período ya cargado. El PDF lo dejamos para después: cada banco lo arma distinto y es frágil de leer.

También diseñamos completo el manejo de **presupuestos** —incluyendo que la app aprenda de tus gastos y te proponga un presupuesto inicial que vos ajustás— pero decidimos **dejarlo fuera de la v1**. No lo descartamos: el diseño está hecho y la base de datos ya lo contempla. Simplemente preferimos que la app primero haga bien una cosa: ayudarte a entender en qué se te va la plata.

El resto (tarjetas, créditos de tiendas tipo Gollo o Monge, préstamos, depósitos a plazo, metas de ahorro, gastos compartidos) queda modelado en la base de datos desde el día uno, pero sin construir todavía.

### **D12 — Cero espionaje (Ni siquiera telemetría básica)**

No vamos a medir dónde haces clic, ni cuánto tiempo usas la app. Nada. Nuestro compromiso con la privacidad es total. Para probar que la app se usa y que los bancos cambian sus correos, nos basta con la actividad pública de la comunidad arreglando los *parsers* en GitHub.

### **D10 — Historial a prueba de errores**

En lugar de tener una tabla normal que se sobreescribe, guardaremos todo como una bitácora de eventos ("hoy entró este gasto", "mañana lo moví de categoría"). Esto hace que la sincronización entre varios de tus dispositivos no tenga conflictos, y nos ayuda a no registrar dos veces el mismo cobro si el banco manda el correo duplicado.

### **D11 — Respetamos la moneda original**

Si pagas en colones, guardamos colones; si pagas en dólares, guardamos dólares. El cálculo al tipo de cambio se hace solo cuando estás viendo la pantalla. Así evitamos que tus datos históricos se deformen cuando el dólar suba o baje.

### **D17 — La app va a estar equivocada, y hay que asumirlo (26 Agosto 2026\)**

*Decisión nueva, surgida al diseñar la arquitectura técnica.*

Si la app suma solo lo que ve en los correos, tarde o temprano te va a decir que tenés ₡100.000 cuando en realidad tenés ₡88.000. No es un error de programación: es inevitable. El correo no cubre el efectivo, ni todos los SINPE, ni las comisiones del banco, ni la suscripción que se rebajó sin avisarte, ni el depósito que alguien te hizo de sorpresa.

En vez de esconder eso, lo mostramos. Vos podés decirle a la app cuánto tenés de verdad en la cuenta (a mano, o importando tu estado de cuenta), y la app te muestra la diferencia y te deja registrar el ajuste.

Esto tiene una ventaja que nos parece importante: **la app se corrige sola.** Un gasto en efectivo que nunca anotaste en marzo deja de arrastrar el error hasta agosto en cuanto le confirmás un saldo real más reciente. Sin esto, los errores se acumulan para siempre — y esa es justamente la razón por la que la gente termina abandonando las apps de finanzas.

## **4\. Nuestro compromiso legal (Ley 8968\)**

Nos tomamos muy en serio la Ley de Protección de la Persona frente al Tratamiento de sus Datos Personales (Ley 8968). Gracias a nuestra arquitectura donde "no sabemos nada" (conocimiento cero), evitamos muchísimos riesgos. Aún así:

* Pediremos tu consentimiento de forma clara y honesta, nada de esconder cosas en términos larguísimos.  
* Podrás llevarte tus datos o borrarlos de verdad cuando quieras.  
* Eliminamos tus correos del servidor inmediatamente después de procesarlos.

**Tareas pendientes para cuando nos acerquemos al lanzamiento:** Necesitamos revisar con un abogado si el historial de gastos se considera un "dato sensible" (porque refleja tu condición socioeconómica) y confirmar si, a pesar de que no guardamos tus cuentas en la nube, nos toca inscribirnos ante la PRODHAB.

*Nota de equipo: Este documento está vivo. Si en el futuro cambiamos de opinión en alguna de estas decisiones, no borraremos el historial. Simplemente agregaremos la fecha, qué decidimos cambiar y por qué, para mantener nuestra historia transparente.*

