## Plan de 5 días (4 horas / día)

### Día 1 – Fundamentos, modelado ER y entorno de desarrollo

** Objetivo:** Comprender el dominio del problema, diseñar el modelo entidad-relación y levantar el stack base.

#### Teoría (1.5h)

- Introducción a sistemas de información geoespaciales.
    
- Ciclo de vida de una BD: requisitos → conceptual → lógico → físico.
    
- **Modelo entidad-relación** para _FleetTrack_:
    
    - Entidades: `vehiculo`, `conductor`, `ubicacion`, `incidente`, `ruta`.
        
    - Relaciones: `conduce` (1:N), `registra_ubicacion` (1:N), `reporta` (N:1 incidente-conductor).
        
    - Atributos clave: geometrías (`punto` para ubicaciones/incidentes, `línea` para rutas).
        
- Tipos de datos espaciales básicos: `POINT`, `LINESTRING`.
    

#### Práctica (2h)

1. **Configurar entorno:**
    
    ~~~bash
    
    # Levantar PostgreSQL + PostGIS con Docker
    docker run --name postgis -e POSTGRES_PASSWORD=admin123 -p 5432:5432 -d postgis/postgis
    ~~~
2. **Inicializar proyecto Spring Boot** con [Spring Initializr](https://start.spring.io/):
    
    - Dependencias: Spring Web, Spring Data JPA, PostgreSQL Driver.
        
3. **Crear entidades JPA** (solo las primeras):
    
    ~~~java
    
    @Entity
    public class Vehiculo {
        @Id @GeneratedValue private Long id;
        private String patente;
        private String modelo;
        // getters/setters
    }
    ~~~
4. **Configurar `application.properties`** con conexión a PostgreSQL.
    
5. **Generar tablas** automáticamente (`ddl-auto: update`).
    
6. **Insertar datos de prueba** usando `CommandLineRunner`.
    

#### Cierre (0.5h)

- Revisar que cada participante tenga el entorno funcionando.
    
- Tarea: completar el diagrama ER incluyendo todos los atributos y cardinalidades.
    

---

### Día 2 – Modelo lógico, normalización y primeros endpoints con Leaflet

** Objetivo:** Transformar el modelo ER a relacional, normalizar y construir la primera API para mostrar vehículos en un mapa.

#### Teoría (1.5h)

- De ER a tablas: reglas de transformación.
    
- Normalización hasta 3FN (ejemplo con `ubicacion` y `vehiculo`).
    
- **Introducción a PostGIS:**
    
    - Columna `geometry(Point, 4326)` para coordenadas.
        
    - Índices espaciales `GIST`.
        
- Diseño final de tablas:
    
    ~~~sql
    
    CREATE TABLE vehiculo (id SERIAL PRIMARY KEY, patente TEXT, modelo TEXT);
    CREATE TABLE ubicacion (
        id SERIAL PRIMARY KEY,
        vehiculo_id INT REFERENCES vehiculo(id),
        coordenadas GEOMETRY(POINT, 4326),
        timestamp TIMESTAMP
    );
    CREATE INDEX idx_ubicacion_coordenadas ON ubicacion USING GIST (coordenadas);
    ~~~

#### Práctica (2h)

1. **Crear repositorios y servicios** en Spring:
    
    ~~~java
    
    @Repository
    public interface UbicacionRepository extends JpaRepository<Ubicacion, Long> {
        // Obtener última ubicación de cada vehículo (consulta nativa)
        @Query(value = "SELECT DISTINCT ON (vehiculo_id) * FROM ubicacion ORDER BY vehiculo_id, timestamp DESC", nativeQuery = true)
        List<Ubicacion> findUltimasUbicaciones();
    }
    ~~~
    
2. **Endpoint REST** `/api/vehiculos/ultimas-ubicaciones` que devuelve JSON con `{id, patente, lat, lon, timestamp}`.
    
3. **Frontend mínimo** con Leaflet:
    
    ~~~html
    
    <div id="map" style="height: 500px;"></div>
    <script>
        var map = L.map('map').setView([-34.6037, -58.3816], 13);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
        fetch('/api/vehiculos/ultimas-ubicaciones')
          .then(res => res.json())
          .then(data => {
              data.forEach(v => {
                  L.marker([v.lat, v.lon]).addTo(map).bindPopup(v.patente);
              });
          });
    </script>
    ~~~

#### Cierre (0.5h)

- Mostrar mapa con marcadores dinámicos.
    
- Explicar cómo mejorar con actualización periódica.
    

---

### Día 3 – Consultas espaciales y simulación de ubicaciones en tiempo real

** Objetivo:** Implementar el registro continuo de ubicaciones y dibujar trayectorias.

#### Teoría (1.5h)

- Funciones espaciales clave:
    
    - `ST_MakePoint(lon, lat)` → crear geometría.
        
    - `ST_Distance(geom1, geom2)` → distancia en metros.
        
    - `ST_Length(linestring)` → longitud de una ruta.
        
- Transacciones y concurrencia al insertar ubicaciones.
    
- Estrategia para simular movimiento: _setInterval_ desde el frontend.
    

#### Práctica (2h)

1. **Endpoint POST** `/api/ubicaciones`:
    
    ~~~java
    
    @PostMapping
    public Ubicacion registrar(@RequestParam Long vehiculoId,
                               @RequestParam Double lat,
                               @RequestParam Double lon) {
        Ubicacion u = new Ubicacion();
        u.setVehiculo(vehiculoRepository.findById(vehiculoId).get());
        u.setCoordenadas(geometryFactory.createPoint(new Coordinate(lon, lat)));
        u.setTimestamp(LocalDateTime.now());
        return ubicacionRepository.save(u);
    }
    ~~~
    
2. **Endpoint GET** `/api/vehiculos/{id}/trayectoria`:
    
    - Devuelve array de `{lat, lon, timestamp}` ordenado.
        
    - Consulta JPQL o nativa con `ST_AsGeoJSON`.
        
3. **Frontend:**
    
    - Simular envío de ubicación cada 3 segundos (lat/lon aleatorios cerca de un centro).
        
    - Obtener trayectoria y dibujar línea con `L.polyline`.
        
4. **Optimización:** agregar índice espacial en `coordenadas` y en `(vehiculo_id, timestamp)`.
    

#### Cierre (0.5h)

- Ver vehículo moviéndose en el mapa con su ruta histórica.
    
- Discutir sobre _batch inserts_ para alto rendimiento.
    

---

### Día 4 – Incidentes, cercanía y cálculo básico de rutas

** Objetivo:** Gestionar incidentes espaciales y consultar los que están cerca de una posición o ruta.

#### Teoría (1.5h)

- Modelo de incidentes: `id, tipo (accidente/tráfico/cierre), descripcion, coordenadas, estado (activo/resuelto)`.
    
- Consulta de cercanía: `ST_DWithin(geom1, geom2, radio_metros)` + índice GIST.
    
- Cálculo de rutas: si no se usa API externa, predefinir rutas como `LINESTRING` en tabla `ruta` y medir distancias.
    
- Optimización de consultas con `EXPLAIN ANALYZE`.
    

#### Práctica (2h)

1. **Crear entidad `Incidente`** con columna `coordenadas` (Point).
    
2. **Endpoint POST** `/api/incidentes` (reportar incidente desde el mapa).
    
3. **Endpoint GET** `/api/incidentes/cercanos?lat=...&lon=...&radio=500`:
    
    ~~~sql
    
    SELECT * FROM incidente 
    WHERE ST_DWithin(coordenadas, ST_SetSRID(ST_MakePoint(:lon, :lat), 4326), :radio)
    AND estado = 'ACTIVO';
    ~~~
4. **Frontend:** al hacer clic en el mapa, abrir formulario para reportar incidente; mostrar incidentes como marcadores rojos.
    
5. **Rutas fijas:** crear tabla `ruta` con `trazado LINESTRING`, endpoint para obtener todas las rutas y mostrarlas en Leaflet con `L.polyline`.
    
6. **Bono:** consultar incidentes cercanos a una ruta (usando `ST_Distance` entre punto y línea).
    

#### Cierre (0.5h)

- Aplicación funcional: vehículos móviles, incidentes visibles, rutas dibujadas.
    
- Discusión sobre índices espaciales y rendimiento.
    

---

### Día 5 – Optimización, reportes y presentación final

** Objetivo:** Ajustar el rendimiento, generar reportes analíticos y consolidar el proyecto.

#### Teoría (1h)

- Particionamiento por tiempo para tabla `ubicacion` (particiones por mes).
    
- Vistas materializadas para reportes (ej. incidentes por zona).
    
- Estrategias de cache (Spring Cache).
    
- Seguridad básica: API con roles (opcional).
    
- Buenas prácticas: diagrama de la BD, documentación con Swagger.
    

#### Práctica (2.5h)

1. **Crear vista materializada** de resumen diario:
    
    ~~~sql
    
    CREATE MATERIALIZED VIEW reporte_diario_vehiculos AS
    SELECT vehiculo_id, DATE(timestamp) AS dia, COUNT(*) AS total_ubicaciones
    FROM ubicacion GROUP BY vehiculo_id, dia;
    ~~~
2. **Implementar endpoint** `/api/reportes/incidentes-por-zona` que agrupe incidentes dentro de celdas de una grilla (usar `ST_SquareGrid`).
    
3. **Optimizar consulta lenta** agregando índices compuestos y usando `EXPLAIN`.
    
4. **Integración final:** dashboard simple con:
    
    - Mapa con vehículos, rutas e incidentes.
        
    - Lista de incidentes cercanos a un vehículo seleccionado.
        
    - Botón para refrescar posiciones.
        
5. **Presentación** (10 min por grupo): mostrar la funcionalidad, explicar el modelo de BD y una mejora que agregaron.
    

#### Cierre (0.5h)

- Retroalimentación grupal.
    
- Recursos: documentación de PostGIS, curso de Spring Data JPA, Leaflet avanzado.
    
- Desafío adicional: conectar a API de routing real (OSRM) en lugar de rutas fijas.
    

---

## Notas para el instructor

- **Requisitos previos de los alumnos:** Java básico, SQL elemental. No se requiere experiencia en Spring Boot ni GIS.
    
- **Entrega de código base:** puedes proporcionar un esqueleto del proyecto (controladores vacíos, configuración de PostGIS) para acelerar la práctica.
    
- **Simulación de datos reales:** usa el generador de posiciones que camine sobre un eje (ej. Av. Corrientes, Buenos Aires) para que la trayectoria sea coherente.
    
- **Evaluación:** se basa en la completitud del proyecto al final del día 5 (repositorio Git con código y un breve informe del diseño de BD).