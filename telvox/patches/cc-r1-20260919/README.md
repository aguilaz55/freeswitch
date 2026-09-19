# Telvox R1 — callback que pierde la asignación

Desplegado el 2026-09-19, con reinicio completo autorizado de FreeSWITCH y ocho
clientes de voz. Módulo cargado verificado por inode/maps/SHA; smoke pasivo33/33.

Este paquete guarda la fuente EXACTA compilada del módulo custom instalado.
No reemplaza `src/mod/applications/mod_callcenter/mod_callcenter.c` del fork:
la rama telvox-patches-1.11 aún no contiene el baseline completo de integración
con conferencia que ya está instalado en producción. Aplicar sólo las seis
líneas al fuente antiguo no reproduce el binario certificado. No implica
sincronización de todo el árbol productivo ni de mod_conference.

## Archivos y aplicación offline

- baseline/: módulo de conferencia/callcenter previo instalado.
- candidate/: misma fuente con R1, seis líneas añadidas.
- patch.diff: diff baseline → candidate, no contra el HEAD antiguo del fork.
- CAUSA.md y REVISION.md: explicación y revisión independiente; las referencias
  de laboratorio /root/migracion son ubicaciones del servidor, no de este repo.
- build-result.json: comandos exactos y hashes de compilación/enlace contra el
  core estable instalado; requiere el entorno/configuración Telvox descrito.

Para verificar offline: `diff -u baseline/mod_callcenter.c candidate/mod_callcenter.c`.
El nuevo bloque cuelga sólo la sesión callback retenida que pierde el claim,
con LOSE_RACE; no alcanza uuid-standby ni la llamada ganadora.

## Evidencia resumida

Baseline:3/10 residuos. Candidato:30/30 cierres limpios, matriz9/9PASS.
Dos callbacks simultáneos: ambos200/ACK, perdedorBYE antes de colgar cliente,
ganador conservado; testigo independiente intacto y final0canales.
Conferencia callback OpusPASS. Standby mixto: baselinePASS, primer candidatoFAIL
por ventana tonal con bloque20ms de ceros, repetición mismo candidatoPASS.
Los bloques aislados también aparecieron en baseline; origen no establecido.
No se afirma cero drift, audio físico, authAPI2owners o nueva ejecución3+.

Módulo:53cb6d7229ba0fd103796b0083f6fc18ec4f34467b89369120c8354d13ccd908.
Core intacto:013e23208ee0abe04f0ec38d72eba8695862f60606129fb2963ffb0b5e783485.
El repo almacena fuentes y evidencia resumida, no binarios ni datos de llamadas.
Las suites completas están en el servidor, /root/migracion/reviews/cc-r1.
Reserva router1B no desplegada por esta entrega.
