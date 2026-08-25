# rayrelay

Rendezvous + relay **ciego** para [takeit](../takeit) y [msg](../msg), escrito en [raylang](https://github.com/roberto-ayala/raylang): resuelve el pendiente compartido de ambas apps (dos peers detrás de NAT que no pueden verse) con códigos cortos de emparejamiento, un relay TCP que copia bytes sin mirarlos, y un respondedor STUN-lite por UDP para intentar hole punching. El relay nunca ve claro: las apps de encima ya cifran de extremo a extremo.

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

Los bytes que un cliente mande pegados a su línea de control (pipelining) se
reenvían al otro lado tras el emparejamiento — no se pierden. Los códigos
(`xxxx-xxxx`, alfabeto sin caracteres confundibles) rutan la sesión, no son
secretos: la confidencialidad es de la capa de arriba.

## Uso

```text
rayrelay serve [--bind H] [--port N] [--idle SECS] [--metrics-port N]
rayrelay stun  [--bind H] [--port N]
rayrelay probe [--server H] [--port N]
rayrelay host  [--server H] [--port N]
rayrelay join CODE [--server H] [--port N]
```

- `--idle`: corte por inactividad por dirección durante el relay (default 300 s;
  `0` = sin límite, con un suelo de 1 h en nativo — ver hallazgos).
- `--metrics-port`: endpoint Prometheus (`/metrics`): conexiones, pares
  activos/total, bytes relayados, códigos en espera, JOIN fallidos.
- Espera máxima de un host por su peer: 10 min. Handshake: 30 s.
- Apagado: SIGTERM/SIGINT vía `signals()`; los pares vivos caen con el proceso.

## Arquitectura (CSP)

Todo el estado compartido (tabla código→host en espera, métricas) vive en **un
actor** (`src/registry.ray`); cada conexión es una fibra que le habla por canal.
El traspaso del socket del joiner al host viaja por el canal `notify` del propio
host (handle + bytes tempranos), el timeout de espera es una fibra-temporizador
que envía en ese mismo canal, y cada dirección del pipe es una fibra `pump` con
contrapresión natural (write bloqueante). Los clientes `host`/`join` son el
patrón netcat: dos fibras lectoras alimentan canales y `main` decide con
`select`.

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
| Tests (códigos, lineio sobre sockets reales, roundtrip con relay vivo) | ✅ 10 |
| Hole punching UDP coordinado (intercambio de ADDR entre peers vía relay) | 📋 v2 |
| Integración takeit (`--relay`) y msg | 📋 v2 |
| TLS del transporte al relay | 📋 opcional |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Esta app existe para operar sockets de larga vida con fibras — y destapó una
cosecha grande (todo anotado en `raylang/IDEAS.md` §64, con repros mínimos):

1. **[RESUELTO — raylang PR #140]** Nativo: `close(h)` con un lector aparcado
   era un no-op silencioso. Ahora hace `shutdown(Both)` y el lector despierta
   con `Err(invalid handle)` byte-idéntico a la VM — el idiom "cierro para
   despertar al otro pump" ES portable ya (rayrelay conserva su esquema
   cierra-tu-propio-src, que también es correcto).
2. **[RESUELTO — raylang PR #140]** Nativo: `Variant(b.campo, f(b), …)` con
   `f` mutando `b` panicaba (`RefCell already borrowed`) — los args de
   literales compuestos se izan a temporales; el workaround manual sobra.
3. **[PARCIALMENTE RESUELTO — raylang M121]** UDP: la nota "bloquea todas
   las fibras" resultó rancia (la VM cede desde M20.11 y el nativo-fibras
   desde F4), y el hueco real — el timeout — está cerrado:
   `net.set_read_timeout` aplica a UDP y un datagrama perdido es
   `Err("read timeout")`, no un `probe` colgado.
4. **No hay `shutdown(SHUT_WR)`** (half-close): el idiom netcat "aviso EOF y
   dreno hasta que el peer cierre" no se puede expresar; el cliente usa un
   periodo de gracia de 2 s tras EOF de stdin.
5. **[RESUELTO — raylang M116/M116.1]** No hay `try_recv` ni select con
   timeout: existen (`try_recv` → Got/Empty/Closed y `select_timeout`);
   solo queda el select de tipos mixtos (se enum-ifica en userland).
6. Menores: no hay `exit(code)` para terminar desde una fibra auxiliar; el
   error de timeout de lectura solo se distingue por string (`"read timeout"`).

Lo que SÍ estuvo a la altura: el patrón actor + canales-en-mensajes (traspaso
de sockets entre fibras incluido) funciona idéntico en VM y nativo; `signals()`
compone limpio; `net/metrics` y `net/log` listos para operar.

## Desarrollo

```sh
ray test                                   # 10 tests
ray run src/main.ray serve --port 17500    # servidor en la VM
ray build --native src/main.ray -o rayrelay --release
```

Estructura: `src/main.ray` (CLI) · `codes.ray` (códigos) · `lineio.ray`
(líneas + leftover sobre sockets) · `registry.ray` (actor de estado) ·
`relay.ray` (rendezvous + pumps + /metrics) · `stun.ray` (UDP) · `client.ray`
(netcat de prueba).
