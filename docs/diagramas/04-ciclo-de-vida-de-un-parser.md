# Ciclo de vida de un parser

Qué pasa cuando tu banco cambia el formato de sus correos y la app deja de entenderlos. Este recorrido cruza cinco decisiones (D19 a D23) y es difícil de reconstruir leyéndolas sueltas.

El caso que guía el diseño: **doña Catalina tiene cuenta en MUCAP, no existe parser para MUCAP, y no va a abrir una cuenta de GitHub.**

```mermaid
flowchart TD
    ROTO["📧 Llega un correo que la app no entiende"] --> IA

    subgraph local["📱 Todo esto pasa en el dispositivo"]
        IA{"¿Hay IA disponible?<br/>modelo local o llave propia"}
        IA -->|"no, o no la querés"| MAP
        IA -->|"sí — D19"| PRE["Pre-llena el mapeo<br/>ella solo confirma o corrige"]
        PRE --> MAP
        MAP["Mapeo — D21<br/>Catalina marca: esto es el monto,<br/>esto el comercio, esto la fecha"]
        MAP --> VER["Verificación<br/>la app muestra qué extrajo<br/>ella confirma que está bien"]
        VER --> INV{"Prueba de invarianza"}
        INV -->|"falla: memorizó datos de ella"| MAP
    end

    INV -->|"pasa: es estructural"| OFR["¿Compartirlo?<br/>anónimo por defecto"]
    OFR -->|"Ahora no"| FIN["Su app funciona igual<br/>sin insistir"]
    OFR -->|"Compartir"| ENV

    subgraph serv["⚙️ Relay"]
        ENV["Propuesta firmada — D22<br/>anónima · límite por llave<br/>la llave se descarta"]
        ENV --> COLA[("Cola de propuestas")]
    end

    COLA --> BOT["🤖 Bot abre PR público"]
    BOT --> REV{"Revisión humana — D20"}
    REV -->|"rechazada"| NADA["Catalina no se entera<br/>su app sigue funcionando"]
    REV -->|"aprobada"| BIB[("📚 Biblioteca de parsers")]
    BIB -->|"índice COMPLETO<br/>nunca uno por banco"| TODOS["📱 Todas las personas de MUCAP"]

    style local fill:#e6f4ff,stroke:#0369a1
    style serv fill:#fff4e6,stroke:#d97706
```

## Las cinco ideas que sostienen esto

**1. El camino principal no usa IA.** Catalina marca los campos con el dedo y la app deduce la regla. Sin llave de API, sin bajar un modelo de varios gigas a un teléfono de gama media, sin que el correo salga del dispositivo. La IA solo **adelanta** el trabajo cuando está disponible; la app nunca se queda esperándola (D21).

**2. Quien valida es ella, no el modelo.** Catalina tiene la verdad en la mano: sabe que ese correo era una compra de ₡45.200 en el AutoMercado. Por eso la calidad del generador importa menos de lo que parece.

**3. La prueba de invarianza reemplaza la detección de datos personales.** En vez de buscar datos de Catalina para borrarlos —lista negra, siempre deja pasar alguno— se aprovecha que **un parser bueno debe funcionar igual con otros valores**:

```mermaid
flowchart LR
    O["Correo de Catalina"] --> M["Copia con TODOS<br/>los valores cambiados"]
    M --> T{"¿El parser<br/>extrae bien?"}
    T -->|sí| E["✅ Es estructural<br/>se puede compartir"]
    T -->|no| N["❌ Memorizó sus datos<br/>NO sale del dispositivo"]
```

Determinista, sin falsos negativos silenciosos. De yapa, atrapa parsers frágiles.

**4. Anónima de verdad, y con freno al spam.** El relay ya sabe autenticar por firma sin conocer identidad (D15), así que la propuesta viaja firmada pero anónima, con límite por llave pública. **La llave se descarta después de validar**: si quedara guardada junto a la propuesta, la cola misma sería el vínculo que el anonimato promete que no existe.

**5. El PR es público a propósito.** No es un detalle de flujo: cada arreglo deja un registro fechado y verificable de que la falta de un estándar de banca abierta le cuesta plata y tiempo a la gente. Esa evidencia es el objetivo 2 del proyecto. Si los aportes murieran en una cola privada, se perdería justo cuando más volumen tendría.

## Dos puertas, cada una para su público

| | Dentro de la app | GitHub |
|---|---|---|
| Para quién | Catalina, que no sabe qué es GitHub | Quien quiere crédito por su aporte |
| Identidad | Anónima siempre | Pública |
| Firma DCO | No aplica — un parser es dato, no código (D22) | — |

## Lo que se cede con el anonimato

Si la propuesta se rechaza, **Catalina no se entera**: no hay canal de vuelta. Pero su app sigue funcionando con la regla que ella hizo, así que no queda peor que antes. El aporte era altruista.

Y cuando llegue el parser oficial de MUCAP, **el de ella no se toca**: lo local se conserva mientras funcione. Si algún día falla, ahí se intenta el oficial. Sin preguntas ni pantallas de elección — Catalina no tiene por qué saber que existen dos.

---

**Decisiones relacionadas:** D6, D7, D19, D20, D21, D22, D23 · [diseño de contribución anónima](../superpowers/specs/2026-08-26-contribucion-anonima-parsers-design.md).
