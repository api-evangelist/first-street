---
name: Serve First Street climate raster tiles
description: Overlay flood, wildfire, heat, wind and air-quality raster tiles on a map — with the API key proxied server-side, which the provider requires.
api: First Street Raster Map API
endpoint: https://api.firststreet.org/v2/maps/tile
contract: openapi/first-street-maps-api-openapi.yml
operations: [getRasterTile]
generated: '2026-09-10'
method: generated
source: https://docs.firststreet.org/api/raster-map-api/getting-started
---

# Serve First Street climate raster tiles

256×256 PNG tiles in Web Mercator, addressed by `{z}/{x}/{y}` — a plain XYZ slippy-map
scheme, not an OGC WMS/WMTS service. There is no `GetCapabilities`; the tile catalogue is
in the docs.

## The URL

```
GET https://api.firststreet.org/v2/maps/tile/{peril}/{...product}/{z}/{x}/{y}.png
```

`{...product}` is a variable-length path segment and differs per peril. Worked example for
global flood depth probability, current year, SSP2-4.5, 500-year return period:

```
/v2/maps/tile/globalflood/probability/depth/0/245/500/{z}/{x}/{y}.png
```

| Segment | Meaning | Values |
|---|---|---|
| relative-year | when | `0` (this year), `30` (in 30 years) |
| ssp | scenario | `245` (SSP2-4.5) |
| return-period | likelihood | `500` (0.2%), `100` (1%), `20` (5%), `5` (20%) |

## Step 1 — proxy the key. This is not optional.

The docs are explicit: "When using the map tile service publically, ensure that your API
key is not leaked to the client. Proxy all requests through your own servers." A key in a
tile URL is a key in every browser's network tab.

Client asks your server for `/tiles/{peril}/{...product}/{z}/{x}/{y}.png`; your server
forwards to `https://api.firststreet.org/v2/maps/tile/...?key=$FSF_API_KEY` and streams the
PNG back. Never put the key in front-end code. If a key does leak, contact your account
executive or security@firststreet.org immediately.

## Step 2 — wire the layer

- **Mapbox / Google Maps** — raster tile source against your proxy URL template.
- **ArcGIS Online** — `(+)` → *Add layer from URL*, paste the template, set Type to
  *Tile layer*.
- **iOS MapKit** — `MKTileOverlay(urlTemplate:)`.

Apply roughly **65% opacity**; the docs recommend it for legibility over a base map.

## Step 3 — get the legend from GraphQL, not from the tiles

The colour ramp is data, and it lives on the Climate Risk API:

```graphql
query { flood { mapLegend { colors { color represents { min max } } } } }
```

`represents.min`/`max` give the value range each colour stands for, in the returned unit.
Render your legend from this rather than hard-coding swatches.

## Cost discipline

Tile requests count toward your overall API usage — including tiles pulled by the docs'
own CodePen and JSFiddle examples. Tiles may carry their own contractual rate limit
separate from the GraphQL limit ("a limit of 150 requests per limit may be applied to our
GraphQL API while a limit of 1000/requests per limit may be applied to our Tiles API").
Cache aggressively at your proxy: a tile for a given peril, product, vintage and z/x/y does
not change between model releases.
