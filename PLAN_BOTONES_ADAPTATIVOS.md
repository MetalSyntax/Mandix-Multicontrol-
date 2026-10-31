# Plan: botones que se adaptan al código elegido

**Problema.** Cada control (`remote_*.xml`) dibuja siempre las mismas teclas. Al cambiar de código con ◀ ▶,
el perfil nuevo (`RemoteProfile.keys`) puede no traer alguna tecla. Al tocarla sale
`toast_code_key_missing` ("ese código no tiene la tecla"), en `SkinRemoteActivity.send()`.

**Objetivo.** Que el control muestre solo lo que el código elegido puede hacer. Si una tecla tiene un
equivalente, debe usarlo y mostrarlo. Las teclas que el código trae pero el dibujo no tiene deben quedar
accesibles.

## 1. Saber si una tecla está disponible (base de todo)

Una sola función en `SkinRemoteActivity`:

```kotlin
private fun available(kind: String, name: String) = when {
    kind == "ac" -> true                                    // el aire lo arma AcCodec
    wifiOn && wifiLink!!.supports(name) -> true             // por red no depende del código IR
    kind == "bt" -> false                                   // Bluetooth sin Wi-Fi: imposible
    isCustom -> true                                        // código personalizado: todo se puede aprender
    else -> key(name) != null                               // ya resuelve aprendidas + ALIASES
}
```

- Se llama desde `render()`, que ya se ejecuta en cada cambio de código (`step()`), Wi-Fi o modo AUTO.
- `bindKeys()` guarda la lista de vistas con tag para no recorrer el árbol en cada `render()`.
- Test unitario: perfil sin `Mute` → no disponible; con Wi-Fi Roku → disponible; personalizado → disponible.

## 2. Cómo se ve una tecla no disponible

- **Atenuar, no ocultar:** `alpha = 0.25f` y `isEnabled = false`. El dibujo conserva la forma del control
  físico, así que nada salta de lugar al pasar de un código a otro.
- Al tocar una tecla atenuada (vía `setOnTouchListener`, porque una vista deshabilitada no recibe clic):
  un aviso corto que diga qué hacer, por ejemplo "Este código no tiene Mute: prueba ◀ ▶ o actívalo por Wi-Fi".
- **Excepción:** las píldoras de apps (Netflix/Disney/Hulu…) sí se ocultan (`GONE`). Ver el punto 4.

## 3. Equivalencias: la tecla cambia en vez de fallar

Se amplía `ALIASES`, que hoy cubre Mode↔Source y Play↔Pause:

| Tecla del dibujo | Si falta, usar |
|---|---|
| Ok | Select, Enter |
| Back | Return, Exit |
| Options | Info, Menu |
| Home | Menu |
| Play (▶❚❚) | Pause; si solo hay Play, el ícono cambia a ▶ |
| Vol_up / Vol_dn | (no hay alias: se atenúan) |

- `key()` devuelve también **qué** nombre resolvió. Si no es el del dibujo, `render()` cambia el ícono o el
  texto de la vista (mapa nombre → `ic_k_*`, con los íconos nuevos) para que el usuario vea qué va a enviar.

## 4. Fila de apps generada, no fija

- Hoy las píldoras de apps están escritas a mano en el XML (Netflix, Disney+, Hulu, Sling / Prime).
- Se cambian por un `LinearLayout` vacío (`@id/appRow`) que `render()` llena con la unión de:
  - las apps de `profile.keys` que estén en una lista conocida (nombre → color/texto, la de los XML actuales);
  - `RokuEcp.APPS` / `FireAdb.APPS` si el Wi-Fi está activo.
- Así, un código Roku TV TCL muestra Sling, un Hisense no, y por Wi-Fi aparecen todas.

## 5. Teclas que el código tiene y el dibujo no

- Botón "⋯ Más teclas" en el selector de código (como ⌨ Escribir), visible solo si existen.
- Abre una hoja con una cuadrícula de botones genéricos (nombre de la tecla) para
  `profile.keys.keys − teclas del dibujo`.
- Mantener presionado sigue sirviendo para aprender, igual que en el dibujo.

## 6. Casos especiales

- **AUTO / barrido** (`sweep:`): durante la búsqueda nada se atenúa (el perfil cambia cada segundo).
- **Aires** (`ac:`): siempre disponibles. Swing/Sleep sin soporte ya avisan con `toast_ac_unsupported`; se
  atenúan con `AcCodec.command(p, action) == null`.
- **Fire TV por IR:** solo Power/Vol/Mute van por IR, y la cruz es `bt:`. Sin Wi-Fi, la cruz queda atenuada
  con el aviso "activa Wi-Fi para navegar".

## Orden de trabajo

1. `available()` + atenuar + aviso al tocar, con su test. Esto solo ya elimina el mensaje de error.
2. Equivalencias + cambio de ícono.
3. Fila de apps dinámica (Roku, Amazon).
4. Hoja "Más teclas".

El paso 1 es pequeño (≈40 líneas). Los pasos 2–4 son independientes y se pueden hacer uno por versión.
