# Heat Map de Perú en R

Este proyecto muestra cómo construir un **mapa de calor** de los departamentos del Perú utilizando R.  

---

## Flujo general

El proceso completo sigue estos pasos:

1. Cargar librerías
2. Leer shapefile (geometría del Perú)
3. Leer base de datos (valores por departamento)
4. Limpiar y transformar los datos
5. Unir datos espaciales y estadísticos
6. Visualizar el mapa de calor

---

## Librerías necesarias

```r
library(sf)         # Manejo de datos espaciales (geometría + atributos)
library(purrr)      # Programación funcional (útil en pipelines)
library(tidyverse)  # Manipulación de datos (dplyr, readr, etc.)
library(ggplot2)    # Visualización
library(ggrepel)    # Mejora de etiquetas (opcional)
library(readxl)     # Lectura de Excel (opcional)
library(readr)      # Lectura de CSV
```

---
## Otros Recursos
- La data del presente heatmap fue obtenida de la Plataforma Nacional de Datos Abiertos (https://www.gob.pe/datosabiertos) 
- El shapefile del Peru descargado de GEO GPS PERÚ (https://www.geogpsperu.com/2014/03/base-de-datos-peru-shapefile-shp-minam.html)

---

## Cargar shapefile de Perú

Cargar el shapefile utilizando la dirección en la que fue guardada

```r
dirmapas <- "C:/Users/fabia/OneDrive/Desktop/cursos_R/biostats/DEPARTAMENTOS_inei_geogpsperu_suyopomalia"
setwd(dirmapas)

peru_d<-st_read("DEPARTAMENTOS_inei_geogpsperu_suyopomalia.shp")
```

Al usar `st_read()`, obtienes un **dataframe espacial** donde cada fila es un departamento y una columna especial (`geometry`) almacena su forma.

---

## Verificación del mapa base

```r
ggplot(data = peru_d) +
  geom_sf()
```

### ¿Qué estás verificando aquí?

- Que el shapefile se cargó correctamente
- Que las geometrías están bien definidas
- Que la proyección espacial es válida

---

## Cargar datos

En el presente tutorial se utiliza data de ciberdelitos denunciados en el año 2020

```r
DF<-read_csv("delitosciber2020.csv")
DF
```
---

## Procesamiento de datos

### 1. Agrupar por departamento

```r
DF1 <- DF %>%
  group_by(dpto_pjfs) %>% 
  summarise(cantidad = sum(cantidad, na.rm = TRUE))
DF1
```
Agrupa todas las observaciones por departamento y suma los casos de delitos. Usa `na.rm = TRUE` para evitar errores por valores faltantes

---

### 2. Calcular porcentaje

```r
DF1 <- DF1 %>%
  mutate(porcentaje = (cantidad / sum(cantidad)) * 100)
DF1
```

Cada departamento ahora tiene:
- `cantidad`: total de casos
- `porcentaje`: proporción respecto al total nacional

---

### 3. Seleccionar variables

```r
delitos<-DF1%>%
  select(dpto_pjfs, porcentaje)
```

Se reduce el dataset solo a lo necesario para el mapa.

---

### 4. Exploración de datos

```r
summary(DF1$porcentaje)
```

Al permigirme ver el rango (mín–máx), se puede definke mejor la escala de colores (`limits`)

---

## Unión de datos espaciales y estadísticos

```r
peru_datos <- peru_d %>% 
  left_join(delitos, by = c("NOMBDEP" = "dpto_pjfs")) 
```

Utilizar left_join mantiene todas las geometrías del mapa, añadiendo la variable `porcentaje` a cada departamento. La unión

La unión depende de coincidencia exacta entre:
- `NOMBDEP` (shapefile)
- `dpto_pjfs` (datos)

Errores comunes:
- "LIMA" vs "Lima"
- Tildes
- Espacios extra

Si falla el join aparecen `NA` en el mapa y se debe realizar una limpieza extra usando por ejemplo `str_trim()`, `tolower()` u otras funciones dependiendo del problema particular

---

## Crear mapa de calor

```r
ggplot(peru_datos) +
  geom_sf(aes(fill = delitos$porcentaje), color = "grey60", size = 0.3) +
  labs(
    title = "Denuncias por ciberdelincuencia en Perú",
    subtitle = "Porcentaje de denuncias según departamento de la Presidencia\nde Junta de Fiscales Superiores del Distrito Fiscal, 2020",
    caption = "Datos: Ministerio Público Fiscalía de la Nación (MPFN) | Elaboración propia",
    x = NULL, y = NULL
  ) +
  scale_fill_viridis_c(
    name = "% de delitos",
    option = "viridis",
    direction = -1,
    na.value = "grey95",
    limits = c(0.5, 35),
    breaks = c(5, 15, 25, 35)
  ) +
  theme_minimal(base_size = 13) +
  theme(
    plot.title = element_text(face = "bold", size = 18, color = "#2c3e50",
                              hjust = 0, margin = margin(b = 6)),
    plot.subtitle = element_text(size = 11, color = "#34495e",
                                 hjust = 0, margin = margin(b = 14)),
    plot.caption = element_text(size = 8, color = "grey40", hjust = 1,
                                margin = margin(t = 10)),
    legend.position = "bottom",
    legend.key.width = unit(2, "cm"),
    legend.key.height = unit(0.4, "cm"),
    legend.title = element_text(size = 10, face = "bold"),
    legend.text = element_text(size = 9),
    panel.grid = element_line(color = "grey90", linewidth = 0.2),
    plot.margin = margin(15, 25, 10, 25)
  )
```

---

### ¿Cómo funciona el mapa?

#### `geom_sf()`
Dibuja los polígonos de cada departamento.

#### `aes(fill = ...)`
Asigna un color a cada departamento según el valor de la variable.

#### `scale_fill_viridis_c()`
Controla:
- Paleta de colores
- Rango (`limits`)
- Cortes (`breaks`)
- Color para valores faltantes

---

### Interpretación del mapa

- Zonas oscuras → mayor proporción de delitos  
- Zonas claras → menor proporción  
- Gris → sin datos o error en el join  

---

## Exportar el mapa

```r
ggsave("heatmap_peru.png", width = 10, height = 6)
```

---

# Producto final

<img width="1014" height="954" alt="image" src="https://github.com/user-attachments/assets/2548eaa0-629e-4aeb-b038-f8d54f929211" />


---
