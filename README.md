# CloudPage SFMC — Limitaciones de CreateSalesforceObject y UpdateSingleSalesforceObject

**Fecha:** 2025-09-01  
**Autor:** Santi

## Resumen (TL;DR)
- Son llamadas **sincrónicas y por registro**: no hay *bulk*.  
- No consumen **API daily limit**, pero sí están sujetos a **concurrencia**, **timeouts** y **locks** en Salesforce.  
- Disparan **Validation Rules/Flows/Triggers** y requieren **FLS/CRUD** correctos del usuario integrado.  
- Manejo de errores **básico**; conviene **loggear** en Data Extension.  
- Úsalas para **pocos registros** (formularios, altas puntuales). Evítalas con **volumen** o **picos**.

---

## Limitaciones clave
1. **Granularidad 1:1 (sin bulk)**  
   Cada invocación crea/actualiza **un** registro. Iterar muchos registros desde una CloudPage acumula latencia y riesgo de *timeout*.

2. **Límites de ejecución y concurrencia**  
   - No cuentan contra el **API daily limit** del org, pero sí aplican límites de **concurrencia** y **tiempos de respuesta**.  
   - Bajo tráfico concurrente, pueden aparecer errores por espera prolongada y saturación.

3. **Record locking / UNABLE_TO_LOCK_ROW**  
   Actualizaciones simultáneas sobre el mismo registro (o padres) pueden bloquearse. Es frecuente con skew de datos o automatizaciones paralelas.

4. **Validaciones y automatizaciones se disparan**  
   Reglas de validación, Flows/Process Builder y Triggers se ejecutan; pueden bloquear, modificar en cascada o aumentar el tiempo de transacción.

5. **Permisos y visibilidad (FLS/CRUD)**  
   La ejecución usa el **usuario integrado**; campos u objetos no visibles/editables ⇒ error. Considera también políticas de red/IP si aplican.

6. **Formato estricto de datos (especialmente fechas)**  
   Usa ISO-8601/UTC para Date/DateTime. Normaliza *picklists*, requeridos y longitudes para evitar rechazos.

7. **Timeouts del runtime de CloudPages**  
   Bucles con muchas llamadas en una sola petición pueden agotar el tiempo de ejecución del entorno de Marketing Cloud.

8. **Observabilidad limitada**  
   - `CreateSalesforceObject()` retorna el **Id**.  
   - `UpdateSingleSalesforceObject()` retorna **1/0**.  
   Log interno es imprescindible (DE de auditoría con payload y resultado).

9. **Rendimiento**  
   Cada llamada tiene *overhead* de red y serialización; el rendimiento decae al escalar.

10. **Compatibilidad/alcance**  
   - Sirven para objetos estándar y *custom*.  
   - No hay **upsert por External Id**; necesitas lógica propia para decidir create vs update (o usar APIs con *upsert* nativo).

---

## Cuándo conviene (y cuándo no)
**Úsalas si…**  
- Hay **pocos registros por interacción** (1–5) y necesitás respuesta **sincrónica** (p. ej. formulario de preferencias, alta simple de caso/cita).

**Evítalas si…**  
- Hay **volumen** y/o **concurrencia** (listas largas, picos de tráfico, múltiples objetos en cascada).

---

## Patrones de mitigación recomendados
1. **Staging asíncrono en Data Extension**  
   Escribe primero en una DE con idempotencia (token anti reenvío) y procesa en **Automation Studio** (SSJS/WSProxy) por **lotes** con **reintentos** y *rate limiting*.

2. **API de Salesforce para volumen**  
   Para grandes cargas, usa **REST/Bulk API 2.0** desde un *worker*/Function con **External IDs (upsert)**, *backoff* exponencial y manejo de **row locks**.

3. **Optimización en Salesforce**  
   Revisa **Validation Rules/Flows/Triggers**: *bulk-safe*, sin recursión y con tiempos acotados para reducir locks y timeouts.

4. **Permisos e IPs**  
   Asegura **FLS/CRUD** del usuario integrado y, si aplica, allowlist de IPs de Marketing Cloud en Salesforce (**Network Access** o políticas de acceso).

5. **Logging y observabilidad**  
   - DE de auditoría (fecha/hora, objeto, *payload*, resultado, error).  
   - Correlación por *request id* y *session id* para *troubleshooting*.  
   - Alertas básicas (Automation + query a errores recientes).

---

## Checklist antes de ir a producción
- [ ] Validar FLS/CRUD del usuario integrado por objeto/campo.  
- [ ] Ensayar con **volumen representativo** y **concurrencia** simulada.  
- [ ] Normalizar **formatos** (fechas, *picklists*, longitudes).  
- [ ] Revisar **Validation Rules/Flows/Triggers** por impacto y tiempos.  
- [ ] Implementar **DE de log** y políticas de **reintento**.  
- [ ] Definir *fallback* (staging en DE si falla la llamada sincrónica).

---

### Nota
La función de actualización en AMPscript es `UpdateSingleSalesforceObject()` (a veces abreviada informalmente como *UpdateSalesforceObject*).
