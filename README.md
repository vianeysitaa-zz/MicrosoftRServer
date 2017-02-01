# MicrosoftRServer
Comparación simple entre Microsoft R Server y Open Source R

# Microsoft R Server vs. Open Source R: Predicción de retrasos de vuelos

Este ejemplo tiene como objetivo comparar el rendimiento y la exactitud de los modelos entre Microsoft R Server (MRS) y Open Source R (obtenido de CRAN) utilizando el ejemplo "Flight Delay Prediction con Microsoft R Server": https://msdn.microsoft.com/en-us/microsoft-r/scaler-getting-started-0-example-airline-data


En el ejemplo anterior, se utilizan datos históricos de tiempo de ejecución y clima para predecir si la llegada de un vuelo de pasajeros programado se retrasará en más de 15 minutos.

La comparación está hecha en una máquina con las siguientes caracterísitcas [Máquina Virtual para Data Science de Microsoft]: 
- Sistema Operativo: Windows
- Tipo de sistema: 64-bit
- Cores: 4
- Memoria RAM: 28.0 GB
- Versión de Microsoft R Server: 8.0.0
- Version de R: 3.2.2

Esta máquina la pueden generar ustedes, lo único que necesitan es una subscripción a Azure [Pueden utilizar un free trial] y comenzar a usarla. Este equipo cuenta con instalación no solo de RServer, sino de Anaconda Python, Jupyter, Power BI Desktop, Visual Studio Community, SQL Server 2016 Developer Edition, etc. Si quieren conocer más de este equipo, o quieren generar uno, acá pueden encontrar las instrucciones: https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-data-science-provision-vm

En este ejemplo, tenemos una comparación paso a paso en los cinco pasos de "Flight Delay Prediction con Microsoft R Server":

- [Paso 1: Importar información](#anchor-1)
- [Paso 2: Pre-procesar datos](#anchor-2)
- [Paso 3: Preparar los conjuntos de datos de entrenamiento y prueba](#anchor-3)
- [Paso 4A: Escoger y aplicar un algoritmo de aprendizaje (Regresión logística)](#anchor-4A)
- [Paso 4B: Escoger y aplicar un algoritmo de aprendizaje (Árbol de decisión)](#anchor-4B)
- [Paso 5A: Predicción (Regresión logística)](#anchor-5A)
- [Paso 5B: Predicción (Árbol de decisión)](#anchor-5B)

Asimismo, comparamos la exactitud de los modelos, utilizando la "Área bajo la Curva (AUC)" como medida.

- [comparación de exactitud de los modelos](#anchor-6)

Datos y Scripts de R que serán utilizados:
- **Flight Delays Data.csv** (unzip the **Flight Delays Data.zip** file)
- **Weather Data.csv**
- **MRS_flight_delays.R**
- **R_flight_delays.R**

Las conclusiones se basan solo en el desempeño de los modelos y la comparación de la exactitud de los modelos.
- [Conclusiones](#anchor-7)


------------------------------------------

## <a name="anchor-1"></a> Paso 1: Importar Datos

Nombres de sub-pasos | Tiempo de ejecución en MRS (en segundos) | Tiempo de ejecución en R (en segundos)
---------------------| ---------------------------------------- | -------------------------------------
Importar datos de vuelos (2,719,418*14) | 10.86 | 15.60
Importar la información del clima y eliminar algunas características(404,914*10) | 2.33 | 1.79

```
### Microsoft R Server:
# Importar la información de vuelos.
system.time(flight_mrs <- rxImport(inData = inputFileFlight, outFile = outFileFlight,
                                   missingValueString = "M", stringsAsFactors = TRUE)
            )  # tiempo: 10.86 segundos

# Importar la información del clima y eliminar datos redundantes.
system.time(weather_mrs <- rxImport(inData = inputFileWeather, outFile = outFileWeather,
                                    missingValueString = "M", stringsAsFactors = TRUE,
                                    varsToDrop = c('Year', 'Timezone', 'DryBulbFarenheit', 'DewPointFarenheit'),
                                    overwrite=TRUE)
            )  # tiempo: 2.33 segundos
```

```
### R Open Source:
# Importar la información de vuelos.
system.time(flight_r <- read.csv(file = inputFileFlight, na.strings = "NA",
                                 stringsAsFactors = TRUE)
            )  # tiempo: 15.60 segundos

# Importar la información del clima y eliminar datos redundantes.
system.time(weather_r <- subset(read.csv(file = inputFileWeather, na.strings = "NA",
                                         stringsAsFactors = TRUE),
                                select = -c(Year, Timezone, DryBulbFarenheit, DewPointFarenheit)
                                )
            )  # tiempo: 1.79 segundos
```


## <a name="anchor-2"></a> Paso 2: Pre-procesar Datos

Nombres de sub-pasos | Tiempo de ejecución en MRS (en segundos) | Tiempo de ejecución en R (en segundos)
---------------------| ---------------------------------------- | -------------------------------------
Remover las filtraciones de destino de los datos de vuelo y 
redondear a la baja la hora de salida programada para obtener una hora completa | 4.75 | 0.03
Renombrar algunas columnas de los datos del tiempo para prepararlo para el cruce de información | 0.34 | <0.01
Unión de los registros de vuelo y datos meteorológicos por OriginAirportID | 28.44 | 43.92
Unión de los registros de vuelo y datos meteorológicos por DestAirportID | 36.25 | 37.53
Normalizar algunas características numéricas y convertir algunas características para ser categóricas | 10.39 | 39.98

```
### Microsoft R Server:
# Remover las filtraciones de destino de los datos de vuelo.
varsToDrop <- c('DepDelay', 'DepDel15', 'ArrDelay', 'Cancelled', 'Year')

# Redondear a la baja la hora de salida programada para obtener una hora completa.
xform <- function(dataList) {
  # Crear una nueva variable continua a partir de una variable continua existente:
  # Redondear a la baja la columna CSRDepTime a la hora más cercana.
  dataList$CRSDepTime <- sapply(dataList$CRSDepTime,
                                FUN = function(x) {floor(x/100)})

  # Regresar la lista adaptada.
  return(dataList)
}
system.time(flight_mrs <- rxDataStep(inData = flight_mrs,
                                   outFile = outFileFlight2,
                                   varsToDrop = varsToDrop,
                                   transformFunc = xform,
                                   transformVars = 'CRSDepTime',
                                   overwrite=TRUE
                                   )
            )  # tiempo: 4.75 segundos

# Renombrar algunas columnas de los datos del tiempo para prepararlo para el cruce de información.
xform2 <- function(dataList) {
  # Crear una nueva columna 'DestAirportID' en los datos del clima.
  dataList$DestAirportID <- dataList$AirportID
  # Rnombrar 'AdjustedMonth', 'AdjustedDay', 'AirportID', 'AdjustedHour'.
  names(dataList)[match(c('AdjustedMonth', 'AdjustedDay', 'AirportID', 'AdjustedHour'),
                 names(dataList))] <- c('Month', 'DayofMonth', 'OriginAirportID', 'CRSDepTime')

  # Regresar la lista adatpada.
  return(dataList)
}
system.time(weather_mrs <- rxDataStep(inData = weather_mrs,
                                      outFile = outFileWeather2,
                                      transformFunc = xform2,
                                      transformVars = c('AdjustedMonth', 'AdjustedDay', 'AirportID', 'AdjustedHour'),
                                      overwrite=TRUE
                                      )
            )  # tiempo: 0.34 segundos

# Concatenar los registros de vuelos y el clima.
# 1). Unir los registros de vuelos y clima utilizando el origen del vuelo (OriginAirportID).
system.time(originData_mrs <- rxMerge(inData1 = flight_mrs, inData2 = weather_mrs, outFile = outFileOrigin,
                                      type = 'inner', autoSort = TRUE, decreasing = FALSE,
                                      matchVars = c('Month', 'DayofMonth', 'OriginAirportID', 'CRSDepTime'),
                                      varsToDrop2 = 'DestAirportID',
                                      overwrite=TRUE
                                      )
            )  # tiempo: 28.44 segundos

# 2). Unir los registros de vuelos y clima utilizando el destino del vuelo (DestAirportID).
system.time(destData_mrs <- rxMerge(inData1 = originData_mrs, inData2 = weather_mrs, outFile = outFileDest,
                                    type = 'inner', autoSort = TRUE, decreasing = FALSE,
                                    matchVars = c('Month', 'DayofMonth', 'DestAirportID', 'CRSDepTime'),
                                    varsToDrop2 = c('OriginAirportID'),
                                    duplicateVarExt = c("Origin", "Destination"),
                                    overwrite=TRUE
                                    )
            )  # tiempo: 36.25 segundos

# Normalizar algunas características numéricas y convertir algunas características para ser categóricas.
system.time(finalData_mrs <- rxDataStep(inData = destData_mrs, outFile = outFileFinal,
                                        transforms = list(
                                                          # Normalizar algunas características numéricas
                                                          Visibility.Origin = scale(Visibility.Origin),
                                                          DryBulbCelsius.Origin = scale(DryBulbCelsius.Origin),
                                                          DewPointCelsius.Origin = scale(DewPointCelsius.Origin),
                                                          RelativeHumidity.Origin = scale(RelativeHumidity.Origin),
                                                          WindSpeed.Origin = scale(WindSpeed.Origin),
                                                          Altimeter.Origin = scale(Altimeter.Origin),
                                                          Visibility.Destination = scale(Visibility.Destination),
                                                          DryBulbCelsius.Destination = scale(DryBulbCelsius.Destination),
                                                          DewPointCelsius.Destination = scale(DewPointCelsius.Destination),
                                                          RelativeHumidity.Destination = scale(RelativeHumidity.Destination),
                                                          WindSpeed.Destination = scale(WindSpeed.Destination),
                                                          Altimeter.Destination = scale(Altimeter.Destination),

                                                          # Convertir 'OriginAirportID', 'DestAirportID' a categóricas
                                                          OriginAirportID = factor(OriginAirportID),
                                                          DestAirportID = factor(DestAirportID)
                                                          ),
                                        overwrite=TRUE
                                        )
            )  # tiempo: 10.39 segundos
```

```
### R Open Source:
# Remover las filtraciones de destino de los datos de vuelo.
# Redondear a la baja la hora de salida programada para obtener una hora completa.
xform_r <- function(df) {
  # Remover las filtraciones de destino de los datos de vuelo.
  varsToDrop <- c('DepDelay', 'DepDel15', 'ArrDelay', 'Cancelled', 'Year')
  df <- df[, !(names(df) %in% varsToDrop)]

  # Redondear a la baja la hora de salida programada para obtener una hora complta.
  df$CRSDepTime <- floor(df$CRSDepTime/100)

  # Regresar el DataFrame.
  return(df)
}
system.time(flight_r <- xform_r(flight_r))  # tiempo: 0.03 segundos

# Renombrar algunas columnas de los datos del tiempo para prepararlo para el cruce de información.
xform2_r <- function(df) {
  # Crear una columna 'DestAirportID' en los datos del clima.
  df$DestAirportID <- df$AirportID

  # Renombrar 'AdjustedMonth', 'AdjustedDay', 'AirportID', 'AdjustedHour'.
  names(df)[match(c('AdjustedMonth', 'AdjustedDay', 'AirportID', 'AdjustedHour'),
                  names(df))] <- c('Month', 'DayofMonth', 'OriginAirportID', 'CRSDepTime')

  # Regresar el DataFrame.
  return(df)
}
system.time(weather_r <- xform2_r(weather_r))  # tiempo: <0.01 segundos

# Concatenar los registros de vuelos y el clima.
1). Unir los registros de vuelos y clima utilizando el origen del vuelo (OriginAirportID).
mergeFunc <- function(df1, df2) {
  # Remover la columna "DestAirportID" antes del cruce de información.
  df2 <- subset(df2, select = -DestAirportID)

  # Concatenar los dos DataFrames.
  dfOut <- merge(df1, df2,
                 by = c('Month', 'DayofMonth', 'OriginAirportID', 'CRSDepTime'))

  # Regresar el DataFrame.
  return(dfOut)
}
system.time(originData_r <- mergeFunc(flight_r, weather_r)) # tiempo: 43.92 segundos

# 2). Unir los registros de vuelos y clima utilizando el destino del vuelo (DestAirportID).
mergeFunc2 <- function(df1, df2) {
  # Remover la columna "OriginAirportID" antes del cruce de información.
  df2 <- subset(df2, select = -OriginAirportID)

  # Concatenar los dos DataFrames.
  dfOut <- merge(df1, df2,
                 by = c('Month', 'DayofMonth', 'DestAirportID', 'CRSDepTime'),
                 suffixes = c(".Origin", ".Destination"))

  # Regresar el DataFrame.
  return(dfOut)
}
system.time(destData_r <- mergeFunc2(originData_r, weather_r)) # tiempo: 37.53 segundos

# Normalizar algunas características numéricas y convertir algunas características para ser categóricas.
scaleVar <- c('Visibility.Origin', 'DryBulbCelsius.Origin', 'DewPointCelsius.Origin',
              'RelativeHumidity.Origin', 'WindSpeed.Origin', 'Altimeter.Origin',
              'Visibility.Destination', 'DryBulbCelsius.Destination', 'DewPointCelsius.Destination',
              'RelativeHumidity.Destination', 'WindSpeed.Destination', 'Altimeter.Destination')

# Características que necesitan convertirse a categóricas.
cateVar <- c('OriginAirportID', 'DestAirportID')

xform3_r <- function(df) {
  # Normalización.
  df[, scaleVar] <- sapply(df[, scaleVar], FUN = function(x) {scale(x)})

  # conversión a categórico.
  df[, cateVar] <- sapply(df[, cateVar], FUN = function(x) {factor(x)})

  # Regresar el DataFrame.
  return(df)
}
system.time(finalData_r <- xform3_r(destData_r))  # tiempo: 39.98 segundos
```


## <a name="anchor-3"></a> Paso 3: Preparar los conjuntos de datos de entrenamiento y prueba

Nombres de sub-pasos | Tiempo de ejecución en MRS (en segundos) | Tiempo de ejecución en R (en segundos)
---------------------| ---------------------------------------- | -------------------------------------
Selección aleatoria 80/20 como entrenamiento/prueba | 9.19 | 0.04

```
### Microsoft R Server:
# División aleatoria del 80% de los datos para entrenar el modelo y el 20%  restante para probarlo.
system.time(rxExec(rxSplit, inData = finalData_mrs,
                   outFilesBase="finalData",
                   outFileSuffixes=c("Train", "Test"),
                   splitByFactor="splitVar",
                   overwrite=TRUE,
                   transforms=list(splitVar = factor(sample(c("Train", "Test"), size=.rxNumRows, replace=TRUE, prob=c(.80, .20)),
                                   levels= c("Train", "Test"))),
                   rngSeed=17,
                   consoleOutput=TRUE
                   )
            )  # tiempo: 9.19 segundos
```

```
### R Open Source:
# División aleatoria del 80% de los datos para entrenar el modelo y el 20%  restante para probarlo.
set.seed(17)
system.time(sub <- sample(nrow(finalData_r), floor(nrow(finalData_r) * 0.8)))  # tiempo: 0.04 segundos
```


## <a name="anchor-4A"></a> Paso 4A: Escoger y aplicar un algoritmo de entrenamiento (Regresión logística)

Nombres de sub-pasos | Tiempo de ejecución en MRS (en segundos) | Tiempo de ejecución en R (en segundos)
---------------------| ---------------------------------------- | -------------------------------------
Construcción de un modelo de regresión logística | 5.50 | 229.32

```
### Microsoft R Sserver:
# Construcción de un modelo de regresión logística.
system.time(logitModel_mrs <- rxLogit(form, data = 'finalData.splitVar.Train.xdf'))  # tiempo: 5.50 segundos
```

```
### R Open Source:
# Construcción de un modelo de regresión logística.
system.time(logitModel_r <- glm(form, data = train, family = "binomial"))  # tiempo: 229.32 segundos
```


## <a name="anchor-5A"></a> Paso 5A: Predicción (Regresión logísica)

Nombres de sub-pasos | Tiempo de ejecución en MRS (en segundos) | Tiempo de ejecución en R (en segundos)
---------------------| ---------------------------------------- | -------------------------------------
Predecir la probabilidad en los datos de prueba | 0.56 | 1.72

```
### Microsoft R Server:
# Predecir la probabilidad en los datos de prueba.
system.time(predictLogit_mrs <- rxPredict(logitModel_mrs, data = 'finalData.splitVar.Test.logit.xdf',
                                          type = 'response', overwrite = TRUE)
            ) # tiempo: 0.56 segundos
```

```
### R Open Source:
# Predecir la probabilidad en los datos de prueba.
system.time(predictLogit_r <- predict(logitModel_r, newdata = test, type = 'response'))  # tiempo: 1.72 segundos
```


## <a name="anchor-4B"></a> Paso 4B: Escoger y aplicar un algoritmo de entrenamiento (Árbol de decisión)

Nombres de sub-pasos | Tiempo de ejecución en MRS (en segundos) | Tiempo de ejecución en R (en segundos)
---------------------| ---------------------------------------- | -------------------------------------
Construcción de un modelo de árbol de decisión| 151.69 | 360.10
Podar un árbol de decisión para devolver el árbol más pequeño | 0.03 | 4.25

```
### Microsoft R Server:
# Construcción de un modelo de árbol de decisión.
system.time(dTree1_mrs <- rxDTree(form, data = 'finalData.splitVar.Train.xdf'))  # tiempo: 151.69 segundos

# Podar un árbol de decisión para devolver el árbol más pequeño.
system.time(dTree2_mrs <- prune.rxDTree(dTree1_mrs, cp = treeCp_mrs))  # tiempo: 0.03 segundos
```

```
### R  Open Source:
# Construcción de un modelo de árbol de decisión.
(if(!require("rpart")) install.packages("rpart"))
library(rpart)
system.time(dTree1_r <- rpart(form, data = train, method = 'class',
                              control=rpart.control(minsplit = 20, minbucket = 1, cp = 0,
                                                    maxcompete = 0, maxSurrogate = 0,
                                                    xval = 2, maxDepth = 10)))  # tiempo: 360.10 segundos

# Podar un árbol de decisión para devolver el árbol más pequeño.
system.time(dTree2_r <- prune(dTree1_r, cp = treeCp_r))   # tiempo: 4.25 segundos
```


## <a name="anchor-5B"></a> Paso 5B: Predicción (Árbol de decisión)


Nombres de sub-pasos | Tiempo de ejecución en MRS (en segundos) | Tiempo de ejecución en R (en segundos)
---------------------| ---------------------------------------- | -------------------------------------
Predecir la probabilidad en los datos de prueba | 1.16 | 13.53

```
### Microsoft R Server:
# Predecir la probabilidad en los datos de prueba.
system.time(predictTree_mrs <- rxPredict(dTree2_mrs, data = 'finalData.splitVar.Test.tree.xdf',
                                         overwrite = TRUE)
            )  # tiempo: 1.16 segundos
```

```
### Open Source R:
# Predecir la probabilidad en los datos de prueba.
system.time(predictTree_r <- predict(dTree2_r, newdata = test, type = 'prob'))  # tiempo: 13.53 segundos
```


## <a name="anchor-6"></a> Comparación de Precisión de los Modelos

EL modelo de regresión logísitca tiene un `AUC` de **0.70** en Microsoft R Server y R Open Source.
```
### Microsoft R Server:
# Calcular el área bajo la curva (AUC).
rxAuc(rxRoc("ArrDel15", "ArrDel15_Pred", predictLogit_mrs))  # AUC = 0.70
```

```
### R Open Source:
# Calcular el área bajo la curva (AUC).
auc <- function(outcome, prob){
  N <- length(prob)
  N_pos <- sum(outcome)
  df <- data.frame(out = outcome, prob = prob)
  df <- df[order(-df$prob),]
  df$above <- (1:N) - cumsum(df$out)
  return( 1- sum( df$above * df$out ) / (N_pos * (N-N_pos) ) )
}
auc(testLogit$ArrDel15, testLogit$ArrDel15_Pred)  # AUC = 0.70
```

Para el modelo de árbol de decisión, utilizamos la funcipn 'rpart' para comparar con la función 'rxDTree' de Microsoft Open Source. Las dos funciones pueden ser utilizadas con árboles de decisión. Sin embargo, existen algunos parámetros como 'minisplit', `minbucket`, `maxcompete`, `maxSurrogate`, `xval`, `maxDepth`, etc., que tienen diferenes valores por defecto en cada una de las funciones.

- `minSplit`: El número mínimo de observaciones que debe existir en un nodo antes de intentar una división.
- `minBucket`: El número mínimo de observaciones en un nodo terminal (o hoja).
- `maxCompete`: El número máximo de divisiones del competidor retenidas en la salida. Estos son útiles diagnósticos de modelos, ya que permiten comparar divisiones en la salida con las alternativas.
- `maxSurrogate`: El número máximo de divisiones de sustitución retenidas en la salida.
- `xVal`: Número de validaciones cruzadas realizadas en el ajuste del modelo.
- `maxDepth`: La profundidad máxima de cualquier nodo de árbol.

Para tener una comparación justa, ajustamos los valores por defecto de estos parámetros en la función 'rpart' para igualar los valores por defecto de 'rxDTree'. Después de podar estos árboles para encontrar el mejor valor de 'cp' (Escalar numérico especificando el parámetro de complejidad), 'rxDTree' encuentra el mejor valor de 'cp' como **2.156921e-05** y R Open Source 'rpart' encuentra el mejor valor de 'cp' como **1.063143e-05**. 

Al usar esos dos valores de 'cp' para podar los árboles, el modelo de Árbol de Decisión, tiene un 'AUC' de **0.73** en Microsoft R Server, y **0.74** en R Open Source. 

## <a name="anchor-7"></a> Conclusiones

Dado que las precisiones del modelo son las mismas tanto en Microsoft R Server como en R Open Source, queremos comparar el tiempo de ejecución general y paso a paso en Microsoft R Server y R. Con base en la ** Comparación de tiempos de ejecución **, podemos ver que Microsoft R Server toma 261.49 segundos, que es sólo el 35% del tiempo total de ejecución en código abierto R.


1. En general, el servidor Microsoft R tiene un tiempo de ejecución menor (** 261,49 segundos **) comparado con R Open Source (747,82 segundos). Esto conduce a una mejora de ** 65% ** en términos del rendimiento. 
2. El servidor Microsoft R tiene un mejor rendimiento al importar conjuntos de datos a gran escala comparados con R Open Source. Los conjuntos de datos a gran escala pueden considerarse como millones de registros. Para el conjunto de datos a nivel de miles, Microsoft R Server parece tener el mismo rendimiento que R Open Source. 
3. Microsoft R Server funciona muy bien en la etapa de pre-procesamiento de datos, especialmente al unir dos conjuntos de datos relativamente grandes y editar los metadatos.
4. Microsoft R Server utiliza un poco más de tiempo al dividir los datos de forma aleatoria en conjuntos de datos de entrenamiento y prueba. Pero como está escribiendo conjuntos de datos en archivos externos `.xdf` al mismo tiempo y R Open Source no tiene este paso intermedio, la pequeña diferencia de tiempo es aceptable. 
5. Microsoft R Server realmente se destaca cuando se trata del aprendizaje de algoritmos (modelos de entrenamiento) y se predice sobre los nuevos datos (evaluación de modelos). Para los modelos de Regresión Logística y Árbol de Decisión, el Servidor Microsoft R ahorra una cantidad significativa en tiempo de ejecución en estos dos pasos, comparado con R Open Source. Además, los algoritmos de aprendizaje y la predicción son los que consumen la mayor parte del tiempo. Por lo tanto, mejora el rendimiento total.



<!-- Links -->
[Flight Delay Prediction with Microsoft R Server]: https://github.com/mezmicrosoft/Microsoft_R_Server/tree/master/Flight_Delay_Prediction
[Microsoft Data Science Virtual Machine]: https://azure.microsoft.com/en-us/marketplace/partners/microsoft-ads/standard-data-science-vm/
