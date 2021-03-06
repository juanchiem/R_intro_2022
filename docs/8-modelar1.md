---
output: html_document
editor_options: 
  chunk_output_type: console
---

# Correlacion 

## Simple


```r
pacman::p_load(tidyverse, GGally, correlation, ggcorrplot, viridis)
```


`GGally`


```r
iris %>% 
  ggpairs()
```


`correlation`

[correlation](%22https://easystats.github.io/correlation/articles/types.html%22) (del ecosistema [easystats](%22https://easystats.github.io/easystats/%22))


```r
iris_corr <- iris %>% 
  correlation(method = "pearson") #spearman
iris_corr
```


```r
iris_corr %>%  
  ggplot() + 
  aes(x=Parameter1, Parameter2, fill= r)+
  geom_tile()
```


```r
iris_corr %>%  
  ggplot(aes(x=fct_relevel(Parameter1, "Sepal.Length", after = 2), 
             y=fct_relevel(Parameter2, "Petal.Width"), 
             fill= r)) + 
  geom_tile() + 
  scale_fill_viridis(discrete=FALSE, direction = -1) + 
  labs(x="", y="") + 
  geom_text(aes(label=r %>% round(2)), col="white") +
  coord_flip()+
  theme_minimal()
```

`ggcorrplot`


```r
iris %>% 
  select_if(is.numeric) %>% 
  cor() %>% 
  round(2) %>%
  ggcorrplot(hc.order = TRUE, type = "lower", lab = TRUE)
```

## Multi-level


```r
iris %>% 
  ggpairs(aes(color = Species))
```


```r
iris_corr_sp <- iris %>%
  group_by(Species) %>%
  correlation() # %>% 

iris_corr_sp # %>% knitr::kable()
```

> Es aqui donde pasa el investigador al frente del analista a interpretar los patrones 

Veamos un ejemplo clasico multi-level: Simpson's paradox 


```r
simpson <- simulate_simpson(n = 100, groups = 10)
simpson

simpson %>% 
  ggplot()+ 
  aes(x = V1, y = V2) +
  geom_point() +
  geom_smooth(method = "lm", se = FALSE)
```


```r
correlation(simpson)
```


```r
simpson %>% 
  ggplot() +
  aes(x = V1, y = V2, col = Group) +
  geom_point() +
  geom_smooth(method = "lm", se = FALSE) +
  geom_smooth(colour = "black", method = "lm", se = FALSE)
```

Parece que, para cada sujeto, la relación es diferente. La tendencia negativa (global) parece ser un producto de las diferencias entre los grupos y podría ser espuria

Las correlaciones multinivel (como en multigrupo) nos permiten dar cuenta de las diferencias entre grupos. Se basa en una parcialización del grupo, ingresada como efecto aleatorio en una regresión lineal mixta.

Puede calcularlos con el paquete de correlaciones configurando el argumento multinivel en VERDADERO.


```r
simpson %>% 
  group_by(Group) %>% 
  correlation()

simpson %>% 
  correlation(multilevel = TRUE)
```

Ojo con las correlaciones!


```r
anscombe_long <- rio::import('https://docs.google.com/spreadsheets/d/1c_FXVNkkj4LD8hVUForaaI24UhRauh_tWsy7sFRSM2k/edit#gid=1709959158')

anscombe_long %>% 
  group_by(set) %>% 
  summarise(x_mean = mean(x), 
            y_mean = mean(y), 
            x_sd = sd(x), 
            y_sd = sd(y))
```


```r
anscombe_long %>% 
  group_by(set) %>% 
  correlation(method = "pearson")
```


```r
anscombe_long %>% 
  ggplot() + 
  aes(x, y) + 
  geom_point() + 
  geom_smooth(method="lm")+ 
  facet_wrap("set")
```

> Conclusion: SIEMPRE plotear los datos crudos y los residuos

:::{#box1 .blue-box}

Como pasar "anscombe" de wide a long


```r
data(anscombe)

anscombe_long <- anscombe %>%
  pivot_longer(
    cols = everything(),
    # apilar solo el valor en la columna "set"
    names_to = c(".value", "set"),
    # indico que las columnas que estoy apilando tienen su primer digito con el nombre de una variable y el segundo digito con el nro de set
    names_pattern = "(.)(.)"
  ) %>% 
  arrange(set, x)
```
:::

# Regresión linear


```r
pacman::p_load(tidyverse, performance, ggeffects)
```

[performance](https://easystats.github.io/performance/) 

[ggeffects](https://strengejacke.github.io/ggeffects/) 


## Simple


```r
lm1 <- lm(y~x, data=anscombe_long %>% filter(set==1))
check_model(lm1)
check_outliers(lm1)
```


```r
anova(lm1)
summary(lm1)
```

:::{#box1 .blue-box}

Agregar formula y R2


```r
library(ggpmisc)
formu <- y ~ x

anscombe_long %>% 
  filter(set==1) %>% 
  ggplot() + 
  aes(x, y) + 
  geom_point() + 
  geom_smooth(method="lm")+ 
  stat_poly_eq(formula = formu,
               aes(label = paste(..eq.label.., ..rr.label.., sep = "~~~")),
               parse = TRUE) 
```
:::

## Regresión polinomial

Es un caso especial de la Regresión Lineal, enriquece el modelo lineal al aumentar predictores adicionales, obtenidos al elevar cada uno de los predictores originales a una potencia.

---

La regresión ajustada parece correcta para el set 1, pero veamos para el set 2


```r
lm2 <- lm(y~x, data=anscombe_long %>% filter(set==2))
check_model(lm2)
check_outliers(lm2)
```


```r
lm2.sq <- lm(y~x + I(x^2), data=anscombe_long %>% filter(set==2))
compare_performance(lm2, lm2.sq)
```

:::{#box1 .blue-box}

Regresiones lineales seriales 


```r
library(broom)  

lm_ans_coef <- anscombe_long %>% 
  group_by(set) %>%
  do(tidy(lm(data = ., formula = y ~ x)))

lm_ans_coef
```


```r
lm_ans_stats <- anscombe_long %>% 
  group_by(set) %>%
  do(glance(
    lm(data = ., formula = y ~ x))
    )
lm_ans_stats
```
:::


Ahora veamos un caso del dataset `cars`


```r
cars %>%
  ggplot()+
  aes(speed, dist)+
  geom_point()

cars %>%
  ggplot()+
  aes(speed, dist)+
  geom_point()+
  # geom_smooth()+
  geom_smooth(method = lm)+
  # geom_smooth(method = lm, formula = y ~ poly(x, 1), ) + 
  geom_smooth(method = lm, formula = y ~ poly(x, 2), col="red")
```

Ajustamos las dos alternativas


```r
fit1 <- lm(dist ~ speed, data = cars)
check_model(fit1)
```


```r
fit2 <- lm(dist ~ speed + I(speed^2), data = cars)

compare_performance(fit1, fit2)
summary(fit2)
```


```r
pred_fit2 <- ggpredict(fit2, terms = c("speed"))

pred_fit2 %>%  
  plot(add.data = TRUE, limit.range = TRUE)  
```

## Regresion multiple

La regresión lineal múltiple permite generar un modelo lineal en el que el valor de la variable dependiente o respuesta (Y) se determina a partir de un conjunto de variables independientes llamadas predictores (X1, X2, X3…)

---

Volvamos a `iris` para tratar de modelar `Sepal.Width ~ Sepal.Length`


```r
model1 <- lm(Sepal.Width ~ Sepal.Length, data = iris)
model2 <- lm(Sepal.Width ~ Sepal.Length * Species, data = iris)
# model2 <- lm(Sepal.Width ~ Sepal.Length + Species + Sepal.Width:Species, data = iris)
model3 <- lm(Sepal.Width ~ Sepal.Length + Species, data = iris)
model4 <- lm(Sepal.Width ~ Sepal.Length:Species, data = iris)

compare_performance(model1, model2, model3, model4)

anova(model2)
check_model(model2)
```


```r
pred_mod <- ggpredict(model2, terms = c("Sepal.Length", "Species"))

pred_mod %>%  
  plot(add.data = TRUE, limit.range = TRUE)  
```

:::{#box1 .blue-box}

Como se corrije la Simspon`s paradox? Atribuyendo efecto aleatorio a los grupos, 


```r
library(lme4)

simpson_lm <- lm(V2 ~ V1, data=simpson) 
simpson_mixed <- lmer(V2 ~ V1 + (1|Group), data=simpson) 

ggpredict(simpson_lm) %>% plot(add.data = TRUE)
ggpredict(simpson_mixed) %>% plot(add.data = TRUE)
```
:::
