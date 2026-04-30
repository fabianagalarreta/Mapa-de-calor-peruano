# 🇵🇪 Heat Map de Perú en R

Este proyecto muestra cómo construir un **mapa de calor (choropleth)** de los departamentos del Perú utilizando R.  
El objetivo es integrar **datos espaciales (shapefile)** con **datos estadísticos** para representar una variable (en este caso, porcentaje de denuncias por ciberdelincuencia) mediante una escala de color.

Este tipo de visualización es ampliamente usado en **epidemiología, salud pública y bioingeniería**, ya que permite identificar patrones espaciales y desigualdades regionales.

---

# 🧠 Flujo general del análisis

El proceso completo sigue estos pasos:

1. Cargar librerías
2. Leer shapefile (geometría del Perú)
3. Leer base de datos (valores por departamento)
4. Limpiar y transformar los datos
5. Unir datos espaciales y estadísticos
6. Visualizar el mapa de calor

---

# 📦 Librerías necesarias

```r
library(sf)         # Manejo de datos espaciales (geometría + atributos)
library(purrr)      # Programación funcional (útil en pipelines)
library(tidyverse)  # Manipulación de datos (dplyr, readr, etc.)
library(ggplot2)    # Visualización
library(ggrepel)    # Mejora de etiquetas (opcional)
library(readxl)     # Lectura de Excel (opcional)
library(readr)      # Lectura de CSV
```

### ¿Por qué estas librerías?

- `sf`: convierte shapefiles en dataframes espaciales
- `dplyr` (dentro de tidyverse): permite agrupar y resumir datos
- `ggplot2`: construye visualizaciones declarativas
- `readr`: lectura eficiente de archivos CSV

---

# 🗺️ Cargar shapefile de Perú

```r
dirmapas <- "C:/Users/fabia/OneDrive/Desktop/cursos_R/biostats/DEPARTAMENTOS_inei_geogpsperu_suyopomalia"
setwd(dirmapas)

peru_d<-st_read("DEPARTAMENTOS_inei_geogpsperu_suyopomalia.shp")
```

### 🔍 ¿Qué es un shapefile?

Un shapefile es un conjunto de archivos que contiene:
- Geometría (polígonos de departamentos)
- Atributos (nombres, códigos, etc.)

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
- Que existe la variable `NOMBDEP` (clave para el join)

---

# 📊 Cargar datos

```r
DF<-read_csv("delitosciber2020.csv")
DF
```

### Buenas prácticas aquí:

- Revisar nombres de columnas (`names(DF)`)
- Ver estructura (`str(DF)`)
- Identificar variable de interés (`cantidad`)
- Confirmar variable de agrupación (`dpto_pjfs`)

---

# 🧹 Procesamiento de datos

## 1. Agrupar por departamento

```r
DF1 <- DF %>%
  group_by(dpto_pjfs) %>% 
  summarise(cantidad = sum(cantidad, na.rm = TRUE))
DF1
```

### ¿Qué ocurre aquí?

- Se agrupan todas las observaciones por departamento
- Se suman los casos de delitos
- `na.rm = TRUE` evita errores por valores faltantes

---

## 2. Calcular porcentaje

```r
DF1 <- DF1 %>%
  mutate(porcentaje = (cantidad / sum(cantidad)) * 100)
DF1
```

### Interpretación

Cada departamento ahora tiene:
- `cantidad`: total de casos
- `porcentaje`: proporción respecto al total nacional

Esto es clave porque:
- Permite comparar regiones en la misma escala
- Evita sesgos por tamaño poblacional

---

## 3. Seleccionar variables

```r
delitos<-DF1%>%
  select(dpto_pjfs, porcentaje)
```

Se reduce el dataset solo a lo necesario para el mapa.

---

## 4. Exploración de datos

```r
summary(DF1$porcentaje)
```

### ¿Por qué es importante?

Te permite:
- Ver el rango (mín–máx)
- Detectar valores extremos
- Definir mejor la escala de colores (`limits`)

---

# 🔗 Unión de datos espaciales y estadísticos

```r
peru_datos <- peru_d %>% 
  left_join(delitos, by = c("NOMBDEP" = "dpto_pjfs")) 
```

### 🧠 Concepto clave: `left_join`

- Mantiene todas las geometrías del mapa
- Añade la variable `porcentaje` a cada departamento

### ⚠️ Punto crítico

La unión depende de coincidencia exacta entre:
- `NOMBDEP` (shapefile)
- `dpto_pjfs` (datos)

Errores comunes:
- "LIMA" vs "Lima"
- Tildes
- Espacios extra

Si falla el join → aparecen `NA` en el mapa

---

# 🔥 Crear mapa de calor

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

# 🎨 ¿Cómo funciona el mapa?

## `geom_sf()`
Dibuja los polígonos de cada departamento.

## `aes(fill = ...)`
Asigna un color a cada departamento según el valor de la variable.

## `scale_fill_viridis_c()`
Controla:
- Paleta de colores
- Rango (`limits`)
- Cortes (`breaks`)
- Color para valores faltantes

---

# ⚠️ Nota técnica importante

Tu código usa:

```r
aes(fill = delitos$porcentaje)
```

Esto funciona, pero no es lo más limpio.

Forma recomendada:

```r
aes(fill = porcentaje)
```

Porque esa variable ya está dentro de `peru_datos`.

---

# 🎯 Interpretación del mapa

- Zonas oscuras → mayor proporción de delitos  
- Zonas claras → menor proporción  
- Gris → sin datos o error en el join  

Este tipo de mapa permite detectar:
- Concentración geográfica
- Desigualdades regionales
- Posibles patrones espaciales

---

# 💡 Recomendaciones avanzadas

- Normalizar nombres antes del join:
```r
mutate(dpto_pjfs = str_to_upper(str_trim(dpto_pjfs)))
```

- Usar cuantiles si hay sesgo:
```r
quantile(DF1$porcentaje)
```

- Añadir etiquetas o centroides si necesitas interpretación más rica

---

# 💾 Exportar el mapa

```r
ggsave("heatmap_peru.png", width = 10, height = 6)
```

---

# 💾 Código completo

```r
library(sf)
library(purrr)
library(tidyverse)
library(ggplot2)
library(ggrepel)
library(readxl)
library(readr)

dirmapas <- "C:/Users/fabia/OneDrive/Desktop/cursos_R/biostats/DEPARTAMENTOS_inei_geogpsperu_suyopomalia"
setwd(dirmapas)
peru_d<-st_read("DEPARTAMENTOS_inei_geogpsperu_suyopomalia.shp")

ggplot(data = peru_d) +
  geom_sf()

DF<-read_csv("delitosciber2020.csv")
DF

DF1 <- DF %>%
  group_by(dpto_pjfs) %>% 
  summarise(cantidad = sum(cantidad, na.rm = TRUE))
DF1

DF1 <- DF1 %>%
  mutate(porcentaje = (cantidad / sum(cantidad)) * 100)
DF1

delitos<-DF1%>%
  select(dpto_pjfs, porcentaje)

summary(DF1$porcentaje)

peru_datos <- peru_d %>% 
  left_join(delitos, by = c("NOMBDEP" = "dpto_pjfs")) 

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
