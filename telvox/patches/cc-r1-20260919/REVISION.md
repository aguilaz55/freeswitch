# Revisión independiente R1 — Astra

Dictamen estático: **PASA CÓDIGO, cero hallazgos nuevos de corrección**.
Validación dinámica en curso al emitir este dictamen.

Fuente e2e39af0e035470713a839de94b3f6931eb08c87cfd8224b4fbe01fe8cb78b57;
módulo 53cb6d7229ba0fd103796b0083f6fc18ec4f34467b89369120c8354d13ccd908.
Un único bloque, seis líneas añadidas.

El hangup actúa sobre el callback que perdió el claim, mediante la sesión
retenida por su hilo. No busca otros UUID ni amplía barridos. Se trazaron
originate, sustitución correlacionada de loopback, locks y liberación en done;
el hangup nativo serializa solicitudes repetidas. Ganador y uuid-standby están
excluidos; conferencia, bridge, grabación y contabilidad ocurren después de esta
salida. Estados/tiers mantienen el recorrido basal. No hay nuevas entradas de
auth, SQL, destinos, secretos, tarifas ni contratos de drivers.

Verificación del revisor, sólo lectura:

```text
DIFF_ARTIFACT_MATCH=PASS
CHANGED_HUNKS=1 (+6/-0)
INDEPENDENT_NEEDED_RPATH_EXPORTS_PARITY=PASS
```

Evidencia dinámica inspeccionada en esa revisión: callback/conferencia PASS;
standby FAIL sólo en fracción de tono de grabación canal0/segundo3, con audio,
identidad y retorno standby PASS. El primer banco R1 abortó por KeyError del
fixture y no constituye reproducción. Se mantienen ambos fallos visibles.

Pendientes al emitir: R1 temprano válido, ring-all dos callbacks y testigo de
otro owner, matriz track/standby y control basal de la grabación. Conferencias
3+ cubiertas estáticamente; no ejecución nueva de 3+ participantes.

Sin producción ni llamadas reales. Conservar módulo anterior f5a3e8ea para
rollback. Instalación requiere reinicio completo autorizado, nunca hot-reload.

## Delta de evidencia revisado independientemente

El revisor contrastó los 40 ledgers SIP y snapshots antes de la limpieza,
hashes, módulos mapeados, namespace y salida del proceso:

```text
baseline: 7/10 limpias; 3 residuos; 10 deltas <2 ms
candidato: 30/30 limpias; 29 deltas <2 ms
matrix: 9/9 PASS; contador externo final 0
READ_ONLY_EVIDENCE_DELTA=PASS
```

Standby basal y repetición candidata PASS, incluido regreso a espera y canal
vivo. Los WAV de las tres ejecuciones contienen bloques aislados de 20ms de
ceros en posiciones diferentes. El FAIL inicial se conserva: no acredita una
regresión del parche ni queda explicado el origen de las discontinuidades.
Ver STANDBY_RECORDING_DIAG.md. No se afirma cero drift global.

Dictamen actualizado: PASA CÓDIGO y evidencia acotada; pendiente únicamente
el banco de dos callbacks simultáneos con testigo independiente de otro owner.
No sustituye aceptación integrada ni comprobaciones posteriores al despliegue.

## Cierre final — PASA R1, listo para despliegue autorizado

Revisión independiente del fixture y ledgers finales de
`smokes/ringall-ownership/candidate-final-run02`, sin hallazgos bloqueantes:

- Ambos callbacks enviaron200 y recibieron ACK.
- BYE del perdedor408,276ms antes del hangup del cliente; ganador21,819ms después.
- Witness owner200 vivo en ambas comprobaciones, sin BYE entrante; ledger final
  de23paquetes RTP incluye uno después del hangup del cliente (snapshot vivo22).
- Espera de cero canales completada antes del cleanup; exit0 y módulo53cb exacto.

```text
INDEPENDENT_OWNERSHIP_LEDGER=PASS
ARTIFACT_MODULE_SCENARIO_ISOLATION=PASS
```

Cerrado el pendiente dinámico R1. El fallo inicial del fixture se conserva y
no cuenta como fallo de producto. Owners son metadatos privados, no auth API.
LOSE_RACE validado estáticamente; no se afirma una cabecera SIP no capturada.
Sin nuevas pruebas ni despliegue por el revisor. Reinicio completo autorizado
y respaldo requeridos; aceptación integrada SIMTEST de1B sigue pendiente.
