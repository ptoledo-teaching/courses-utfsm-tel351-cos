# COS - Modelos de costo

Este laboratorio utiliza mediciones de una aplicación que integra Amazon Polly para dimensionar su consumo y construir un presupuesto mediante AWS Pricing Calculator. Una plantilla de CloudFormation proporciona la infraestructura completa. El trabajo consiste en ejecutar pruebas controladas, interpretar sus resultados y fundamentar una configuración de operación a partir de su costo y de la calidad del audio obtenido.

La actividad se realiza desde AWS Management Console, con la interfaz en inglés. No requiere desarrollar funciones, instalar herramientas ni escribir scripts. La limpieza es obligatoria y se realiza después del laboratorio.

La plantilla [infrastructure.yaml](infrastructure.yaml) contiene las funciones y los cinco casos de prueba. Los eventos JSON para iniciar las comparaciones y consultar sus resultados están incluidos en este README. Solo se carga la plantilla en CloudFormation; los eventos se utilizan posteriormente en la consola de Lambda.

## Resultados esperados

Al completar el laboratorio, será posible:

1. Identificar las dimensiones de cobro de una aplicación que utiliza varios servicios gestionados
2. Distinguir caracteres procesados, solicitudes, tiempo de ejecución, almacenamiento y transferencia de datos
3. Diseñar comparaciones que modifiquen una variable a la vez y conservar evidencia de las mediciones
4. Relacionar el consumo observado con los parámetros utilizados por AWS Pricing Calculator
5. Estimar un costo mensual y un costo medio por reporte para distintos niveles de demanda
6. Comparar configuraciones considerando costo, tiempo de respuesta y calidad del audio
7. Reconocer los supuestos y las limitaciones de un presupuesto construido a partir de una muestra pequeña

## Contexto

La HoloNet permite transmitir comunicaciones holográficas y datos entre estaciones distribuidas en distintos sistemas. Sus equipos de operaciones reciben reportes sobre interrupciones de enlaces, degradación de señal y fallas de equipamiento. Algunos reportes describen brevemente el incidente; otros incluyen antecedentes adicionales sobre el mismo problema.

El personal de terreno necesita escuchar estos reportes mientras realiza inspecciones y no puede mantener la vista en una pantalla. Para apoyar ese trabajo, las estaciones utilizan un servicio de síntesis de voz basado en inteligencia artificial, que convierte los textos en audio mediante Amazon Polly. El servicio conserva los archivos para que puedan escucharse nuevamente. No resume ni modifica el reporte: el audio debe transmitir el texto proporcionado.

Antes de extender el servicio a todas las estaciones, se necesita estimar cuánto costará mantenerlo en operación. Además de la cantidad de reportes, influyen su extensión, el motor de síntesis, el formato de audio, las reproducciones y el tiempo durante el cual se conservan los archivos. La decisión debe considerar si una diferencia de calidad aporta valor suficiente para justificar su costo.

La infraestructura incluye una función generadora que conserva los reportes en S3 y envía sus identificadores a una cola SQS. Una segunda función recibe los mensajes, obtiene cada reporte, solicita la síntesis a Polly y conserva el audio y sus mediciones en S3. CloudWatch almacena los registros de ejecución. El generador también permite consultar el avance y reunir las mediciones de cada prueba.

```text
Prueba desde la consola de Lambda
                |
        Función generadora
                |
             Cola SQS
                |
       Función de síntesis ---- Polly
                |
       Audio y resultados en S3

CloudWatch: registros y mediciones de ejecución
```

## Interacción y comprobación

Las pruebas se inician desde **Test** en la función generadora. Cada ejecución envía un conjunto finito de reportes y devuelve un `runId`, que identifica esa prueba. La respuesta del generador confirma el envío, no el término de la síntesis. Los resultados se consultan posteriormente utilizando ese mismo identificador.

Los cinco casos contienen incidentes preparados para el laboratorio. Sus versiones breve y extensa mantienen los mismos hechos principales. Las comparaciones entre motores y formatos utilizan los mismos textos, no reportes elegidos al azar en cada ejecución.

El generador limita la cantidad de reportes y no mantiene un envío periódico. Consultar un resultado no vuelve a llamar a Polly. Ejecutar nuevamente la acción de generación sí crea otra prueba y produce consumo adicional. No aumentar el volumen para simular un mes completo: la estimación mensual se obtiene mediante una proyección.

Los registros entregan consumos observados, no un presupuesto resuelto. Las tarifas, las conversiones de unidades y los supuestos deben incorporarse en la estimación. Conservar los valores realmente obtenidos y no esperar duraciones idénticas entre ejecuciones.

## Actividad

### 1. Desplegar y reconocer el sistema

1. Ingresar a AWS Management Console con su identidad administrativa y seleccionar **US East (N. Virginia)**, `us-east-1`
2. Descargar `infrastructure.yaml` desde este repositorio mediante **Download raw file**
3. Abrir **CloudFormation → Stacks → Create stack → With new resources (standard)**
4. Seleccionar **Choose an existing template → Upload a template file**, cargar la plantilla y continuar
5. Utilizar `tel351-cos` como **Stack name**. La plantilla no requiere parámetros adicionales
6. Revisar los recursos IAM que creará la plantilla, marcar la confirmación correspondiente y seleccionar **Submit**
7. Esperar hasta que el stack alcance `CREATE_COMPLETE` y abrir **Outputs**

Se utilizará una única región para los recursos del laboratorio y para sus estimaciones. Las pruebas emplean la voz **Lupe**, en español, con motores **Standard** y **Neural**. Se mantiene una frecuencia de muestreo de **16.000 Hz** tanto en MP3 como en PCM. La función de síntesis utiliza arquitectura ARM64 y 512 MB de memoria. Conservar estas características al construir el presupuesto.

MP3 es un formato comprimido. PCM conserva las muestras sin esa compresión; la función agrega un encabezado WAV al resultado de Polly para facilitar su reproducción y lo guarda con extensión `.wav`. No se requiere convertir los archivos ni instalar un editor de audio.

Reconocer los siguientes outputs:

| Output | Uso |
| --- | --- |
| `GeneratorFunctionName` | Iniciar las pruebas y consultar sus resultados |
| `SynthesizerFunctionName` | Reconocer la configuración de la función que utiliza Polly |
| `QueueUrl` | Identificar la cola que recibe los reportes |
| `DeadLetterQueueUrl` | Identificar entregas que no pudieron completar su procesamiento por una falla de infraestructura |
| `ResultsBucketName` | Consultar los archivos y descargar las mediciones |
| `SynthesizerLogGroupName` | Revisar los registros de procesamiento |
| `VoiceId` y `SampleRate` | Confirmar la voz y la frecuencia de muestreo |
| `MeasurementsQueryName` | Localizar la consulta preparada para revisar las ejecuciones de Lambda |

Abrir **Resources** y relacionar los recursos del stack con el flujo descrito en el contexto. Identificar qué componentes se utilizan para producir un audio y cuáles sirven únicamente para generar y consultar las pruebas.

Antes de ejecutar, formular una predicción: ¿qué dimensiones de consumo deberían cambiar al ampliar el texto, cambiar el motor o cambiar el formato? Conservar esa predicción para contrastarla con las mediciones.

### 2. Ejecutar pruebas controladas

#### 2.1 Iniciar la prueba base

1. Abrir la función indicada en `GeneratorFunctionName` y seleccionar **Test**
2. Crear un evento privado con el nombre `cos-generate`
3. Reemplazar el contenido del evento por el siguiente JSON

```json
{
  "action": "generate",
  "scenario": "brief",
  "reports": 5,
  "engine": "standard",
  "outputFormat": "mp3"
}
```

4. Guardar el evento y seleccionar **Test** una sola vez
5. Revisar **Execution result**, comprobar la cantidad de reportes aceptados y conservar el `runId`

`scenario` selecciona la versión de los reportes; `engine`, el motor de síntesis; y `outputFormat`, el formato solicitado a Polly. La voz y la frecuencia de muestreo permanecen fijas.

#### 2.2 Consultar el avance

1. Crear otro evento privado, denominado `cos-summary`
2. Incorporar el siguiente contenido y reemplazar el identificador por el obtenido en la prueba

```json
{
  "action": "summary",
  "runId": "identificador-obtenido-al-generar"
}
```

3. Ejecutar el evento y distinguir `pending`, `completed`, `failed` e `incomplete`. Este último indica que no existe un resultado definitivo después del plazo de observación; no equivale a una ejecución sin consumo
4. Si quedan reportes pendientes, esperar brevemente y repetir únicamente la consulta
5. Cuando termine el procesamiento, abrir el bucket indicado en `ResultsBucketName` y localizar el prefijo informado por el resumen
6. Revisar un reporte original en `inputs/` y sus mediciones en `results/`
7. Descargar el archivo correspondiente desde `audio/`, reproducirlo y comprobar que su contenido corresponda al reporte

Los tres directorios se encuentran dentro del prefijo de la prueba. El campo `audioKey` del resultado identifica el archivo de audio. Reproducir un archivo ya descargado no invoca el servicio; descargarlo nuevamente desde S3 realiza otra operación de lectura.

Las fallas de Polly quedan registradas como resultados fallidos y no provocan llamadas automáticas adicionales de síntesis. Las entregas duplicadas no repiten una síntesis que ya tiene resultado. Si una ejecución se interrumpe y no permite conocer el resultado, la herramienta conserva esa incertidumbre. La DLQ almacena entregas que no pudieron resolverse por fallas de infraestructura y no tiene un consumidor.

Si aparece una falla, revisar `errors` en el resumen y utilizar el `runId` y el identificador del reporte para localizar sus registros en CloudWatch. No interpretar una prueba sin resultados como una configuración de costo cero ni repetir envíos sin identificar primero la causa. Repetir `summary` no corrige una síntesis fallida.

#### 2.3 Comparar las configuraciones

Editar el evento `cos-generate` para realizar las pruebas de la tabla. Utilizar cinco reportes por prueba y conservar todos los identificadores. La primera fila corresponde a la prueba ya realizada; no volver a ejecutarla.

| Prueba | `scenario` | `engine` | `outputFormat` | Comparación |
| --- | --- | --- | --- | --- |
| Estándar breve | `brief` | `standard` | `mp3` | Configuración base |
| Estándar extensa | `extended` | `standard` | `mp3` | Efecto de ampliar el texto |
| Neural breve | `brief` | `neural` | `mp3` | Efecto de cambiar el motor |
| Neural extensa | `extended` | `neural` | `mp3` | Comparación de motores con texto extenso |
| Estándar PCM | `brief` | `standard` | `pcm` | Efecto de cambiar el formato |

Iniciar la siguiente prueba después de completar la anterior. Mantener sin cambios la memoria de Lambda, la concurrencia y las demás configuraciones. Se realizarán 25 síntesis en total; las consultas de resultados no forman parte de esa cantidad.

Para un mismo reporte, comparar los audios estándar y neural. Revisar la claridad de los identificadores de equipos, las cifras, las pausas y la naturalidad de la voz. Comparar también MP3 y WAV manteniendo el mismo motor. Conservar al menos una observación que fundamente una diferencia de utilidad o la ausencia de una diferencia relevante. No suponer que la opción de mayor precio será necesariamente la más apropiada.

### 3. Interpretar las mediciones

1. Consultar el resumen de cada prueba y descargar desde S3 sus mediciones
2. Importar los CSV en una planilla utilizando codificación UTF-8, coma como separador de columnas y punto como separador decimal. Comprobar que las duraciones y los tamaños se reconozcan como números y mantener separadas las cinco configuraciones
3. Abrir **CloudWatch → Logs Insights**, seleccionar el log group de síntesis y un intervalo que abarque las pruebas
4. Abrir la consulta guardada indicada en `MeasurementsQueryName` y ejecutarla
5. Relacionar las invocaciones con los reportes mediante sus identificadores y revisar duración facturada y memoria asignada

El resumen publica los nombres de tres archivos bajo el prefijo de la prueba:

| Archivo | Contenido |
| --- | --- |
| `measurements.csv` | Una fila por reporte, con resultado, caracteres, latencia, tamaños de objetos e identificador de invocación |
| `invocations.csv` | Duración facturada, memoria y GB-segundo de cada invocación identificada en los logs |
| `summary.json` | Conteos, estadísticas, disponibilidad de mediciones y alcance del registro de operaciones |

Los registros de CloudWatch pueden aparecer después de que termine la síntesis. Si `measurementStatus` es `PARTIAL`, revisar los campos disponibles y repetir `summary` cuando falten registros. No volver a generar la prueba para esperar los logs. Los CSV se actualizan al consultar y las celdas vacías representan datos desconocidos. `READY` indica que están disponibles las mediciones previstas, no que la calidad de la voz sea satisfactoria.

Conservar, como mínimo, esta información para cada configuración:

| Medición | Uso en el análisis |
| --- | --- |
| Reportes completados, fallidos y llamadas de síntesis | Distinguir trabajo útil de operaciones realizadas |
| `requestCharacters` | Caracteres facturables informados por Polly |
| `pollyLatencyMs` | Tiempo observado desde la solicitud hasta la recepción completa del audio |
| Duración facturada y memoria de Lambda | Estimar GB-segundo de ejecución |
| `audioObjectBytes` y tamaños de originales, controles y resultados | Estimar almacenamiento y tamaño de las descargas |
| Volumen observado de registros | Establecer un supuesto de ingestión y conservación de logs |

Polly cobra por caracteres procesados, con tarifas distintas según el motor. No cobra la síntesis por segundos de audio ni por tamaño del archivo. `requestCharacters` proviene de la respuesta del servicio, mientras que `textCharacters` es el conteo local del texto. Usar el valor informado por Polly para la estimación. La [documentación de SynthesizeSpeech](https://docs.aws.amazon.com/polly/latest/APIReference/API_SynthesizeSpeech.html) describe la respuesta.

`audioBytes` corresponde al contenido recibido desde Polly y `audioObjectBytes`, al archivo conservado en S3. En PCM, el segundo incluye el encabezado WAV. `audioDurationSeconds` se calcula a partir de las muestras PCM y queda vacío para MP3; no es un dato necesario para presupuestar Polly.

La latencia de Polly y la duración facturada de Lambda son mediciones diferentes. La función también recibe el mensaje, obtiene el texto y conserva el resultado. Identificar qué ocurre con su duración mientras espera el audio.

`billedDurationMs` se obtiene de la línea `REPORT` publicada por Lambda; `processingMs` mide una parte del trabajo dentro del código y no la reemplaza. `observedLogBytes` suma los bytes UTF-8 de los mensajes vinculados a las invocaciones. No incluye toda la información auxiliar ni la sobrecarga de ingestión de CloudWatch y no representa un volumen exacto de facturación. Registrar el margen utilizado para presupuestar esa diferencia.

Calcular promedios por reporte y observar el rango de los resultados. Revisar si una primera ejecución o un reintento afecta la comparación. No descartar silenciosamente los casos fallidos. Si una llamada no entrega información de consumo, marcar ese dato como desconocido, no como cero.

Contrastar las predicciones iniciales con los resultados. Explicar qué diferencias se observaron y cuáles no pueden atribuirse con certeza a la configuración debido al tamaño de la muestra.

### 4. Construir el presupuesto

#### 4.1 Definir la demanda y los supuestos

Utilizar los siguientes escenarios de operación:

| Supuesto | Operación habitual | Crecimiento |
| --- | ---: | ---: |
| Reportes nuevos por mes | 10.000 | 100.000 |
| Reportes breves | 80% | 80% |
| Reportes extensos | 20% | 20% |
| Descargas completas del audio por reporte | 3 | 3 |
| Conservación de originales, audios y resultados | 30 días | 30 días |
| Conservación de logs | 7 días | 7 días |

Cada reporte se sintetiza una sola vez. Las tres descargas reutilizan ese archivo y corresponden a transferencias desde S3 hacia Internet. No se utiliza CloudFront. Las reproducciones locales posteriores no representan nuevas descargas.

Considerar un mes de 30 días, llegadas uniformes y un sistema que ya se encuentra operando de manera estable. El volumen almacenado debe corresponder a esa condición, no a un bucket inicialmente vacío que recién comienza a acumular archivos. Registrar las conversiones utilizadas para pasar de bytes a la unidad solicitada por la calculadora.

Presupuestar uso bajo demanda, sin compromisos de capacidad ni descuentos promocionales. Separar el costo del servicio de los créditos de la cuenta. Si la calculadora aplica una franquicia o beneficio, dejarlo identificado para que las comparaciones utilicen el mismo criterio.

#### 4.2 Convertir las mediciones en consumo mensual

Construir primero una tabla de consumos, sin tarifas. Para cada motor en MP3, combinar los promedios de sus pruebas breve y extensa con la proporción indicada. Por ejemplo:

```text
caracteres por reporte = 0,8 × promedio breve + 0,2 × promedio extenso
caracteres mensuales = reportes nuevos mensuales × caracteres por reporte
```

Aplicar el mismo criterio a la duración y los tamaños de archivo. No usar el promedio simple de las cinco pruebas: tienen configuraciones y proporciones diferentes. La prueba PCM permite comparar formatos para los reportes breves; no se dispone de una medición PCM de los reportes extensos.

Identificar las dimensiones necesarias para cada servicio:

| Servicio | Consumo que debe representarse |
| --- | --- |
| Polly | Caracteres procesados por motor |
| Lambda | Invocaciones, memoria asignada y duración facturada de la función de síntesis |
| SQS | Operaciones de envío, recepción y eliminación, considerando tamaño de mensajes y agrupación |
| S3 | Volumen promedio almacenado, escrituras, lecturas y transferencia de audio hacia Internet |
| CloudWatch | Ingestión y conservación de registros |

Las solicitudes de SQS no equivalen directamente al número de reportes: también intervienen la recepción de mensajes, su eliminación, las consultas sin mensajes y las unidades de tamaño facturables. Utilizar la configuración y el registro de operaciones que entregue la herramienta para establecer una estimación explícita; no presentar ese cálculo como un conteo exacto de la factura.

La ruta normal de un reporte conserva cuatro objetos: original, control de procesamiento, audio y resultado. También lee el original para sintetizarlo. Las descargas del audio son lecturas adicionales. Distinguir esas operaciones de los manifiestos, consultas y exportaciones utilizados para reunir las mediciones del laboratorio.

Separar las operaciones del servicio de las funciones de generación y consulta. El presupuesto representa reportes recibidos por el servicio, no la repetición mensual de las pruebas ni la descarga reiterada de sus resúmenes. La conservación de 30 días es un supuesto de operación y no reemplaza la limpieza del despliegue de prueba.

#### 4.3 Utilizar AWS Pricing Calculator

1. Abrir [AWS Pricing Calculator](https://calculator.aws/#/) y crear una estimación
2. Agregar Amazon Polly y seleccionar el motor estándar. Incorporar los caracteres mensuales calculados, revisando las unidades solicitadas
3. Agregar Lambda, SQS, S3 y CloudWatch en `us-east-1` con los consumos y configuraciones determinados previamente
4. Incorporar las descargas de audio en las solicitudes GET y en la transferencia de salida de S3. Identificar los beneficios de transferencia gratuita que aplique la calculadora
5. Revisar el desglose y registrar el costo mensual de cada servicio y el total
6. Preparar una segunda estimación equivalente para el motor neural en MP3, utilizando sus propias mediciones
7. Comparar ambos resultados y recalcular la alternativa seleccionada para el escenario de crecimiento
8. Guardar o exportar las estimaciones y conservar la fecha, región, motores, supuestos y valores utilizados

Calcular el costo medio dividiendo el costo mensual por la cantidad mensual de reportes nuevos. Distinguir este valor del costo de la síntesis individual, que no incluye los demás componentes del sistema.

Comparar, además, dos políticas para un reporte breve: conservar el audio y descargarlo tres veces, o sintetizarlo nuevamente antes de cada descarga. Calcular el efecto sobre Polly y explicar qué otras operaciones cambiarían. No es necesario generar nuevas pruebas ni presentar el costo total de la segunda política como si se hubiese medido su implementación.

AWS Pricing Calculator entrega una estimación basada en los parámetros ingresados, no una garantía del monto de la factura. Revisar las unidades y los [supuestos de la calculadora](https://aws.amazon.com/calculator/calculator-assumptions/), especialmente al comparar períodos, regiones y beneficios gratuitos.

### 5. Fundamentar una recomendación

Preparar una recomendación breve que responda:

1. ¿Qué motor conviene utilizar para la mezcla de reportes propuesta y qué evidencia sustenta esa decisión?
2. ¿Qué componente explica la mayor parte del presupuesto? ¿Coincide con la predicción inicial?
3. ¿Qué diferencia de tamaño y calidad se observó entre MP3 y PCM? ¿Qué partes del presupuesto cambiarían al usar uno u otro?
4. ¿Cómo influyen la conservación y la reutilización del audio en el consumo? Distinguir síntesis, descarga y reproducción local
5. ¿Cómo cambian el costo total y el costo medio al aumentar la demanda? Identificar los supuestos que se mantuvieron y las condiciones que podrían dejar de cumplirse
6. ¿Qué medición adicional sería necesaria antes de comprometer un presupuesto de operación? Considerar la variación de los textos, los errores, los reintentos y la capacidad para atender la demanda

Conservar la planilla de mediciones, las estimaciones y la recomendación como resultado de la actividad. La comparación debe permitir reconstruir el origen de los valores utilizados; una cifra final sin supuestos ni mediciones no permite evaluar la decisión.

## Limpieza

Realizar la limpieza después del laboratorio. Conservar primero las mediciones y estimaciones necesarias para revisar los resultados.

1. No iniciar nuevas pruebas y consultar que las ya iniciadas hayan terminado
2. Revisar las colas del stack. Si quedan mensajes por procesar, deshabilitar el event source mapping de la función y esperar que terminen las invocaciones en curso antes de vaciar el bucket. Deshabilitar el origen no interrumpe las ejecuciones que ya comenzaron
3. Abrir únicamente el bucket indicado en `ResultsBucketName`, descargar los resultados que se conservarán y utilizar **Empty** para eliminar sus objetos. Confirmar el nombre del bucket antes de aceptar
4. Regresar a **CloudFormation**, seleccionar exclusivamente el stack `tel351-cos` y utilizar **Delete**
5. Seguir **Events** hasta confirmar la eliminación. Si falla, identificar el recurso y resolver la causa antes de reintentar
6. Comprobar que no permanezcan las funciones, colas, roles, bucket ni log groups creados por este stack
7. Eliminar los eventos privados de prueba que permanezcan disponibles en la consola

No eliminar recursos de otros laboratorios. Las estimaciones guardadas en AWS Pricing Calculator no despliegan recursos y pueden conservarse.
