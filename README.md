# rayrelay

Rendezvous + relay **ciego** para [takeit](../takeit) y [msg](../msg), escrito en [raylang](https://github.com/ray-language/raylang): resuelve el pendiente compartido de ambas apps (dos peers detrás de NAT que no pueden verse) con códigos cortos de emparejamiento, un relay TCP que copia bytes sin mirarlos, y un respondedor STUN-lite por UDP para intentar hole punching. El relay nunca ve claro: las apps de encima ya cifran de extremo a extremo.

```text
# En un VPS
$ rayrelay serve --port 7440 --metrics-port 9100
$ rayrelay stun --port 7441          # proceso aparte (ver hallazgo UDP)

# Peer A (detrás de NAT)
$ rayrelay host --server vps.example.com
code: rsp3-zceb
peer should run: rayrelay join rsp3-zceb --server vps.example.com --port 7440
peer connected
hola →                                ← lo que escriba el otro

# Peer B
$ rayrelay join rsp3-zceb --server vps.example.com

# ¿Cómo me ve el mundo? (STUN-lite)
$ rayrelay probe --server vps.example.com
ADDR 203.0.113.7 51824
```

## Protocolo

Una línea de control en texto y después bytes crudos:

```text
host   → "HOST\n"           ← "CODE <xxxx-xxxx>\n"  …espera…  ← "PEER\n"  → pipe
joiner → "JOIN <code>\n"    ← "OK\n" | "ERR <motivo>\n"                   → pipe
UDP    → "WHOAMI"           ← "ADDR <ip-observada> <puerto-observado>\n"
```

Con `--authorized-keys`, solo las claves de la lista pueden hacer `HOST` (ver
[Autorización](#autorización)):

```text
host   → "HOST <key-id>\n"  ← "CHALLENGE <nonce>\n"
       → "AUTH <relay-host> <firma>\n"  ← "CODE <xxxx-xxxx>\n" | "ERR <motivo>\n"  …
```

Los bytes que un cliente mande pegados a su línea de control (pipelining) se
reenvían al otro lado tras el emparejamiento — no se pierden. Los códigos
(`xxxx-xxxx`, alfabeto sin caracteres confundibles) rutan la sesión, no son
secretos: la confidencialidad es de la capa de arriba.

## Uso

```text
rayrelay serve [--bind H] [--port N] [--idle SECS] [--metrics-port N]
               [--authorized-keys FILE] [--name HOST[,HOST…]]
rayrelay stun  [--bind H] [--port N]
rayrelay probe [--server H] [--port N]
rayrelay host  [--server H] [--port N] [--key FILE]
rayrelay join CODE [--server H] [--port N]
rayrelay keygen FILE [--comment TEXT]
```

- `--idle`: corte por inactividad por dirección durante el relay (default 300 s;
  `0` = sin límite; los sockets llevan `SO_KEEPALIVE`, así que un peer que
  desaparece sin FIN se recoge igualmente).
- `--metrics-port`: endpoint Prometheus (`/metrics`): conexiones, pares
  activos/total, bytes relayados, códigos en espera, JOIN fallidos.
- Espera máxima de un host por su peer: 10 min. Handshake: 30 s.
- Apagado: SIGTERM/SIGINT vía `signals()`; los pares vivos caen con el proceso.

## Autorización

Sin `--authorized-keys` el relay es **abierto**: cualquiera puede hacer `HOST`. Con él, solo las
claves Ed25519 de la lista, al estilo de `authorized_keys` de SSH:

```text
# /etc/rayrelay/keys — una línea por usuario: clave, nombre y límites opcionales
ed25519:988de305…0b32  roberto  pairs=4  daily=200GiB  rate=50MiB/s
ed25519:9c01d4e2…7a1f  maria    pairs=2  daily=20GiB
```

```sh
rayrelay serve --authorized-keys /etc/rayrelay/keys --name relay.example.com
rayrelay keygen ~/.config/rayrelay/key --comment roberto   # imprime la línea para el archivo
kill -HUP <pid>                                            # relee el archivo sin cortar nada
```

- **Desafío por conexión.** El host anuncia su key-id (16 hex de `sha256(clave pública)`), el
  relay responde con 32 bytes aleatorios y el host firma
  `"rayrelay-auth-v1\n" ‖ relay-host ‖ "\n" ‖ key-id ‖ "\n" ‖ nonce`. Una firma capturada no sirve
  para otra conexión. Un key-id desconocido también recibe desafío: la respuesta no revela qué
  claves existen.
- **Ligada al relay.** La firma incluye el nombre con que el cliente marcó el relay; con
  `--name`, el relay solo acepta firmas para sus nombres (un relay falso no puede reenviarle el
  desafío a este). Sin `--name` se acepta cualquiera y se avisa al arrancar.
- **El relay solo guarda claves públicas**: robarle el archivo no da acceso.
- **Quien hace `JOIN` no necesita clave**: el código, de un solo uso y con caducidad de 10 min,
  es su autorización.
- **Límites por clave** (0 u omitido = sin límite): `pairs` (códigos en espera + pares activos a
  la vez), `daily` (bytes por día UTC; se comprueba al registrar, así que la última sesión puede
  pasarse) y `rate` (bytes/s por dirección de cada par).
- **Por IP:** 20 fallos (`AUTH` rechazado, `JOIN` con código desconocido o mal formado) en 60 s
  y el relay deja de atender a esa IP hasta que acabe la ventana.
- **SIGHUP** relee el archivo: las claves nuevas entran, las borradas dejan de poder hacer `HOST`
  (las sesiones en curso terminan). Un archivo con errores no se aplica: siguen las de antes.
- **Métricas:** `rayrelay_authorized_keys`, `rayrelay_auth_failures_total`,
  `rayrelay_quota_rejections_total`, `rayrelay_blocked_connections_total` y
  `rayrelay_user_bytes_total{user}`.
- **Sin TLS** (todavía) se ven en la red el key-id, el código y lo que las apps manden en claro;
  la clave privada y el contenido E2E no.

Un relay abierto acepta `HOST <key-id>` como `HOST` a secas, así que un cliente con clave
funciona contra los dos.

## Arquitectura (CSP)

Todo el estado compartido (tabla código→host en espera, métricas) vive en **un
actor** (`src/registry.ray`); cada conexión es una fibra que le habla por canal.
El traspaso del socket del joiner al host viaja por el canal `notify` del propio
host (handle + bytes tempranos), el timeout de espera es una fibra-temporizador
que envía en ese mismo canal, y cada dirección del pipe es una fibra `pump` con
contrapresión natural (write bloqueante). Los clientes `host`/`join` son el
patrón netcat: dos fibras lectoras alimentan canales y `main` decide con
`select`. Al llegar EOF de stdin el cliente hace half-close
(`net.shutdown_write`) y sigue drenando hasta que el otro lado cierre; el relay
propaga ese half-close al peer (cada `pump` reenvía el EOF como `SHUT_WR`), así
que el patrón "envío, cierro mi escritura, leo la respuesta" funciona extremo a
extremo. Los sockets del par van con `TCP_NODELAY` (sesión interactiva) y
`SO_KEEPALIVE`.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Rendezvous TCP con códigos cortos + relay ciego bidireccional | ✅ |
| Pipelining (bytes pegados al JOIN/HOST no se pierden) | ✅ |
| Timeouts: handshake 30 s, espera de peer 10 min, idle configurable | ✅ |
| STUN-lite UDP (`WHOAMI` → dirección observada) + `probe` | ✅ |
| Métricas Prometheus + logs JSON estructurados (`net/log`) | ✅ |
| Clientes de prueba `host`/`join` (netcat sobre el relay) | ✅ |
| Apagado por señales (`signals()`) | ✅ |
| Binario nativo + VM (E2E verificado en ambos) | ✅ |
| Half-close (`SHUT_WR`) propagado por el relay + `TCP_NODELAY`/keepalive | ✅ |
| Autorización por claves Ed25519 con desafío, límites por clave y por IP, recarga con SIGHUP | ✅ |
| Tests (códigos, lineio sobre sockets reales, roundtrip y half-close con relay vivo, timeout de `probe`, autorización) | ✅ 24 |
| Hole punching UDP coordinado (intercambio de ADDR entre peers vía relay) | 📋 v2 |
| Integración takeit (`--relay`, desde takeit 0.5.0) | ✅ |
| Integración msg | 📋 v2 |
| TLS del transporte al relay | 📋 opcional |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Esta app existe para operar sockets de larga vida con fibras — y destapó una
cosecha grande (todo anotado en `raylang/IDEAS.md` §64, con repros mínimos):

1. **[RESUELTO — raylang PR #140]** Nativo: `close(h)` con un lector aparcado
   era un no-op silencioso. Ahora hace `shutdown(Both)` y el lector despierta
   con `Err(invalid handle)` byte-idéntico a la VM — el idiom "cierro para
   despertar al otro pump" ES portable ya, y rayrelay lo usa: ante un error
   o el timeout de inactividad, cerrar ambos sockets despierta al otro pump.
2. **[RESUELTO — raylang PR #140]** Nativo: `Variant(b.campo, f(b), …)` con
   `f` mutando `b` panicaba (`RefCell already borrowed`) — los args de
   literales compuestos se izan a temporales; el workaround manual sobra.
3. **[RESUELTO — raylang M121]** UDP: la nota "bloquea todas las fibras"
   resultó rancia (la VM cede desde M20.11 y el nativo-fibras desde F4), y el
   hueco real — el timeout — está cerrado: `net.set_read_timeout` aplica a
   UDP y `probe` falla con `Err("read timeout")` a los 3 s en vez de colgarse.
   `stun` sigue siendo un proceso aparte, ahora solo por despliegue.
4. **[RESUELTO — raylang M130]** No había `shutdown(SHUT_WR)`: con
   `net.shutdown_write` el cliente ya no usa el periodo de gracia de 2 s tras
   EOF de stdin, y el relay propaga el half-close en vez de tirar el par.
5. **[RESUELTO — raylang M116/M116.1]** No hay `try_recv` ni select con
   timeout: existen (`try_recv` → Got/Empty/Closed y `select_timeout`);
   solo queda el select de tipos mixtos (se enum-ifica en userland).
6. Menores: `exit(code)` ya existe (M130); el error de timeout de lectura
   sigue distinguiéndose solo por string (`"read timeout"`, estable).

Los repros mínimos de `docs/repros/` se re-ejecutaron con raylang 1.27.11
(septiembre 2026): los ocho dan salida idéntica en VM y en binario nativo
(`close` cruzado despierta al lector aparcado, `Variant(b.campo, f(b), …)`
ya no panica, el timeout de lectura es `"read timeout"` en ambos). Se
conservan como tests de regresión históricos:
`ray run docs/repros/<x>.ray` y `ray build --native docs/repros/<x>.ray`.

Lo que SÍ estuvo a la altura: el patrón actor + canales-en-mensajes (traspaso
de sockets entre fibras incluido) funciona idéntico en VM y nativo; `signals()`
compone limpio; `net/metrics` y `net/log` listos para operar.

## Desarrollo

Requiere raylang ≥ 1.27.65 (`[package] raylang`); la dependencia `net` viene del índice de paquetes
(`net = "^0.3.7"` en `ray.toml`, versión exacta fijada en `ray.lock`).

```sh
ray test                                   # 24 tests
ray run src/main.ray serve --port 17500    # servidor en la VM
ray build --native src/main.ray -o rayrelay --release
```

Estructura: `src/main.ray` (CLI) · `codes.ray` (códigos) · `lineio.ray`
(líneas + leftover sobre sockets) · `registry.ray` (actor de estado) ·
`relay.ray` (rendezvous + pumps + /metrics) · `auth.ray` (claves, desafío, límites) ·
`stun.ray` (UDP) · `client.ray` (netcat de prueba).
