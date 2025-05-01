# Historical US Fires

This is a year by year overview of fires sourced from the data provided by the [NIFC](https://data-nifc.opendata.arcgis.com/search?tags=cy_wildlandfire_opendata%2CCategory)

This dataset only contains fires greater than 1K acres in size.

Processing the data into something the project can use:

```sh
# Break down data by year
mkdir -p wildfire_geojson && for yr in {1984..2024}; do \
  ogr2ogr -f GeoJSON \
    -where "Ig_Date >= '${yr}-01-01' AND Ig_Date < '$((yr+1))-01-01' AND Incid_Type='Wildfire'" \
    wildfire_geojson/us_fires_${yr}.geojson \
    mtbs_perims_DD.shp; \
done
```

```sh
npm i topojson-server

mkdir -p topos && for geo in wildfire_geojson/us_fires_*.geojson; do \
  npx topojson-server -o topos/"$(basename "${geo%.geojson}").topojson" "$geo"; \
done
```

![Zoomed out map](image.png)

![Zoomed in map](image-1.png)
