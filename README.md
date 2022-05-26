PAV - P4: reconocimiento y verificación del locutor
===================================================

Obtenga su copia del repositorio de la práctica accediendo a [Práctica 4](https://github.com/albino-pav/P4)
y pulsando sobre el botón `Fork` situado en la esquina superior derecha. A continuación, siga las
instrucciones de la [Práctica 2](https://github.com/albino-pav/P2) para crear una rama con el apellido de
los integrantes del grupo de prácticas, dar de alta al resto de integrantes como colaboradores del proyecto
y crear la copias locales del repositorio.

También debe descomprimir, en el directorio `PAV/P4`, el fichero [db_8mu.tgz](https://atenea.upc.edu/mod/resource/view.php?id=3508877?forcedownload=1)
con la base de datos oral que se utilizará en la parte experimental de la práctica.

Como entrega deberá realizar un *pull request* con el contenido de su copia del repositorio. Recuerde
que los ficheros entregados deberán estar en condiciones de ser ejecutados con sólo ejecutar:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~.sh
  make release
  run_spkid mfcc train test classerr verify verifyerr
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Recuerde que, además de los trabajos indicados en esta parte básica, también deberá realizar un proyecto
de ampliación, del cual deberá subir una memoria explicativa a Atenea y los ficheros correspondientes al
repositorio de la práctica.

A modo de memoria de la parte básica, complete, en este mismo documento y usando el formato *markdown*, los
ejercicios indicados.

## Ejercicios.

### SPTK, Sox y los scripts de extracción de características.

- Analice el script `wav2lp.sh` y explique la misión de los distintos comandos involucrados en el *pipeline*
  principal (`sox`, `$X2X`, `$FRAME`, `$WINDOW` y `$LPC`). Explique el significado de cada una de las 
  opciones empleadas y de sus valores.
  
  **SOX:** Permite realizar la conversión de una señal de entrada sin cabecera a una del formato adecuado. Para la conversión se puede elegir cualquier formato de la señal de entrada y los bits utilizados entre otras cosas. Por ejemplo: sox, permite la conversión de una señal de entrada a reales en coma flotante de 32 bits con o sin cabecera. Además sox también permite la conversión de señales guardadas en un programa externo, para ello, solo hay que poner el URL de dicha señal, como señal de entrada.
  
  **X2X:** Es el programa de SPTK que permite la conversión entre distintos formatos de datos, tal como se puede observar en la siguiente imagen, permite bastantes tipos de conversión, desde convertir a un formato de caracteres, hasta un unsigned long long de 8 bytes. En el caso de convertir a valores numéricos, hay que especificar hasta dónde se quiere que se redondean los valores de salida.
  
  ![image](https://user-images.githubusercontent.com/100692200/169974160-91510516-64c0-4a5a-a6ad-f621bf69c0fa.png)

  **FRAME:** Divide la señal de entrada en tramas de “l” muestras con desplazamiento de ventana de periodo “p” muestras que se le indiquen y también puede elegir si el punto de comienzo es centrado o no. El valor máximo de “l” es de 256 y el de “p” es de 100. Un ejemplo sería poner: sptk frame -l 200 -p 40. En este caso le estamos pidiendo que nos divida la señal en tramas de 200 muestras, con un desplazamiento de 40 muestras.
  
  ![image](https://user-images.githubusercontent.com/100692200/169975611-bee4fba3-c85d-44fc-8ae8-d5a35b16fd50.png)

  
  **WINDOW:** Multiplica cada trama por una ventana. Se puede elegir el número “l” de muestras por trama del fichero de entrada y “L” de muestras por trama del fichero de salida, ambos solo pueden ser como mucho de 256, el tipo de normalización (si no tiene normalización, si tiene normalización power o magnitude) y el tipo de ventana que se desea utilizar, pudiendo escoger entre 6 opciones distintas de ventana, siendo las 6 ventanas más utilizadas. El tipo de dato de entrada y de salida ha de ser de tipo float.
  
  ![image](https://user-images.githubusercontent.com/100692200/169975216-fe943316-f3ff-48b6-83a2-59649133da53.png)
  
  **LPC:** Realiza un análisis de la señal, usando el método de Levinson-Durbin. Calcula los lpc_order de orden “m” (siendo como mucho 25) de los primeros coeficientes de predicción lineal, precedidos por el factor de ganancia del predictor. Se puede escoger el número “l” de muestras por trama (siendo como mucho 256) y el valor mínimo del determinante de la matriz normal (siendo como mucho 10^-6). Tanto la señal de entrada como la señal de salida tienen un formato float.
  
  ![image](https://user-images.githubusercontent.com/100692200/169975505-2901d835-1d2d-41d2-9390-86f8fd8c776f.png)



- Explique el procedimiento seguido para obtener un fichero de formato *fmatrix* a partir de los ficheros de
  salida de SPTK (líneas 45 a 47 del script `wav2lp.sh`).
  
  ![image](https://user-images.githubusercontent.com/100692200/169977226-fcc8d6cb-f1a4-4adc-a1a5-bc4ebc7b8904.png)
  
  EL procedimiento para obtener un fichero fmatrix es el detallado a continuación:
  1. Mediante el comando sox pasamos de .wav a una señal signed int de 16 bits, sin cabecera ni ningún formato adicional.
  2. Con el comando X2X convertimos los datos de formato short a float.
  3. Aplicando FRAME dividimos la señal en tramas de 240 muestras con un desplazamiento de ventana de 80 muestras.
  4. Hacemos WINDOW para enventanar la señal con una ventana Blackman con 240 muestras de entrada y de salida. 
  5. Con el comando LPC realizamos el cálculo de los ‘lpc_order’ de los primeros coeficientes de predicción lineal con el método de Levison-Durbin pasados por el parámetro -m y con un frame -l de 240 muestras. 
  6. Una vez finalizado el proceso ponemos un comando ‘>$base.lp’ para redireccionar la salida al fichero $base.lp.
  7. Una vez procesada la señal y guardada en un fichero, procedemos a calcular el número de columnas y el número de filas. 
  8. Para ello, sabiendo que en primer elemento del vector de predicción se almacena la ganancia del predictor, calculamos las columnas sumando uno al orden del predictor. 
  9. Teniendo en cuenta que queremos que en cada fila se almacene cada trama de la señal y cada columna para cada uno de los coeficientes con los que se parametriza la trama. Para realizar el cálculo del número de filas hemos de tener en cuenta la longitud de la señal y la longitud y desplazamiento de la ventana aplicada. Por eso, primero con el comando sox realizamos la conversión de datos del tipo float al tipo ascii. Y a continuación, contamos las líneas con el comando wc-1. 
  10. Finalmente, ya tenemos creada la matriz, y la imprimimos.
 

  * ¿Por qué es conveniente usar este formato (u otro parecido)? Tenga en cuenta cuál es el formato de
    entrada y cuál es el de resultado.
    
    Se usa este formato, porque para poder identificar correctamente los coeficientes por trama, la matriz nos es muy útil. Y el fichero fmatrix lo utilizamos, porque en este fichero se realiza la parametrización de una señal de voz usando coeficientes de predicción lineal, en el que hay que poner el número de filas y de columnas, seguidos por los datos. Los datos se almacenan en nrow filas de ncol columnas, en los que cada fila corresponde a una trama de señal, y cada columna a cada uno de los coeficientes con los se parametriza la trama. Así tenemos todos coeficientes de la señal a analizar en las diferenes columnas y podemos mostrarlos con el programas fmatrix_show y elegir los coeficientes 2 y 3, que son los que nos piden, con fmatrix_cut. Es decir, este formato, nos facilita trabajar de forma más eficiente con los datos.
    

- Escriba el *pipeline* principal usado para calcular los coeficientes cepstrales de predicción lineal
  (LPCC) en su fichero <code>scripts/wav2lpcc.sh</code>:
  
  <img width="573" alt="image" src="https://user-images.githubusercontent.com/101046951/170018269-b430d8b3-40f0-4f26-8200-dd24131dfc0f.png">


- Escriba el *pipeline* principal usado para calcular los coeficientes cepstrales en escala Mel (MFCC) en su
  fichero <code>scripts/wav2mfcc.sh</code>:
  
  <img width="574" alt="image" src="https://user-images.githubusercontent.com/101046951/170018355-1c7877fb-e3b6-4329-9b00-17d684a958d9.png">


### Extracción de características.

- Inserte una imagen mostrando la dependencia entre los coeficientes 2 y 3 de las tres parametrizaciones
  para todas las señales de un locutor.
  
  ![image](https://user-images.githubusercontent.com/100692200/170026247-cae66a34-05b7-4b53-80ef-02275c5f3ebe.png)

  ![image](https://user-images.githubusercontent.com/100692200/170023320-0a8c9b03-1c0b-47c9-9dc5-3b6926baf116.png)
  
  ![image](https://user-images.githubusercontent.com/100692200/170025250-17b78bad-672e-4205-8afa-6e1059f72a84.png)


  
  + Indique **todas** las órdenes necesarias para obtener las gráficas a partir de las señales 
    parametrizadas.
    
    `FEAT=lp run_spkid lp train`
    
    `fmatrix_show -H work/lp/BLOCK00/SES000/*.lp | cut -f3,4 | tr . , > lp_2_3.txt`
    
    Hacemos la gráfica
    
    `FEAT=lpcc run_spkid lpcc train`
    
    `fmatrix_show -H work/lpcc/BLOCK00/SES000/*.lpcc | cut -f3,4 | tr . , > lpcc_2_3.txt`
    
    Hacemos la gráfica
    
    `FEAT=lp run_spkid lp train`
    
    `fmatrix_show -H work/mfcc/BLOCK00/SES000/*.mfcc | cut -f3,4 | tr . , > mfcc_2_3.txt`
    
    Hacemos la gráfica
    
  + ¿Cuál de ellas le parece que contiene más información?
    
    Nos parece que usando la parametrización LP obtenemos más información.

- Usando el programa <code>pearson</code>, obtenga los coeficientes de correlación normalizada entre los
  parámetros 2 y 3 para un locutor, y rellene la tabla siguiente con los valores obtenidos.
  
  <img width="503" alt="image" src="https://user-images.githubusercontent.com/101046951/170028447-fe87631a-c824-4d5f-940d-181326454dd9.png">


  |                        | LP   | LPCC | MFCC |
  |------------------------|:----:|:----:|:----:|
  | &rho;<sub>x</sub>[2,3] |   -0,818326   |   0,181637   |   0,0810048   |
  
  + Compare los resultados de <code>pearson</code> con los obtenidos gráficamente.
  
- Según la teoría, ¿qué parámetros considera adecuados para el cálculo de los coeficientes LPCC y MFCC?

### Entrenamiento y visualización de los GMM.

Complete el código necesario para entrenar modelos GMM.

- Inserte una gráfica que muestre la función de densidad de probabilidad modelada por el GMM de un locutor
  para sus dos primeros coeficientes de MFCC.
  
- Inserte una gráfica que permita comparar los modelos y poblaciones de dos locutores distintos (la gŕafica
  de la página 20 del enunciado puede servirle de referencia del resultado deseado). Analice la capacidad
  del modelado GMM para diferenciar las señales de uno y otro.

### Reconocimiento del locutor.

Complete el código necesario para realizar reconociminto del locutor y optimice sus parámetros.

- Inserte una tabla con la tasa de error obtenida en el reconocimiento de los locutores de la base de datos
  SPEECON usando su mejor sistema de reconocimiento para los parámetros LP, LPCC y MFCC.

### Verificación del locutor.

Complete el código necesario para realizar verificación del locutor y optimice sus parámetros.

- Inserte una tabla con el *score* obtenido con su mejor sistema de verificación del locutor en la tarea
  de verificación de SPEECON. La tabla debe incluir el umbral óptimo, el número de falsas alarmas y de
  pérdidas, y el score obtenido usando la parametrización que mejor resultado le hubiera dado en la tarea
  de reconocimiento.
 
### Test final

- Adjunte, en el repositorio de la práctica, los ficheros `class_test.log` y `verif_test.log` 
  correspondientes a la evaluación *ciega* final.

### Trabajo de ampliación.

- Recuerde enviar a Atenea un fichero en formato zip o tgz con la memoria (en formato PDF) con el trabajo 
  realizado como ampliación, así como los ficheros `class_ampl.log` y/o `verif_ampl.log`, obtenidos como 
  resultado del mismo.
