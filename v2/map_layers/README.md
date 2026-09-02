# Capas geográficas de esta carpeta

Los `.json` de esta carpeta (`puebla.min.geojson` y los 217 `Pue_M{número}.min.json`, uno por
municipio) son versiones **simplificadas** de los límites geográficos de Puebla, usadas por
[`RegionalizacionTerritorialView.vue`](../../../../src/views/20242030/RegionalizacionTerritorialView.vue).

Los originales (sin simplificar) viven en el repo externo `sdti-ippi.github.io/SIEPI`
(`.../multimedia/20192024/map_layers/`). Se generó una copia local más liviana porque:

- El límite estatal original pesaba **1.07 MB** (38,106 puntos) para un solo polígono.
- Los 217 municipios juntos pesaban **8.48 MB** (343,007 puntos).
- La vista tiene un `maxScale` limitado (no se puede hacer zoom infinito), así que ese nivel de
  detalle nunca se llega a apreciar — es peso de red y de parseo desperdiciado.

Después de simplificar, todo junto pesa **~0.44 MB** (~20x menos), sin pérdida visible de forma.

## La herramienta: mapshaper

[mapshaper](https://mapshaper.org) es un CLI (y también una web app) para simplificar,
convertir y editar datos geográficos (GeoJSON, Shapefile, TopoJSON, etc.). No hace falta
instalarlo: se corre directo con `npx`, que lo descarga solo la primera vez.

```bash
npx mapshaper --version
```

## El comando que se usó

```bash
mapshaper Pue_M*.json combine-files -simplify 5% keep-shapes -o carpeta_salida/ format=geojson
```

Desglosado:

| Parte | Qué hace |
|---|---|
| `Pue_M*.json` | Todos los archivos de entrada (uno por municipio). También funciona con un solo archivo, como el límite estatal. |
| `combine-files` | **La parte importante.** Ver la sección de abajo — evita huecos entre municipios vecinos. |
| `-simplify 5%` | Reduce los polígonos a un 5% de su "resolución" original (menos puntos, misma silueta general). Ver [nivel de simplificación](#eligiendo-el-nivel-de-simplificación) más abajo. |
| `keep-shapes` | Evita que un polígono muy pequeño desaparezca por completo al simplificar (sin esto, un municipio chico podría quedar reducido a nada). |
| `-o carpeta_salida/ format=geojson` | Exporta cada capa a su **propio archivo**, conservando el nombre original (`Pue_M001.min.json` sigue siendo `Pue_M001.min.json`). |

## Por qué `combine-files` (y no `batch-mode`)

Si le pasas varios archivos a mapshaper sin decirle qué hacer, te va a advertir algo como:

```
Note: implicit batch processing is deprecated. Add `batch-mode` to keep this
behavior, or `combine-files` to import the files as a group of layers.
```

Son dos modos muy distintos:

- **`batch-mode`**: procesa cada archivo **por separado**, como si corrieras mapshaper una vez
  por archivo. Cuando dos municipios vecinos comparten un borde, cada uno simplifica *su copia*
  de ese borde de forma independiente — el resultado no coincide exactamente en ambos lados y
  quedan **huecos** (o encimes) triangulares en las intersecciones. Esto es justo lo que pasó la
  primera vez que se generaron estos archivos.

- **`combine-files`**: importa todos los archivos como capas de **una sola topología
  compartida**. mapshaper detecta que el borde entre, por ejemplo, Acajete y su vecino es la
  *misma línea* (el mismo "arco") y la simplifica **una sola vez** para ambos — así el borde
  compartido siempre coincide perfectamente, sin importar qué tanto se simplifique.

Para una sola capa (como `puebla.min.geojson`, un único polígono) no hay borde compartido con
nada más, así que `combine-files` o `batch-mode` dan el mismo resultado — pero para varios
archivos que comparten frontera, **siempre usar `combine-files`**.

## Eligiendo el nivel de simplificación

`-simplify 5%` es un porcentaje **relativo** a cada forma (no una distancia fija), así que
conviene probar en la forma más pequeña del set antes de aplicarlo a todo — es la más sensible a
perder su silueta:

```bash
# cuenta puntos rápido para encontrar la más chica
node -e "
const fs=require('fs');
fs.readdirSync('.').filter(f=>f.endsWith('.json')).forEach(f=>{
  const g=JSON.parse(fs.readFileSync(f));
  console.log(f, JSON.stringify(g).match(/\[-?\d+\.\d+,-?\d+\.\d+\]/g)?.length||0);
});
"
```

Referencia de lo que se probó en esta carpeta (217 municipios, municipio más chico = 110 puntos
originales):

| `-simplify` | Peso total | Municipio más chico | Resultado |
|---|---|---|---|
| 10% | 0.80 MB | 12 pts | bien |
| **5%** ✅ | **0.44 MB** | 7→13 pts* | bien — nivel usado aquí |
| 3% | 0.28 MB | 5 pts | la silueta se pierde (queda un pentágono genérico) |
| 1–2% | 0.13–0.20 MB | 4 pts | irreconocible |

\* con `combine-files` el conteo cambia un poco respecto a simplificar cada archivo aparte,
porque los puntos de los bordes compartidos ahora se cuentan una sola vez.

También existe `-simplify dp interval=100m` (simplificar por distancia fija en vez de
porcentaje) — se probó pero dio *peor* resultado aquí, porque la mayoría de los municipios
tienen curvas de terreno reales por encima de esos umbrales, así que terminó pesando más que el
5% por porcentaje.

## Verificar el resultado

1. **Cuantitativo**: comparar tamaño en bytes y cantidad de puntos antes/después (scripts arriba).
2. **Visual — silueta**: renderizar el original vs. el simplificado lado a lado (un script chico
   con `path` SVG y las coordenadas normalizadas al bounding box es suficiente).
3. **Visual — huecos entre municipios**: renderizar todos los municipios de una región **juntos**
   sobre un fondo de color muy distinto al relleno (rojo de fondo + polígonos verdes, por
   ejemplo). Cualquier hueco en una intersección se ve inmediatamente como una línea del color
   de fondo.
4. **En la app real**: correr `npm run serve`, abrir `/regionalizacion-territorial` y revisar con
   zoom que los bordes entre municipios vecinos queden sin separaciones.

## Regenerar estos archivos

```bash
# 1. Descargar los originales (ejemplo con los municipios; el límite estatal es un solo archivo)
#    ver files/layers.json para la lista completa de URLs (campo "url")

# 2. Simplificar con topología compartida
npx mapshaper municipios_raw/*.json combine-files -simplify 5% keep-shapes \
  -o municipios_simplificados/ format=geojson

# 3. Copiar el resultado a esta carpeta (public/assets/20242030/map_layers/)
```

El código (`RegionalizacionTerritorialView.vue`) arma la URL de cada municipio a partir de su
`number` (`Pue_M{number}.min.json`) — no depende del campo `url` de `layers.json`, así que basta
con que los archivos existan aquí con ese nombre.
