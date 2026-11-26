# mercadito-shop: diagnóstico y recomendaciones

Ejercicio DevOps, 25 y 26 de noviembre de 2026. Clúster k3s de 3 nodos (cp-1 con taint de control plane,
worker-1 y worker-2 de 2 CPU cada uno), namespace `marcos-shop`, tráfico generado por k6.

Todas las horas están en UTC. Cada hallazgo indica síntoma, evidencia, causa, corrección y resultado medido.
Las correcciones son solo configuración y objetos de Kubernetes: el código de la aplicación no se tocó.

## Resumen

| # | Problema | Corrección | Estado |
|---|---|---|---|
| 1 | Carritos en memoria: con 2 réplicas cada pod tenía los suyos | `CART_STORE: redis` | Verificado |
| 2 | catalog satura la CPU: recomendaciones lentas, 502 en páginas de producto, timeouts | Proxy de caché `catalog-cache` delante de catalog | Verificado: página de producto p50 15 ms, 0 errores |
| 3 | Caché de miniaturas sin límite y anchos arbitrarios: cualquiera puede agotar la memoria de catalog | `THUMB_CACHE_MAX_MB: 64`; el proxy cachea los anchos de la tienda y limita a 4 simultáneas las generaciones de anchos poco comunes | Verificado: todo ancho válido se sirve; límite activo |
| 4 | checkout retiene conexión de BD y bloqueos de stock mientras espera el pago (timeout 30 s, 2 conexiones, sin reintentos) | Timeout 2 s, 2 reintentos, 6 conexiones (dentro del límite de Postgres) | Aplicado; latente hasta que se inyecten fallos de pago |
| 5 | Reparto entre nodos roto tras cada rollout (los 2 pods de cart en el mismo nodo) | `matchLabelKeys: [pod-template-hash]` | Verificado: un pod por worker en cada servicio |
| 6 | Un nodo caído tarda ~5 min en reemplazar sus pods | `tolerationSeconds: 30` | Aplicado |
| 7 | Sitio solo por HTTP: cookies y token de `/admin` en texto plano | Certificado de Let's Encrypt en el Ingress | Verificado |
| 8 | `/admin/faults` accesible desde Internet | Bloqueo listo en `deploy/optional/`, no aplicado (ver 4.2) | Documentado |
| 9 | Cuota de CPU agotada por un request mal dimensionado (error propio) | Request de catalog 500m → 300m | Verificado |

## 1. Pipeline (`.github/workflows/ci-cd.yml`)

1. **Tests:** `go vet`, tests unitarios y de integración contra un Postgres de servicio.
2. **Imágenes:** una por servicio (5), etiquetadas con el SHA del commit, publicadas en GHCR con `GITHUB_TOKEN`.
3. **Despliegue:** Job de migración → `kubectl apply -k deploy/prod` (el pipeline agrega un bloque `images:`
   con el SHA) → `rollout status` de cada servicio → prueba de humo por HTTPS contra el host público.

Decisiones:

- **No se usa `make ci`.** Su objetivo `hygiene` falla si existe `.github/`: es una verificación de los autores
  del ejercicio para no dejar pistas en el repositorio. Se ejecutan `go vet` y los tests directamente.
- **Los despliegues no se cancelan entre sí** (`concurrency` sin `cancel-in-progress`): cancelar un
  `kubectl apply` a medias puede dejar el clúster con servicios en versiones mezcladas.
- **Imágenes privadas.** Los paquetes de GHCR heredan la visibilidad del fork (público). Se pasaron a privados
  y los pods descargan con el Secret `ghcr-pull` (token `read:packages`, creado fuera del pipeline). Verificado:
  un pull anónimo devuelve 403.
- **La prueba de humo usa HTTPS sin `-k`:** si el certificado deja de ser válido, el despliegue falla.

### Seguridad del pipeline

Endurecimiento aplicado al final (commit `e5b5c61`, 05:18), verificado con actionlint y con un despliegue completo:

- **Acciones de terceros fijadas por SHA** (con la versión en un comentario). Este repositorio no tiene Dependabot, así que esos SHA se actualizan a mano. Una etiqueta como `@v4` se puede
  mover; si el repositorio de la acción se viera comprometido, el código nuevo correría junto a las credenciales
  del clúster.
- **Secretos por `env:`** en los pasos de shell, en lugar de interpolarlos en el script.
- **`permissions: contents: read`** declarado arriba; solo el job que publica imágenes agrega `packages: write`.

Auditoría (los repositorios son públicos, así que también lo son los logs de Actions):

- Ningún secreto en el historial de git (patrones de tokens, claves privadas y campos de kubeconfig: 0).
- Logs de despliegue revisados: los tokens aparecen enmascarados (`***`) y el kubeconfig decodificado nunca se
  imprime.
- Los despliegues no corren en PR ni desde forks ajenos, y no se usa `pull_request_target`: un PR malicioso no
  alcanza los secretos del clúster.
- El `GITHUB_TOKEN` por defecto del repositorio es de solo lectura; los kubeconfig de CI solo actúan en su
  namespace.

## 2. Lista para producción

| Requisito | Implementación |
|---|---|
| Requests y limits | Por servicio. Sin límites de CPU (ver 3.2: un límite produjo throttling del 78 %). Límites de memoria en todos. |
| Probes | Readiness en `/readyz` (cubre el warmup de 45 s de catalog y el drenaje al apagar); liveness en `/healthz` (siempre 200: nunca mata un pod que aún calienta). Timeout de readiness de catalog: 3 s (`/readyz` hace ping a Postgres). |
| Autoescalado | HPA en storefront y catalog (2–4 réplicas, CPU al 70 %). |
| Pérdida de un nodo | 2+ réplicas por servicio; `topologySpreadConstraints` obligatorio (`DoNotSchedule`) entre worker-1 y worker-2 con `nodeTaintsPolicy: Honor` (sin esto cp-1 cuenta como dominio vacío y una tercera réplica no se puede programar) y `matchLabelKeys: [pod-template-hash]`; PDB `minAvailable: 1`; `tolerationSeconds: 30`. |
| Despliegues seguros | `maxUnavailable: 0`, `maxSurge: 1`, `SHUTDOWN_DRAIN_SECONDS: 10` (la app deja de estar lista y termina las peticiones en curso antes de apagarse). |
| Seguridad | UID numérico (la imagen distroless usa el usuario `nonroot`, un nombre que `runAsNonRoot` no puede verificar), raíz de solo lectura, todas las capabilities eliminadas. |

**Evidencia:** durante un reinicio completo de todos los deployments, 174/174 peticiones a la tienda
devolvieron 200. Borrando a la vez todos los pods de un worker, la home siguió respondiendo al 100 %.

**Límite:** el kubeconfig del ejercicio no puede hacer drain ni cordon, así que la pérdida real de un nodo no se
pudo probar; se simuló borrando pods (ver 3.6).

## 3. Problemas encontrados bajo tráfico

### 3.0 Contexto

- k6 (namespace `ops`) envió olas de ~5 min con ~4 min de silencio hasta las 03:06, y carga continua de
  50–75 peticiones/s desde las 03:08. k6 solo navega: no compra.
- El entorno inyectó `ADMIN_TOKEN` y `REDIS_URL` como valores literales en nuestros Deployments
  (`kubectl set env`, 00:50). Es el inyector de fallos del ejercicio (`/admin/faults`: `cpu_burn`,
  `thumb_burst`, latencia y fallos de pagos); Redis comparte esos interruptores entre réplicas. Nuestros
  manifiestos no declaran esas variables, así `kubectl apply` no las toca y no hay conflicto.

### 3.1 Carritos en memoria

- **Configuración original:** `CART_STORE: memory`. Con 2 réplicas, cada pod guarda sus propios carritos: los
  productos aparecen y desaparecen según el pod que responda.
- **Corrección:** `CART_STORE: redis` (la URL la inyecta el entorno y coincide con el Secret `shop-redis`,
  comparado por hash).
- **Verificado con una compra real en el navegador:** las lecturas del carrito las atendieron los dos pods de
  cart y siempre devolvieron los mismos 3 productos; el pedido se creó sin errores.

### 3.2 catalog satura la CPU

- **Síntoma:** páginas de producto con p50 de 3,7–5,9 s y p95 de 11–15 s; 2–17 % de 502; timeouts en el
  navegador.
- **Medición (Grafana y logs):** 17,4 recomendaciones/s; catalog consume ~2,5 núcleos → **~144 ms de CPU por
  recomendación** (el código dice 40–80 ms, calibrado en otra máquina). Tiempo medio por recomendación 3,8 s,
  el 96 % en espera. Con carga, cada pod de catalog usa ~0,9 núcleos y hay 2 por worker.
- **Causa:**
  1. Las recomendaciones son caras y la demanda (~2,5 núcleos) se acerca a la capacidad de los dos workers.
  2. Con llegadas en ráfaga se forman colas; las que superan los 10 s de storefront se convierten en 502.
  3. catalog sigue calculando las recomendaciones abandonadas (el código no cancela): CPU desperdiciada.
  4. El detalle de producto (`/products/{id}`, ~1 ms) espera su turno de CPU detrás de las recomendaciones; al
     cancelarlo storefront, `storeError` lo registra como 500 aunque no sea un fallo de la base de datos.
- **Intentos que no funcionaron (y se revirtieron):**
  - Límite de 1 CPU por pod con HPA máx. 3: throttling del 78 % en un pod, página de producto más lenta
    (p95 11 s → 15 s). Revertido.
  - Request de 500m para reflejar el uso real: agotó la cuota (ver 3.7). Bajado a 300m.
- **Corrección final:** proxy de caché `catalog-cache` (ver 3.4).
- **Resultado:** medido de 03:56 a 04:01, 5 min de carga continua, tras el despliegue del proxy.

  | | Antes de la caché | Después |
  |---|---|---|
  | Página de producto | p50 3,7–5,9 s, p95 11–15 s | **p50 15 ms, p95 79 ms** |
  | Home y búsqueda | p95 ~0,8 s | **p95 55–61 ms** |
  | Errores 502 | 2–17 % | **0 de ~31 000 peticiones** |
  | Recomendaciones calculadas por catalog | ~17/s | **0** (todas desde la caché) |
  | CPU de catalog | ~0,9 núcleos por pod | ~1m por pod; el HPA bajó a 2 réplicas |
  | Carga atendida | ~17 páginas de producto/s | **~34/s** (k6 recibe respuestas más rápidas y envía más) |

  Residual: casos aislados de hasta ~5 s (muy por debajo del 1 %, dado un p95 de 79 ms).

### 3.3 Caché de miniaturas sin límite

- **Configuración original:** `THUMB_CACHE_MAX_MB: "0"` = sin límite. La clave es (producto, ancho) y se acepta
  cualquier ancho entre 16 y 2048: 500 × 2033 combinaciones, accesibles desde Internet en `/img/{id}?w=`.
- **Medición:** una miniatura de 2048 px pesa 2,6 MB y tarda 11 s en generarse.
- **Prueba controlada (sin llegar a OOM, para no tumbar pods con tráfico real):** 20 miniaturas distintas de
  ~2048 px subieron la caché 50 MB sin desalojar nada y la memoria de los pods de ~16 a ~120 MiB (la generación
  usa además un búfer temporal de ~12 MB por imagen). Con ~40–50 por pod se alcanzaría el límite de 384 MiB.
- **Corrección:**
  - `THUMB_CACHE_MAX_MB: "64"`: la caché de catalog desaloja lo menos usado y deja de crecer sin límite.
  - En el proxy, los anchos que usan las plantillas (160, 240, 480, o sin ancho) se cachean.
  - Cualquier otro ancho válido **se sirve igual**, sin caché, pero con como máximo 4 generaciones simultáneas
    por pod del proxy (8 en total); el exceso recibe 503. Así un enlace válido con un ancho poco común funciona,
    y una ráfaga de miniaturas grandes no agota la CPU ni la memoria de catalog: 8 generaciones de ~12 MB
    repartidas entre los pods de catalog, más 64 MB de caché, quedan muy por debajo de 384 MiB.
- **Primera versión descartada:** rechazar con 400 los anchos que no usa la tienda. Protegía igual, pero rompía
  enlaces válidos; se reemplazó por el límite de concurrencia.
- **Cuánto aporta cachear imágenes en el proxy:** poco. catalog ya sirve sus miniaturas cacheadas en ~8 ms
  (p50) y k6 casi no pide imágenes (~40 en 5 minutos frente a ~10 000 páginas de producto). La mejora de
  rendimiento viene de las recomendaciones; para las imágenes, el valor del proxy es la protección.
- **Resultado:** verificado en el clúster (05:06, commit `5b38727`). Por la tienda pública: `w=2048` → 200 (2,6 MB, 11 s, generación nueva), `w=37` y `w=480` → 200; `w=99999` → 400 de catalog (fuera de 16–2048); producto inexistente → 404. En el proxy: `w=480` → `MISS` y luego `HIT`; `w=777` → `BYPASS` las dos veces. Con 10 anchos poco comunes simultáneos contra el catalog real, 7 → 200 y 3 → 503: el límite solo corta cuando hay más de 4 generándose a la vez en ese pod (las que terminan rápido liberan su lugar).
- **Mantenimiento:** si las plantillas de la tienda agregan un ancho nuevo, conviene agregarlo a la regla del proxy; si no, esas imágenes se servirían sin caché y sujetas al límite de 4 simultáneas.

### 3.4 Proxy de caché `catalog-cache`

- nginx entre storefront y catalog (`CATALOG_URL: http://catalog-cache:8081`). Configuración en
  `deploy/base/catalog-cache.yaml`.
- **Recomendaciones:** caché de 10 min; `proxy_cache_lock` para que peticiones simultáneas del mismo producto
  disparen un solo cálculo. Son deterministas: como máximo 500 cálculos por réplica del proxy.
- **Miniaturas:** los anchos de la tienda se cachean 60 min; los demás se sirven sin caché, máximo 4 a la vez
  por pod (`limit_conn`), 503 por encima.
- **Sin caché:** detalle de producto (lleva stock en vivo), búsqueda y listados.
- **Por qué no en Redis:** el Redis del namespace tiene 128 MiB y guarda carritos e interruptores de fallos;
  llenarlo con imágenes pondría en riesgo los carritos. nginx cachea en su propio `emptyDir` (máx. 100 MB).
- **Verificado antes de cada despliegue:** `nginx -t` con la imagen real en solo lectura; pruebas funcionales
  contra un catalog falso (lento a propósito, para ejercitar el límite) y contra el real por port-forward.
- **Sin corte al desplegar:** el proxy se creó y se esperó a que estuviera listo antes de que storefront
  apuntara a él.

### 3.5 checkout y pagos: trampa latente

- **Código:** checkout abre una transacción, descuenta stock con `UPDATE` (bloquea esas filas), inserta el
  pedido y **llama al proveedor de pagos con la transacción abierta**; hace commit al responder el pago.
- **Configuración original:** `PAYMENT_TIMEOUT_MS: 30000`, `DB_MAX_CONNS: 2`, `PAYMENT_RETRIES: 0`. Con pagos
  lentos, 4 pagos bloquean todo checkout; storefront abandona a los 10 s mientras checkout retiene conexión y
  bloqueos 20 s más (el cobro puede ocurrir sin que el cliente vea la confirmación); un fallo transitorio
  cancela la compra.
- **Por qué no se veía:** con pagos a 50 ms todo fluye. Ráfaga de 20 compras concurrentes: 20/20 pedidos,
  checkout 82–211 ms. Aparece en cuanto el entorno inyecte latencia o fallos de pago.
- **Presupuesto de conexiones (medido):** Postgres permite 50 (47 utilizables); 26 en uso (catalog 4 por pod,
  checkout 2 por pod, internas ~6). Peor caso durante un despliegue: catalog 5 × 4 + checkout 3 × X + 6 ≤ 47 →
  X ≤ 7. Los 10 propuestos inicialmente podían agotar Postgres en un despliegue.
- **Corrección:** `PAYMENT_TIMEOUT_MS: 2000`, `PAYMENT_RETRIES: 2` (peor caso ≈ 6,3 s, por debajo de los 10 s
  de storefront), `DB_MAX_CONNS: 6` (peor caso 44 de 47).
- **Riesgo de los reintentos:** payments-mock no deduplica por `order_ref`; con un proveedor real, reintentar
  un cobro expirado que sí se procesó podría cobrar dos veces. Por eso solo 2.
- **Hallazgo menor:** los 20 cobros de la ráfaga fueron a un solo pod de payments-mock (keep-alive + balanceo
  por conexión del Service). En parte azar (~1/8 con 3–4 conexiones). No se cambia: un balanceador L7 interno
  (Traefik) expondría el servicio de pagos, y una service mesh requiere permisos de clúster.

### 3.6 Pérdida de un nodo

- **Hallazgo:** los dos pods de cart estaban en worker-2 y los dos de payments-mock en worker-1. La regla de
  reparto contaba también los pods de la versión anterior durante cada rollout.
- **Simulación:** se borraron a la vez los 8 pods de aplicación de worker-2 con tráfico: la home respondió 45/45.
  Matiz: Kubernetes los recreó en segundos porque el nodo seguía vivo; con el nodo realmente caído, sus pods
  habrían quedado ~5 min sin reemplazo y cart, con ambos pods allí, caído.
- **Corrección:** `matchLabelKeys: [pod-template-hash]` y `tolerationSeconds: 30` en todos los Deployments
  (parche común en `deploy/prod/kustomization.yaml`).
- **Resultado:** tras el despliegue, cart, checkout, payments-mock y catalog-cache tienen un pod en cada worker; catalog y storefront, dos y dos.

### 3.7 Cuota de CPU agotada (error propio)

- **Síntoma:** un despliegue falló: el rollout de catalog quedó en "1 de 4" 5 minutos. El HPA oscilaba y
  storefront no podía crear su cuarta réplica.
- **Evidencia:** `exceeded quota: candidate-quota, requested: requests.cpu=500m, used: requests.cpu=3,
  limited: requests.cpu=3`.
- **Causa:** al subir el request de catalog a 500m no se tuvo en cuenta la cuota del namespace (3 CPU en
  requests, compartida por todos los servicios). El enunciado lo advertía.
- **Corrección:** 300m; peor caso con rollouts ~2,4 de 3. Sin errores de cuota desde las 03:04.

## 4. Riesgos y recomendaciones

### 4.1 Fuera de nuestro alcance (código o permisos de clúster)

1. **Recomendaciones:** cachearlas en la aplicación y cancelar el trabajo cuando el cliente se va.
2. **Errores 500 falsos:** distinguir la cancelación del cliente (499) de los fallos reales.
3. **Pagos:** no llamar al proveedor con la transacción abierta (reservar, confirmar, cobrar, compensar) y exigir
   una clave de idempotencia (`order_ref`).
4. **Miniaturas:** limitar los anchos en la propia app y la concurrencia de generación.
5. **Redirección HTTP → HTTPS:** requiere un Middleware de Traefik; no tenemos permiso para crearlo.
6. **Renovación del certificado:** se emitió a mano (no hay cert-manager); vence el 2027-02-24.
7. **Postgres y Redis únicos, ambos en cp-1:** si cp-1 cae, se pierden carritos y base de datos.
8. **Capacidad:** los workers (2 × 2 CPU) quedan justos para catalog; la cuota (3 CPU) también.

### 4.2 `/admin` expuesto y credenciales en texto plano

- `/admin/faults` es accesible desde Internet a través del Ingress, protegido solo por `ADMIN_TOKEN`.
- `ADMIN_TOKEN` y `REDIS_URL` (con la contraseña de Redis) son valores literales en el Deployment: los ve
  cualquiera con permiso de lectura sobre Deployments, sin necesitar permiso sobre Secrets.
- **No se corrigió a propósito:** los gestiona el entorno del ejercicio, y bloquear `/admin` podría romper la
  inyección de fallos si el entorno la hace por la URL pública. El bloqueo está listo en
  `deploy/optional/block-admin-ingress.yaml` (validado con `--dry-run=server`) con instrucciones para activarlo.
- **Recomendación:** mover ambos valores a un Secret con `secretKeyRef` y activar el bloqueo.

## 5. Proceso y lecciones

- El primer diagnóstico de catalog se hizo solo con `kubectl` y logs; Grafana se revisó después. A partir de ahí
  cada hallazgo se validó primero en el dashboard `mercadito-shop`.
- Dos cambios no funcionaron (límite de CPU y request de 500m) y se revirtieron con la medición que lo mostraba.
- Probar la API con `curl` no basta: los problemas más graves de snapshare solo aparecieron usando la app en
  el navegador.
