Equity in Athletics - Cohen
================
Lisandro Carrasco
7/6/2022

El deporte universitario en Estados Unidos resulta un mercado de alto
atractivo para el consumo masivo, así como una fuente de ingreso para
las universidades tanto públicas como privadas de ese país. Si bien en
la mayoría de los casos no constituyen partes centrales de sus ingresos
y presupuestos, sí son un aporte considerable en términos económicos y,
sobre todo, reputacionales.

El departamento de Educación de los Estados Unidos ofrece una gran
cantidad de información sobre las universidades involucradas en
diferentes asociaciones deportivas y la participación de sus estudiantes
en varios deportes, así como el impacto económico que esto genera.

Se analizará un subconjunto de esos datos, disponibilizado por la
comunidad de [R for Data Science](https://bit.ly/3avHYUs), que incluye
datos de más de 2.200 universidades entre los años 2015 y 2019.

``` r
sports <- read.csv('https://raw.githubusercontent.com/rfordatascience/tidytuesday/master/data/2022/2022-03-29/sports.csv')
reactable(head(sports))
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

El dataset preprocesado para la actividad semanal de ‘Tidy Tuesday’
cuenta con más de 132 mil observaciones correspondientes a 28 variables,
que incluyen información sobre

  - Universidades: nombre, estado, cógido postal, cantidad de
    estudiantes
  - Ligas universitarias: asociaciones y divisiones en las que
    participan
  - Atletas: cantidad de atletas según género y deporte
  - Economía: ingresos y gastos según género y deporte

En base a esta información, este análisis buscará responder preguntas en
torno a 3 ejes:

**1) Deportes**: ¿Cuáles son los deportes que más ingresos generan? ¿Son
los más rentables? ¿Hay oportunidades para las universidades en deportes
menos masivos?

**2) Universidades**: ¿Pueden el sector privado y el público competir en
el deporte universitario? ¿Alguno de los dos hace un uso más eficiente
de sus recursos?

**3) Género**: ¿Cómo se expresa la brecha de género en el deporte
universitario?

## Exploración y preprocesamiento

#### Valores faltantes

Un primer punto a explorar rápidamente es el de los valores faltantes en
cada variable

``` r
col_nan <- data.frame((apply(is.na(sports), MARGIN = 2, FUN=sum)))
names(col_nan)[1] <- "faltantes"
col_nan <- col_nan%>%
  rownames_to_column("columna")%>%
  filter(faltantes > 0)
reactable(col_nan)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Faltantes-1.png)<!-- -->

Pueden apreciarse dos niveles de valores faltates que se pueden explicar
por diferentes orígenes: por un lado, algunas columnas asociadas a la
información de las universidades presentan 45 observaciones con valores
faltantes; por otro, las variables de participación de los atletas y de
su información económica, presentan valores más altos de faltantes y
repetidos para algunas columnas.

``` r
universidades_nan <- filter(sports, is.na(city_txt) == TRUE & is.na(state_cd) == TRUE & is.na(zip_text) == TRUE & is.na(sector_name) == TRUE)
reactable(universidades_nan)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

Los 45 valores faltantes se corresponden a una **univerisdad de Canadá**
que, si bien participa de la NCAA Division II, puede contar con un
mercado diferente del resto de las universidades. Para evitar
distorciones, serán excluida de este análisis.

``` r
sports <- filter(sports, institution_name != "Simon Fraser University")
rm(universidades_nan)
```

Así como también las correspondientes a **territorios no incorporados**
que claramente presentan condiciones económicas muy disímiles respecto
al territorio nacional de los Estados Unidos.

``` r
estados <- unique(sports$state_cd)
eeuu <- read.csv('https://gist.githubusercontent.com/dantonnoriega/bf1acd2290e15b91e6710b6fd3be0a53/raw/11d15233327c8080c9646c7e1f23052659db251d/us-state-ansi-fips.csv')
str(eeuu)
```

    ## 'data.frame':    51 obs. of  3 variables:
    ##  $ stname: chr  "Alabama" "Alaska" "Arizona" "Arkansas" ...
    ##  $ st    : int  1 2 4 5 6 8 9 10 11 12 ...
    ##  $ stusps: chr  " AL" " AK" " AZ" " AR" ...

``` r
eeuu <- rename(eeuu, state_name=stname)
eeuu$stusps <- str_replace(eeuu$stusps, " ", "")


(dif <- estados[!(estados %in% c(eeuu$stusps))])
```

    ## [1] "PR" "VI"

``` r
sports <- filter(sports, !state_cd%in%dif)
sports <- left_join(sports, eeuu, by=c('state_cd'='stusps'))

rm(estados, eeuu, dif)
```

Retomando el análisis de los valores faltantes, el resto de las
observaciones corresponden a varibles donde no se han registrado atletas
de alguno de los dos sexos participando de un deporte en particular y
por tanto, no puede registrarse información. Estos valores faltantes
continuarán como tal, dado que una imputación por 0 alteraría los
resultados y otras formas de imputación no tendrían sentido.

#### Nuevas variables

Considerando las variables al momento, se definirán otras nuevas en base
a la información ya disponible para ampliar así el espectro de lo que se
puede observar en los datos. Estas nuevas variables buscarán expresar
otras dimensiones del plano económico, estando asociadas a la
rentabilidad, las ganancias o los gastos par atleta, según su sexo.
También se apartarán otras que no serán utilizadas.

``` r
sports_fin <- sports%>%
  mutate(sector = case_when(sector_cd == 1 | sector_cd == 4 ~ "Public", 
                            TRUE ~ "Private"))%>%
  mutate(rent_women = round((rev_women/exp_women),3),
         rent_men = round((rev_men/exp_men),3),
         exp_per_women = round((exp_women/sum_partic_women),2),
         exp_per_men = round((exp_men/sum_partic_men),2), 
         rev_per_women = round((rev_women/sum_partic_women),2),
         rev_per_men = round((rev_men/sum_partic_men),2),
         rentabilidad = round((total_rev_menwomen/total_exp_menwomen),3), 
         participantes = sum_partic_men+sum_partic_women,
         ganancia=total_rev_menwomen-total_exp_menwomen,
         ingreso_pc = round((total_rev_menwomen/participantes),2),
         gasto_pc = round((total_exp_menwomen/participantes),2),
         ganancia_pc = ingreso_pc-gasto_pc)%>%
  select(-unitid, -city_txt, -zip_text, -classification_code,
         -classification_other, -sector_cd, -sector_name, -sportscode, -st)

reactable(head(sports_fin))
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Variables-1.png)<!-- -->

#### Asociaciones y ligas

Estados Unidos cuenta una amplia variedad de asociaciones y ligas
universitarias, presentando grandes diferencias entre sí en relación a
la cantidad de deportistas que participan en ellas pero, principalmente,
en los ingresos que generan. Este conjunto de datos cuenta con
universidades que participan de 19 divisiones diferentes
correspondientes a 7 asociaciones y 2 grupos independientes o no
clasificados. Las 7 asociaciones son: NCAA, NJCAA, USCAA, NAIA, NCCAA,
CCCAA, NWAC. Puede verse una clara brecha en los ingresos que han
generado las diferentes asociaciones y divisiones en los últimos años:

``` r
divisiones <- sports_fin%>%
  group_by(classification_name)%>%
  summarise(ingresos=sum(total_rev_menwomen, na.rm = TRUE))


fig_ingresos_div <- plot_ly(divisiones, type='bar', x = ~classification_name, y = ~ingresos, text = ~classification_name, 
               hovertemplate = paste('%{x}', '<br>Ingresos: %{y:.2s}<br>'))
fig_ingresos_div <- fig_ingresos_div%>% layout(uniformtext=list(minsize=4, mode='hide'))%>% 
  layout(xaxis = list(categoryorder = "total ascending"))
fig_ingresos_div <- fig_ingresos_div %>% layout(title = 'Ingresos acumulados por asociación')
fig_ingresos_div
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Divisiones-1.png)<!-- -->

El gráfico de barras que muestra los ingresos acumulados por cada
división en los últimos años, marcando una diferencia notable entre las
divisiones correspondientes a la NCAA, principalmente en la División I -
FBS, y el resto. Esto se debe a que la NCAA es la asociación más grande
de los Estados Unidos en términos de cantidad de institcuiones, pero
también a que los datos incluyen información de asociaciones regionales
o de escala menor que no tienen un peso comparable con la NCAA. Estas
asociaciones también serán excluídas del análisis junto con las
independientes o las catalogadas como ‘Otras’, que corresponden a
universidades que participan de más de una división a la vez y son parte
de una polémica en la regulación deportiva de aquel país, como la
Universidad John Hopkins.

``` r
asoci <- sports_fin$classification_name%>%
  str_split(" ", simplify = TRUE)
asoci <- data.frame(asoci)        
sports$asociaciones <- asoci$X1

excluidas <- c('NWAC', 'CCCAA', 'USCAA', 'Independent', 'Other')

sports <- filter(sports, !asociaciones %in%excluidas)
unique(sports$asociaciones)
```

    ## [1] "NCAA"  "NJCAA" "NAIA"  "NCCAA"

``` r
rm(asoci, excluidas, divisiones)
```

``` r
asociaciones <- sports%>%
  group_by(asociaciones)%>%
  summarise(ingresos=sum(total_rev_menwomen, na.rm = TRUE))


fig_ingresos_aso <- plot_ly(asociaciones, type='bar', x = ~asociaciones, y = ~ingresos, text = ~asociaciones, 
               hovertemplate = paste('%{x}', '<br>Ingresos: %{y:.2s}<br>'))
fig_ingresos_aso <- fig_ingresos_aso%>% layout(uniformtext=list(minsize=4, mode='hide'))%>% 
  layout(xaxis = list(categoryorder = "total ascending"))
fig_ingresos_aso <- fig_ingresos_aso %>% layout(title = 'Ingresos acumulados por asociación')
rm(asociaciones)
fig_ingresos_aso
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Ingresos%20por%20asociaciones-1.png)<!-- -->

Agrupando por asociaciones queda más en evidencia la hegemonía de la
NCAA, al menos durante los 5 años analizados. La diferencia tiene
sentido a sabiendas del tamaño de cada una: según sus datos oficiales,
mientras que la NCAA cuenta con 1137 instituciones educativas asociadas,
la NAIA solo cuenta con 265 y la NJCAA y la NCCAA con 512 y 59,
respectivamente.

## La NCAA y el fútbol americano

La primera serie de preguntas de este análisis se enfoca en averiguar
qué deportes son los que más ingresos generan para las universidades
estadounidenses y si hay otros deportes de una potencial alta
rentabilidad que no estén siendo tan explotados.

El gráfico de ingresos generado por cada división muestra que la
**División I - FBS** de la NCAA es, por mucho, la que más ingresos ha
generado en el último tiempo. Esta es una división marcada por una
composición particular: participan las universidades que cuenten con los
mejores equipos de fútbol americano. Puede apreciarse que la segunda
división que más ingresos genera es también de la NCAA y es la
**División I - FCS**. Curiosamente, también delimitada por el fútbol
americano, con el resto de los equipos que no están en la FBS.

Esta jerarquización dirige a la conclusión de que **el fútbol americano
es claramente el deporte universitario más importante** en muchos
sentidos: define la organización de las asociaciones universitaria dado
que es el deporte que más ingresos ha registrado en los últimos 5 años.
También resulta ser el deporte que más ganancias ha generado:

``` r
deportes_group <- sports_fin%>%
  group_by(sports)%>%
  summarise(ingresos=sum(total_rev_menwomen, na.rm = TRUE),
            ganancia=sum(ganancia, na.rm=TRUE))%>%
  arrange(desc(ingresos))
reactable(deportes_group)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Tabla%20-%20Ingresos%20y%20ganancias%20por%20deporte-1.png)<!-- -->

Esto puede apreciarse con un gráfico de tortas que muestra el aporte de
cada actividad a la masa total de ingresos del deporte universitario
entre 2015 y 2019:

``` r
plot_tortadep <- plot_ly(deportes_group, labels = ~sports, values = ~ingresos, type = 'pie')
plot_tortadep <- plot_tortadep %>% layout(title = 'Generación de ingresos por deporte',
                      xaxis = list(showgrid = FALSE, zeroline = FALSE, showticklabels = FALSE),
                      yaxis = list(showgrid = FALSE, zeroline = FALSE, showticklabels = FALSE))

plot_tortadep
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Torta%20-%20Ingresos%20por%20deportes-1.png)<!-- -->

``` r
participantes <- sports_fin%>%
  group_by(sports,year)%>%
  summarise(hombres=sum(sum_partic_men, na.rm = TRUE),
            mujeres=sum(sum_partic_women, na.rm = TRUE),
            participantes=hombres+mujeres)

jugados_2019 <- filter(participantes, year==2019)
```

Hay una cuestión a destacar en relación a la generación de ingresos.
Mientras que **el fútbol americano aportó más del 40% de los ingresos**
del deporte universitario en el período analizado, esto no implica que
el 40% de los estudiantes se dedique a este deporte. La proporción de
atletas de este deporte es menor: **un 24.19 de los atletas
universitarios hombres** se dedicaron a esta actividad en 2019.

Puede apreciarse la dedicación de los estudiantes a cada deporte durante
el último año analizado (2019) en el siguiente mapa de árbol:

``` r
treemap_hombres <- ggplot(jugados_2019, aes(area = hombres, fill = hombres, label=sports)) +
  geom_treemap()+
  geom_treemap_text(colour = "white",
                    place = "centre",
                    size = 15)+
  labs(fill="Atletas", title="Proporción de atletas masculinos según deporte practicado")
treemap_hombres
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Treemap%20hombres-1.png)<!-- -->

Hay que destacar que no se registran mujeres jugando al football
americano. Por tanto, la participación de los jugadores de football
sobre el total de la masa universitaria será menor a la mencionada solo
entre hombres. En la comparación de los dos gráficos puede apreciarse la
aparición del **‘soccer’** cuando se mira el gráfico de árbol que
incluye a ambos deportes, así como del softball y el volley.

``` r
treemap_ambos <- ggplot(jugados_2019, aes(area = participantes, fill = participantes, label=sports)) +
  geom_treemap()+
  geom_treemap_text(colour = "white",
                    place = "centre",
                    size = 15)+
  labs(fill="Atletas", title="Proporción de atletas de ambos sexos según deporte practicado")

treemap_ambos
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Treemap%20ambos-1.png)<!-- -->

Esto se debe a que el *soccer* resulta ser un deporte muy jugado por las
estudiantes universitarias, posicionándolo entre los 5 más jugados de
ambos géneros. La siguiente tabla muestra el promedio de participantes
que ha tenido cada deporte en los años analizados.

``` r
participantes_prom <- participantes%>%
  group_by(sports)%>%
  summarise(hombres = round(mean(hombres, na.rm = TRUE)),
            mujeres = round(mean(mujeres, na.rm =TRUE)),
            participantes = round(mean(participantes, na.rm = TRUE)))%>%
  arrange(desc(participantes))

reactable(participantes_prom)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Tabla%20de%20jugadores%20por%20deporte%20según%20sexo-1.png)<!-- -->

En resumen, las ganancias generales, a pesar del registro de varios
deportes deficitarios, se deben a que el **futbol americano genera un
ingreso y una ganancia per cápita muy sobre la media**, dado el enorme
mercado que generan los fanáticos del juego.

``` r
deportes_pc <- sports_fin%>%
  group_by(sports, year)%>%
  summarise(ingresos=round((mean(total_rev_menwomen, na.rm = TRUE)),2), 
            gasto=round((mean(total_exp_menwomen, na.rm = TRUE)),2),
            rentabilidad =round((mean(rentabilidad, na.rm = TRUE)),3), 
            participantes=round((mean(participantes, na.rm = TRUE))),
            ingreso_pc=round((mean(ingreso_pc, na.rm = TRUE)),2), 
            gasto_pc=round((mean(gasto_pc, na.rm = TRUE)),2))%>%
  mutate(ganancia_pc = round((ingreso_pc-gasto_pc),2))

reactable(deportes_pc)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Ganancia%20per%20cápita%20por%20deporte-1.png)<!-- -->

De hecho, el promedio de ganancia por atleta en los últimos años ha sido
negativo para las universidades en general: la ganancia por atleta en
los últimos 5 años fue de -2092.91, mientras que en el caso del fútbol
americano se registró una ganancia por atleta del 16676.28. Esto muestra
que **las ganancias del fútbol y el basket pueden compensar a otros
deportes deficitarios**.

La siguiente serie de tiempo muestra cómo el fútbol americano genera
sostenidamente más ganancia por atleta que los otros 6 deportes más
practicados tanto por hombres como por mujeres. Estos otros deportes son
el baseball, atletismo combinado (*“all track comined”*), soccer,
basket, softball y volley.

``` r
lista=c('Football', 'Baseball', 'All Track Combined', 'Soccer', 'Basketball', 'Softball', 'Volleyball')
deportes_seleccion <- deportes_pc%>%
  filter(sports %in% lista)%>%
  select(sports, year, ganancia_pc)
deportes_t <- deportes_seleccion %>%
  pivot_wider(names_from = sports, values_from = ganancia_pc)


names(deportes_t)[2] <- 'Combined'
deportes_t <- deportes_t%>%
  mutate(year=as.character(year))


plot_gpc <- plot_ly(deportes_t, x = ~year, y = ~Football, name = 'Football', type = 'scatter', mode = 'lines+markers',
               hovertemplate = paste('%{x}', '<br>Ganancia: %{y:.5}<br>')) 
plot_gpc <- plot_gpc %>% add_trace(y = ~Baseball, name = 'Baseball', mode = 'lines+markers')
plot_gpc <- plot_gpc %>% add_trace(y = ~Combined, name = 'All Track Combined', mode = 'lines+markers') 
plot_gpc <- plot_gpc %>% add_trace(y = ~Basketball, name = 'Basketball', mode = 'lines+markers') 
plot_gpc <- plot_gpc %>% add_trace(y = ~Softball, name = 'Softball', mode = 'lines+markers') 
plot_gpc <- plot_gpc %>% add_trace(y = ~Volleyball, name = 'Volleyball', mode = 'lines+markers') 
plot_gpc <- plot_gpc %>% add_trace(y = ~Soccer, name = 'Soccer', mode = 'lines+markers') 
plot_gpc <- plot_gpc %>% layout(title = "Ganancia per cápita de cada deporte",
                      xaxis = list(title = "Año"),
                      yaxis = list (title = "Ganancia por atleta"))

plot_gpc
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Serie%20de%20tiempo%20-%20Ganancia%20por%20atleta-1.png)<!-- -->

Una última observación sobre el rendimiento económico de cada deporte es
que si bien el football es el deporte que más ingresos y ganancias
genera, no es necesariamente en el que más se invierte por atleta (sí en
general). De hecho, el buceo, el basket y la gimnasia han presentado
niveles de gasto por atleta más altos que el football en los últimos
años, incluso a pesar de ser deportes que no tienen recompensas
económicas significativas.

``` r
deportes_gasto <- sports_fin%>%
  group_by(sports, year)%>%
  summarise(gasto=sum(total_exp_menwomen, na.rm = TRUE),
            participantes=sum(participantes, na.rm = TRUE))%>%
  mutate(gasto_pc=gasto/participantes)%>%
  arrange(desc(gasto_pc))

reactable(deportes_gasto)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Gasto%20por%20atleta-1.png)<!-- -->

## Universidades

Los ingresos mencionados anteriormente, forman parte de un esquema
económico mucho más amplio para las universidades estadounidenses, que
cuentan con múltiples formas de financiación que hacen que su
presupuesto no esté extremadamente comprometido con el negocio
deportivo. A diferencia de lo que sucede en el sistema educativo
argentino, las universidades públicas de EEUU pueden acceder a las
mismas formas de financiación que las privadas, permitiéndoles una
competencia mayor en el campo deportivo. De hecho, es interesante ver
que en el período evaluado, **las universidades públicas registraron una
masa total de ingresos más grande que las privadas**.

``` r
universidades <- sports_fin%>%
  group_by(institution_name, year, sector)%>%
  summarise(participantes=sum(participantes, na.rm = TRUE),
            ingresos = sum(total_rev_menwomen, na.rm = TRUE),
            gastos = sum(total_exp_menwomen, na.rm = TRUE),
            ganancia = sum(ganancia, na.rm = TRUE),
            ganancia_pc = ganancia/participantes,
            rentabilidad=ingresos/gastos)

universidades_group <- universidades%>%
  group_by(sector, year)%>%
  summarise(ingresos=sum(ingresos, na.rm = TRUE),
            ganancia=sum(ganancia, na.rm = TRUE),
            gastos=sum(gastos, na.rm=TRUE))


fig_ingresos_uni<- ggplot(universidades_group)+
  geom_col(aes(x=sector, y=ingresos, fill=sector), alpha=0.65)+
  theme_minimal()+
  scale_fill_manual(values=c('blue', 'red'))+
  labs(y="", x="Sector", title="Ingresos acumulados", subtitle = "Según sector universitario", legend=FALSE)+
  theme(legend.position='none')
fig_ingresos_uni <- fig_ingresos_uni+facet_grid(~year, scales="free_y")
ggplotly(fig_ingresos_uni)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Facetado%20por%20año%20público%20privado-1.png)<!-- -->

La diferencia se acentúa mucho cuando se pasa al plano de las ganancias:

``` r
fig_ganancia_uni<- ggplot(universidades_group)+
  geom_col(aes(x=sector, y=ganancia, fill=sector), alpha=0.65)+
  theme_minimal()+
  scale_fill_manual(values=c('blue', 'red'))+
  labs(y="", x="Sector", title="Ganancias acumuladas", subtitle = "Según sector universitario", legend=FALSE)+
  theme(legend.position='none')
fig_ganancia_uni <- fig_ganancia_uni+facet_grid(~year, scales="free_y")
ggplotly(fig_ganancia_uni)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

Contando con 983 unviersidades privadas y 1206 públicas, podemos ver que
la diferencia de estos volúmenes no radica en los pesos de la muestra,
si no que las universidades públicas en promedio generan mucho más
ingreso y ganancia.

De hecho, la gestión de **recursos por atleta en las universidades
públicas es más eficiente**, dado que el sector público cuenta con
menos estudiantes que el privado, (en promedio, cuentan con 305681 y
361411 por año, respectivamente),

En términos de **ingreso por atleta**, nuevamente puede verse que cada
año del período, las universidades públicas tienen mejores resultados.

``` r
universidades_group <- universidades_group%>%
  mutate(ingresos_pc =round((ifelse(sector== "Public", ingresos/sum(filter(universidades, sector=='Public')$participantes)/5, ingresos/sum(filter(universidades, sector=='Private')$participantes)/5)),2))%>%
  mutate(gastos_pc =round((ifelse(sector== "Public", gastos/sum(filter(universidades, sector=='Public')$participantes)/5, gastos/sum(filter(universidades, sector=='Private')$participantes)/5)),2))%>%
  mutate(rentabilidad=round((ingresos/gastos),3))
```

``` r
fig_ingreso_pc_uni<- ggplot(universidades_group)+
  geom_col(aes(x=sector, y=ingresos_pc, fill=sector), alpha=0.65)+
  theme_minimal()+
  scale_fill_manual(values=c('blue', 'red'))+
  labs(y="", x="Sector", title="Ingresos por atleta", subtitle = "Según sector universitario", legend=FALSE)+
  theme(legend.position='none')
fig_ingreso_pc_uni <- fig_ingreso_pc_uni+facet_grid(~year, scales="free_y")
ggplotly(fig_ingreso_pc_uni)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->
La **rentabilidad** general entonces del deporte universitario en el
sector público es más alta que en el privado. Por cada dólar que
invierte el sector público en un deportista universitario, genera entre
USD1.104 y USD1.132, mientras que las universidades privadas generan
entre USD1.026 y USD1.045. Es decir, en un sector, se puede generar un
retorno máximo del 13% y en otro del 4,5%.

``` r
fig_rentabilidad_uni<- ggplot(universidades_group)+
  geom_col(aes(x=sector, y=rentabilidad, fill=sector), alpha=0.65)+
  theme_minimal()+
  scale_fill_manual(values=c('blue', 'red'))+
  labs(y="", x="Sector", title="Rentabilidad", subtitle = "Según sector universitario", legend=FALSE)+
  theme(legend.position='none')
fig_rentabilidad_uni <- fig_rentabilidad_uni+facet_grid(~year, scales="free_y")
ggplotly(fig_rentabilidad_uni)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->
Los datos pueden verse en la siguiente tabla y el gráfico de burbujas
muestra la relación entre ingresos y ganancias por atleta de las 50
universidades que más recaudan, así como su rentabilidad (dada por el
tamaño de la burbuja), destacando por mucho la University of Texas at
Austin.

``` r
reactable(universidades_group)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

``` r
destacadas <- sports_fin%>%
  group_by(institution_name, sector)%>%
  summarise(ingresos=sum(total_rev_menwomen, na.rm = TRUE),
            gastos=sum(total_exp_menwomen, na.rm = TRUE),
            participantes=sum(participantes, na.rm = TRUE),
            ganancia=sum(ganancia, na.rm = TRUE),
            rentabilidad=mean(rentabilidad, na.rm = TRUE),
            ingresos_pc=round((ingresos/participantes),2),
            gastos_pc=round((gastos/participantes),2),
            ganancia_pc=round((ganancia/participantes),2))%>%
  arrange(desc(ingresos))

destacadas <- destacadas[1:50,]

fig_univ_dest <- plot_ly(data = destacadas, x = ~ingresos_pc, y = ~ganancia_pc, color = ~sector, size=~rentabilidad,
               text = ~paste(institution_name, "$<br>Ingresos por atl: ", ingresos_pc, '$<br>Ganancia por atl:', ganancia_pc))%>%
  layout(title = "Universidades más exitosas",
         xaxis = list(title = "Ingreso por atleta"),
         yaxis = list (title = "Ganancia por atleta"),
         legend = list(title='Sector'))

fig_univ_dest
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Universidades%20destcadas-1.png)<!-- -->

Una última salvedad en torno a las apuestas de las universidades tiene
que ver con el **potencial de ganancias por atleta** que se esconde más
allá de los grandes ingresos del football y el basket. Si bien esos dos
deportes son los que constituyen el principal mercado de deporte
universitario por el fenómeno de masas que generan sus equipos, llama la
atención que las mejores ganancias por atleta registradas por las
universidades han sido en otros deportes: **tenis y golf**.

Esto es un fenómeno interesante a evaluar en un sentido de inversión:
mientras que un equipo de football implica una inversión en una gran
cantidad de jugadores, en el tenis y el golf esto se reduce a la
inversión en el único jugador. En este sentido, al comparar deportes de
equipo con deportes individuales, se trata de la comparación entre **dos
estrategias de inversión** en el mundo deportivo universitario,
identificables en un **trade-off riesgo-rendimiento**. Mientras que la
inversión en deportes como el football o el basket pueden tener retornos
más bajos por jugador, se trata de una situación de menor riesgo dado
que la dinámica del juego y otros componentes de un equipo pueden suplir
los fallos de una persona; mientras que los deportes individuales pueden
generar ganancias por atleta exponencialmente más altas cuando alcanzan
un determinado logro, pero la exposición de la universidad al
comportamiento y desempeño de ese único atleta resulta más arriesgada.

De hecho, si bien el *soccer* y el baseball se presentan como el tercer
y cuarto deporte que más ingreso generan en total, esto no se da en la
misma medida cuando se habla de ingreso por atleta.

Un caso de éxito es el de la Greensboro College, que ha registrado el
mayor ingreso de su serie histórica gracias al golf en el año 2018,
superando por mucho lo generado por cualquier otro deporte. En ese año,
la institución ingreso 11695028 dólares en esta actividad. Lo más
destaclable aún: proviene de una de sus 4 golfistas mujeres. Esto
disparó su rentabilidad general, posicionando a la institución como la
más rentable de todas.

``` r
universidades_rent <- sports_fin%>%
  group_by(institution_name, sector)%>%
  summarise(rentabilidad=mean(rentabilidad, na.rm = TRUE))%>%
  arrange(desc(rentabilidad))
reactable(universidades_rent)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Rentabilidad%20universidades-1.png)<!-- -->
En la misma dirección, la segunda institución más rentable, la West
Virginia University, ha registrado sus mayores ingresos por atleta en el
**tenis**, mientras las ganancias más grandes las ha visto en **buceo y
natación** y las mejores relaciones ingreso/gasto se dan en el **tenis
para las mujeres** y en el **golf para los hombres**.

``` r
west_virginia <- sports_fin%>%
  filter(institution_name=='West Virginia University')%>%
  select(institution_name, sports, rentabilidad, ingreso_pc, ganancia, rent_women, rent_men)%>%
  arrange(desc(rentabilidad))
reactable(west_virginia)
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/West%20Virginia-1.png)<!-- -->

## La brecha de género en el deporte

El conjunto de datos ofrece información interesante para evaluar si hay
una brecha de género en el deporte universitario, si esta diferencia ha
variado a lo largo de los últimos años y si es que se puede avanzar en
un esquema más igualitario y a la vez rentable para las instituciones a
partir de una mayor inversión en atletas mujeres.

Varios informes ya han estudiado en estos años que el gasto en atletas
hombres ha sido mayor por persona que en el caso de las mujeres.

``` r
gasto_genero <- sports_fin%>%
  group_by(year)%>%
  summarise(gasto_hombres=sum(exp_men, na.rm = TRUE),
            gasto_mujeres=sum(exp_women, na.rm = TRUE),
            hombres = sum(sum_partic_men, na.rm = TRUE),
            mujeres=sum(sum_partic_women, na.rm = TRUE),
            gasto_ph = gasto_hombres/hombres,
            gasto_pm = gasto_mujeres/mujeres,
            brecha=gasto_ph-gasto_pm)


fig_brecha <- plot_ly(gasto_genero, x = ~year, y = ~gasto_ph, type = 'bar', name = 'Gasto por hombre')
fig_brecha <- fig_brecha %>% add_trace(y = ~gasto_pm, name = 'Gasto por mujer')
fig_brecha <- fig_brecha %>% layout(yaxis = list(title = 'Gasto por atleta'),
                                    xaxis = list(title = 'Año'),
                                    barmode = 'group')
fig_brecha <- fig_brecha %>% add_trace(y = ~brecha, type = 'scatter', mode = 'lines', line = list(color = 'rgb(205, 12, 24)', width = 4), name="Brecha de gasto")
fig_brecha
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

De hecho, puede apreciarse que entre 2015 y 2018 esta brecha no hizo más
que creer. Esta brecha importante puede deberse a que los dos deportes
más importantes para el mercado, el fútbol americano y el basket, no
son jugados por mujeres. Es interesante entonces analizar el caso del
soccer, un deporte que en EEUU tiene un gran peso femenino y que puede
resultar más equitativo en términos de inversión y generación.

``` r
soccer <- sports_fin%>%
  filter(sports=='Soccer' & year==2019)

soccer_res <- data.frame(
  mujeres=sum(soccer$sum_partic_women, na.rm = TRUE),
  hombres=sum(soccer$sum_partic_men, na.rm = TRUE),
  ingresos_m=sum(soccer$rev_women, na.rm = TRUE),
  ingresos_h=sum(soccer$rev_men, na.rm = TRUE))

soccer_res <- soccer_res%>%
  mutate(ingresos_pm = round((ingresos_m/mujeres),2),
         ingresos_ph = round((ingresos_h/hombres),2))
```

Una cuestión a observar es que el número de hombres y mujeres es muy
similar: tomando los datos del último de estudio, 2019, vemos que hay
41305 mujeres que practican el deporte y 41937 que lo hacen.

Y si bien la cantidad de atletas es similar, no lo es la masa de
ingresos generada. Mientras que en los **hombres generaron un total de
415373515 dólares** durante el 2019, **las mujeres generaron 525334467
dólares**, es decir, un 26.47% más. Esta diferencia puede ser un primer
indicio del valor que podría tener profundizar la inversión en otros
deportes donde las mujeres de momento no participan, como el baseball, o
en otras donde perciben una menor inversión de parte de las
universidades, como el basket.

#### ¿Demócratas o republicanos?

Una última salvedad tiene que ver con la cuestión cultural detrás de la
brecha de género. Casi todo el conjunto de datos con el que se trabajó
fue recolectado durante la presidencia de Donald Trump en los Estados
Unidos. Desde su campaña, el feminismo fue una cuestión central en la
agenda de los medios, así como lo fue también durante su gestión. Es
interesante ver que la asociación genérica que se planteaba durante la
campaña de 2016, marcando una polaridad entre los candidatos y sus
posiciones sobre las desigualdades de género, no se expresó en el plano
del deporte universitario al menos de manera lineal.

Esto ha de ser complejizado, pero con una primera impresión puede verse
que en 7 de 10 de los estados que más dinero invierten por atleta mujer,
el ganador de la elección de 2016 fue Trump. Es decir: las posiciones
políticas mayoritarias (o incluso culturales) de los estados al momento
de la elección no se reflejaron lineal ni directamente en la inversión
deportiva universitaria.

``` r
estados_eleccion <- sports_fin%>%
  filter(year==2016)%>%
  group_by(state_name)%>%
  summarise(women=sum(sum_partic_women, na.rm = TRUE),
            men = sum(sum_partic_men, na.rm = TRUE),
            exp_women = sum(exp_women, na.rm = TRUE),
            exp_men = sum(exp_men, na.rm = TRUE))%>%
  mutate(exp_per_women = exp_women/women,
         exp_per_men = exp_men/men)


elecciones <- read.csv('https://raw.githubusercontent.com/kjhealy/us_election_2016/0aafba6c5784a0c30d692637836ddf312437b0df/data/election_state_16.csv')
names(elecciones)
```

    ##  [1] "state"        "st"           "fips"         "total_vote"   "vote_margin" 
    ##  [6] "winner"       "party"        "pct_margin"   "r_points"     "d_points"    
    ## [11] "pct_clinton"  "pct_trump"    "pct_johnson"  "pct_other"    "clinton_vote"
    ## [16] "trump_vote"   "johnson_vote" "other_vote"   "ev_dem"       "ev_rep"      
    ## [21] "ev_oth"       "census"

``` r
elecciones <- select(elecciones, state, winner)
estados_eleccion <- left_join(estados_eleccion, elecciones, by=c('state_name'='state'))

fig_elecciones <- plot_ly(estados_eleccion, type='bar', x = ~state_name, y = ~exp_per_women, text = ~winner, 
               color=~winner, colors = c(Trump='red', Clinton='royalblue3'),
               hovertemplate = paste('%{x}','<br>%{text}', '<br>exp_per_women: %{y:.2s}<br>'))
fig_elecciones <- fig_elecciones %>% layout(uniformtext=list(minsize=4, mode='hide'))%>% 
  layout(xaxis = list(categoryorder = "total ascending"))
fig_elecciones
```

![](Equity-in-Athletics---Cohen_files/figure-gfm/Elecciones-1.png)<!-- -->
