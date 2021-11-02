#Correlacion


```r
library(tidyverse)
```



```r
pacman::p_load(GGally, correlation, report)
```


```r
# GGally
iris %>% 
  ggpairs()
```

Paquete [correlation]("https://easystats.github.io/correlation/articles/types.html") (del ecosistema [easystats]("https://easystats.github.io/easystats/"))


```r
# correlation
iris %>% 
  correlation(method = "pearson") #spearman
```


```r
iris %>% 
  ggpairs(aes(color = Species))
```


```r
iris %>%
   dplyr::select(Species, Sepal.Length, Sepal.Width, Petal.Width) %>%
  group_by(Species) %>%
  correlation()
```

# Regresión linear 


```r
pacman::p_load(jtools, report)
```


```r
model <- iris %>%
  lm(formula = Sepal.Length ~ Petal.Width)

# R base
anova(model)
summary(model)

# jtools
summ(model)
```


```r
library(ggfortify)
autoplot(model)
```


Paquete [report](https://easystats.github.io/report/)


```r
report(model)
```

Agregar formula y R2


```r
library(ggpmisc)
formu <- y ~ x

ggplot(iris, aes(x=Petal.Width, y=Sepal.Length))+
  geom_point() +
  geom_smooth(method="lm")+
  stat_poly_eq(formula = formu,
               aes(label = paste(..eq.label.., ..rr.label.., sep = "~~~")),
               parse = TRUE) 
  # geom_abline(slope = coef(model)[[2]], intercept = coef(model)[[1]])
  # geom_line(data = fortify(model), aes(x=Petal.Width, y = .fitted))
```

Paquete [jtools](https://jtools.jacob-long.com/)


```r
effect_plot(model, 
            pred=Petal.Width, 
            plot.points = TRUE, 
            interval = TRUE, 
            colors = "red")+
  theme_bw()
```

Paquete [ggeffects](https://strengejacke.github.io/ggeffects/index.html)


```r
res_reg <- ggpredict(model, terms ="Petal.Width")

res_reg %>% 
  ggplot()+
  aes(x=x, y=predicted)+
  geom_line()

plot(res_reg, ci = FALSE) + 
  labs(x = "Ancho de pétalos") + 
  geom_point(data=iris, 
             aes(x=Petal.Width, y=Sepal.Length))
```

Regresión polinomial


```r
fit2 <- lm(dist ~ speed + I(speed^2), data = cars)
summary(fit2)
# fit3 <- lm(dist ~ poly(speed,2, raw = TRUE), data = cars)
# summary(fit3)

effect_plot(fit2, 
            pred=speed, 
            plot.points = TRUE, 
            interval = TRUE, 
            colors = "red")
```


```r
ggplot(cars, aes(x=speed, y=dist) ) +
  geom_point() +
  stat_smooth(method = lm, formula = y ~ poly(x, 2, raw = TRUE))
```
