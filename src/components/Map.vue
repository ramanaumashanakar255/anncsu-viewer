
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

let map: maplibregl.Map | null = null

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
    const metaQuery = `
      SELECT ST_AsGeoJSON(ST_Point(0,0)) as geometry, value
      FROM parquet_kv_metadata(${parquetFile})
      WHERE key = 'geo'
    `
    const result = await executeQuery(metaQuery)

    if (result && result.features.length > 0) {
      const meta = JSON.parse(
        (result.features[0].properties as any).value
      )
      const bbox = meta.columns.geometry.bbox
      return [[bbox[0], bbox[1]], [bbox[2], bbox[3]]]
    }
  } catch {}

  try {
    const result = await executeQuery(`
      SELECT
        ST_AsGeoJSON(ST_Point(0,0)) as geometry,
        MIN(longitude) as min_x,
        MIN(latitude) as min_y,
        MAX(longitude) as max_x,
        MAX(latitude) as max_y
      FROM read_parquet(${parquetFile})
    `)

    if (result?.features.length) {
      const p = result.features[0].properties as any
      return [[p.min_x, p.min_y], [p.max_x, p.max_y]]
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

function addBuildingLayer(m: maplibregl.Map) {
  if (m.getSource('osm-buildings')) return

  m.addSource('osm-buildings', {
    type: 'vector',
    url: 'https://tiles.openfreemap.org/planet',
  })

  m.addLayer({
    id: 'osm-buildings-fill',
    type: 'fill',
    source: 'osm-buildings',
    'source-layer': 'building',
    minzoom: 15,
    paint: {
      'fill-color': '#d8c7b9',
      'fill-opacity': 0.65,
    },
  })

  m.addLayer({
    id: 'osm-buildings-outline',
    type: 'line',
    source: 'osm-buildings',
    'source-layer': 'building',
    minzoom: 15,
    paint: {
      'line-color': '#927e70',
      'line-width': 1.2,
    },
  })

  m.on('click', 'osm-buildings-fill', (e) => {
    const f = e.features?.[0]
    if (!f) return

    const p = f.properties || {}

    const html = `
      <div style="font-size:13px;min-width:190px">
        <b>BUILDING INFORMATION</b>
        <hr/>
        <div>OSM ID: ${safe(f.id)}</div>
        <div>Building: ${safe(p.class || p.building)}</div>
        <div>Height: ${safe(p.render_height || p.height)}</div>
        <div>Levels: ${safe(p.levels)}</div>
        <div>Address: ${safe(p.housenumber)}</div>
        <div>Street: ${safe(p.street)}</div>
        <hr/>
        <div style="color:#b45309">
          ANNCSU matching requires verification.
        </div>
      </div>
    `

    new maplibregl.Popup()
      .setLngLat(e.lngLat)
      .setHTML(html)
      .addTo(m)
  })

  m.on('mouseenter', 'osm-buildings-fill', () => {
    m.getCanvas().style.cursor = 'pointer'
  })

  m.on('mouseleave', 'osm-buildings-fill', () => {
    m.getCanvas().style.cursor = ''
  })
}

onMounted(async () => {
  if (appConfig.isNazionale) {
    const protocol = new Protocol()
    maplibregl.addProtocol('pmtiles', protocol.tile)
  }

  const bounds = await getBounds()

  map = new maplibregl.Map({
    container: 'map',
    style: {
      version: 8,
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

  m.addControl(new maplibregl.NavigationControl(), 'bottom-right')

  m.on('load', () => {
    // Add building footprints first
    addBuildingLayer(m)

    // Add ANNCSU points above buildings
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
            18, 10,
          ],
          'circle-color': [
            'case',
            ['==', ['get', 'out_of_bounds'], true],
            '#E63946',
            '#4c9b82',
          ],
          'circle-stroke-width': 1,
          'circle-stroke-color': '#ffffff',
        },
      })
    }
  })

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
        ${p.NOME_COMUNE
          ? `<div>${safe(p.NOME_COMUNE)}</div>`
          : ''}
        ${p.category
          ? `<div>${safe(p.category)}</div>`
          : ''}
        ${p.out_of_bounds
          ? '<div style="color:red">Fuori confine comunale</div>'
          : ''}
      </div>
    `).addTo(m)
  })

  m.on('mouseleave', 'places-points', () => {
    m.getCanvas().style.cursor = ''
    popup.remove()
  })

  const store = useAppStore()
  const { query, selectedCoordinates, resetView } = storeToRefs(store)

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

  watch(selectedCoordinates, (coords) => {
    if (m.getLayer('selected-address-point')) {
      m.removeLayer('selected-address-point')
    }

    if (m.getSource('selected-address')) {
      m.removeSource('selected-address')
    }

    if (coords) {
      m.flyTo({ center: coords, zoom: 18 })

      m.addSource('selected-address', {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: [{
            type: 'Feature',
            geometry: {
              type: 'Point',
              coordinates: coords,
            },
            properties: {},
          }],
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

  watch(resetView, (shouldReset) => {
    if (shouldReset) {
      m.fitBounds(bounds, { padding: 20 })
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

<style scoped></style>
