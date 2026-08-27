# Identidad, llaves y recuperación

BancaLink no te pide correo, ni contraseña, ni nombre, ni cédula. Eso tiene una consecuencia que hay que decir de frente y no en letra chica: **si perdés tus llaves y no dejaste método de recuperación, nadie puede devolverte tu historial.** Ni nosotros.

## Cómo nace tu identidad

```mermaid
flowchart TD
    INST["📱 Primera vez que abrís la app"] --> GEN["Se genera un par de llaves<br/>en tu dispositivo"]
    GEN --> POST["POST /provision con tu llave pública"]
    POST --> DIR["El relay devuelve una dirección<br/>a7f3…@relay.bancalink.cr"]
    DIR --> ID["📮 Esa dirección ES tu identidad"]

    GEN --> USO1["Cifra los correos hacia vos"]
    GEN --> USO2["Firma el desafío al descargar blobs"]

    style ID fill:#dcfce7,stroke:#16a34a
```

El relay solo guarda `dirección aleatoria ↔ llave pública`. No hay nombre, ni correo, ni nada que te identifique.

**Por qué se firma para descargar.** Como el relay entrega cada blob una sola vez y después lo borra, alguien que descubriera una dirección ajena podría provocarle pérdida de transacciones —aunque nunca pudiera leer su contenido. Por eso exige firma válida antes de entregar o borrar nada.

## Las dos llaves y sus papeles

```mermaid
flowchart LR
    subgraph asim["🔑 Par asimétrico"]
        A1["Cifra los correos<br/>que llegan al relay"]
        A2["Firma la autenticación"]
    end

    subgraph sim["🗝️ Llave maestra simétrica"]
        S1["Cifra la bitácora completa<br/>para el respaldo"]
    end

    PASS["🔒 Tu contraseña o passkey"] -->|"envuelve"| sim

    style asim fill:#e6f4ff,stroke:#0369a1
    style sim fill:#fef3c7,stroke:#d97706
```

**La envoltura importa.** Cambiar la contraseña solo re-envuelve la llave maestra; **no re-cifra tu historial**. En una app local-first, re-cifrar todo significaría reescribir la base entera en cada dispositivo — lento y con riesgo de fallar a medias.

## Si perdés el dispositivo

```mermaid
flowchart TD
    P["😰 Perdiste el teléfono"] --> Q1{"¿Tenés otro<br/>dispositivo con la app?"}
    Q1 -->|Sí| N1["✅ Nivel 1<br/>Vinculás con QR o código corto<br/>NO perdés nada"]
    Q1 -->|No| Q2{"¿Tenés contraseña<br/>+ respaldo en la nube?"}
    Q2 -->|Sí| N2["✅ Nivel 2<br/>Instalás, restaurás, desbloqueás<br/>NO perdés nada"]
    Q2 -->|No| N3["⚠️ Nivel 3<br/>No hay forma de recuperarlo"]

    N3 --> R["Podés empezar de nuevo y reenviar<br/>los correos viejos del banco.<br/>Se pierde lo que hiciste a mano:<br/>categorías, gastos en efectivo"]

    style N1 fill:#dcfce7,stroke:#16a34a
    style N2 fill:#dcfce7,stroke:#16a34a
    style N3 fill:#fee2e2,stroke:#dc2626
```

**Reinstalar o borrar los datos en el mismo dispositivo cuenta como "no tengo otro dispositivo"** y cae directo al Nivel 2.

**El Nivel 3 es la consecuencia directa e innegociable del conocimiento cero:** nadie, en ningún lado, tiene una copia legible de tu llave. Por eso el onboarding tiene que empujar activamente a configurar al menos un método real de recuperación — si no, el Nivel 3 deja de ser la excepción y se vuelve lo normal.

## Dónde vive cada secreto

Un detalle que parece menor y no lo es: **los secretos no van en la bitácora de eventos.**

| | Bitácora de eventos | Almacén de secretos |
|---|---|---|
| Estructura | Append-only | Mutable, borrado real |
| Cifrado | Llave maestra | Llave maestra |
| ¿Entra al respaldo? | Sí | **No** — opt-in explícito |
| ¿Se fusiona entre dispositivos? | Sí | No |

**Por qué es obligatorio y no una preferencia:** la bitácora es *append-only*, así que **lo que entra ahí no se puede borrar nunca**. Una llave de API guardada como evento sería irrevocable: "revocada" sería apenas un evento posterior, con el valor original intacto en el historial, copiado a cada dispositivo y a cada respaldo (D19).

---

**Decisiones relacionadas:** D9 (passkeys), D10 (bitácora), D15 (sin registro), D16 (recuperación), D19 (llave de IA).
