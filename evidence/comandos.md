# Registro de comandos — Laboratorio 3

Equipo: Christian Navarro, Emmanuel Calvo, Jimmy Feng. Horas en UTC−6 (Costa Rica), 3-nov-2026.
Las horas de cada paso provienen de la hora de modificación de los archivos de `evidence/` (`ls -l --time-style`) y de las marcas de las salidas.

## Paso 0 — preparar
```bash
docker compose version
make build
mkdir -p evidence
printf '# Registro de comandos — Laboratorio 3\n\n' > evidence/comandos.md
```
Resultado: sesión iniciada a las 12:15; el primer `make build` falló (Docker daemon aún no iniciado: `failed to connect to the docker API`); tras iniciar Docker Desktop, `make build` terminó con `Image ti4601:local Built`. Terminado antes de las 12:27. **Paso 0: OK**

## Paso 1 — levantar y estado inicial
```bash
date --iso-8601=seconds | tee evidence/cluster-start.txt
make lab3-up
make lab3-status | tee -a evidence/cluster-start.txt
```
Resultado: 3 contenedores `ti4601-mongo-N` sanos; nodos arrancados a las 12:27:54. Evidencia: `cluster-start.txt` (12:28:03). **Paso 1: OK**

## Paso 2 — `rs.initiate()`
```bash
make lab3-shell
```
```javascript
rs.initiate({ _id: "rs-lab3", members: [
  { _id: 0, host: "mongo-1:27017" }, { _id: 1, host: "mongo-2:27017" }, { _id: 2, host: "mongo-3:27017" } ] })
```
Resultado: `{ ok: 1 }` a las 12:29:58; `make lab3-status` → `mongo-1 PRIMARY`, `mongo-2/3 SECONDARY`. **Paso 2: OK**

## Paso 3 — inspección
```bash
docker exec ti4601-mongo-1 mongosh --quiet --eval '...' | tee evidence/rs-inspect.txt
```
Resultado: `term=1`, `electionTimeoutMillis=10000`, `heartbeatIntervalMillis=2000`, `votes=1`/`priority=1` en los 3; ambos secundarios con `syncSource: mongo-1:27017`. Hora: 12:34:55. **Paso 3: OK**

## Paso 4 — semilla
```bash
docker compose --profile lab3 run --rm --no-deps app-mongo python3 labs/lab3-mongo-replicas/generar_datos.py | tee evidence/datos-manifest.txt
```
Nota: ejecutado primero en el host falló (`No module named 'pymongo'`); se repitió en el contenedor. Resultado: `semilla 2026`, 400 eventos, 1189 zonas, `capacidad_total 162810`, `sha256 9fc6a2db…4774a`. Hora: 12:37:02. **Paso 4: OK**

## Paso 5 — carga e índices (TODO 1 y 2)
```bash
docker compose --profile lab3 run --rm --no-deps app-mongo python3 labs/lab3-mongo-replicas/cargar.py
set -o pipefail
make lab3-check | tee evidence/config-check.txt
docker exec ti4601-mongo-3 mongosh --quiet --eval '... explain("executionStats") ...' | tee evidence/explain-indices.txt
```
Resultado: 5 líneas `OK:`, carga 0.31 s (1276 docs/s); `lab3-check` → **6/6** (12:38:14); `explain`: con índice 67 docs examinados (`IXSCAN`) frente a 400 sin índice (`COLLSCAN`). `explain-indices.txt` se generó a las 13:31:39, con `mongo-3` como primario. **Paso 5: OK**

## Paso 6 — oplog
```bash
docker exec ti4601-mongo-3 mongosh --quiet --eval '... insertOne ORD-MANUAL-2 ... oplog.rs ... printSecondaryReplicationInfo()' | tee evidence/oplog.txt
```
Resultado: `ORD-MANUAL-1` (12:39:11, `t:1`) y `ORD-MANUAL-2` (13:32:38, `t:3`) en el oplog; retraso 0 s y 6 s en dos mediciones. Escritura en un secundario (`insertOne` en `mongo-1`/`mongo-2`): `NotWritablePrimary: not primary` (`escritura-secundario.txt`, 14:36:36). **Paso 6: OK**

## Paso 7 — lecturas obsoletas
```bash
docker compose --profile lab3 run --rm --no-deps app-mongo python3 labs/lab3-mongo-replicas/lectura_secundario.py --n 500 --csv evidence/lectura-secundario.csv | tee evidence/lectura-secundario.txt
```
Resultado: `w1_secundario` 96.0 %, `majority_secundario` 0.6 %, `causal_majority` 0.0 %. Hora: 12:42:28. **Paso 7: OK**

## Paso 8 — compra atómica (TODO 3)
```bash
docker compose --profile lab3 run --rm --no-deps app-mongo python3 labs/lab3-mongo-replicas/comprar.py | tee evidence/comprar.txt
```
Resultado: 6 líneas `OK:` y `todas las verificaciones pasaron` (12:43:28). Versión rota a propósito (14:52:18): `... comprar.py | tee evidence/comprar-roto.txt` → `confirmadas=200, agotadas=0`, `quedan 116`, VIP `3 confirmadas, quedan 6`; restaurada con `cp comprar_correcta.py comprar.py` y repetida (14:53:52) → `todas las verificaciones pasaron`. **Paso 8: OK**

## Paso 9 — write concern (TODO 4)
```bash
docker compose --profile lab3 run --rm --no-deps app-mongo python3 labs/lab3-mongo-replicas/bench_write_concern.py --n 300 --csv evidence/write-concern.csv | tee evidence/write-concern.txt
wc -l evidence/write-concern.csv
```
Resultado: p50 en ms: `w1` 6.65, `w1_journal` 28.78, `majority` 23.74, `w3` 12.89; CSV con **1201** líneas. Hora: 12:44:56. **Paso 9: OK**

## Paso 10 — RPO (TODO 5) y línea base
```bash
... verificar_rpo.py --autotest
... sonda_escrituras.py --duration 10 --label "Base" --csv evidence/rpo-baseline.csv | tail -5
... verificar_rpo.py --csv evidence/rpo-baseline.csv | tee evidence/rpo-baseline.txt
```
Resultado: 3 casos sintéticos `OK`; base: 20/20 confirmados, **PERDIDOS 0**. Hora: 12:47:02 (sonda) y 12:47:29 (`rpo-baseline.txt`). **Paso 10: OK**

## Paso 11 — Falla A (cae `mongo-2`, secundario)
```bash
make lab3-status | tee evidence/falla-a-before.txt
# Terminal A: sonda --duration 40 --label "Falla A" --signal-file evidence/falla-a-stop.epoch --csv evidence/falla-a.csv
# Terminal B:
docker stop --timeout 0 ti4601-mongo-2
date +%s.%N > evidence/falla-a-stop.epoch
date --iso-8601=ns | tee evidence/falla-a-stop.txt      
make lab3-status NODE=mongo-1 | tee evidence/falla-a-status.txt
docker start ti4601-mongo-2
... verificar_rpo.py --csv evidence/falla-a.csv | tee evidence/falla-a-rpo.txt
```
Resultado: 79/79 confirmados, 0 errores, **PERDIDOS 0**; `mongo-2` volvió `SECONDARY` (esa recuperación quedó registrada en `falla-b-before.txt`, tomado justo después; no se generó `falla-a-recovered.txt`). **Paso 11: OK**

## Paso 12 — Falla B (cae `mongo-1`, primario)
```bash
make lab3-status | tee evidence/falla-b-before.txt
# Terminal A: sonda --duration 60 --label "Falla B" ... --csv evidence/falla-b.csv
# Terminal B:
docker kill ti4601-mongo-1
date +%s.%N > evidence/falla-b-stop.epoch
date --iso-8601=ns | tee evidence/falla-b-stop.txt     
sleep 15
make lab3-status NODE=mongo-2 | tee evidence/falla-b-status.txt
docker exec ti4601-mongo-2 mongosh --quiet --eval 'print("term=" + rs.status().term)' | tee -a evidence/falla-b-status.txt
sleep 15
docker start ti4601-mongo-1
make lab3-status | tee evidence/falla-b-recovered.txt
... verificar_rpo.py --csv evidence/falla-b.csv | tee evidence/falla-b-rpo.txt
```
Resultado: nuevo primario `mongo-3`, `term=2`; RTO 11.87 s; 96/99 confirmados, **PERDIDOS 0**; `mongo-1` regresó `SECONDARY`. **Paso 12: OK**

## Paso 13 — Falla C (caen los dos secundarios)
```bash
make lab3-status | tee evidence/falla-c-before.txt
# Terminal A: sonda --duration 90 --label "Falla C" --w majority --wtimeout-ms 2000 ... --csv evidence/falla-c.csv
# Terminal B:
docker kill ti4601-mongo-1 ti4601-mongo-2
date +%s.%N > evidence/falla-c-stop.epoch
date --iso-8601=ns | tee evidence/falla-c-stop.txt      
sleep 20
make lab3-status NODE=mongo-3 | tee evidence/falla-c-status.txt
sleep 10
docker start ti4601-mongo-1 ti4601-mongo-2
make lab3-status | tee evidence/falla-c-recovered.txt
... verificar_rpo.py --csv evidence/falla-c.csv | tee evidence/falla-c-rpo.txt
```
Resultado: `mongo-3` pasó a `SECONDARY`; RTO 42.59 s; 94 confirmados, 13 errores, **PERDIDOS 0**, **5 errores sí se escribieron** (seq 25–29). **Paso 13: OK**

## Estado final
`make lab3-status` → 1 `PRIMARY` (`mongo-3`) y 2 `SECONDARY` (`mongo-1`, `mongo-2`). Conjunto restaurado.