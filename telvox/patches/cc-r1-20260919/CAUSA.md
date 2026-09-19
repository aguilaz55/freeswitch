# R1 — callback B residual al perder el claim

Fecha: 2026-09-19. Alcance: copia aislada de `mod_callcenter.c`; sin compilar,
instalar, reiniciar ni ejecutar llamadas desde esta tarea.

## Causa exacta

En `outbound_agent_thread_run`, `switch_ivr_originate` puede devolver
`SWITCH_STATUS_SUCCESS` con un `agent_session` callback ya contestado. El flujo
borra inmediatamente `cc_member_pre_answer_uuid` de ese canal y después intenta
adjudicarse el miembro en estrategias `ring-all`/`ring-progressively`.

Si el `UPDATE members ... serving_agent = <agente>` no obtiene el claim, el
`SELECT count(*)` devuelve cero y la rama salta directamente a `done`. Esa salida
actualiza tiers/estado y libera los locks de las sesiones, pero no cuelga el
callback B creado por este hilo. Además, el B ya no conserva
`cc_member_pre_answer_uuid`, por lo que ni el `hupall` del ganador ni el `hupall`
del abandono del miembro pueden encontrarlo. Esto explica el control009: B
contestado, ningún bridge contabilizado y canal residual tras el BYE inmediato
del miembro.

## Cambio mínimo

En la única rama `count == 0`, antes de `goto done`, se cuelga con
`SWITCH_CAUSE_LOSE_RACE` el `agent_channel` que el mismo hilo obtuvo, solamente
cuando `agent_type` es `callback`.

La operación no busca por UUID ni por una variable global. Actúa sobre la sesión
que sigue bajo el lock del hilo, así que no puede seleccionar otra oferta del
mismo miembro. El ganador no entra en la rama. `uuid-standby` no se cuelga. La
preparación de conferencia, el bridge y el inicio de grabación ocurren después
del claim y no cambian.

## Archivos y hashes

- Fuente de partida:
  `changes/recording-conference-integration/candidate/mod_callcenter/mod_callcenter.c`
- Baseline exacto:
  `changes/cc-r1/baseline/mod_callcenter.c`
- Candidato:
  `changes/cc-r1/candidate/mod_callcenter.c`
- Diff portable dentro de este directorio: `changes/cc-r1/patch.diff`
- SHA-256 fuente/baseline:
  `f81983cafaa62f8a16bf2847767b7bc2d264db505de77c557e7c208be12d4de5`
- SHA-256 candidato:
  `e2e39af0e035470713a839de94b3f6931eb08c87cfd8224b4fbe01fe8cb78b57`

## Evidencia estática cruda

```text
$ sha256sum changes/cc-r1/baseline/mod_callcenter.c \
    changes/cc-r1/candidate/mod_callcenter.c \
    changes/recording-conference-integration/candidate/mod_callcenter/mod_callcenter.c
f81983cafaa62f8a16bf2847767b7bc2d264db505de77c557e7c208be12d4de5  changes/cc-r1/baseline/mod_callcenter.c
e2e39af0e035470713a839de94b3f6931eb08c87cfd8224b4fbe01fe8cb78b57  changes/cc-r1/candidate/mod_callcenter.c
f81983cafaa62f8a16bf2847767b7bc2d264db505de77c557e7c208be12d4de5  changes/recording-conference-integration/candidate/mod_callcenter/mod_callcenter.c

$ cmp -s changes/cc-r1/baseline/mod_callcenter.c \
    changes/recording-conference-integration/candidate/mod_callcenter/mod_callcenter.c
BASELINE_CMP_EXIT=0

$ diff -u --label baseline/mod_callcenter.c --label candidate/mod_callcenter.c \
    changes/cc-r1/baseline/mod_callcenter.c changes/cc-r1/candidate/mod_callcenter.c
@@ -2333,6 +2333,12 @@
 			switch_safe_free(sql);

 			if (atoi(res) == 0) {
+				/* This thread owns the callback session returned by originate.
+				 * It cleared the pre-answer marker before losing the member claim,
+				 * so the winner/member cleanup cannot find this exact B leg. */
+				if (!strcasecmp(h->agent_type, CC_AGENT_TYPE_CALLBACK)) {
+					switch_channel_hangup(agent_channel, SWITCH_CAUSE_LOSE_RACE);
+				}
 				goto done;
 			}
 			switch_core_session_hupall_matching_var("cc_member_pre_answer_uuid", h->member_uuid, SWITCH_CAUSE_LOSE_RACE);

$ cd changes/cc-r1/baseline
$ patch --dry-run -p1 -i ../patch.diff
checking file mod_callcenter.c

$ cd /root/migracion
$ git apply --check --directory=changes/cc-r1/baseline changes/cc-r1/patch.diff
# exit 0, sin salida
```

## Validación dinámica requerida por el principal

El cambio necesita compilación aislada y repetir el banco que ya posee el
principal:

1. control009 sin `callcenter_track`, con BYE inmediato: debe aparecer BYE al
   device y cero canales dentro de cinco segundos; el agente debe quedar
   `Waiting`, contador cero y sin bridge contabilizado.
2. run006 con barrera `In a queue call`: debe conservar bridge, audio, cierre y
   contabilidad normales.
3. Cobertura de `ring-all` con dos callbacks: el perdedor recibe `LOSE_RACE` y el
   ganador permanece activo.
4. Cobertura `uuid-standby` y conferencia/grabación custom: no deben recibir el
   hangup nuevo porque el guard exige `callback` y la rama precede esos flujos.

No se necesita instrumentación adicional para el primer intento: el ledger SIP,
el snapshot de canales y el estado CC distinguen el resultado. Si control009
siguiera dejando B residual, la instrumentación mínima sería un único log en
esta rama con el UUID de `agent_session`, tipo de agente y causa, sin datos de
cliente ni dialstring, para comprobar que se ejecutó la compensación.

## Limitaciones

La causa está demostrada por control009 más la salida concreta de fuente. Esta
tarea no compiló el módulo ni ejecutó el banco; por tanto, el comportamiento
dinámico del binario candidato queda pendiente del principal. El parche no
corrige otras posibles fallas de `switch_ivr_uuid_bridge` ni añade guardas en esa
ruta, porque no son necesarias para la reproducción adjudicada.
