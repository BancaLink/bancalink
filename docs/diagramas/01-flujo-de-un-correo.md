# Flujo de un correo, de punta a punta

Este es el recorrido completo de una transacción: desde que el banco manda la notificación hasta que aparece como un gasto en tu app y queda respaldada.

Si vas a leer un solo diagrama de este proyecto, que sea este.

```mermaid
flowchart TD
    B["🏦 Banco"] -->|"notificación por correo"| M["📬 Tu buzón personal<br/>Gmail, iCloud, corporativo"]
    M -->|"regla de reenvío que vos configurás"| RX

    subgraph relay["⚙️ Relay — servidor de BancaLink"]
        RX["Recepción SMTP"] --> CIF["Cifrado con tu llave pública"]
        CIF --> DEL["Borrado del texto original"]
        DEL --> BLOB[("Blob cifrado<br/>ilegible para el servidor")]
    end

    BLOB -->|"descarga autenticada por firma<br/>se entrega una vez y se borra"| DES

    subgraph disp["📱 Tu dispositivo — acá pasa todo lo importante"]
        DES["Descifrado"] --> NOR["Normalización del HTML"]
        NOR --> PAR["Parsers"]
        PAR --> LOG[("Bitácora de eventos<br/>append-only")]
        LOG --> UI["Saldos, reportes, categorías"]
    end

    LOG -->|"cifrado con tu llave maestra"| NUBE[("☁️ TU nube<br/>Drive · OneDrive · Dropbox")]
    NUBE -.->|"restaurar o sincronizar<br/>otro dispositivo"| DES

    style relay fill:#fff4e6,stroke:#d97706
    style disp fill:#e6f4ff,stroke:#0369a1
```

## Lo que hay que entender de este dibujo

**El relay recibe pero no comprende.** Cifra apenas recibe, borra el original y nunca interpreta el contenido. No sabe de qué banco es, ni de cuánto fue el movimiento. Los límites exactos de esa afirmación están en [`02-limites-de-confianza.md`](02-limites-de-confianza.md) — vale leerlo, porque hay matices.

**Nunca pedimos acceso a tu buzón.** Vos configurás una regla de reenvío y la borrás cuando querás. Esa fue una decisión deliberada: OAuth de Gmail arrastraba auditorías CASA que habrían hecho el proyecto inviable (D3).

**Todo lo que requiere entender tus datos ocurre en tu dispositivo.** Descifrado, parsing, categorías, reportes. El servidor no participa.

**La nube es almacenamiento tonto.** El respaldo va cifrado con tu llave maestra, en la carpeta de aplicación del proveedor. Google puede leer esa carpeta; lo que encuentra es ruido (D8).

## Qué pasa si algo falla

| Falla | Qué ocurre |
|---|---|
| Relay caído o sin red | La app funciona completa con datos locales. Solo se atrasan los correos nuevos |
| Blob de nube corrupto | Nunca se sobrescribe el respaldo a ciegas ni se toca lo local. Se avisa y se preserva |
| Correo duplicado | Produce el mismo ID determinista y se descarta al fusionar (D10) |

---

**Decisiones relacionadas:** D2 (relay ciego), D3 (ingesta por reenvío), D8 (respaldos en tu nube), D10 (bitácora de eventos), D15 (sin cuentas).
