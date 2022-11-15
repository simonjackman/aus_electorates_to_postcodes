---
title: "NSW 2023 legislative districts to postcodes: correspondence file & mapping application"
title-block-banner: true
description: |
  We derive the correspondence between NSW legislative districts and postcodes via a spatial merge, using the G-NAF and a MapInfo file of district boundaries from the NSW Electoral Commission.  We also provide an interactive tabular summary and map of the correspondence.
author: "Simon Jackman"
date: now
date-format: "h:mmA D MMMM YYYY"
format:
  html:
    theme:
      - cosmo
      - custom.scss
    mainfont: Avenir
    fontsize: 14px
    toc: true
    fig-width: 8
    fig-height: 6
    echo: true
    code-tools: true
    code-fold: true
    smooth-scroll: true
    page-layout: full
    embed-resources: true
    standalone: true
tbl-cap-location: bottom    
crossref:
  tbl-title: Table
knitr: 
  opts_knit: 
    echo: TRUE
    include: TRUE
    warnings: FALSE
    message: FALSE
execute:
  keep-md: true
  warning: false
  error: false
  echo: true
---





# Inputs





Not included in this repo for size and/or license restrictions:

-   NSW Electoral Commission [MapInfo file](https://www.elections.nsw.gov.au/NSWEC/media/NSWEC/Maps/index-maps/StateElectoralDistrict2021_GDA2020.zip)

-   List of NSW addresses from the August 2022 G-NAF (the [Geocoded National Address File](https://geoscape.com.au/data/g-naf-core/)).

For making maps, we use the (very good) approximations to [postcode geographies](https://www.abs.gov.au/statistics/standards/australian-statistical-geography-standard-asgs-edition-3/jul2021-jun2026/non-abs-structures/postal-areas) provided in [shapefiles](https://www.abs.gov.au/statistics/standards/australian-statistical-geography-standard-asgs-edition-3/jul2021-jun2026/access-and-downloads/digital-boundary-files/POA_2021_AUST_GDA94_SHP.zip) published by the Australian Bureau of Statistics.

# Methodology

-   Every GNAF address in NSW is geocoded using [GDA94](https://epsg.io/4939) or [GDA 2020 CRS](https://epsg.io/7844).
-   Locate each geo-coded NSW address from G-NAF in a NSW legislative district, using functions in the [`sf` R package](https://journal.r-project.org/archive/2018/RJ-2018-009/RJ-2018-009.pdf).
-   For each postcode, compute and report count and what proportion of its geo-coded addresses lie in each district.
-   Use [`leaflet`](https://leafletjs.com "https://leafletjs.com") to build a mapping/visualization application.

# Read G-NAF


::: {.cell}

```{.r .cell-code}
gnaf <- read_delim(file = gnaf_core, delim = "|")
nsw_addresses <- gnaf %>%
    rename_with(tolower) %>%
    filter(state == "NSW")
write.fst(nsw_addresses, path = here("data/gnaf_subset.fst"))
rm(gnaf)
```
:::

::: {.cell}

```{.r .cell-code}
nsw_addresses <- read.fst(path = here("data/gnaf_subset.fst"))
nsw_geo <- nsw_addresses %>% 
  distinct(postcode,longitude,latitude)

nsw_geo <- st_as_sf(nsw_geo,
                    coords = c('longitude', 'latitude'),
                    crs = st_crs(CRS("+init=EPSG:7843"))
                    )
```
:::


We subset this file to NSW addresses, yielding 4,772,933 addresses, spanning 622 postcodes and 2,894,120 distinct geo-codes (latitude/longitude pairs).

We now compute the enclosing NSW state legislative district for each address.

# NSW state legislative districts

We read the [MapInfo file](https://www.elections.nsw.gov.au/NSWEC/media/NSWEC/Maps/index-maps/StateElectoralDistrict2021_GDA2020.zip) supplied by the [NSW Electoral Commission](https://www.elections.nsw.gov.au/redistribution/Final-boundaries-and-names); this employs the `GDA2020` CRS and contain the boundaries for NSW's 93 electorate districts, to be used in the 2023 election (gazetted and proclaimed on 26 August 2021).


::: {.cell}

```{.r .cell-code}
nsw_shp <-
  st_read(
    dsn = here("data/StateElectoralDistrict2021_GDA2020.MID"),
    query = sprintf(
      "SELECT objectid, cadid, districtna from StateElectoralDistrict2021_GDA2020"
    )
  )
```

::: {.cell-output .cell-output-stdout}
```
Reading query `SELECT objectid, cadid, districtna from StateElectoralDistrict2021_GDA2020'
from data source `/Users/jackman/Library/CloudStorage/GoogleDrive-simonjackman@icloud.com/Shared drives/C200 TEAM: ANALYTICS/HubSpot migration/aus_electorates_to_postcodes/data/StateElectoralDistrict2021_GDA2020.MID' 
  using driver `MapInfo File'
Simple feature collection with 93 features and 3 fields
Geometry type: MULTIPOLYGON
Dimension:     XY
Bounding box:  xmin: 8714579 ymin: 4024167 xmax: 10446780 ymax: 5045061
Projected CRS: unnamed
```
:::

```{.r .cell-code}
nsw_shp <- st_transform(nsw_shp, crs = CRS("+init=EPSG:7843"))
nsw_shp <- as(nsw_shp, "sf") %>% st_make_valid() 
geojsonio::geojson_write(nsw_shp,file = here("data/nsw_shp.json"))
```

::: {.cell-output .cell-output-stdout}
```
<geojson-file>
  Path:       /Users/jackman/Library/CloudStorage/GoogleDrive-simonjackman@icloud.com/Shared drives/C200 TEAM: ANALYTICS/HubSpot migration/aus_electorates_to_postcodes/data/nsw_shp.json
  From class: geo_list
```
:::
:::


# Read postcode shapefiles

For map-making later on, we read the ABS postal areas (postcode) shapefiles; the coordinates use the `GDA2020` geodetic CRS, which corresponds to `EPSG:7843` CRS used in the other data sets we utilize.


::: {.cell}

```{.r .cell-code}
poa_shp <-
  st_read(
    here("data/POA_2021_AUST_GDA2020_SHP/POA_2021_AUST_GDA2020.shp"),
  )
st_crs(poa_shp) <- CRS("+init=EPSG:7843")
poa_shp <- st_transform(poa_shp, crs = CRS("+init=EPSG:7843"))
poa_shp <- as(poa_shp, "sf") %>% st_make_valid() 
```
:::


# Matching addresses to districts

We now match the unique, geo-coded NSW addresses to districts, the real "work" of this exercise being done by the call to `sf::st_intersects` in the function `pfunc`; we group the data by `postcode` and use processing these batches of data in parallel via `furrr::future_map`.


::: {.cell hash='postcode_nsw_legislative_district_cache/html/st-intersect-work_a5db038b0cf052ca5f20db22c0dbd5bc'}

```{.r .cell-code}
pfunc <- function(obj){
  z <- st_intersects(obj$geometry,nsw_shp)
  return(as.integer(z))
}

nsw_geo <- nsw_geo %>% 
  group_nest(postcode) %>% 
  mutate(
    intersection = future_map(.x = data,
                              .f = ~pfunc(.x))
    ) %>% 
  ungroup()

nsw_geo <- nsw_geo %>% 
  unnest(c(data,intersection))

nsw_geo <- nsw_geo %>% 
  mutate(district = nsw_shp$districtna[intersection])
```
:::


We merge the results back against the GNAF addresses for NSW:


::: {.cell}

```{.r .cell-code}
coords <- st_coordinates(nsw_geo$geometry)
nsw_geo <- nsw_geo %>% 
  mutate(longitude = coords[,1],
         latitude = coords[,2]) %>% 
  select(-geometry) 

nsw_addresses <- left_join(nsw_addresses,
                           nsw_geo,
                           by = c("postcode", "longitude", "latitude"))
rm(nsw_geo,coords)
```
:::


We also filter down to the subset of Australian postcodes that intersect NSW legislative districts:


::: {.cell}

```{.r .cell-code}
poa_shp <- poa_shp %>% 
  semi_join(nsw_addresses %>% distinct(postcode),
            by = c("POA_CODE21" = "postcode")) 
poa_shp_small <- poa_shp %>% 
  st_simplify(preserveTopology = TRUE, dTolerance = 5)

geojsonio::geojson_write(poa_shp_small,file = here("data/nsw_poa_shp.json"))
```
:::


# Counts of addresses by legislative district

We compute counts of addresses with postcodes within districts; we also compute

-   percentage of a district's addresses in a given postcode (`per_of_district`)
-   percentage of a postcode's addresses in a given district (`per_of_postcode`)


::: {.cell}

```{.r .cell-code}
out <- nsw_addresses %>% 
  count(district,postcode) %>% 
  group_by(district) %>% 
  mutate(per_of_district = n/sum(n)*100) %>% 
  ungroup() %>% 
  group_by(postcode) %>% 
  mutate(per_of_postcode = n/sum(n)*100) %>% 
  ungroup() %>% 
  arrange(district,desc(per_of_district))

rm(nsw_addresses)
```
:::

::: {.cell}

```{.r .cell-code}
ojs_define(out_raw=out)
```
:::


# Linked table and map




:::{.cell}

```{.js .cell-code code-fold="undefined" startFrom="227" source-offset="-0"}
out = transpose(out_raw)
viewof theDistrict = Inputs.select(out.map(d => d.district),
    {
      label: "District: ",
      sort: true,
      unique: true
    }
  )
out_small = out.filter(d => d.district == theDistrict)
```

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-1-1 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-1-2 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-1-3 nodetype="declaration"}
:::
:::
:::
:::

:::{.cell}

```{.js .cell-code code-fold="undefined" startFrom="238" source-offset="0"}
Inputs.table(
  out_small,
  {
   format: {
    per_of_district: x => x.toFixed(1),
    per_of_postcode: x => x.toFixed(1)
  },
  rows: out_small.length + 10
  }
)
```

:::{.cell-output .cell-output-display}

:::{#ojs-cell-2 nodetype="expression"}
:::
:::
:::
```{=html}
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.2/dist/leaflet.css"
     integrity="sha256-sA+zWATbFveLLNqWO2gtiw3HL/lh1giY/Inf1BJ0z14="
     crossorigin=""/>
<script src="https://unpkg.com/leaflet@1.9.2/dist/leaflet.js"
     integrity="sha256-o9N1jGDZrf5tS+Ft4gbIK7mYMipq9lqpVJ91xHSyKhg="
     crossorigin=""></script>

<style>
  .leaflet-container {
      font-family: "Avenir", "Helvetica Neue", Arial, Helvetica, sans-serif;
      font-size: 12px;
      font-size: 0.75rem;
      line-height: 1.5;
    }

  .leaflet-tooltip-left:before {
    right: 0;
    margin-right: -12px;
    border-left: 0px;
    border-left-color: rgba(0, 0, 0, 0);
}
.leaflet-tooltip-right:before {
    left: 0;
    margin-left: -12px;
    border-right: 0px;
    border-right-color: rgba(0, 0, 0, 0);
    }
.leaflet-tooltip-own {
    position: absolute;
    padding: 4px;
    background-color: rgba(0, 0, 0, 0);
    border: 0px solid #000;
    color: #000;
    white-space: nowrap;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    pointer-events: none;
    box-shadow: 0 1px 3px rgba(0,0,0,0.4);
}
</style>     
```

:::{.cell}

```{.js .cell-code code-fold="undefined" startFrom="295" source-offset="0"}
nsw_shp_json = await FileAttachment("nsw_shp.json").json()
```

:::{.cell-output .cell-output-display}

:::{#ojs-cell-3 nodetype="declaration"}
:::
:::
:::

:::{.cell}

```{.js .cell-code code-fold="undefined" startFrom="300" source-offset="-0"}
width = 800
height = 1000
poa_shp_json = FileAttachment("nsw_poa_shp.json").json()

thePostcodes = out_small.map(d => d.postcode)
```

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-4-1 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-4-2 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-4-3 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-4-4 nodetype="declaration"}
:::
:::
:::
:::

:::{.cell}

```{.js .cell-code code-fold="undefined" startFrom="308" source-offset="-0"}
lat_default = -33.8727778
long_default = 151.2258333
L = require('leaflet@1.9.2')

map2 = {
  let container = DOM.element ('div', { style: `width:${width}px;height:${height}px` });
  yield container;
  
  let map = L.map(container)
  let osmLayer = L.tileLayer('https://stamen-tiles-{s}.a.ssl.fastly.net/toner/{z}/{x}/{y}{r}.{ext}', {
      attribution: 'Map tiles by <a href="http://stamen.com">Stamen Design</a>, <a href="http://creativecommons.org/licenses/by/3.0">CC BY 3.0</a> &mdash; Map data &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
  subdomains: 'abcd',
	minZoom: 0,
	maxZoom: 25,
	ext: 'png'
  }).addTo(map);
  
  function districtFilter(feature) {
    if(feature.properties.districtna === theDistrict) return true
  }
  
  function postCodeFilter(feature) {
    if(thePostcodes.includes(feature.properties.POA_CODE21)) return true;
  }

  function style(feature) {
    return {
        fillColor: "blue",
        weight: 2,
        opacity: 0.7,
        color: 'blue',
        fillOpacity: 0.0
    };
  }

  // highlight function
  function highlightFeature(e) {
    var layer = e.target;

    layer.setStyle({
        fillOpacity: 0.5,
        opacity: 1.0,
        weight: 3
    });
    
    layer.openTooltip();
    
    layer.bringToFront();
  }

  // mouseout
  function resetHighlight(e) {
    poaLayer.resetStyle(e.target);
  }
  
  function onEachFeature(feature, layer) {
    layer.bindTooltip("<div style='background:white; padding:1px 3px 1px 3px'><b>" + feature.properties.POA_CODE21 + "</b></div>",
                     {
                        direction: 'right',
                        permanent: false,
                        sticky: true,
                        offset: [10, 0],
                        opacity: 1,
                        className: 'leaflet-tooltip-own'
                     });
    layer.on({
        mouseover: highlightFeature,
        mouseout: resetHighlight
    });
  }

  let DistLayer  = L.geoJson(nsw_shp_json, 
    {
      filter: districtFilter,
      weight: 5, 
      color: "#cc00007f",
    }).bindPopup(function (Layer) {
        return Layer.feature.properties.districtna;
    }).addTo(map);

  let poaLayer = L.geoJson(poa_shp_json,
   {
      filter: postCodeFilter,
      style : style,
      onEachFeature: onEachFeature
    })
    .addTo(map);

  map.fitBounds(DistLayer.getBounds());
}
```

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-5-1 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-5-2 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-5-3 nodetype="declaration"}
:::
:::
:::

:::{.cell-output .cell-output-display}

:::{}

:::{#ojs-cell-5-4 nodetype="declaration"}
:::
:::
:::
:::



# Write to file


::: {.cell}

```{.r .cell-code}
write.csv(out, file = here("aux_data/nsw/district_postcode_counts.csv"))
```
:::