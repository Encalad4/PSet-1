# README - DM-PSet-01: Backfill Histórico de QuickBooks Online

## Descripción del Proyecto

Este proyecto implementa un pipeline de datos para realizar un backfill histórico desde QuickBooks Online (QBO) hacia una base de datos PostgreSQL. El sistema extrae datos de las entidades **Invoices**, **Customers** e **Items** y los almacena en un esquema `raw` con trazabilidad completa. La orquestación se realiza con **Mage AI**, el despliegue con **Docker Compose**, y la gestión de credenciales mediante **Mage Secrets**.




## Estructura del Pipeline

El pipeline implementado consta de tres bloques principales:

### 1. **access-token_getter** - Obtención de Token de Acceso
- Renueva el access token usando OAuth 2.0 con refresh token
- Maneja reintentos con backoff exponencial
- Gestiona errores de autenticación (401, 400)
- Actualiza el refresh token en variables globales
- Retorna: `access_token`, `expires_in`, `token_obtained_at`, `realm_id`

### 2. **json_getter** - Extracción de Datos de QBO
- Consulta la API de QuickBooks con filtros temporales
- Implementa paginación automática (page_size = 10)
- Maneja rate limits y errores de API
- Parámetros configurables: `fecha_inicio`, `fecha_fin` (UTC)
- Estructura de salida por chunk:
  ```json
  {
    "qb_Invoices": "qb_Invoices",
    "payload": {...},
    "ingested_at_utc": "...",
    "extract_window_start_utc": "...",
    "extract_window_end_utc": "...",
    "page_number": 1,
    "page_size": 10,
    "request_payload": "SELECT ..."
  }
  ```

### 3. **uploading_to_db** - Carga a PostgreSQL
- Conexión segura a PostgreSQL usando credenciales de Mage Secrets
- Implementa upsert (INSERT/UPDATE) por clave primaria `id`
- Valida existencia previa de registros
- Almacena payload completo en formato JSONB
- Incluye metadatos de ingesta requeridos

## Configuración y Despliegue

### Prerrequisitos
- Docker y Docker Compose instalados
- Cuenta de desarrollador QBO (sandbox) con:
  - Client ID y Client Secret
  - Realm ID
  - Refresh Token
- Mage AI básico

### Configuración de Mage Secrets

| Nombre del Secreto | Propósito | Responsable |
|-------------------|-----------|-------------|
| `client_id` | Autenticación OAuth 2.0 con QBO | Equipo de Desarrollo |
| `client_secret` | Autenticación OAuth 2.0 con QBO | Equipo de Desarrollo |
| `realm_id` | Identificador de compañía QBO | Equipo de Desarrollo |
| `postgres_host` | Conexión a PostgreSQL | Administrador de BD |
| `postgres_db` | Base de datos destino | Administrador de BD |
| `postgres_user` | Usuario de PostgreSQL | Administrador de BD |
| `postgres_password` | Contraseña de PostgreSQL | Administrador de BD |

**Política de Rotación**: Los tokens de QBO rotan automáticamente cada 100 días. Las credenciales de BD se rotan manualmente cada 90 días.



## Diseño del Esquema RAW

### Tabla `qb_invoices`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | VARCHAR(255) PRIMARY KEY | ID único del invoice (QBO) |
| `payload` | JSONB | Documento completo del invoice |
| `ingested_at_utc` | TIMESTAMP | Fecha/hora de ingesta (UTC) |
| `extract_window_start_utc` | TIMESTAMP | Inicio de ventana de extracción |
| `extract_window_end_utc` | TIMESTAMP | Fin de ventana de extracción |
| `page_number` | INTEGER | Número de página en paginación |
| `page_size` | INTEGER | Tamaño de página procesada |
| `request_payload` | TEXT | Query original a la API |

**Índices**: `id` (PRIMARY KEY)

## Configuración del Trigger One-Time

### Parámetros del Pipeline
- **Tipo**: One-time execution
- **Fecha/hora inicio UTC**: 2025-12-27T12:54:08
- **Equivalente America/Guayaquil**: 2025-12-28 16:16:17 
- **Fecha/hora fin UTC**: 2025-12-28T13:16:17
- **Equivalente America/Guayaquil**: 2025-12-28 08:16:17


### Política de Deshabilitación
El trigger se deshabilita automáticamente después de la ejecución exitosa. Para reejecutar, se debe configurar un nuevo trigger one-time con los parámetros deseados.

## Validaciones y Calidad de Datos

### Validaciones Implementadas
1. **Integridad de Claves Primarias**: Verificación de `id` no nulo y único
2. **Consistencia Temporal**: Todas las marcas de tiempo en UTC
3. **Idempotencia**: Reejecución no genera duplicados (via UPSERT)
4. **Volumetría**: Conteo por chunk y total

### Scripts de Validación
```sql
-- Verificar idempotencia
SELECT id, COUNT(*) as duplicados 
FROM qb_invoices 
GROUP BY id 
HAVING COUNT(*) > 1;

-- Verificar volumetría por día
SELECT DATE(ingested_at_utc) as dia, COUNT(*) as registros
FROM qb_invoices
GROUP BY DATE(ingested_at_utc)
ORDER BY dia;
```

## Troubleshooting

### Problemas Comunes y Soluciones

#### 1. Error de Autenticación OAuth
```
Síntoma: "Refresh token inválido o revocado"
Solución:
1. Verificar que el refresh_token en Mage Secrets sea válido
2. Asegurar que Client ID y Secret sean correctos
3. Verificar que la app QBO tenga permisos necesarios
```

#### 2. Timeout en Consultas API
```
Síntoma: Timeout después de 15 segundos
Solución:
1. Reducir page_size en json_getter (línea 47)
2. Aumentar timeout en session.post()
3. Implementar chunking más pequeño
```

#### 3. Duplicados en PostgreSQL
```
Síntoma: Registros duplicados con mismo id
Solución:
1. Verificar que ON CONFLICT (id) funcione correctamente
2. Confirmar que la tabla tenga PRIMARY KEY en id
3. Validar estructura de upsert_sql
```



## Runbook de Operación

### Reanudación de Procesos Interrumpidos
1. Identificar último chunk exitoso en logs
2. Configurar nuevo trigger con:
   - `fecha_inicio`: última fecha procesada + 1 segundo
   - `fecha_fin`: fecha original final
3. Ejecutar pipeline con trigger one-time



### Verificación de Resultados
1. Consultar métricas en logs de Mage
2. Validar conteos en PostgreSQL:
   ```sql
   SELECT COUNT(*) as total FROM qb_invoices;
   ```
3. Revisar logs de error (si existen)

## Evidencias Requeridas

### Archivos de Evidencia
1. Configuración de Mage Secrets (nombres visibles, valores ocultos)
![Mage Secrets](capturas/Mage%20secrets.png)
2. Trigger one-time configurado en Mage
![Trigger one time](capturas/Completed%20customers%20backfill.png)
![Trigger one time](capturas/Completed%20invoice%20backfill.png)
![Trigger one time](capturas/Completed%20items%20backfill.png)
3. Estructura de tabla `qb_invoices` en PgAdmin
![Tabla invoices](capturas/Ejemplos%20invoices%20en%20tabla.png)
4. Reporte de volumetría por chunk
![Volumetria](capturas/Idempotencia%20logs.png)
5. Prueba de idempotencia 
![Volumetria logs](capturas/Idempotencia%20sql.png)



## Checklist de Aceptación

- [x] Mage y Postgres se comunican por nombre de servicio
- [x] Todos los secretos (QBO y Postgres) están en Mage Secrets
- [x] Pipeline acepta `fecha_inicio` y `fecha_fin` (UTC) y segmenta el rango
- [x] Trigger one-time configurado, ejecutado y deshabilitado
- [x] Esquema raw con tablas, payload completo y metadatos
- [x] Idempotencia verificada: reejecución no genera duplicados
- [x] Paginación y rate limits manejados y documentados
- [x] Volumetría y validaciones registradas como evidencia
- [x] Runbook de reanudación y reintentos disponible

---

