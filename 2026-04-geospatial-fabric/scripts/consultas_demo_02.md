# Análisis Geoespacial de Flota con T-SQL
Este documento contiene ejemplos prácticos utilizando funciones espaciales en SQL Server aplicadas a análisis de flota, geolocalización, alertas y geofencing.

## STDistance()

Calcula la distancia más corta (en metros) entre dos puntos geográficos.

Uso típico:
- Distancia de eventos a una base
- Distancia promedio de operación por vehículo

### Distancia de alertas de velocidad a la base

~~~sql
SELECT
    PLACA,
    TIMESTAMP,
    MOTIVO,
    VELOCIDAD,
    LATITUD,
    LONGITUD,
    ROUND(
        geography::Point(LATITUD, LONGITUD, 4326)
            .STDistance(geography::Point(9.9811, -84.2801, 4326))
        / 1000.0
    , 2) AS KM_DESDE_BASE
FROM gold_eventos_alertas
WHERE CATEGORIA_EVENTO = 'ALERTA_VELOCIDAD'
ORDER BY KM_DESDE_BASE DESC;
~~~

### Distancia promedio de operación por camión

~~~sql
SELECT
    PLACA,
    COUNT(*) AS TOTAL_DIAS,
    ROUND(AVG(
        geography::Point(LAT_CENTRO_DIA, LON_CENTRO_DIA, 4326)
            .STDistance(geography::Point(9.9811, -84.2801, 4326))
        / 1000.0
    ), 2) AS KM_PROMEDIO_DE_BASE,
    ROUND(MAX(
        geography::Point(LAT_CENTRO_DIA, LON_CENTRO_DIA, 4326)
            .STDistance(geography::Point(9.9811, -84.2801, 4326))
        / 1000.0
    ), 2) AS KM_MAXIMO_DE_BASE
FROM gold_kpi_diario
GROUP BY PLACA
ORDER BY KM_MAXIMO_DE_BASE DESC;
~~~

## STBuffer()

Crea un área circular alrededor de un punto geográfico.

Uso típico:
- Geofencing
- Zonas de control (bodegas, clientes, zonas sensibles)

### Alertas dentro de 2 km de la base

~~~sql
DECLARE @base_la_garita geography = geography::Point(9.9811, -84.2801, 4326);
DECLARE @zona_base geography = @base_la_garita.STBuffer(2000);

SELECT
    PLACA,
    TIMESTAMP,
    CATEGORIA_EVENTO,
    SEVERIDAD,
    MOTIVO,
    LATITUD,
    LONGITUD,
    ROUND(
        geography::Point(LATITUD, LONGITUD, 4326).STDistance(@base_la_garita) / 1000.0
    , 0) * 1000 AS METROS_DESDE_BASE
FROM gold_eventos_alertas
WHERE @zona_base.STContains(geography::Point(LATITUD, LONGITUD, 4326)) = 1
ORDER BY TIMESTAMP;
~~~

### Paradas cerca del centro de distribución

~~~sql
DECLARE @walmart_cd geography = geography::Point(9.9850, -84.2750, 4326);
DECLARE @zona_cd geography = @walmart_cd.STBuffer(500);

SELECT
    PLACA,
    INICIO_PARADA,
    FIN_PARADA,
    DURACION_MIN,
    TIPO_PARADA,
    LUGAR,
    LATITUD,
    LONGITUD
FROM gold_paradas
WHERE @zona_cd.STContains(geography::Point(LATITUD, LONGITUD, 4326)) = 1
ORDER BY PLACA, INICIO_PARADA;
~~~

### Alertas en zona escolar (300m)

~~~sql
DECLARE @zona_escolar geography =
    geography::Point(10.0633, -84.6247, 4326).STBuffer(300);

SELECT
    SEVERIDAD,
    COUNT(*) AS TOTAL_ALERTAS
FROM gold_eventos_alertas
WHERE @zona_escolar.STContains(geography::Point(LATITUD, LONGITUD, 4326)) = 1
GROUP BY SEVERIDAD
ORDER BY TOTAL_ALERTAS DESC;
~~~

## STIntersects() / STWithin()

Uso típico:
- Saber si un camión operó dentro de una zona
- Detectar eventos fuera de rutas autorizadas

### Camiones dentro de Puntarenas

~~~sql
DECLARE @puntarenas geography =
    geography::Point(9.9773, -84.8308, 4326).STBuffer(60000);

SELECT
    PLACA,
    FECHA,
    COUNT(*) AS PINGS_EN_PUNTARENAS
FROM gold_kpi_diario
WHERE @puntarenas.STIntersects(
    geography::Point(LAT_CENTRO_DIA, LON_CENTRO_DIA, 4326)
) = 1
GROUP BY PLACA, FECHA;
~~~

### Alertas fuera de autopista

~~~sql
DECLARE @autopista geography =
    geography::Point(9.9900, -84.1500, 4326).STBuffer(3000);

SELECT
    PLACA,
    VELOCIDAD,
    CASE
        WHEN @autopista.STIntersects(geography::Point(LATITUD, LONGITUD, 4326)) = 1
            THEN 'EN AUTOPISTA'
        ELSE 'FUERA DE AUTOPISTA'
    END AS ZONA_ALERTA
FROM gold_eventos_alertas;
~~~

## STIntersection()

~~~sql
DECLARE @zona_C202945 geography =
    geography::Point(10.0631, -84.6247, 4326).STBuffer(30000);

DECLARE @zona_C202946 geography =
    geography::Point(9.9861, -84.2729, 4326).STBuffer(15000);

DECLARE @zona_comun geography =
    @zona_C202945.STIntersection(@zona_C202946);

SELECT
    @zona_comun.STArea() / 1000000.0 AS KM2_ZONA_COMUN;
~~~

## STUnion()

~~~sql
DECLARE @cobertura_C202945 geography =
    geography::Point(10.0631, -84.6247, 4326).STBuffer(30000);

DECLARE @cobertura_C202946 geography =
    geography::Point(9.9861, -84.2729, 4326).STBuffer(15000);

DECLARE @cobertura_total geography =
    @cobertura_C202945.STUnion(@cobertura_C202946);

SELECT
    @cobertura_total.STArea() / 1000000.0 AS KM2_COBERTURA_TOTAL;
~~~

## Resumen

Estas funciones permiten implementar análisis geoespacial avanzado directamente en SQL Server para control de flota, geofencing y detección de anomalías.
