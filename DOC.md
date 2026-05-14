# Documento de Proceso: Phishing con Embeddings Granite + MLP

Este documento explica, paso a paso y en lenguaje simple, qué hace el notebook `phishing_granite_mlp.ipynb` y por qué cada paso existe. La idea es que alguien que no tenga mucha experiencia en inteligencia artificial o machine learning pueda entender el flujo completo sin tener que conocer teoría avanzada antes.

## 1. Problema que se quiere resolver

El objetivo es construir un sistema que mire un correo electrónico y decida si parece ser:

- `0`: correo normal, no phishing
- `1`: correo de phishing

Phishing significa que un mensaje intenta engañar a la persona para que haga algo peligroso, por ejemplo:

- entrar a un enlace falso,
- escribir contraseñas en una página fraudulenta,
- descargar un archivo malicioso,
- responder con información personal o bancaria.

En vez de tratar de escribir reglas manuales para detectar estos mensajes, el notebook usa datos reales de correos y aprende patrones a partir de ellos.

## 2. Qué datos se usan

El proyecto trabaja con el archivo:

- `dataset/phishing_email.csv`

Ese archivo ya viene preparado y unificado. Contiene dos columnas principales:

- `text_combined`: el texto del correo
- `label`: la clase del correo, donde `0` significa no phishing y `1` significa phishing

Dentro de la carpeta `dataset/` también hay otros CSV que representan fuentes parciales del mismo conjunto:

- `CEAS_08.csv`
- `Enron.csv`
- `Ling.csv`
- `Nazario.csv`
- `Nigerian_Fraud.csv`
- `SpamAssasin.csv`

El notebook usa `phishing_email.csv` como fuente principal porque ya reúne esas piezas en una sola tabla. Eso hace el flujo más simple y evita tener que combinar archivos cada vez que se abre el notebook.

## 3. Qué hace la primera parte del notebook

La primera parte del notebook prepara el terreno. Antes de entrenar cualquier modelo, primero se verifica qué hay dentro del dataset y se limpia lo necesario.

### 3.1 Cargar el archivo

El notebook abre `dataset/phishing_email.csv` y revisa:

- cuántas filas tiene,
- qué columnas contiene,
- cuántos ejemplos hay de cada clase,
- si hay valores vacíos,
- qué tan largos son los textos.

Esto es importante porque no conviene entrenar “a ciegas”. Antes de modelar, hay que saber si los datos están completos, balanceados y en qué estado vienen.

### 3.2 Entender la estructura

Cada fila del archivo representa un correo electrónico.

Un ejemplo simple sería:

- texto del correo
- etiqueta: `0` o `1`

En machine learning, a ese conjunto de ejemplos se le llama `dataset`. Cada fila es una observación que el sistema puede usar para aprender.

## 4. Limpieza del texto

Los correos vienen con detalles que no ayudan al modelo y que conviene normalizar. El notebook hace una limpieza conservadora, es decir, no destruye información útil, solo ordena el texto.

### 4.1 Qué se limpia

Se hace lo siguiente:

- si el texto está vacío o es `NaN`, se trata como cadena vacía,
- se eliminan espacios repetidos,
- se quitan espacios al inicio y al final.

En términos simples, esto convierte textos “desordenados” en textos más consistentes.

### 4.2 Qué no se borra

El notebook no borra los correos duplicados por defecto.

Esto es una decisión importante. A veces un mismo correo aparece más de una vez en el dataset. Eliminar duplicados podría parecer correcto, pero también puede quitar ejemplos reales que forman parte del problema. En cambio, aquí se conserva el contenido y se controla el riesgo de fuga de información con otra técnica: el `text_hash`.

## 5. Qué es `text_hash` y por qué se usa

Después de limpiar cada texto, el notebook genera un hash SHA-256 para ese correo.

Un hash es una especie de huella digital del texto. Si dos correos son idénticos, su hash también será idéntico. Si cambian, aunque sea un poco, el hash cambia por completo.

Eso sirve para dos cosas:

1. Detectar textos repetidos.
2. Evitar que el mismo correo termine en train, validation y test al mismo tiempo.

Esto último es muy importante. Si el mismo texto aparece en entrenamiento y evaluación, el modelo puede parecer mejor de lo que realmente es. Sería una ilusión de rendimiento.

## 6. Por qué se separa el dataset en train, validation y test

En machine learning no se entrena y evalúa sobre los mismos datos.

El conjunto se divide en tres partes:

- `train`
- `val`
- `test`

### 6.1 Train

Es la parte que el modelo ve durante el entrenamiento. Aquí aprende patrones.

### 6.2 Validation

Es una parte separada que se usa para revisar cómo va el aprendizaje mientras el modelo se entrena. Ayuda a decidir si el modelo mejora o si empieza a sobreajustarse.

### 6.3 Test

Es la parte final. Se guarda para la evaluación definitiva. El modelo no debe usarla durante el entrenamiento.

## 7. Cómo se hace el split sin fuga

El notebook no parte las filas al azar sin más. Usa `text_hash` como grupo.

Eso significa que:

- si dos filas tienen el mismo correo, van juntas,
- no se permiten hashes iguales repartidos entre splits distintos.

Así se protege la evaluación. El objetivo no es memorizar correos repetidos, sino aprender a reconocer señales generales de phishing.

Además, el split intenta mantener una distribución parecida de clases en cada partición. Esto importa porque si una parte tuviera casi todos los phishing y otra casi todos los normales, la evaluación sería poco representativa.

## 8. Qué se guarda después de preparar los datos

Una vez terminada la limpieza y separación, el notebook crea un archivo de metadatos:

- `artifacts/phishing_clean_metadata.parquet`

Ese archivo no contiene los embeddings todavía. Contiene información de trazabilidad, como:

- `row_id`
- `text`
- `label`
- `text_hash`
- `split`

Esto permite reconstruir qué pasó con cada correo sin tener que rehacer toda la preparación.

## 9. Qué viene después de la preparación

Una vez listo el dataset, el notebook pasa a la parte de embeddings.

Aquí ya no se trabaja con texto “en crudo”. Se transforma cada correo en un vector numérico que resume su contenido.

## 10. Qué es un embedding

Un embedding es una representación numérica de un texto.

En vez de guardar el correo como palabras, el modelo lo convierte en una lista de números. Esos números capturan información semántica, es decir, el significado general del texto.

La ventaja es que un clasificador posterior, como un MLP, puede trabajar mejor con esos vectores que con texto bruto.

## 11. Modelo usado para embeddings

El notebook usa:

- `ibm-granite/granite-embedding-97m-multilingual-r2`

Este modelo toma texto y lo convierte en un embedding de dimensión fija.

La lógica usada en el notebook es:

1. cargar el tokenizer,
2. cargar el modelo,
3. tokenizar el texto,
4. pasar el texto por el modelo,
5. tomar el vector del token inicial (`CLS pooling`),
6. normalizar el vector.

### 11.1 Tokenizer

El tokenizer parte el texto en unidades que el modelo entiende.

### 11.2 CLS pooling

El modelo devuelve representaciones para todos los tokens. El notebook toma la primera representación, que funciona como un resumen del texto.

### 11.3 Normalización

Luego se normaliza el vector. Eso hace que todos los embeddings tengan una escala comparable.

## 12. Por qué guardar embeddings en disco

Generar embeddings con un modelo grande toma tiempo y recursos. Si cada vez que se entrena el clasificador hubiera que recalcularlos, el proceso sería lento y caro.

Por eso el notebook hace dos fases:

1. primero extrae y guarda embeddings,
2. después entrena el MLP usando esos embeddings ya guardados.

Esto permite reentrenar el clasificador varias veces sin repetir la parte pesada.

## 13. Qué es el MLP

MLP significa `Multi-Layer Perceptron`.

En términos simples, es una red neuronal pequeña y directa que toma el embedding y decide si el correo es phishing o no.

No recibe texto. Recibe números.

En este proyecto el MLP cumple el rol de clasificador final.

## 14. Cómo aprende el MLP

El MLP ve vectores de embeddings y etiquetas correctas.

Durante el entrenamiento:

- compara su predicción con la verdad,
- calcula un error,
- ajusta sus pesos internos para reducir ese error.

Ese proceso se repite muchas veces, por lotes.

## 15. Qué hace el entrenamiento offline

El término “offline” aquí significa que el clasificador se entrena usando los tensores ya guardados, sin volver a pasar por Granite.

Eso hace el entrenamiento más rápido y más simple.

En esta fase se cargan:

- `artifacts/granite_embeddings_full.pt`
- `artifacts/phishing_clean_metadata.parquet`

Con eso se reconstruyen las particiones `train`, `val` y `test`.

## 16. Cómo se evalúa el modelo

Al final, el notebook prueba el modelo con el conjunto `test`.

Se calculan métricas como:

- accuracy
- precision
- recall
- F1
- ROC AUC
- matriz de confusión

### 16.1 Accuracy

Mide cuántas predicciones fueron correctas en total.

### 16.2 Precision

De los correos que el modelo marcó como phishing, cuántos realmente lo eran.

### 16.3 Recall

De todos los correos phishing reales, cuántos logró encontrar.

### 16.4 F1

Combina precision y recall en una sola métrica.

### 16.5 Matriz de confusión

Muestra cuántos casos fueron:

- verdaderos positivos,
- verdaderos negativos,
- falsos positivos,
- falsos negativos.

Eso ayuda a ver en qué falla el modelo.

## 17. Qué se guarda al final

Cuando el entrenamiento termina, el notebook guarda el mejor modelo en:

- `artifacts/mlp_model.pt`

También puede guardar métricas adicionales, como:

- `artifacts/test_metrics.json`

Así se puede revisar el resultado sin volver a ejecutar todo.

## 18. Cómo clasificar un correo nuevo

El notebook incluye una función de inferencia para texto nuevo.

El proceso es:

1. recibir un correo,
2. limpiarlo igual que los datos de entrenamiento,
3. convertirlo en embedding con Granite,
4. pasar el embedding por el MLP,
5. obtener una probabilidad para cada clase.

La salida final dice si el modelo cree que el correo es phishing o no, junto con una probabilidad aproximada.

## 19. Resumen del flujo completo

De principio a fin, el notebook hace esto:

1. carga el dataset unificado,
2. inspecciona su estructura,
3. limpia texto vacío y normaliza espacios,
4. crea `row_id` y `text_hash`,
5. separa train, validation y test sin fuga entre duplicados,
6. guarda metadatos preparados,
7. convierte cada correo en embeddings con Granite,
8. guarda los embeddings en disco,
9. entrena un MLP sobre esos embeddings,
10. evalúa el resultado,
11. guarda el modelo entrenado,
12. deja una función lista para inferencia.

## 20. Idea principal para recordar

Si hubiera que resumir el proyecto en una sola frase, sería esta:

primero se transforma el texto en números útiles con un modelo grande, y luego se usa una red neuronal pequeña para decidir si el correo es phishing.

Esa separación hace que el proceso sea más claro, más reutilizable y más fácil de mantener.
