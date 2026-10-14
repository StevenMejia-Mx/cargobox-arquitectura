# Matriz de trazabilidad

La matriz conecta cada necesidad de la Unidad 2 (matriz TR del Modelo de dominio y restricciones del BRD) con las historias, las reglas, la actividad del BPMN, los componentes del diseño y el ADR que la justifica. Las necesidades TR-03 y TR-05 quedan fuera de las cinco historias de esta entrega, pero el diseño ya deja el punto de extensión.

| Necesidad (U2) | Historia | Reglas | Actividad del BPMN | Componentes del diseño | ADR |
|---|---|---|---|---|---|
| BRD §6: límites de peso y dimensiones; tarifa por nivel | US01 | RN-01, RN-02, RN-03 | Pesar, medir y cotizar → ¿Cumple límites? → Cobrar, generar guía e imprimir / Rechazar – carga especial | EnvioApplicationService.registrarGuia, Guia.generar, Peso, Dimensiones, Cotizador, PoliticaTarifa, CargaEspecialRequerida | ADR-001, ADR-005 |
| TR-01 Reducir llamadas por falta de información | US09 | RN-08 | Consulta de estado (TR-01; no es actividad de un carril) | ConsultaRastreoQueryService.consultarHistorial, HistorialRastreoReader, HistorialRastreoProjection, HistorialEnvioView | ADR-003, ADR-004 |
| TR-02 Cada cambio de estado genera un evento de rastreo | US04, US05, US06 | RN-04, RN-05 | Escaneo de ingreso y clasificación → Despacho a ruta → Cargar paquete y salir a ruta | RastreoApplicationService.registrarEscaneo, Guia.registrarEscaneo, Guia.cambiarEstado, EventoRastreo, EstadoGuia, TransicionEstadoInvalida | ADR-002 |
| BRD §6: máximo 2 intentos y resguardo | US05 | RN-06 | Registrar «Intento fallido» → Resguardo en almacén (máx. 5 días hábiles) | EntregaApplicationService.registrarIntentoFallido, Guia.registrarIntentoFallido, IntentoEntrega, EN_RESGUARDO, GuiaEnResguardo, LimiteIntentosExcedido, NotificacionHandler | ADR-001, ADR-002 |
| TR-04 Centralizar la evidencia de entrega | US06 | RN-07 | ¿Destinatario o tercero presente? → Capturar foto + firma digital → Fin: entrega exitosa | EntregaApplicationService.capturarEvidenciaEntrega, AlmacenEvidencias, Guia.registrarEntrega, EvidenciaEntrega, EvidenciaIncompleta, EntregaYaRegistrada, EntregaCompletada | ADR-002, ADR-005 |
| TR-03 Formalizar el manejo de incidencias | US14, US15 (fuera de esta entrega) | — | Timer 10 días sin rastro → Declarar pérdida y compensar | Pendiente: agregado Incidencia en el trabajo final; reaccionará a los Domain Events vía PublicadorEventos | ADR-004 |
| TR-05 Detectar envíos retrasados (24 h / 48 h) | US12, US13 (fuera de esta entrega) | — | Timer 24 h sin cambio → Generar alerta por retraso | Pendiente: tarea programada que lea la fecha del último EventoRastreo | ADR-004 |

## Recorrido del BPMN

Para responder si el diseño está de acuerdo con el BPMN, se recorrió cada actividad y compuerta del diagrama de la Unidad 2 y se ubicó el componente que la atiende. Todas las actividades del flujo principal y del flujo de excepción de las cinco historias quedan cubiertas; las que dependen de otras historias se marcan como fuera de esta entrega.

| Actividad o compuerta del BPMN | Carril | Dónde la atiende el diseño | Resultado |
|---|---|---|---|
| Inicio: Entrega paquete | Remitente | EnvioApplicationService.registrarGuia (US01) | Cubierta |
| Pesar, medir y cotizar | Sucursal | registrarGuia: Peso y Dimensiones + Cotizador / PoliticaTarifa | Cubierta |
| ¿Cumple límites? (≤ 30 kg y ≤ 100 cm) | Sucursal | Guia.generar: Peso y Dimensiones validan RN-01 | Cubierta |
| Cobrar, generar guía e imprimir | Sucursal | registrarGuia obtiene el NumeroGuia y devuelve GuiaRegistradaDTO | Cubierta; el cobro se asume (BRD §4 y §7) |
| Rechazar / Carga especial | Sucursal | Error CargaEspecialRequerida; el UnitOfWork se deshace | Cubierta |
| Escaneo de ingreso y clasificación | Centro de distribución | RastreoApplicationService.registrarEscaneo con tipo INGRESO (US04) | Cubierta |
| Despacho a ruta / unidad | Centro de distribución | registrarEscaneo con tipo SALIDA (US04) | Cubierta |
| Cargar paquete y salir a ruta | Repartidor | Salida del centro que atiende el destino: EN_REPARTO + EventoRastreo | Cubierta |
| ¿Destinatario o tercero presente? | Repartidor | Decide qué método se llama: capturarEvidenciaEntrega (US06) o registrarIntentoFallido (US05) | Cubierta |
| Capturar foto + firma digital | Repartidor | EntregaApplicationService.capturarEvidenciaEntrega + AlmacenEvidencias; RN-07 en la Guía | Cubierta |
| Registrar «Intento fallido» | Repartidor | EntregaApplicationService.registrarIntentoFallido; RN-06 en la Guía | Cubierta |
| ¿Cliente reagenda o recoge en hub? | Centro de distribución | US10 | Fuera de esta entrega (ver hallazgo H-01) |
| Resguardo en almacén (máx. 5 días hábiles) | Centro de distribución | EstadoGuia.EN_RESGUARDO y evento GuiaEnResguardo | Cubierta; el conteo de 5 días va con US10 |
| Registrar devolución a remitente | Atención al cliente | US10 | Fuera de esta entrega |
| Timer 24 h sin cambio → Generar alerta | Atención al cliente | US12 | Fuera de esta entrega; se conectará a los Domain Events |
| Timer 10 días → Declarar pérdida y compensar | Atención al cliente | US15 | Fuera de esta entrega |
| Fin: Entrega exitosa | — | Evento EntregaCompletada | Cubierta |

## Hallazgos de continuidad con la Unidad 2

| ID | Hallazgo en la Unidad 2 | Decisión del equipo |
|---|---|---|
| H-01 | El BPMN no muestra el conteo de dos intentos: después del primer intento fallido pasa directo a «¿Cliente reagenda o recoge en hub?». | Se siguió la regla del BRD §6 y de US05 (máximo 2 intentos), que es más precisa. La Guía lleva el número de intento; el BPMN debe actualizarse para mostrar el segundo intento antes del resguardo. |
| H-02 | La imagen del modelo de dominio en el PDF de la U2 (página 2) muestra Pasajero, Conductor, Viaje y Vehículo, que no son de CargoBox. | Se usaron el texto, las reglas y el glosario de ese mismo documento, que sí son de CargoBox. La imagen debe reemplazarse por el mapa de dominio de esta entrega. |
| H-03 | El modelo de dominio de la U2 trata Remitente, Destinatario, EvidenciaEntrega y NivelServicio como entidades. | En este diseño son Value Objects: no tienen ciclo de vida propio en las cinco historias. Se registra el cambio para mantener la trazabilidad. |
