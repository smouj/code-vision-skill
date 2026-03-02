name: code-vision
description: Motor de visualización interactiva de estructura de código y mapeo de dependencias
version: 2.4.1
author: SMOUJBOT
type: analysis
dependencies:
  - graphviz>=2.50.0
  - python3>=3.9
  - pygments>=2.14.0
  - networkx>=3.0
  - matplotlib>=3.7.0
  - clang>=14.0 (opcional, para C/C++)
required_env:
  - CV_OUTPUT_DIR: ${HOME}/.openclaw/code-vision/output
  - CV_MAX_NODES: 500
optional_env:
  - CV_THEME: dark|light (predeterminado: dark)
  - CV_ENGINE: dot|neato|fdp|sfdp|twopi|circo
  - CV_DEBUG: 0|1 (predeterminado: 0)
tags:
  - visualization
  - architecture
  - graphs
  - dependencies
  - structure
  - analysis
```

# Code Vision Skill

Motor de visualización interactiva de estructura de código y mapeo de dependencias que genera grafos en tiempo real de relaciones de código, cadenas de importación/exportación y patrones arquitectónicos.

## Purpose

Casos de uso reales:
- **Análisis de dependencias de módulos**: Detectar dependencias circulares en proyectos Python/TypeScript/Go antes del merge
- **Detección de desviación arquitectónica**: Comparar el grafo de dependencias real contra la arquitectura por capas prevista
- **Análisis de impacto**: Visualizar todos los archivos afectados al modificar un módulo central (ej. modificar servicio de autenticación)
- **Incorporación al codebase**: Generar mapas de estructura de alto nivel para nuevos desarrolladores que se incorporan a monorepos grandes
- **Identificación de deuda técnica**: Encontrar "módulos dios" con >50 importaciones/exportaciones a través de múltiples directorios
- **Validación de límites de microservicios**: Asegurar que servicios en `services/` no importen accidentalmente de otros servicios
- **Auditoría de salud de paquetes**: Identificar módulos huérfanos con cero dependencias entrantes

## Alcance

### Comandos Principales

**`code-vision analyze <path>`**
- Analiza codebase y genera grafo de dependencias interactivo
- `--lang <python|ts|js|go|java|rust|cpp>` (auto-detectado si se omite)
- `--output <formato>`: `html` (predeterminado), `svg`, `png`, `json`, `dot`
- `--depth <n>`: Seguir dependencias hasta N niveles profundos (predeterminado: 3, máximo: 10)
- `--filter <patrón>`: Solo incluir archivos que coincidan con patrón glob (ej. `src/**/*.ts`)
- `--exclude <patrón>`: Excluir archivos que coincidan con patrón (ej. `node_modules`, `*.test.*`)
- `--group-by <dir|namespace|file>`: Estrategia de agrupación de nodos (predeterminado: dir)
- `--cluster`: Habilitar clustering por detección de comunidades (método Louville)
- `--min-degree <n>`: Ocultar nodos con < n conexiones (predeterminado: 1)
- `--focus <archivo>`: Resaltar ruta hacia archivo específico y sus dependientes/dependencias
- `--layout <motor>`: Sobrescribir variable de entorno CV_ENGINE para esta ejecución

**`code-vision compare <commit1> <commit2>`**
- Comparar grafos de dependencias entre commits de git
- `--metric <cyclomatic|imports|exports|coupling>`: Qué comparar (predeterminado: imports)
- `--threshold <0-1>`: Solo mostrar cambios por encima del umbral de similitud (predeterminado: 0.1)
- `--output <html|terminal>`: Formato de diff visual (predeterminado: html)

**`code-vision serve <puerto>`**
- Lanzar dashboard web interactivo (puerto predeterminado: 8080)
- `--host <ip>`: Dirección de enlace (predeterminado: 127.0.0.1)
- `--reload`: Observar filesystem y actualizar automáticamente
- `--open`: Abrir navegador automáticamente

**`code-vision export <id-grafo>`**
- Exportar grafo previamente generado a varios formatos
- `<id-grafo>`: UUID de análisis anterior o `latest`
- `--format <svg|png|pdf|json|dot|gexf>`
- `--zoom <1.0-5.0>`: Factor de escala para formatos rasterizados
- `--with-legend`: Incluir leyenda del grafo en la salida

**`code-vision cycles <ruta>`**
- Detectar dependencias circulares con rastreo de ruta exacto
- `--min-cycle <3>`: Longitud mínima de ciclo a reportar (predeterminado: 3)
- `--format <text|json|dot>`: Formato de salida (predeterminado: text)
- `--fix-suggestions`: Mostrar refactorización sugerida para romper ciclos

### Comandos de Soporte

**`code-vision metrics <ruta>`**
- Generar reporte de texto: nodos totales, aristas, densidad, grado promedio, máximo fan-in/fan-out
- `--top <n>`: Mostrar top N módulos más conectados (predeterminado: 10)
- `--json`: Salida legible por máquina

**`code-vision validate <archivo-arquitectura>`**
- Validar codebase actual contra restricciones arquitectónicas definidas en YAML/JSON
- Ejemplo de archivo de arquitectura:
  ```yaml
  layers:
    - presentation: ['ui/', 'components/']
    - domain: ['core/', 'models/']
    - infrastructure: ['services/', 'repositories/']
  rules:
    - \"presentation -> domain\"
    - \"domain -> infrastructure\"
    - \"forbidden: presentation -> infrastructure\"
  ```

## Proceso de Trabajo Detallado

**1. Descubrimiento Pre-análisis**
```
% code-vision analyze src/ --lang python --output json
```
- Escanea filesystem usando `find` con patrones `--include` por lenguaje
- Extrae importaciones usando:
  - Python: módulo `ast` (seguro), fallback a regex para errores de sintaxis
  - TypeScript/JS: parser `esprima`, recoge `import`, `require`, `export`
  - Go: `go list -json` + parser personalizado para importaciones
  - Java: `javap` o regex para sentencias `import`
  - Rust: `cargo metadata` + parsing con crate `syn`
  - C/C++: `clang -Xclang -ast-dump` para includes
- Construye grafo dirigido: nodo = archivo/módulo, arista = importación/dependencia

**2. Limpieza y Filtrado del Grafo**
- Elimina:
  - Dependencias externas/stdlib (configurable via `--external`)
  - Archivos de test (a menos que se use bandera `--include-tests`)
  - Nodos con grado 0 (inalcanzables) o > `CV_MAX_NODES`
  - Aristas duplicadas (colapso de multi-arista)
- Aplica filtro `--min-degree`

**3. Disposición y Renderizado**
- Usa motores de disposición de Graphviz:
  - `dot`: Jerárquico (predeterminado para arquitecturas por capas)
  - `neato`: Modelo de resorte (predeterminado para grafos no dirigidos o complejos)
  - `fdp`: Dirigido por fuerza para grafos grandes (>200 nodos)
  - `sfdp`: Dirigido por fuerza escalable para grafos muy grandes (>1000 nodos)
  - `twopi`: Disposición radial (bueno para núcleo vs periferia)
  - `circo`: Disposición circular (para arquitecturas en anillo)
- Esquema de colores (tema oscuro predeterminado):
  - Azul: capa de presentación
  - Verde: lógica de negocio/dominio
  - Naranja: infraestructura/persistencia
  - Rojo: utilidades/compartidas
  - Púrpura: dependencias externas
  - Gris: no clasificado
- Tamaño de nodo proporcional a fan-in + fan-out

**4. Generación de Salida**
- HTML: Visualización interactiva D3.js con:
  - Pan/zoom
  - Click en nodo para resaltar vecinos
  - Hover para tooltip con tamaño de archivo, LOC, conteo de importaciones
  - Caja de búsqueda para filtrar nodos
  - Alternar leyenda
- SVG/PNG: Exportación estática de alta resolución
- JSON: Datos del grafo para herramientas descendientes
- DOT: Fuente para ajustes manuales de Graphviz

**5. Dashboard Interactivo (`serve`)**
- Backend Flask/FastAPI en puerto 8080
- Sirve:
  - `/`: Dashboard con historial de grafos
  - `/graph/<uuid>`: Renderizar análisis específico
  - `/api/analyze`: Disparar nuevo análisis vía HTTP POST
  - `/api/compare`: Comparar dos análisis
- WebSocket para actualizaciones en vivo en modo `--reload`

## Reglas de Oro

1. **Siempre excluir `node_modules`, `vendor`, `target`, `build`, `.git` por defecto** a menos que se sobrescriba explícitamente
2. **Respetar límites de lenguaje**: No inferir dependencias entre ecosistemas de lenguajes a menos que se proporcione bandera `--cross-lang`
3. **Limitar tamaño del grafo**: Aplicar límite duro `CV_MAX_NODES` (predeterminado 500). Sugerir `--filter` si se excede.
4. **No destructivo**: Nunca modificar archivos fuente. Todos los artefactos escritos en `CV_OUTPUT_DIR` (predeterminado: `~/.openclaw/code-vision/output/` con subdirectorios timestampeados)
5. **Seguridad**: No parsear archivos con extensiones sospechosas (`.exe`, `.bin`, `.so`) más allá de dump hex para verificación de entropía
6. **Rendimiento**: Abortar análisis si archivo individual > 10MB (probablemente binario). Advertir y omitir.
7. **Git-aware**: Cuando se usa `code-vision compare`, verificar que ambos commits existan localmente; fetch desde remoto si está configurado
8. **Detección de lenguaje**: Usar extensión de archivo primero, luego shebang, luego detección por contenido. Siempre registrar lenguaje detectado.
9. **Detección de ciclos**: Usar algoritmo de Johnson (todos los ciclos simples) pero limitar resultados a top 50 ciclos más largos para evitar explosión combinatoria
10. **Salida determinística**: Ejecuciones de Graphviz deben sembrar generadores aleatorios consistentemente para diseños reproducibles entre ejecuciones en código sin cambios

## Ejemplos

### Ejemplo 1: Encontrar dependencias circulares en microservicio Python
```bash
% code-vision cycles services/auth/ --lang python --format text
Dependencias circulares detectadas (3 ciclos):

Ciclo 1: services/auth/models.py -> services/auth/api.py -> services/auth/decorators.py -> services/auth/models.py
Ciclo 2: shared/utils.py -> services/auth/validators.py -> shared/utils.py

Romper ciclo #1:
  - Mover lógica de decorators.py a módulo separado
  - O inyectar decorators como callable en lugar de importar

--file-count: 127
--cycle-count: 2
```

### Ejemplo 2: Generar grafo HTML interactivo para frontend TypeScript
```bash
% code-vision analyze apps/web/ --lang ts --output html \
  --filter 'apps/web/src/**/*.tsx' \
  --exclude '**/*.test.*,**/*.spec.*' \
  --depth 2 \
  --cluster \
  --focus apps/web/src/components/Button.tsx \
  --theme dark

Análisis completo: 234 nodos, 512 aristas
Salida guardada en: /home/user/.openclaw/code-vision/output/20260101_143022/graph.html
Grafo interactivo: file:///home/user/.openclaw/code-vision/output/20260101_143022/graph.html
```

### Ejemplo 3: Detectar violaciones arquitectónicas contra diseño por capas
Dado `architecture.yaml`:
```yaml
layers:
  - ui: ['src/ui/', 'src/components/']
  - domain: ['src/core/', 'src/models/']
  - infra: ['src/services/', 'src/repositories/']
rules:
  - \"ui -> domain\"
  - \"domain -> infra\"
  - \"forbidden: ui -> infra\"
```

```bash
% code-vision validate architecture.yaml
Resultado de validación: FALLÓ
Violaciones:
  1. ui/src/components/Navigation.tsx -> infra/src/services/api.ts (importación directa prohibida)
  2. infra/src/services/auth.ts -> domain/src/models/User.ts (dependencia inversa)
  3. domain/src/models/Order.ts -> infra/src/repositories/ (infra no debe ser importado por dominio)
  3 violaciones encontradas. Considerar refactorización a inversión de dependencias.
```

### Ejemplo 4: Comparar cambios de dependencias después de refactor
```bash
% code-vision compare abc123def def456ghi --metric coupling --threshold 0.05
Comparando commits:
  Base:  abc123def (2025-12-01)
  Head:  def456ghi (2025-12-02)

Cambios de acoplamiento significativos (>5% umbral):
  - src/core/Engine.ts: +12 importaciones (de 8 a 20) [CRÍTICO]
  - src/utils/helpers.ts: -9 exportaciones (de 15 a 6) [OK]
  - src/repositories/*: No hay cambio significativo

Recomendación: Revisar aumento de fan-in en Engine.ts; puede indicar fuga de responsabilidad.
```

### Ejemplo 5: Servir dashboard en vivo para revisión de equipo
```bash
% code-vision serve 8080 --reload --open
Iniciando dashboard Code Vision...
Escuchando: http://127.0.0.1:8080
Observando: /home/user/projects/myapp (recursivo)
Auto-recarga: habilitada
Abrir navegador: http://127.0.0.1:8080

Dashboard listo. Análisis recientes:
  1. 2026-01-01 14:30 - myapp (analyze) - 234 nodos
  2. 2026-01-01 13:15 - myapp (analyze) - 221 nodos
```

### Ejemplo 6: Identificar "módulos dios" con dependencias excesivas
```bash
% code-vision metrics src/ --top 10 --json
{
  \"total_nodes\": 456,
  \"total_edges\": 1234,
  \"density\": 0.012,
  \"average_degree\": 5.41,
  \"top_modules_by_degree\": [
    {\"file\": \"src/core/Container.ts\", \"degree\": 87, \"type\": \"fan-out\"},
    {\"file\": \"src/utils/Global.ts\", \"degree\": 65, \"type\": \"fan-in\"},
    {\"file\": \"src/services/Api.ts\", \"degree\": 54, \"type\": \"fan-out\"}
  ]
}
```

## Comandos de Rollback

Si un análisis o generación de visualización produce resultados indeseables:

**1. Eliminar ejecución de análisis específica**
```bash
% rm -rf ~/.openclaw/code-vision/output/20260101_143022
# o usar timestamp del más reciente
% rm -rf ~/.openclaw/code-vision/output/$(ls -t ~/.openclaw/code-vision/output | head -1)
```

**2. Restablecer configuración a valores predeterminados**
```bash
% unset CV_OUTPUT_DIR CV_MAX_NODES CV_THEME CV_ENGINE CV_DEBUG
# o editar overrides personalizados:
% nano ~/.openclaw/config/ext/custom-overrides.json
# Eliminar cualquier clave de code-vision y re-ejecutar análisis
```

**3. Limpiar caché de graphviz (si diseños obsoletos)**
```bash
% rm -rf ~/.cache/graphviz/
# Graphviz recreará caché de disposición en siguiente ejecución
```

**4. Deshacer sobrescritura accidental de exportación de grafo**
```bash
# Si usaste --output graph.svg y quieres versión anterior:
% git checkout HEAD -- docs/architecture/graph.svg
# (asumiendo que está trackeado en git)
```

**5. Detener dashboard en ejecución**
```bash
% pkill -f \"code-vision serve\"
# u obtener PID y matar:
% ps aux | grep code-vision
% kill <pid>
```

**6. Restaurar desde snapshot de análisis conocido-bueno**
```bash
% code-vision export snapshot-20251201 --format html --output restored.html
# donde snapshot-20251201 es un graph-id previamente guardado
```

**7. Deshabilitar recarga en vivo si causa problemas de rendimiento**
```bash
# Simplemente omitir bandera --reload en invocaciones subsiguientes
# O configurar variable de entorno:
% export CV_RELOAD=0
```

## Solución de Problemas

**Problema**: `Graphviz engine 'dot' no encontrado`
**Solución**: Instalar graphviz: `apt-get install graphviz` o `brew install graphviz`

**Problema**: `Máximo de nodos (500) excedido. Análisis abortado.`
**Solución**: Usar `--filter` para limitar alcance o aumentar límite:
```bash
% export CV_MAX_NODES=2000
% code-vision analyze . --filter 'src/main/**/*.py'
```

**Problema**: `Error de parser en línea 234 en archivo X.py`
**Solución**: Verificar archivo por errores de sintaxis: `python -m py_compile X.py`. Usar `--skip-errors` para continuar con grafo parcial.

**Problema**: Grafo aparece desconectado o con aristas faltantes
**Solución**: Asegurarse de no excluir archivos que proporcionan importaciones. Verificar patrones `--exclude`. Usar `--include-tests` si archivos de test definen fixtures críticos.

**Problema**: Motor de disposición se cuelga en grafo grande
**Solución**: Cambiar motor: `--layout fdp` para grafos grandes. O usar `--min-degree 2` para podar nodos hoja.

**Problema**: Ciclos reportados pero el código se ve bien
**Solución**: Falsos positivos de importaciones dinámicas (ej. `importlib.import_module`). Usar `--strict` para reportar solo ciclos de análisis estático o `--ignore-dynamic`.

**Problema**: Colores no coinciden con expectativas de capa
**Solución**: Proporcionar YAML de arquitectura personalizada via `code-vision analyze . --architecture mi-arquitectura.yaml`. Definir mapeos de capa.

**Problema**: Dashboard no accesible en red
**Solución**: Enlazar a 0.0.0.0: `code-vision serve 8080 --host 0.0.0.0`. Verificar reglas de firewall.

**Problema**: Memoria agotada durante análisis
**Solución**: Reducir profundidad: `--depth 1`. O analizar subdirectorios por separado. Asegurar `ulimit -n` (archivos abiertos) > 4096.

**Problema**: `pygments` no encontrado, resaltado de sintaxis roto en HTML
**Solución**: `pip install pygments` o `apt-get install python3-pygments`

**Problema**: Comparación de commits de git falla con "revisión desconocida"
**Solución**: Fetch commits faltantes: `git fetch origin <commit>` o asegurar que commit existe localmente. Usar `git log --oneline --all` para verificar.
```