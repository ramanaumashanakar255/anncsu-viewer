
<script setup lang="ts">
import maplibregl from 'maplibre-gl'
import 'maplibre-gl/dist/maplibre-gl.css'
import { Protocol } from 'pmtiles'
import { onMounted, onUnmounted, watch, watchEffect } from 'vue'
import { storeToRefs } from 'pinia'
import { addGeoJsonLayerAndReturnLegend } from '../services/layer'
import { useAppStore } from '@/store/app.store.ts'
import { executeQuery } from '@/services/duckdb.ts'
import { appConfig } from '@/config'

const dataBase =
  import.meta.env.MODE === 'production'
    ? appConfig.dataBaseUrl
    : 'http://localhost:5173/data'

const parquetFile = `'${dataBase}/anncsu-indirizzi.parquet'`
const pmtilesUrl = `pmtiles://${dataBase}/anncsu-indirizzi.pmtiles`

type Bounds = [[number, number], [number, number]]
type Position = [number, number]

let map: maplibregl.Map | null = null
let protocolRegistered = false

const EMPTY: any = {
  type: 'FeatureCollection',
  features: [],
}

async function getBoundsFromPmtiles(): Promise<Bounds> {
  try {
    const { PMTiles } = await import('pmtiles')
    const pm = new PMTiles(`${dataBase}/anncsu-indirizzi.pmtiles`)
    const header = await pm.getHeader()

    return [
      [header.minLon, header.minLat],
      [header.maxLon, header.maxLat],
    ]
  } catch {
    return [[6.63, 35.49], [18.52, 47.09]]
  }
}

async function getBoundsFromParquet(): Promise<Bounds> {
  try {
    const result = await executeQuery(`
      SELECT
        ST_AsGeoJSON(ST_Point(0,0)) AS geometry,
        value
      FROM parquet_kv_metadata(${parquetFile})
      WHERE key = 'geo'
    `)

    if (result?.features?.length) {
      const meta = JSON.parse(
        (result.features[0].properties as any).value
      )

      const bbox = meta.columns.geometry.bbox

      return [
        [bbox[0], bbox[1]],
        [bbox[2], bbox[3]],
      ]
    }
  } catch {}

  try {
    const result = await executeQuery(`
      SELECT
        ST_AsGeoJSON(ST_Point(0,0)) AS geometry,
        MIN(longitude) AS min_x,
        MIN(latitude) AS min_y,
        MAX(longitude) AS max_x,
        MAX(latitude) AS max_y
      FROM read_parquet(${parquetFile})
    `)

    if (result?.features?.length) {
      const p = result.features[0].properties as any

      return [
        [Number(p.min_x), Number(p.min_y)],
        [Number(p.max_x), Number(p.max_y)],
      ]
    }
  } catch {}

  return [[6.63, 35.49], [18.52, 47.09]]
}

async function getBounds(): Promise<Bounds> {
  return appConfig.isNazionale
    ? getBoundsFromPmtiles()
    : getBoundsFromParquet()
}

function safe(value: unknown): string {
  return String(value ?? 'Not available').replace(
    /[&<>"']/g,
    (c) => ({
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#39;',
    })[c]!
  )
}

// --------------------------------------
// GEOMETRY FUNCTIONS
// --------------------------------------

function buildingParts(feature: any): Position[][][] {
  const geometry = feature.geometry

  if (!geometry) return []

  if (geometry.type === 'Polygon') {
    return [geometry.coordinates]
  }

  if (geometry.type === 'MultiPolygon') {
    return geometry.coordinates
  }

  return []
}

function pointInRing(
  point: Position,
  ring: Position[]
): boolean {
  let inside = false

  for (
    let i = 0, j = ring.length - 1;
    i < ring.length;
    j = i++
  ) {
    const xi = ring[i][0]
    const yi = ring[i][1]
    const xj = ring[j][0]
    const yj = ring[j][1]

    const intersects =
      (yi > point[1]) !== (yj > point[1]) &&
      point[0] <
        ((xj - xi) * (point[1] - yi)) /
          (yj - yi + 1e-12) +
          xi

    if (intersects) inside = !inside
  }

  return inside
}

function pointInPolygon(
  point: Position,
  rings: Position[][]
): boolean {
  if (!rings.length) return false

  const insideOuter = pointInRing(point, rings[0])

  if (!insideOuter) return false

  for (let i = 1; i < rings.length; i++) {
    if (pointInRing(point, rings[i])) {
      return false
    }
  }

  return true
}

function distanceMeters(
  a: Position,
  b: Position
): number {
  const R = 6371000
  const rad = Math.PI / 180

  const lat1 = a[1] * rad
  const lat2 = b[1] * rad
  const dLat = (b[1] - a[1]) * rad
  const dLon = (b[0] - a[0]) * rad

  const h =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1) *
      Math.cos(lat2) *
      Math.sin(dLon / 2) ** 2

  return 2 * R * Math.asin(Math.sqrt(h))
}

function distanceToRing(
  point: Position,
  ring: Position[]
): number {
  let minimum = Infinity

  const latFactor = 111320
  const lonFactor =
    111320 * Math.cos(point[1] * Math.PI / 180)

  for (let i = 0; i < ring.length - 1; i++) {
    const ax = (ring[i][0] - point[0]) * lonFactor
    const ay = (ring[i][1] - point[1]) * latFactor

    const bx =
      (ring[i + 1][0] - point[0]) * lonFactor
    const by =
      (ring[i + 1][1] - point[1]) * latFactor

    const dx = bx - ax
    const dy = by - ay

    const length = dx * dx + dy * dy

    const t =
      length === 0
        ? 0
        : Math.max(
            0,
            Math.min(
              1,
              -(ax * dx + ay * dy) / length
            )
          )

    const x = ax + t * dx
    const y = ay + t * dy

    minimum = Math.min(
      minimum,
      Math.sqrt(x * x + y * y)
    )
  }

  return minimum
}

function polygonCentre(ring: Position[]): Position {
  let area = 0
  let x = 0
  let y = 0

  for (let i = 0; i < ring.length - 1; i++) {
    const cross =
      ring[i][0] * ring[i + 1][1] -
      ring[i + 1][0] * ring[i][1]

    area += cross

    x +=
      (ring[i][0] + ring[i + 1][0]) * cross

    y +=
      (ring[i][1] + ring[i + 1][1]) * cross
  }

  if (Math.abs(area) < 1e-12) {
    const points = ring.slice(0, -1)

    return [
      points.reduce((s, p) => s + p[0], 0) /
        Math.max(points.length, 1),

      points.reduce((s, p) => s + p[1], 0) /
        Math.max(points.length, 1),
    ]
  }

  return [
    x / (3 * area),
    y / (3 * area),
  ]
}

function buildingKey(
  feature: any,
  centre: Position
): string {
  if (feature.id !== undefined && feature.id !== null) {
    return `id-${feature.id}`
  }

  return `xy-${centre[0].toFixed(6)}-${centre[1].toFixed(6)}`
}

// Find the best building for an ANNCSU point.
// A point inside a footprint is preferred.
// Otherwise, a nearby footprint within 12 metres is considered.
function findBuilding(
  point: Position,
  buildings: any[]
): any | null {
  let best: any = null

  for (const feature of buildings) {
    for (const rings of buildingParts(feature)) {
      const outer = rings[0]

      if (!outer || outer.length < 4) continue

      const centre = polygonCentre(outer)

      const inside = pointInPolygon(point, rings)

      const distance = inside
        ? 0
        : distanceToRing(point, outer)

      if (distance > 12) continue

      if (!best || distance < best.distance) {
        best = {
          feature,
          centre,
          distance,
          key: buildingKey(feature, centre),
          labelPosition: inside ? point : centre,
        }
      }
    }
  }

  return best
}

// --------------------------------------
// DYNAMIC ADDRESS / CANDIDATE LAYERS
// --------------------------------------

let previousSignature = ''

function updateAddressAndCandidateLayers(
  m: maplibregl.Map
) {
  if (
    m.getZoom() < 16 ||
    !m.getLayer('places-points') ||
    !m.getLayer('microsoft-buildings-fill')
  ) {
    return
  }

  const addressFeatures = m.queryRenderedFeatures({
    layers: ['places-points'],
  })

  const buildingFeatures = m.queryRenderedFeatures({
    layers: ['microsoft-buildings-fill'],
  })

  const labels: any[] = []
  const unmatchedAddresses: any[] = []
  const candidates: any[] = []

  const matchedBuildings = new Set<string>()
  const addressKeys = new Set<string>()
  const candidateKeys = new Set<string>()

  // Match ANNCSU address points with buildings
  for (const addressFeature of addressFeatures) {
    if (addressFeature.geometry.type !== 'Point') {
      continue
    }

    const p = addressFeature.properties || {}

    const coordinates = (
      addressFeature.geometry as any
    ).coordinates as Position

    const civico = String(p.CIVICO ?? '').trim()
    const esponente = String(p.ESPONENTE ?? '').trim()

    const addressLabel =
      `${civico}${esponente ? ' ' + esponente : ''}`

    const addressKey =
      `${coordinates[0].toFixed(7)}_${coordinates[1].toFixed(7)}_${addressLabel}`

    if (addressKeys.has(addressKey)) continue
    addressKeys.add(addressKey)

    const match = findBuilding(
      coordinates,
      buildingFeatures
    )

    if (match) {
      matchedBuildings.add(match.key)

      if (civico) {
        labels.push({
          type: 'Feature',
          geometry: {
            type: 'Point',
            coordinates: match.labelPosition,
          },
          properties: {
            label: addressLabel,
            street: p.ODONIMO || '',
            municipality: p.NOME_COMUNE || '',
            source: 'ANNCSU',
          },
        })
      }
    } else {
      // No suitable building: keep the real ANNCSU point
      unmatchedAddresses.push({
        type: 'Feature',
        geometry: {
          type: 'Point',
          coordinates,
        },
        properties: {
          label: addressLabel,
          street: p.ODONIMO || '',
          municipality: p.NOME_COMUNE || '',
          source: 'ANNCSU',
        },
      })
    }
  }

  // Find Microsoft buildings without an ANNCSU match
  for (const feature of buildingFeatures) {
    for (const rings of buildingParts(feature)) {
      const outer = rings[0]

      if (!outer || outer.length < 4) continue

      const centre = polygonCentre(outer)
      const key = buildingKey(feature, centre)

      if (matchedBuildings.has(key)) continue

      const candidateKey =
        `${centre[0].toFixed(5)}_${centre[1].toFixed(5)}`

      if (candidateKeys.has(candidateKey)) continue
      candidateKeys.add(candidateKey)

      candidates.push({
        type: 'Feature',
        geometry: {
          type: 'Point',
          coordinates: centre,
        },
        properties: {
          status: 'Building without matched ANNCSU address',
          source: 'Microsoft',
          building_id: feature.id ?? '',
        },
      })
    }
  }

  const result = {
    labels,
    unmatchedAddresses,
    candidates,
  }

  const signature = JSON.stringify(result)

  if (signature === previousSignature) return
  previousSignature = signature

  const labelSource = m.getSource(
    'anncsu-building-labels'
  ) as maplibregl.GeoJSONSource

  const fallbackSource = m.getSource(
    'anncsu-unmatched-points'
  ) as maplibregl.GeoJSONSource

  const candidateSource = m.getSource(
    'microsoft-building-candidates'
  ) as maplibregl.GeoJSONSource

  labelSource?.setData({
    type: 'FeatureCollection',
    features: labels,
  } as any)

  fallbackSource?.setData({
    type: 'FeatureCollection',
    features: unmatchedAddresses,
  } as any)

  candidateSource?.setData({
    type: 'FeatureCollection',
    features: candidates,
  } as any)
}

// --------------------------------------
// MAIN MAP
// --------------------------------------

onMounted(async () => {
  if (!protocolRegistered) {
    const protocol = new Protocol()
    maplibregl.addProtocol('pmtiles', protocol.tile)
    protocolRegistered = true
  }

  const bounds = await getBounds()

  const appBase = new URL(
    import.meta.env.BASE_URL,
    window.location.origin
  )

  const microsoftBuildingsUrl = new URL(
    'data/microsoft-italy.pmtiles',
    appBase
  ).toString()

  map = new maplibregl.Map({
    container: 'map',

    style: {
      version: 8,

      glyphs:
        'https://tiles.openfreemap.org/fonts/{fontstack}/{range}.pbf',

      sources: {
        osm: {
          type: 'raster',
          tiles: [
            'https://tile.openstreetmap.org/{z}/{x}/{y}.png',
          ],
          tileSize: 256,
          attribution:
            '&copy; OpenStreetMap contributors',
        },
      },

      layers: [
        {
          id: 'osm',
          type: 'raster',
          source: 'osm',
          minzoom: 0,
          maxzoom: 19,
        },
      ],
    },

    bounds,
    fitBoundsOptions: { padding: 20 },
    bearing: 0,
  })

  const m = map

  m.addControl(
    new maplibregl.NavigationControl(),
    'bottom-right'
  )

  // --------------------------------------
  // LOAD SOURCES AND LAYERS
  // --------------------------------------

  m.on('load', () => {
    // Microsoft Italy building PMTiles
    m.addSource('microsoft-buildings', {
      type: 'vector',
      url: `pmtiles://${microsoftBuildingsUrl}`,
    })

    // Building polygons
    m.addLayer({
      id: 'microsoft-buildings-fill',
      type: 'fill',
      source: 'microsoft-buildings',
      'source-layer': 'microsoft_buildings',
      minzoom: 14,

      paint: {
        'fill-color': '#d7b994',
        'fill-opacity': 0.65,
      },
    })

    // Building outlines
    m.addLayer({
      id: 'microsoft-buildings-outline',
      type: 'line',
      source: 'microsoft-buildings',
      'source-layer': 'microsoft_buildings',
      minzoom: 14,

      paint: {
        'line-color': '#80654c',
        'line-width': 1.1,
      },
    })

    // Original ANNCSU source
    if (appConfig.isNazionale) {
      m.addSource('anncsu-addresses', {
        type: 'vector',
        url: pmtilesUrl,
      })

      m.addLayer({
        id: 'places-points',
        type: 'circle',
        source: 'anncsu-addresses',
        'source-layer': 'addresses',

        paint: {
          'circle-radius': [
            'interpolate',
            ['linear'],
            ['zoom'],
            6, 2,
            12, 5,
            18, 5,
          ],

          'circle-color': [
            'case',
            ['==', ['get', 'out_of_bounds'], true],
            '#E63946',
            '#4c9b82',
          ],

          'circle-opacity': [
            'interpolate',
            ['linear'],
            ['zoom'],
            14, 1,
            16, 0.15,
            19, 0.1,
          ],

          'circle-stroke-width': 1,
          'circle-stroke-color': '#ffffff',
        },
      })
    }

    // House-number labels
    m.addSource('anncsu-building-labels', {
      type: 'geojson',
      data: EMPTY,
    })

    m.addLayer({
      id: 'anncsu-building-numbers',
      type: 'symbol',
      source: 'anncsu-building-labels',
      minzoom: 16,

      layout: {
        'text-field': ['get', 'label'],
        'text-font': ['Open Sans Bold'],
        'text-size': [
          'interpolate',
          ['linear'],
          ['zoom'],
          16, 10,
          18, 14,
          20, 17,
        ],
        'text-anchor': 'center',
        'text-allow-overlap': true,
        'text-ignore-placement': true,
      },

      paint: {
        'text-color': '#172b20',
        'text-halo-color': '#ffffff',
        'text-halo-width': 2,
      },
    })

    // Green fallback points for ANNCSU addresses
    // that do not have a suitable building
    m.addSource('anncsu-unmatched-points', {
      type: 'geojson',
      data: EMPTY,
    })

    m.addLayer({
      id: 'anncsu-unmatched-points-layer',
      type: 'circle',
      source: 'anncsu-unmatched-points',

      paint: {
        'circle-radius': 7,
        'circle-color': '#16a34a',
        'circle-stroke-width': 2,
        'circle-stroke-color': '#ffffff',
      },
    })

    // Orange candidates for buildings without ANNCSU points
    m.addSource('microsoft-building-candidates', {
      type: 'geojson',
      data: EMPTY,
    })

    m.addLayer({
      id: 'microsoft-building-candidates-layer',
      type: 'circle',
      source: 'microsoft-building-candidates',
      minzoom: 16,

      paint: {
        'circle-radius': 6,
        'circle-color': '#f59e0b',
        'circle-stroke-width': 2,
        'circle-stroke-color': '#ffffff',
      },
    })

    // Recalculate when map tiles finish loading
    m.on('idle', () => {
      updateAddressAndCandidateLayers(m)
    })

    m.on('zoomend', () => {
      updateAddressAndCandidateLayers(m)
    })

    m.on('moveend', () => {
      updateAddressAndCandidateLayers(m)
    })

    // Microsoft building popup
    m.on('click', 'microsoft-buildings-fill', (e) => {
      const f = e.features?.[0]
      if (!f) return

      const p = f.properties || {}

      new maplibregl.Popup()
        .setLngLat(e.lngLat)
        .setHTML(`
          <div style="font-size:13px">
            <b>BUILDING INFORMATION</b>
            <hr/>
            <div>Source: Microsoft</div>
            <div>Building ID: ${safe(f.id)}</div>
            <div>Height: ${safe(p.height)}</div>
            <div>Confidence: ${safe(p.confidence)}</div>
            <div>No official house number is assigned by this layer.</div>
          </div>
        `)
        .addTo(m)
    })

    // Candidate marker popup
    m.on(
      'click',
      'microsoft-building-candidates-layer',
      (e) => {
        const f = e.features?.[0]
        if (!f) return

        new maplibregl.Popup()
          .setLngLat(e.lngLat)
          .setHTML(`
            <div style="font-size:13px">
              <b>BUILDING CANDIDATE</b>
              <hr/>
              <div>Source: Microsoft building footprints</div>
              <div>No matching ANNCSU address point found.</div>
              <div style="color:#b45309">
                Civic number not verified.
              </div>
            </div>
          `)
          .addTo(m)
      }
    )

    m.on('mouseenter', 'microsoft-buildings-fill', () => {
      m.getCanvas().style.cursor = 'pointer'
    })

    m.on('mouseleave', 'microsoft-buildings-fill', () => {
      m.getCanvas().style.cursor = ''
    })

    m.on(
      'mouseenter',
      'microsoft-building-candidates-layer',
      () => {
        m.getCanvas().style.cursor = 'pointer'
      }
    )

    m.on(
      'mouseleave',
      'microsoft-building-candidates-layer',
      () => {
        m.getCanvas().style.cursor = ''
      }
    )
  })

  // --------------------------------------
  // EXISTING POPUPS
  // --------------------------------------

  const popup = new maplibregl.Popup({
    closeButton: false,
    closeOnClick: false,
  })

  m.on('mousemove', 'places-area', (e) => {
    const f = e.features?.[0]
    if (!f) return

    m.getCanvas().style.cursor = 'pointer'

    const p = f.properties || {}

    popup.setLngLat(e.lngLat).setHTML(`
      <div>
        <b>${safe(p.name || 'Area')}</b>
        <div>Count: ${safe(p.count)}</div>
      </div>
    `).addTo(m)
  })

  m.on('mouseleave', 'places-area', () => {
    m.getCanvas().style.cursor = ''
    popup.remove()
  })

  m.on('mousemove', 'places-points', (e) => {
    const f = e.features?.[0]
    if (!f) return

    m.getCanvas().style.cursor = 'pointer'

    const p = f.properties || {}

    const address = p.ODONIMO
      ? `${p.ODONIMO}${p.CIVICO ? ' ' + p.CIVICO : ''}${p.ESPONENTE ? ' ' + p.ESPONENTE : ''}`
      : p.name || 'Punto'

    popup.setLngLat(e.lngLat).setHTML(`
      <div>
        <b>${safe(address)}</b>
        ${
          p.NOME_COMUNE
            ? `<div>${safe(p.NOME_COMUNE)}</div>`
            : ''
        }
        ${
          p.category
            ? `<div>${safe(p.category)}</div>`
            : ''
        }
        ${
          p.out_of_bounds
            ? '<div style="color:red">Fuori confine comunale</div>'
            : ''
        }
      </div>
    `).addTo(m)
  })

  m.on('mouseleave', 'places-points', () => {
    m.getCanvas().style.cursor = ''
    popup.remove()
  })

  // --------------------------------------
  // EXISTING STORE / SEARCH FUNCTIONALITY
  // --------------------------------------

  const store = useAppStore()

  const {
    query,
    selectedCoordinates,
    resetView,
  } = storeToRefs(store)

  if (!appConfig.isNazionale) {
    watchEffect(async () => {
      if (!query.value) return

      const legend = await addGeoJsonLayerAndReturnLegend(
        m,
        query.value
      )

      if (legend) {
        store.setLegend(legend)
      } else {
        store.setQueryError(true)
      }

      store.setQueryLoading(false)
    })
  }

  // Selected address highlight
  watch(selectedCoordinates, (coords) => {
    if (m.getLayer('selected-address-point')) {
      m.removeLayer('selected-address-point')
    }

    if (m.getSource('selected-address')) {
      m.removeSource('selected-address')
    }

    if (coords) {
      m.flyTo({
        center: coords,
        zoom: 18,
      })

      m.addSource('selected-address', {
        type: 'geojson',

        data: {
          type: 'FeatureCollection',

          features: [
            {
              type: 'Feature',

              geometry: {
                type: 'Point',
                coordinates: coords,
              },

              properties: {},
            },
          ],
        },
      })

      m.addLayer({
        id: 'selected-address-point',
        source: 'selected-address',
        type: 'circle',

        paint: {
          'circle-radius': 12,
          'circle-color': '#FF6B35',
          'circle-stroke-width': 2,
          'circle-stroke-color': '#ffffff',
        },
      })
    }
  })

  // Reset map
  watch(resetView, (shouldReset) => {
    if (shouldReset) {
      m.fitBounds(bounds, {
        padding: 20,
      })

      store.clearResetView()
    }
  })
})

onUnmounted(() => {
  map?.remove()
  map = null
})
</script>

<template>
  <div id="map" class="h-full w-full"></div>
</template>

<style scoped>
#map {
  width: 100%;
  height: 100%;
}
</style>
