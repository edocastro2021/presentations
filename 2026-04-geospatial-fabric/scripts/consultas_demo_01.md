# Geospatial Queries – Microsoft Fabric / SQL

Este documento contiene ejemplos de consultas geoespaciales aplicadas a monitoreo de flota, utilizando tipos `geography` en SQL.

---

## Distancia entre alertas y base de operaciones

Calcula la distancia (en kilómetros) desde cada alerta hasta la base en La Garita.

```sql
SELECT
    PLACA,
    TIMESTAMP,
    CATEGORIA_EVENTO,
    SEVERIDAD,
    geography::Point(LATITUD, LONGITUD, 4326).STDistance(
        geography::Point(9.9811, -84.2801, 4326)   -- base La Garita
    ) / 1000.0 AS KM_DESDE_BASE
FROM gold_eventos_alertas
WHERE SEVERIDAD IN ('ALTA', 'CRITICA')
ORDER BY KM_DESDE_BASE DESC;
```

---

## Geofence — alertas dentro de radio de 500m

Identifica alertas que ocurren dentro de un radio de 500 metros desde una ubicación específica.

```sql
DECLARE @zona geography = geography::Point(9.9811, -84.2801, 4326).STBuffer(500);

SELECT
    PLACA,
    TIMESTAMP,
    MOTIVO,
    LATITUD,
    LONGITUD
FROM gold_eventos_alertas
WHERE @zona.STContains(
    geography::Point(LATITUD, LONGITUD, 4326)
) = 1;
```

---

## Longitud total de ruta por camión por día

Agrega métricas de operación por camión y fecha.

```sql
SELECT
    PLACA,
    FECHA,
    COUNT(*)            AS VIAJES,
    SUM(KM_VIAJE)       AS KM_TOTALES,
    AVG(VELOCIDAD_PROM) AS VEL_PROMEDIO,
    MAX(DURACION_MIN)   AS VIAJE_MAS_LARGO_MIN
FROM gold_segmentos_viaje
GROUP BY PLACA, FECHA
ORDER BY PLACA, FECHA;
```

---

## Paradas dentro de zona geográfica (Puntarenas)

Filtra paradas dentro de un área aproximada de la provincia de Puntarenas.

```sql
DECLARE @puntarenas geography = geography::Point(9.9773, -84.8308, 4326).STBuffer(50000);

SELECT
    PLACA,
    INICIO_PARADA,
    DURACION_MIN,
    TIPO_PARADA,
    LUGAR
FROM gold_paradas
WHERE @puntarenas.STContains(
    geography::Point(LATITUD, LONGITUD, 4326)
) = 1
ORDER BY DURACION_MIN DESC;
```

---

## Notas técnicas

- Sistema de referencia: SRID 4326 (WGS84)
- STDistance devuelve distancia en metros
- STBuffer define radios en metros
- STContains evalúa si un punto está dentro de un polígono

---

## Uso recomendado

Estas consultas son útiles para:

- Monitoreo de flota
- Detección de eventos por ubicación
- Análisis de operación
- Implementación de geofencing
