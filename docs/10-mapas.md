---
output: html_document
editor_options: 
  chunk_output_type: inline
---
# Mapas

Son muchos los paquetes relacionados a GIS en R con multiplicidad de funciones 

- https://rspatialdata.github.io/ 


```r
library(tidyverse)
library(viridis)
```

Obtención de datos georeferenciaodos


```r
remotes::install_gitlab("dickoa/rgeoboundaries")
library(rgeoboundaries)
library(raster)
library(rnaturalearth)
# install.packages("rnaturalearthdata")
```

Generacion de mapas con sintaxis acoplable a `ggplot`


```r
library(sf)
library(leaflet)
library(htmltools)
library(ggspatial)
```

Hay otros muy interesantes como `tmap`, `mapview` que pueden encontrar mucha informacion en internet

## Paquete `sf`

Obtenemos limites politicos de Africa 

```r
africa <- ne_countries(continent = "Africa", returnclass = "sf", scale = "medium")
africa
```

Primer grafico 


```r
africa %>%  
  ggplot()+
  geom_sf() 
```
- https://docs.qgis.org/2.8/es/docs/gentle_gis_introduction/coordinate_reference_systems.html


```r
africa %>%  
  ggplot()+
  geom_sf() +
  coord_sf(crs = st_crs("ESRI:54030"))   # Robinson
```


```r
africa %>%  
  ggplot()+
  geom_sf() +
  coord_sf(crs = st_crs("EPSG:3395"))   # Mercator
```

Manipualacion con `dplyr`


```r
head(africa)
zambia <- africa %>% filter(sovereignt=="Zambia")
```



```r
zambia %>%  
  ggplot()+
  geom_sf()
```

Una capa por dataframe 


```r
africa %>%  
  ggplot()+
  geom_sf() + 
  geom_sf(data=zambia, fill="gray50")
```

Obtenemos limites provinciales con `rgeoboundaries`


```r
zambia_adm1 <- geoboundaries(c("Zambia"), "adm1")
```


```r
zambia_adm1 %>% 
ggplot() +
  geom_sf()
```


```r
zambia_adm2 <- geoboundaries(c("Zambia"), "adm2")
zambia_adm2
```


```r
zambia_adm2 %>% 
ggplot() +
  geom_sf() #+ 
  # geom_sf_text(aes(label = shapeName), size=2)
  # coord_sf(xlim = c(22, 30),  ylim = c(-10, -16))
```

Hasta ahora veniamos con graficos estáticos

Veamos uno dinámico con `leaflet`


```r
library(htmltools)

zambia_adm2 %>%
  leaflet() %>%
  addTiles() %>%
  addPolygons(label = zambia_adm1$shapeName, weight=1, popup = ~htmlEscape(shapeID))
```

## Choropleth maps

Agreguemos una variable ficticia 


```r
zambia_adm1 <- zambia_adm1 %>% 
  mutate(y=rnorm(10, 1000, 200))

zambia_adm1 %>% 
  ggplot() +
  geom_sf(aes(fill=y)) +
  scale_fill_viridis(direction=-1) 
  # scale_fill_viridis(option = "plasma") 
```

Por ejemplo, podemos practicar esto mismo con datos reales poblacionales con el paquete `wopr`

- https://rspatialdata.github.io/population.html 

## Mapa de clima

https://rspatialdata.github.io/temperature.html

Descarguemos temperatura maxima mensual con `raster`


```r
tmax_data <- getData(name = "worldclim", var = "tmax", res = 10)
```


```r
# Converting temperature values to Celcius
gain(tmax_data) <- 0.1
# Calculating mean of the monthly maximum temperatures
tmax_mean <- mean(tmax_data)
```

Extraemos los datos de zambia


```r
tmax_mean_zam <- raster::mask(tmax_mean, as_Spatial(zambia_adm1))
```

Converting the raster object into a dataframe

```r
tmax_mean_zambia_df <- as.data.frame(tmax_mean_zam, xy = TRUE, na.rm = TRUE)
```


```r
tmax_mean_zambia_df %>%
  ggplot(aes(x = x, y = y)) +
  geom_raster(aes(fill = layer)) +
  geom_sf(data = zambia_adm1, inherit.aes = FALSE, fill = NA) +
  labs(
    title = "Mean monthly maximum temperatures in zambia",
    subtitle = "For the years 1970-2000"
  ) +
  xlab("Longitude") +
  ylab("Latitude") +
  scale_fill_gradient(
    name = "Temperature (°C)",
    low = "#FEED99",
    high = "#AF3301"
  )
```

## Mapa de Muestreos

Supongamos que queremos plotear muestreos de especies realizados entre 2020 y 2021 en las provincias de Misiones y Corrientes

- Descargamos limites politicos a nivel de municipiocon `raster`


```r
# arg_adm2 <- geoboundaries(c("Argentina"), "adm2")
# arg_adm2

mis_corr  <-  getData(country = "ARG", level = 2) %>% 
  st_as_sf() %>% 
  filter(NAME_1 %in% c("Misiones", "Corrientes")) %>% 
  st_transform(crs = 4326)
mis_corr
```


```r
mis_corr %>% 
  ggplot() + 
  geom_sf() +
  theme_minimal()
```

Generamos 120 puntos de muestreo aleatorios, donde solo registramos su especie (A o B), las coordenadas y el tamaño del especimen (media=10, sd=2)


```r
sampling_points <- 
  expand_grid(year=as.integer(2020:2021), sp=LETTERS[1:2], rep=1:30)
sampling_points

sampling_points <- sampling_points %>% 
  mutate(coord=st_sample(mis_corr, 120),    
         size = rnorm(120, 10, 2)) %>% 
  st_sf()  
sampling_points
```


Ahora queremos fusionar con el dataset con los municipios donde cayeron los puntos de muestreo


```r
sampling_points_muni <- st_join(sampling_points, mis_corr)
sampling_points_muni
```

Veamos que tal cayeron los puntos 


```r
ggplot()+
  geom_sf(data=mis_corr)+
  geom_sf(data=sampling_points, aes(col=sp))+
  facet_wrap("year") + 
  theme_void()
      # theme_map() 
```

Y si hubo algun patron en el tamaño de los especimenes registrados 


```r
ggplot()+
  geom_sf(data=mis_corr, col="gray70")+
  geom_sf(data=sampling_points, aes(color=size))+
  facet_grid(sp~year) +
  scale_color_viridis_c() +
  theme_minimal()
```

Y algunos graficos con medidas aritmeticas 


```r
sampling_points_muni %>%  
  count(year, NAME_1, sp) %>% 
  ggplot() + 
  aes(x=year, y=n, col=sp) +
  geom_line() +
  facet_wrap("NAME_1")
```


```r
sampling_points_muni %>%  
  count(NAME_1, NAME_2, sp) %>% 
  ggplot() + 
  aes(x=NAME_2, y=n, fill=sp) +
  geom_bar(position="stack", stat="identity") + 
  facet_wrap("NAME_1", scales = "free")+
  coord_flip()
```


Y si no nos interesa los municipios y tenemos los datos exprtados de una planilla de excel...

El paquete `ggspatial` nos provee un mapa base 


```r
samp_xls <- sampling_points %>% 
  mutate(lon = sf::st_coordinates(.)[,1],
               lat = sf::st_coordinates(.)[,2]) %>% 
  as_tibble() %>%  
  dplyr::select(-coord)

samp_xls

samp_xls %>% 
  ggplot(aes(x = lon, y = lat, col=sp)) +
  annotation_map_tile(type = "cartolight",
                      zoomin = 0) +
  geom_spatial_point() +
  # ylim(c(-52, 40)) +
  # xlim(c(-115, -40)) +
  facet_wrap("year") + 
  guides(size="none") + 
  coord_sf(crs = 4326)
```



```r
library(sf)

cities <- data.frame(
  longitude = c(-58.3612, -58.3173, -58.2940, -58.7228, -57.9556, -58.2395), 
  latitude  = c(-38.0165, -37.9428, -37.5447, -37.6301, -37.8666, -37.6467), 
  city = c("San Agustin", "Los Pinos", "Ramos Otero", "Napaleofu", "La Brava", "Bosch")
) 
cities %>% 
ggplot( aes(longitude, latitude)) +
  annotation_map_tile(zoom = 10) +
  geom_spatial_point() +
  geom_spatial_label_repel(aes(label = city))+
  fixed_plot_aspect() 

# coord_sf(crs = 3995)
```


Referencias 

- https://afrimapr.github.io/afrimapr.website/
- https://afrimapr.github.io/afrilearnr/
- https://mgimond.github.io/Spatial/index.html
- https://paleolimbot.github.io/ggspatial/articles/ggspatial.html#sample-data-1
- https://docs.ropensci.org/rnaturalearth/
- https://datavizs21.classes.andrewheiss.com/example/12-example/
- https://luisdva.github.io/rstats/mapassf/
- https://bookdown.org/nicohahn/making_maps_with_r5/docs/introduction.html
  

```r
library(rgeoboundaries)
# https://rspatialdata.github.io/admin_boundaries.html

library(rnaturalearth)
# https://docs.ropensci.org/rnaturalearth/articles/rnaturalearth.html

library(sf)
# https://keen-swartz-3146c4.netlify.app/

library(ggspatial)
# https://paleolimbot.github.io/ggspatial/

library(rworldmap)
# https://bookdown.org/angelborrego/ciencia_datos/mapas.html#mapas-con-rworldmap

library(pointdensityP)
# https://mgimond.github.io/Spatial/chp11_0.html
```
