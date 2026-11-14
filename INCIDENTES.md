# Incidentes y Soluciones — Oracle Always Free VM

Registro acumulado de errores reales detectados en este proyecto, con su causa raíz y la solución aplicada. **Revisar antes de tocar cualquier cosa.**

Última actualización: 2026-11-13

---

## INC-001 — Registro Oracle rechaza tarjeta de débito

**Síntoma:** `Your transaction was declined` en el paso de verificación de pago del registro.

**Causa raíz:** Oracle (vía CyberSource) solo acepta tarjetas físicas de banco; el débito debe tener *credit facility*. Los débitos básicos peruanos no lo tienen, aunque tengan "compras por internet" habilitadas y saldo.

**Solución:** Registrar con **tarjeta de crédito** (Falabella). Nombre y dirección de facturación exactamente como figuran en el estado de cuenta del banco.

**Lección:** "Compras por internet" ≠ "compras internacionales" ≠ "credit facility". No repetir intentos con el mismo débito: cada rechazo suma señales de fraude.

---

## INC-002 — `Out of host capacity` al crear la VM ARM

**Síntoma:** `500 InternalError … Out of host capacity` en `SA-SAOPAULO-1-AD-1`.

**Causa raíz:** São Paulo tiene **una sola Availability Domain** y la capacidad de `VM.Standard.A1.Flex` está saturada. Reducir RAM (12→10→6 GB) NO resuelve una región completamente saturada.

**Solución (parcial):** Escalera de configuraciones `2 OCPU/6GB` → `1 OCPU/6GB`, y redimensionar después (A1.Flex permite cambiar shape con la instancia en RUNNING). Alternativa definitiva: `VM.Standard.E2.1.Micro` (AMD, Always Free, 2 instancias de 1 GB).

**Lección:** No insistir con multi-AD en Sudamérica. No confundir capacidad con credenciales/permisos.

---

## INC-003 — "Run exitoso" que nunca creó la VM

**Síntoma:** Runs marcados `success` en GitHub, pero cero instancias en Oracle.

**Causa raíz:** El workflow salía con `exit 0` al **encadenar** el siguiente run, no al crear la VM. El estado verde medía "el script terminó", no "la VM existe".

**Solución:** El workflow ahora imprime una línea `RESULT:` explícita (`VM_CREATED`, `VM_ALREADY_RUNNING`, `RETRY_LATER`, `THROTTLED`, `LIST_FAILED`) y solo reporta éxito tras confirmar `lifecycle-state = RUNNING`.

**Lección:** Un wrapper verde no es evidencia. Verificar siempre la instancia real (OCID + estado + IP pública).

---

## INC-004 — GitHub deshabilitó Actions en el repo (cron muerto)

**Síntoma:**
- `gh workflow run` → `HTTP 422: Actions has been disabled for this repository`
- Último run: 2026-10-08 23:18; después **0 runs** pese a cron cada 6 h
- Jobs con `status: queued`, `started_at` horas después, `steps: []` (0 pasos)

**Causa raíz:** El repo es un **fork** (`LuDev-tech/oci-free-arm-instance` ← `chrisshieh2012/...` ← `heykapil/...`). GitHub muestra el banner:

> "Workflows are not currently being run on this fork due to the scale of GitHub Actions usage. A repository maintainer can choose to re-enable these."

El patrón anterior — job de 5 horas **auto-encadenándose 24/7** — disparó el límite de uso de Actions para forks.

**Solución aplicada:**
1. Reescrito el workflow a **bajo consumo**: cron `*/30 * * * *`, **un solo intento por ejecución**, `timeout-minutes: 10`. Sin auto-encadenamiento. (~72 intentos/día con ~1-2 min de runner cada uno).
2. Re-habilitar workflows en la UI: **Actions → banner → "Enable"** (Settings → Actions → General también sirve).

**Lección:** En forks, GitHub puede desactivar Actions por volumen de uso, incluso con `state: active` en la API. Nunca usar loops de 5 h auto-encadenados. Preferir muchos runs cortos y baratos.

---

## INC-005 — `state: active` engañoso en la API

**Síntoma:** `gh api .../actions/workflows` devuelve `state: "active"` mientras el dispatch falla con 422 y el cron no dispara.

**Causa raíz:** `state` describe el archivo del workflow, **no** si Actions está habilitado para el repositorio/fork. `GET /actions/permissions` puede devolver `enabled: true` y aun así el dispatch fallar.

**Solución:** Probar siempre con un dispatch real (`gh workflow run`) y revisar la respuesta HTTP. Un 422 con `Actions has been disabled` es la prueba definitiva.

**Lección:** No confiar en `state` ni en `enabled`. Verificar con un dispatch real + presencia de runs `event=schedule` recientes.

---

## INC-006 — Supabase: `Unsupported phone provider` (proyecto MVP_DELIVERY)

**Síntoma:** El login por celular falla con `Unsupported phone provider`.

**Causa raíz:** El proveedor Phone está habilitado en Supabase pero sin un SMS provider (Twilio/MessageBird) configurado.

**Solución aplicada:** Login alternativo por **email OTP** (funciona sin configuración). Para SMS: Supabase → Authentication → Providers → Phone → configurar Twilio (Account SID, Auth Token, Message Service SID).

**Lección:** No prometer OTP por SMS sin verificar que el proveedor SMS esté configurado. Probar el flujo real, no solo el build.

---

## Checklist antes de reportar "funciona"

1. ¿Existe la instancia en Oracle? (`oci compute instance list` → OCID)
2. ¿Está en `RUNNING`? (no `PROVISIONING`)
3. ¿Tiene IP pública?
4. ¿El workflow se ejecutó de verdad? (`event=schedule`, timestamp reciente)
5. ¿El dispatch responde 2xx? (no 422)
6. ¿El resultado es `RESULT: VM_CREATED` / `VM_ALREADY_RUNNING`, no solo un run verde?

---

## INC-007 — `TooManyRequests` al lanzar la VM (auto-throttling)

**Síntoma:** `"code": "TooManyRequests", "message": "Too many requests for the user"`.

**Causa raíz:** el loop de reintentos cada 5 minutos, 24/7 durante meses, agota el rate limit de la API de lanzamiento de OCI por usuario.

**Solución aplicada (2026-11-13):** subir el intervalo de reintento ARM a 15 minutos (`sleep 900`) y bajar el cron del fallback AMD a cada 2 horas. Nunca correr dos workflows de lanzamiento con la misma frecuencia sobre la misma tenancy.

**Lección:** si OCI devuelve 429, el problema es NUESTRO ritmo, no la capacidad. Espaciar antes de volver a intentar.

---

## INC-008 — el cron del fallback AMD no dispara (schedule poco fiable)

**Sintoma:** el workflow `create-amd-vm.yml` (cron `0 */2 * * *`) no genero ningun run para los ciclos 02:00Z ni 04:00Z del 2026-11-14. El unico run programado del dia (00:41:31Z) llego 41 min tarde. Total historico del workflow: 3 runs (2 manuales).

**Evidencia:** `gh api repos/LuDev-tech/oracle-arm-vps/actions/workflows/417618289/runs` -> runs `43535882633` (schedule, 00:41Z), `43487024781` y `43485661383` (workflow_dispatch). Nada a las 02:00Z/04:00Z.

**Causa raiz:** depender solo de `schedule` en GitHub Actions. El scheduler demora o descarta ejecuciones; el encadenamiento por REST (`workflow_dispatch` via API) si es fiable: lleva ~38 h encadenando el workflow ARM cada 5 h sin fallar (`Chained next run and exiting` a las 02:50:21Z, run nuevo 5 s despues).

**Solucion aplicada (2026-11-14 05:07Z, commit `3faab60`):** cada run ARM dispara `create-amd-vm.yml` una vez al arrancar (3 intentos, 15 s de espera, nunca fatal). El cron del AMD queda solo como respaldo.

**Verificacion:** dispatch manual del mismo endpoint -> run `43559943310` creado con `head_sha=3faab60` y ejecutado; sus 2 lanzamientos E2.1.Micro fallaron con `InternalError: Out of host capacity` (05:09-05:11Z), o sea el mecanismo funciona y la capacidad sigue en cero.

**Leccion:** si un path critico depende solo de `schedule`, asumir que se va a caer. Encadenar por API y verificar la existencia real del run, no el verde del YAML.

