# Mapas




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


Manipualacion con `dplyr`


```r
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
  geom_sf(data=zambia, fill="black")
```

Obtenemos limites provinciales con `rgeoboundaries`


```r
zambia_boundaries <- geoboundaries(c("Zambia"), "adm1")
```


```r
zambia_boundaries %>% 
ggplot() +
  geom_sf()
```

Hasta ahora veniamos con graficos estáticos

Veamos uno dinámico con `leaflet`


```r
zambia_boundaries %>%
  leaflet() %>%
  addTiles() %>%
  addPolygons(label = zambia_boundaries$shapeName, weight=1)
```


Agreguemos una variable ficticia 


```r
zambia_boundaries <- zambia_boundaries %>% 
  mutate(y=rnorm(10, 1000, 200))

zambia_boundaries %>% 
  ggplot() +
  geom_sf(aes(fill=y)) +
  scale_fill_viridis(direction=-1) 
  # scale_fill_viridis(option = "plasma") 
```

## Mapa de clima

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
tmax_mean_zam <- raster::mask(tmax_mean, as_Spatial(zambia_sf))
```

Converting the raster object into a dataframe

```r
tmax_mean_zambia_df <- as.data.frame(tmax_mean_zam, xy = TRUE, na.rm = TRUE)
```


```r
tmax_mean_zambia_df %>%
  ggplot(aes(x = x, y = y)) +
  geom_raster(aes(fill = layer)) +
  geom_sf(data = zambia_sf, inherit.aes = FALSE, fill = NA) +
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
mis_corr  <-  getData(country = "ARG", level = 2) %>% 
  st_as_sf() %>% 
  filter(NAME_1 %in% c("Misiones", "Corrientes")) %>% 
  st_transform(crs = 4326)
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
cities <- data.frame(
  x = c(-63.58595, 116.41214), 
  y = c(44.64862, 40.19063), 
  city = c("Halifax", "Beijing")
)

ggplot(cities, aes(x, y)) +
  annotation_map_tile() +
  geom_spatial_point() +
  geom_spatial_label_repel(aes(label = city), box.padding = 1) +
  coord_sf(crs = 3995)
```


Referencias 

- https://afrimapr.github.io/afrimapr.website/
- https://afrimapr.github.io/afrilearnr/
- https://rspatialdata.github.io/
- https://rspatialdata.github.io/admin_boundaries.html
- https://mgimond.github.io/Spatial/index.html
- https://paleolimbot.github.io/ggspatial/articles/ggspatial.html#sample-data-1
- https://docs.ropensci.org/rnaturalearth/
- https://datavizs21.classes.andrewheiss.com/example/12-example/
- https://luisdva.github.io/rstats/mapassf/
  



```r
library(rgeoboundaries)
# https://rspatialdata.github.io/admin_boundaries.html


library(rnaturalearth)
# https://docs.ropensci.org/rnaturalearth/articles/rnaturalearth.html

pacman::p_load(sf)
# https://keen-swartz-3146c4.netlify.app/

pacman::p_load(sf, ggspatial)
# https://paleolimbot.github.io/ggspatial/

pacman::p_load(rgeos, rworldmap)
# https://bookdown.org/angelborrego/ciencia_datos/mapas.html#mapas-con-rworldmap

pacman::p_load(pointdensityP)
# https://mgimond.github.io/Spatial/chp11_0.html
```