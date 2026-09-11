
  MODELADO DE METAMATERIALES MEDIANTE REDES NEURONALES

--------------------------------------------------------------------------------
DESCRIPCION DEL PROYECTO
--------------------------------------------------------------------------------

Los metamateriales elasticos son estructuras artificiales cuyas propiedades
mecanicas dependen de la geometria de su celda unitaria, no de su composicion
quimica. Una de sus propiedades mas relevantes es el "band gap": la capacidad
de bloquear rangos de frecuencias de vibracion, caracterizado por su frecuencia
central (BandGapLocation) y su anchura (BandGapWidth).

Determinar que configuraciones geometricas producen un band gap especifico
mediante simulacion por elementos finitos puede tomar horas por diseno. Este
proyecto entrena una red neuronal artificial como modelo sustituto, capaz de
predecir instantaneamente las propiedades de nuevas geometrias.

Dataset: 2D Elastodynamic Metamaterials — UCI Machine Learning Repository
  - 1274 muestras utilizadas (el resto presentaba la columna de entrada en
    notacion cientifica, imposibilitando recuperar los 15 pixeles binarios)
  - Entrada : 15 pixeles binarios de la celda unitaria (codificados como entero)
  - Salidas : BandGapLocation y BandGapWidth (valores continuos de simulacion)

Tipo de problema: regresion supervisada con multiples salidas


--------------------------------------------------------------------------------
PREPROCESAMIENTO
--------------------------------------------------------------------------------

1. Se toman los primeros 1274 registros del dataset.
2. Los datos se aleatorizan con semilla fija (random_state=42).
3. La columna CondensedBinary2DGeometry se decodifica: cada entero se convierte
   a su representacion binaria de 15 digitos (zfill(15)), produciendo el vector
   de entrada real para la red.
4. Division: 75% entrenamiento / 25% prueba.
5. Normalizacion min-max aplicada SOLO a las variables de salida. Los parametros
   (min y max) se calculan exclusivamente sobre el conjunto de entrenamiento y
   luego se aplican al conjunto de prueba para evitar fuga de informacion.
6. La columna de entrada no se normaliza por ser binaria.


--------------------------------------------------------------------------------
MODELOS
--------------------------------------------------------------------------------

--- SINGLE OUTPUT ---

Se entrenan dos redes independientes, una por variable de salida.

  BandGapLocation:
    Capas ocultas  : 2
    Neuronas/capa  : 16
    Learning rate  : 0.001
    Funcion perdida: MSE

  BandGapWidth:
    Capas ocultas  : 3
    Neuronas/capa  : 256
    Learning rate  : 0.0075
    Funcion perdida: MAE (elegida por la distribucion sesgada de la variable)

--- MULTIPLE OUTPUT (modelo seleccionado) ---

Una sola red predice ambas variables simultaneamente.

    Capas ocultas  : 3
    Neuronas/capa  : 128
    Learning rate  : 0.01
    Funcion perdida: MAE
    Epochs maximos : 150
    Batch size     : 50
    Optimizador    : Adam

Elementos fijos en todos los modelos:
  - Activacion capas ocultas : ReLU
  - Activacion capa de salida: lineal (regresion de valores continuos)
  - Porcentaje entrenamiento  : 75%


--------------------------------------------------------------------------------
ESTUDIO DE HIPERPARAMETROS
--------------------------------------------------------------------------------

Se evaluaron sistematicamente las siguientes combinaciones:

  Single output : 18 configuraciones por variable
    n_capas     : 1, 2, 3
    neuronas    : 16, 64, 256
    lr          : 0.001, 0.005

  Multiple output : 16 configuraciones
    n_capas     : 2, 3
    neuronas    : 16, 32, 64, 128
    lr          : 0.001, 0.01

La comparacion se realizo por val_loss final, complementada con graficas de
curvas de entrenamiento para detectar sobreajuste o convergencia insuficiente.


--------------------------------------------------------------------------------
RESULTADOS
--------------------------------------------------------------------------------

--- SINGLE OUTPUT ---

  Margen de error    BandGapLocation    BandGapWidth
  -------------------------------------------------------
  5%                 24.5%              11.3%
  10%                46.9%              19.8%
  20%                74.5%              35.8%
  Error prom.        14.6%              86.4%

--- MULTIPLE OUTPUT ---

  Margen de error    BandGapLocation    BandGapWidth
  -------------------------------------------------------
  5%                 43.1%               8.8%
  10%                62.3%              14.8%
  20%                84.0%              30.2%
  Error prom.        11.4%              89.8%

El modelo de multiple output supera al de single output en BandGapLocation
(11.4% vs 14.6%) y presenta desempeño comparable en BandGapWidth. Al predecir
ambas variables con una sola red, se selecciono como modelo final.

Nota: el alto error en BandGapWidth se atribuye principalmente a dos factores:
  1. Distribucion sesgada: la mayoria de valores se concentra cerca de cero,
     lo que infla el error porcentual incluso con diferencias absolutas pequenas.
  2. Dataset reducido: solo 1274 muestras (~3.9% del espacio de diseño posible),
     insuficientes para capturar la relación altamente no lineal de esta variable.


--------------------------------------------------------------------------------
ANALISIS DE SENSIBILIDAD POR PIXEL
--------------------------------------------------------------------------------

Se evaluo el impacto de invertir (flip) cada uno de los 15 pixeles de la celda
unitaria sobre las predicciones del modelo, para 4 disenos seleccionados al azar.

  BandGapLocation: la influencia se concentra en la region central de la celda
    (pixeles 7 a 12). El pixel 8 fue el mas influyente en varios disenos.

  BandGapWidth: la influencia no muestra un patron geografico claro; los pixeles
    mas determinantes varian entre disenos (14, 15, 11 y 13 respectivamente).

Conclusion: los pixeles que controlan Location y Width son distintos entre si,
y ninguno es universalmente dominante; la influencia depende del contexto
geometrico especifico de cada diseno.


--------------------------------------------------------------------------------
RED NEURONAL INVERSA
--------------------------------------------------------------------------------

No es viable con la formulacion actual por dos razones:

  1. El mapeo no es inyectivo: multiples geometrias pueden producir los mismos
     valores de BandGapLocation y BandGapWidth, lo que hace que la red inversa
     promediaria geometrias validas y produciria salidas incorrectas.

  2. BandGapWidth presenta un error promedio de ~89%, lo que propagaria errores
     inaceptables al intentar reconstruir la geometria a partir de esa variable.


--------------------------------------------------------------------------------
REQUISITOS
--------------------------------------------------------------------------------

  Python >= 3.9
  tensorflow >= 2.x
  numpy
  pandas
  matplotlib
  scikit-learn


  pip install tensorflow numpy pandas matplotlib scikit-learn


--------------------------------------------------------------------------------
USO
--------------------------------------------------------------------------------

1. Colocar data.csv en la carpeta data/.

2. Ejecutar el notebook de interes en Google Colab o localmente:
     - multiple_output.ipynb  (modelo recomendado)
     - single_output.ipynb

3. El pipeline completo incluye: carga, decodificacion, normalizacion,
   busqueda de hiperparametros, entrenamiento, validacion y analisis de
   sensibilidad.
