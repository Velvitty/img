<p align="center"><a href="README.md">한국어</a> · <a href="README.en.md">English</a> · <a href="README.ja.md">日本語</a> · <a href="README.es.md">Español</a></p>

# Compresor de imágenes en 400 caracteres

Una herramienta que convierte una foto en un texto lo bastante corto para caber en **un solo comentario (menos de 400 caracteres)**. Está pensada para sitios como los comentarios de DCInside, donde los mensajes son cortos y no se pueden publicar imágenes. Quien lo recibe pega el texto en esta página, o abre el enlace, para ver la foto.

- Todo está en un único archivo `index.html` y se ejecuta en el navegador. Sin servidor y sin subidas.
- El resultado es una vista previa borrosa. 400 caracteres solo contienen entre 300 y 650 bytes, suficiente para reconocer formas, colores y ambiente.

## Cómo se usa

1. En la pestaña **만들기 (Crear)**, arrastra una foto, haz clic para elegirla o pégala con Ctrl/⌘+V.
2. Pulsa **복사 (Copiar)** para copiar una sola línea de menos de 400 caracteres, como `https://g.yeon.at/?뗈앏괟샡…`. Pégala tal cual en tu comentario.
3. Quien lo recibe pega esa línea en la barra de direcciones del navegador y ve la foto al instante. También puede pegarla en la pestaña **보기 (Ver)**; el texto de alrededor se ignora.

Funcionan tanto `https://g.yeon.at/?código` como `https://g.yeon.at/index.html?código`.

## Ajustes

| Ajuste | Descripción |
|---|---|
| Letras del código: hangul | 13 bits por carácter; con la dirección, 381 caracteres guardan unos 619 bytes. La opción más nítida. |
| Letras del código: ASCII | Solo letras, dígitos, `-` y `_`, 6 bits por carácter. Para sitios que no aceptan hangul. |
| Dirección delante | Por defecto `https://g.yeon.at/?` (19 caracteres); el `https://` hace que se convierta en un enlace pulsable en los comentarios, que se descuentan del espacio para la foto. Déjala vacía para generar solo el código. |
| Límite de caracteres | Por defecto 400 (un comentario de DCInside). |

## Cómo funciona

1. La foto se reduce a una cuadrícula de 64 píxeles en el lado largo y se convierte a YCbCr. Con un presupuesto de caracteres amplio, la cuadrícula crece a 96–512 píxeles.
2. Se aplica una DCT 2D a toda la imagen y los coeficientes se ordenan de frecuencia baja a alta.
3. Las frecuencias altas y los canales de color (crominancia) se cuantizan de forma más gruesa.
4. Los niveles se comprimen con un codificador aritmético binario adaptativo al contexto (un range coder al estilo de LZMA).
5. Una búsqueda binaria encuentra el cuantizador más fino que cabe en el límite, y se recortan los caracteres finales mientras la decodificación no cambie.
6. El visor evalúa la serie de cosenos a la resolución de la pantalla, así que la imagen ampliada se ve suave y sin bloques.

## Pestaña de enlace JPG data

Donde los enlaces `data:` se muestran directamente como imágenes (sitios que admiten imágenes en HTML o Markdown), se puede usar un JPG normal como enlace `data:`. Con el mismo número de caracteres, el código de esta página es más nítido (una imagen de 1200×1200 en 7000 caracteres: JPG 25,6 dB, código hangul 28,0 dB). Solo la cabecera de un JPG ocupa varios cientos de caracteres, así que nunca cabe en 400.

## Notas

- Los comentarios de DCInside no muestran imágenes ni enlaces `data:`, así que quien lo recibe debe abrir esta página por su cuenta.
- El código hangul usa 8192 sílabas precompuestas a partir de `가` (U+AC00). En sitios que limitan bytes en lugar de caracteres, usa el código ASCII.

---

## Documentación detallada

### Índice

1. [Descripción general](#descripción-general)
2. [Pantallas](#pantallas)
3. [Formatos de código y enlace](#formatos-de-código-y-enlace)
4. [Calidad y número de caracteres](#calidad-y-número-de-caracteres)
5. [Privacidad y requisitos](#privacidad-y-requisitos)
6. [Abrir en otras direcciones](#abrir-en-otras-direcciones)
7. [Funcionamiento interno](#funcionamiento-interno)
8. [Limitaciones](#limitaciones)
9. [Cómo leer el PSNR](#cómo-leer-el-psnr)
10. [Solución de problemas](#solución-de-problemas)
11. [Preguntas frecuentes](#preguntas-frecuentes)
12. [Material de desarrollo](#material-de-desarrollo)
13. [Notas de diseño](#notas-de-diseño)
    - [Notas de diseño: planteamiento del problema](#notas-de-diseño-planteamiento-del-problema)
    - [Notas de diseño: direcciones y transporte](#notas-de-diseño-direcciones-y-transporte)
    - [Notas de diseño: alojamiento estático](#notas-de-diseño-alojamiento-estático)
    - [Notas de diseño: preprocesado](#notas-de-diseño-preprocesado)
    - [Notas de diseño: transformada](#notas-de-diseño-transformada)
    - [Notas de diseño: cuantización](#notas-de-diseño-cuantización)
    - [Notas de diseño: codificación entrópica](#notas-de-diseño-codificación-entrópica)
    - [Notas de diseño: formato del flujo de bits](#notas-de-diseño-formato-del-flujo-de-bits)
    - [Notas de diseño: conversión a texto](#notas-de-diseño-conversión-a-texto)
    - [Notas de diseño: control de tasa y elección de cuadrícula](#notas-de-diseño-control-de-tasa-y-elección-de-cuadrícula)
    - [Notas de diseño: decodificación y ampliación](#notas-de-diseño-decodificación-y-ampliación)
    - [Notas de diseño: diseño de la página](#notas-de-diseño-diseño-de-la-página)
    - [Notas de diseño: compilación](#notas-de-diseño-compilación)
    - [Notas de diseño: método de evaluación](#notas-de-diseño-método-de-evaluación)
    - [Notas de diseño: las matemáticas del PSNR](#notas-de-diseño-las-matemáticas-del-psnr)
14. [Referencias](#referencias)

### Descripción general

Los comentarios de DCInside admiten como máximo 400 caracteres, no permiten subir imágenes y no muestran los enlaces `data:` como imágenes. Un JPG normal convertido en enlace `data:` necesita unos 800 caracteres solo para la cabecera del archivo, así que no cabe en 400.

Esta página usa un método de compresión propio que reduce una foto a unos 300–620 bytes. Escribe el resultado como sílabas hangul o como letras y dígitos ASCII para que quepa en una línea de comentario. Quien lo recibe abre el enlace o pega el código para ver la foto.

- Dirección: https://g.yeon.at/
- Este repositorio contiene solo los archivos publicados (`index.html`, `CNAME`, README). El código fuente y el material de pruebas están en [Velvitty/img-dev](https://github.com/Velvitty/img-dev).

### Pantallas

La página tiene tres pestañas.

#### Crear (만들기)

1. Arrastra una foto, haz clic en el recuadro para elegirla o pégala con Ctrl/⌘+V. Se pueden añadir varias a la vez.
2. Cada foto genera una tarjeta: el original a la izquierda y, a la derecha, lo que verá quien la reciba.
3. Los números bajo la tarjeta significan:

   | Etiqueta | Significado |
   |---|---|
   | 원본 (Original) | Tamaño del archivo original |
   | 코드 (Código) | Caracteres que contienen la foto (sin la dirección) |
   | 전체 (Total) | Caracteres con la dirección, frente al límite. En verde si está dentro del límite. |
   | 압축 단계 (Paso) | 0–127. Cuanto más bajo, más fina la compresión. |
   | 격자 (Cuadrícula) | Tamaño con el que se guardó la foto: 64 px con 400 caracteres, mayor con más caracteres (hasta 512 px). |

4. Hay tres botones.
   - **복사 (Copiar):** copia la dirección y el código en una sola línea. Donde copiar está bloqueado, el texto queda seleccionado para pulsar Ctrl/⌘+C.
   - **보기 탭에서 확인 (Comprobar en Ver):** muestra la pantalla de quien lo recibe.
   - **삭제 (Borrar):** elimina la tarjeta.

Al cambiar un ajuste (letras del código, límite de caracteres, dirección delante) se vuelven a comprimir todas las fotos. La primera vez hay un ejemplo de atardecer marcado como `예시 (ejemplo)`.

#### Ver (보기)

Pega un código o enlace recibido y la foto aparece.

- El texto de alrededor se ignora. En los códigos hangul se usa el tramo más largo de al menos 4 caracteres; en los ASCII, el tramo más largo después del último `?` o `#`.
- Bajo la foto se muestran la longitud del código, su tipo y el paso de compresión.
- Si el código está cortado o no es un código, la página lo indica.

Al llegar por un enlace (`https://g.yeon.at/?código`) esta pestaña se abre directamente.

#### Enlace JPG data

Una herramienta para sitios con límites amplios. Vuelve a codificar la foto como JPG y genera un enlace `data:image/jpeg;base64,…`.

| Ajuste | Descripción |
|---|---|
| Longitud objetivo | Busca la calidad más alta que cabe y solo reduce la imagen cuando la calidad bajaría de 35. Tiene botones para YouTube/Reddit (9900), Instagram (2150) y GitHub (65 000), con un poco de margen para etiquetas. |
| Calidad | Codifica una sola vez con la calidad elegida (5–95). |
| Lado largo máximo | Original, 1920, 1280, 800 o 480 píxeles. |

El resultado se puede copiar como enlace, como etiqueta `<img>` o en Markdown.

### Formatos de código y enlace

| Formato | Ejemplo | Caracteres para el código | Capacidad |
|---|---|---|---|
| Hangul + dirección (por defecto) | `https://g.yeon.at/?뗈앏괟…` | 381 | unos 619 bytes |
| ASCII + dirección | `https://g.yeon.at/?TqVTweO…` | 381 | unos 285 bytes |
| Solo código hangul | `뗈앏괟…` | 400 | 650 bytes |
| Solo código ASCII | `TqVTweO…` | 400 | 300 bytes |

- **Código hangul:** 13 bits por carácter, unas 2,2 veces más que el ASCII. Usa los 8192 caracteres de `가` (U+AC00) a `쯿` (U+CBFF).
- **Código ASCII:** solo `A–Z a–z 0–9 - _`. Úsalo donde no se admita hangul o donde la longitud se cuente en bytes.
- **Por qué `https://`:** los comentarios de DCInside solo se convierten en enlaces pulsables cuando empiezan por `https://`.
- **Hangul en la dirección:** la barra de direcciones muestra hangul, pero el navegador envía cada sílaba como 9 caracteres, por ejemplo `%EB%97%88`. 381 sílabas suman unos 3,4 KB, muy por debajo de los límites de longitud de URL.
- **Forma con `#`:** `https://g.yeon.at/#código` también funciona. Cuando el código es tan largo que la dirección pasaría de unos 7 KB (códigos hangul de unos 770 caracteres o más), Copiar usa `#` en lugar de `?` automáticamente. Todo lo que va después de `?` se envía al servidor, y GitHub Pages rechaza direcciones de más de 8 KB (error 414).

### Calidad y número de caracteres

Ocho fotos comprimidas en los 381 caracteres que quedan después de `https://g.yeon.at/?`. El PSNR mide cuánto se parece el resultado al original; cuanto más alto, más parecido (consulta [Cómo leer el PSNR](#cómo-leer-el-psnr)).

| Foto | Código ASCII | Código hangul |
|---|---|---|
| Carlino en una cama | 25,8 dB | 28,0 dB |
| Montañas con niebla y un excursionista | 26,5 dB | 27,6 dB |
| Persona en un campo | 24,7 dB | 26,7 dB |
| Oso | 24,2 dB | 26,3 dB |
| Morsas | 22,6 dB | 23,9 dB |
| Carlino con manta (bosque) | 21,9 dB | 23,1 dB |
| Fiordo y acantilado | 21,1 dB | 22,4 dB |
| Cachorro negro sobre suelo de madera | 20,8 dB | 22,0 dB |
| **Media** | **23,4 dB** | **25,0 dB** |

Algunas fotos funcionan claramente mejor que otras.

- **Funcionan bien:** fotos hechas de formas grandes y cambios de color suaves, como niebla, cielo o primeros planos de caras o animales.
- **Funcionan mal:** fotos cubiertas de textura fina, como vetas de madera, hierba o roca. La textura consume los caracteres y el resultado sigue borroso.
- **Recorta antes:** dejar solo la parte que quieres mostrar la hace mucho más nítida con la misma longitud.
- **Texto:** con 400 caracteres no se puede leer texto.
- **Subir el límite:** la cuadrícula que guarda la foto crece y el resultado es más nítido. Medido a 512×512 con una imagen de 1200×1200: 400 caracteres dieron una cuadrícula de 64 px y 21,6 dB; 1367, 96 px y 24,0 dB; 4000, 192 px y 26,6 dB; y 7000, 384 px y 28,0 dB. La cuadrícula llega como máximo a 512 px, así que por encima de unos 10 000 caracteres apenas mejora.

### Privacidad y requisitos

- **Sin subidas:** las fotos nunca salen del navegador. La compresión y la decodificación las hace el propio script de la página.
- **El código es la foto:** no se guarda nada en ningún servidor, así que cualquiera con el código puede ver la misma foto. Si borras el código, la foto desaparece.
- **Conexiones externas:** solo las fuentes (Google Fonts) se cargan de fuera. Sin conexión se usan las fuentes por defecto.
- **Valores recordados:** las letras del código, la dirección delante y la última pestaña se guardan en el almacenamiento del navegador (localStorage). No se guardan fotos ni códigos.
- **Navegadores:** versiones recientes de Chrome, Edge, Safari y Firefox, también en el móvil.

### Abrir en otras direcciones

Esta página solo funciona en `g.yeon.at`.

- Si el repositorio se copia en otra dirección (por ejemplo `tu-usuario.github.io`), dejan de funcionar la creación y la visualización de fotos, y la página solo muestra un aviso y un botón para ir a g.yeon.at.
- Si la dirección llevaba un código, el botón lo pasa a g.yeon.at, así que la foto se puede ver allí.
- Abierta como archivo local o en `localhost`, la página funciona para hacer pruebas.

### Funcionamiento interno

```
foto → cuadrícula de 64–512 píxeles → YCbCr → DCT 2D de toda la imagen → cuantización → codificación aritmética → texto hangul o ASCII
```

1. **Reducir:** la foto se reduce a una cuadrícula con 64 píxeles en el lado largo. Si el presupuesto supera lo que caben 64 píxeles, se prueban por turnos cuadrículas de 96, 128, 192, 256, 384 y 512 píxeles y se queda la que más se parece al original. Se divide a la mitad varias veces para evitar bordes dentados, y las zonas transparentes se rellenan de blanco.
2. **Separar colores:** un canal de brillo (Y) y dos de diferencia de color (Cb, Cr). El ojo es más sensible al brillo, así que el color se trata de forma más gruesa.
3. **Frecuencias:** en lugar de los bloques 8×8 de JPEG, una sola DCT cubre toda la imagen. No hay bordes de bloque, y primero se guardan las frecuencias bajas (las formas grandes).
4. **Cuantización:** las frecuencias altas y los canales de color se redondean de forma más gruesa. El paso de compresión (0–127) fija el intervalo de redondeo.
5. **Codificación aritmética:** cada valor se divide en preguntas como «¿es cero?» y «¿es mayor que 1?», y se comprime con probabilidades que se adaptan a lo anterior (un range coder al estilo de LZMA).
6. **Escribir como texto:** los bytes se escriben de 13 en 13 bits como sílabas hangul, o de 6 en 6 bits como ASCII.
7. **Ajustar al límite:** una búsqueda binaria encuentra el paso de compresión más fino que cabe. Después se quitan caracteres finales mientras el código se siga decodificando igual.
8. **Decodificar:** el navegador de quien lo recibe suma las ondas coseno directamente a la resolución de la pantalla. Como no estira una imagen del tamaño de la cuadrícula, la foto se difumina con suavidad a cualquier tamaño, sin escalones de píxeles.

El comienzo de un código (62 bits, unos 5 caracteres hangul, con la cuadrícula de 64 píxeles) guarda la versión, el tamaño de la cuadrícula, la proporción, el paso de compresión, el color medio y el número de coeficientes. El formato a nivel de bits y los parámetros están en la [documentación detallada de img-dev](https://github.com/Velvitty/img-dev/blob/main/README.es.md#diseño-del-códec).

### Limitaciones

- **Calidad:** una vista previa borrosa. No se distinguen textos ni texturas finas.
- **Ondas:** aparecen ondulaciones junto a los bordes de brillo marcados, porque se recortaron frecuencias.
- **Techo:** la cuadrícula llega como máximo a 512 píxeles, así que por encima de unos 10 000 caracteres apenas mejora.
- **Tiempo:** con límites de 7000 caracteres o más, la compresión tarda unos segundos.
- **Proporción:** las fotos más alargadas que unos 3,8:1 se aplastan a esa proporción.
- **Transparencia:** las zonas transparentes quedan en blanco.
- **GIF animados:** solo se guarda el primer fotograma.
- **Versiones distintas:** si una versión posterior cambia los parámetros de compresión, los códigos antiguos pueden decodificarse de otra forma.
- **Enlaces automáticos de DCInside:** todavía no se ha comprobado si DCInside incluye en el enlace el hangul que va después de `?`. Si el enlace se corta, usa el código ASCII, o pide a quien lo recibe que pegue el código en la pestaña Ver.

### Cómo leer el PSNR

El PSNR (relación señal-ruido de pico) expresa en dB cuánto se parece el resultado al original.

`PSNR = 10 × log10(255² ÷ error cuadrático medio)`

| PSNR | Impresión |
|---|---|
| ∞ | Idéntico al original |
| 45 dB o más | Prácticamente indistinguible |
| 35–40 dB | JPG típico |
| Unos 30 dB | Se ve algo emborronado si se mira de cerca |
| Unos 25 dB | Borroso pero reconocible (media del código hangul) |
| Unos 20 dB | Solo quedan formas y colores |

- Si las imágenes son idénticas, el error es cero y el PSNR es infinito.
- Cada vez que el error se multiplica por 10, el PSNR baja exactamente 10 dB.
- Las cifras de este documento comparan fotos reducidas a 160×120. Comparar a un tamaño mayor da valores más bajos.

### Solución de problemas

| Síntoma | Solución |
|---|---|
| «이 글자로는 사진을 만들 수 없습니다» (no se puede crear una foto con este texto) | El código se cortó o se mezclaron otros caracteres. Copia de nuevo la línea entera. |
| El enlace abre la pestaña Crear sin foto | Puede que el enlace automático del comentario se cortara. Copia el código del comentario y pégalo en la pestaña Ver. Quien envía puede usar el código ASCII. |
| El botón de copiar no hace nada | El texto ya está seleccionado; pulsa Ctrl/⌘+C. |
| La foto no se parece en nada al original | El código puede venir de otra versión de la página. Ábrelo con la misma versión con la que se creó. |
| El total queda un poco por debajo del límite | Es normal. Los pasos de compresión van a saltos, así que pueden sobrar algunos caracteres. |

### Preguntas frecuentes

**¿Se puede recuperar la foto original a partir de un código?**
No. Un código solo contiene la información de una vista previa borrosa.

**¿La misma foto da siempre el mismo código?**
En el mismo navegador, sí. Cada navegador reduce las imágenes de forma un poco distinta, así que el código puede variar ligeramente. Pero un código ya creado se decodifica en la misma foto en cualquier navegador.

**¿Y en sitios que permiten más de 400 caracteres?**
Subir el límite de caracteres agranda la cuadrícula y da un resultado más nítido (hasta 512 píxeles). Usa la pestaña de enlace JPG data donde los enlaces `data:` se muestran como imágenes.

**¿Por qué el código hangul es más nítido que el ASCII?**
Porque el mismo número de caracteres guarda unas 2,2 veces más información. Un carácter ASCII es uno entre 64 valores; uno hangul, uno entre 8192.

**El código hangul parece hecho de caracteres raros.**
Incluye sílabas poco usadas como `뗈` y `쯿`. Los caracteres no tienen significado a propósito, y la mayoría de las fuentes los incluyen todos.

### Material de desarrollo

El código fuente, el script de compilación, las fotos y scripts de prueba y el registro de experimentos están en [Velvitty/img-dev](https://github.com/Velvitty/img-dev). El `index.html` de aquí es una copia de la carpeta `dist/` que genera `python3 build.py` en img-dev.

### Notas de diseño

Estas notas son las mismas que acompañan a cada parte de la documentación detallada de [Velvitty/img-dev](https://github.com/Velvitty/img-dev). Las rutas de código (`src/`, `lab/`, `build.py`) se refieren a img-dev.

#### Notas de diseño: planteamiento del problema

- **Qué:** Un formato de imagen con pérdida que cabe en una línea de comentario (400 caracteres) y una página de navegador que lo crea y lo decodifica.
- **Por qué:** Los comentarios de DCInside tienen un límite de 400 caracteres, no admiten adjuntos y no muestran los enlaces `data:` como imágenes. Para enviar una foto, hay que escribir la foto misma como texto.
- **Cómo (método y técnica):** La cadena clásica de compresión con pérdida, codificación por transformada (DCT) → cuantización → codificación aritmética binaria adaptativa al contexto → conversión a texto, rediseñada para un presupuesto extremadamente pequeño de unos cientos de bytes.
- **Enfoque matemático:** Un carácter transporta `log2(tamaño del alfabeto)` bits: `log2 64 = 6` en base64url y `log2 8192 = 13` con las 8192 sílabas hangul que se usan. Tras los 19 caracteres de la dirección, 381 caracteres × 13 bits = 4953 bits ≈ 619 bytes es todo lo que recibe la foto. Según el teorema de codificación de fuente de Shannon [7], la longitud media del código no puede bajar de la entropía del modelo, así que hacen falta un modelo que describa las fotos con poca entropía (transformada y modelo de contexto) y un codificador que se acerque a esa entropía (codificación aritmética).
- **Enfoque de ingeniería:** Debe funcionar desde un único archivo estático sin servidor, y el código debe ser todos los datos (autocontenido). Por eso no hay almacenamiento, ni subidas, ni cuentas.
- **Por qué así (alternativas):** (1) Los enlaces a un alojamiento de imágenes son cortos, pero dependen de una subida y de la vida del servidor, lo que choca con el requisito de no subir nada. (2) Un enlace JPEG `data:` gasta unos 600 bytes solo en tablas de cuantización y de Huffman, más de 800 caracteres en base64 [3][4][26]. (3) BlurHash [33] y ThumbHash [34] están pensados para marcadores de pocas decenas de caracteres con un número fijo de componentes (BlurHash admite como máximo 9 por eje), así que no pueden aprovechar 400 caracteres. Por eso se creó un códec propio cuya calidad crece con el presupuesto.
- **Valor:** Sin coste de servidor ni riesgo de privacidad, y mientras se copie el texto, se decodifica en la misma foto en cualquier sitio y para siempre.
- **Limitaciones:** El techo de información es firme, así que el resultado no pasa de una vista previa borrosa. Todavía no se ha comprobado si DCInside convierte en enlace un texto con hangul hasta el final.
- **Fuentes:** [3], [4], [7], [26], [33], [34]

#### Notas de diseño: direcciones y transporte

- **Qué:** Los enlaces de la forma `https://g.yeon.at/?código` y la regla que cambia a `#` con códigos largos.
- **Por qué:** Los comentarios de DCInside solo se convierten en enlaces pulsables cuando empiezan por `https://`. Con el código después de `?`, GitHub Pages ignora esa parte y sirve el mismo `index.html`, así que cada dirección puede abrir una foto distinta sin configurar el servidor.
- **Cómo (método y técnica):** Al cargarse, la página lee `location.search`, o `location.hash` si no hay consulta, para obtener el código. Al copiar, si la dirección codificada como URL supera 7000 bytes, `?` se cambia por `#`.
- **Enfoque matemático:** Una sílaba hangul ocupa 3 bytes en UTF-8, y en una dirección cada byte se convierte en los 3 caracteres `%XX`, así que una sílaba cuesta 9 bytes. Un código de 381 caracteres son 381 × 9 = 3429 bytes más los 19 de la dirección. El umbral de 7000 bytes equivale a `(7000 − 19) / 9 ≈ 775` sílabas.
- **Enfoque de ingeniería:** En la sintaxis URI [27], la consulta (tras `?`) se envía al servidor, mientras que el fragmento (tras `#`) se queda en el navegador. HTTP permite responder 414 a un destino demasiado largo [28]. Las peticiones a g.yeon.at devolvieron 200 con 6000 bytes y 414 con 9000 (medido el 2026-10-26), así que el corte se puso en 7000 bytes para dejar margen al resto de la línea de petición.
- **Por qué así (alternativas):** Usar siempre `#` evitaría el 414 por completo, pero `?` es la forma por defecto y basta para los enlaces cortos. Así que los códigos cortos usan `?` y los largos `#`.
- **Valor:** Los enlaces de 400 caracteres mantienen la forma con `?`, y los de miles de caracteres se abren sin errores. Lo que va después de `#` tampoco aparece en los registros del servidor.
- **Limitaciones:** El límite del 414 es una medición del comportamiento actual de GitHub Pages (y su CDN) y puede cambiar. Hasta dónde incluye caracteres no ASCII el enlazado automático de cada mensajería o foro depende del servicio.
- **Fuentes:** [27], [28], [32]

#### Notas de diseño: alojamiento estático

- **Qué:** Publicar solo `index.html` y `CNAME` en GitHub Pages con el dominio propio `g.yeon.at`.
- **Por qué:** La compresión y la decodificación ocurren en el navegador, así que el servidor solo tiene que entregar un archivo.
- **Cómo (método y técnica):** `build.py` en img-dev genera `dist/`, que se copia al repositorio img y se sube a `main`.
- **Enfoque de ingeniería:** El material de desarrollo (img-dev) y la copia publicada (img) están en repositorios separados, para que el de publicación contenga solo los archivos que necesita la página. El archivo `CNAME` fija el dominio propio [32].
- **Por qué así (alternativas):** Una función sin servidor o un servidor propio podrían aplicar la restricción de dirección o recoger estadísticas de uso en el servidor, pero añadirían coste y responsabilidad sobre los datos. El alojamiento estático es lo que mejor y más simplemente cumple los requisitos.
- **Valor:** Sin coste de funcionamiento, y GitHub Pages es el único punto de fallo.
- **Limitaciones:** Sin código de servidor, comprobaciones como la restricción de dirección solo pueden hacerse en el navegador.
- **Fuentes:** [32]

#### Notas de diseño: preprocesado

- **Qué:** Reducir la foto a la cuadrícula, componerla sobre blanco y convertirla a YCbCr.
- **Por qué:** Para ajustar el tamaño de la transformada al presupuesto, con poco cálculo, y para separar brillo y color de modo que el color pueda tratarse de forma más gruesa.
- **Cómo (método y técnica):** En un canvas, la foto se divide a la mitad hasta quedar entre 2 y 4 veces el objetivo y luego se dibuja una vez más al tamaño de la cuadrícula. La proporción se guarda en 6 bits como `a = round(16 · log2(ancho/alto))`. La conversión de color usa los coeficientes BT.601 de rango completo de JFIF (`Y = 0.299R + 0.587G + 0.114B`, etc.) [5][6].
- **Enfoque matemático:** Según el teorema de muestreo [36], reducir por un factor k pliega dentro de la imagen las frecuencias superiores a `1/(2k)` de la resolución original (aliasing). El filtro bilineal del navegador solo mira un entorno de 2×2, así que una reducción grande de una vez equivale a muestrear puntos. Reducir a la mitad paso a paso hace que cada paso se parezca a una media de 2×2, con el mismo efecto paso bajo que una pirámide de mipmaps [18]. El código de proporción tiene 16 pasos por octava, así que el error de proporción es como mucho `2^(1/32) ≈ 2,2 %`, y ±31 pasos llegan a `2^(31/16) ≈ 3,83:1`.
- **Enfoque de ingeniería:** Una cuadrícula de 64 píxeles tiene 64 × 48 = 3072 coeficientes por canal: suficientes para los pocos cientos distintos de cero que caben en un presupuesto de 400 caracteres (unos 619 bytes), con una transformada de menos de un milisegundo. El ojo es menos sensible al contraste de color que al de brillo [17], así que separar los canales de color permite tratarlos de forma distinta al cuantizar.
- **Por qué así (alternativas):** Comprimir en RGB gasta bits en información repetida en los tres canales. En lugar de submuestrear el color en el espacio al estilo 4:2:0, los canales de color usan un paso 1,7 veces mayor en la misma cuadrícula. Con una DCT de toda la imagen, los coeficientes de color de alta frecuencia acaban en cero de todos modos, así que el efecto es parecido y el código, más simple.
- **Valor:** Fotos de cualquier tamaño siguen el mismo camino, y los escalones de la reducción no se cuelan en el resultado.
- **Limitaciones:** La reducción en canvas varía un poco entre navegadores, así que la misma foto puede dar códigos algo distintos en navegadores distintos. Las fotos más alargadas que 3,8:1 se aplastan. La transparencia se pierde en blanco.
- **Fuentes:** [5], [6], [17], [18], [30], [36]

#### Notas de diseño: transformada

- **Qué:** Una única DCT-II 2D sobre toda la cuadrícula y la regla que ordena los coeficientes por frecuencia.
- **Por qué:** Los píxeles vecinos de una imagen natural están muy correlacionados, así que tras una DCT la energía se concentra en unos pocos coeficientes de baja frecuencia. El resto puede ponerse a cero con poca pérdida visible.
- **Cómo (método y técnica):** La transformada se calcula de forma separable, primero en horizontal y luego en vertical. Los coeficientes se ordenan por `r = √((u·s/W)² + (v·s/H)²)` con `s = √(W·H)`.
- **Enfoque matemático:** La normalización siguiente hace que `A[0,0]` sea el brillo medio y que cada otro coeficiente sea la amplitud de su onda coseno.

  ```math
  A[u,v] = \frac{\alpha_u \alpha_v}{WH} \sum_{x=0}^{W-1}\sum_{y=0}^{H-1} f(x,y)\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H},\quad \alpha_0=1,\ \alpha_{k>0}=2
  ```

  ```math
  f(x,y) = \sum_{u,v} A[u,v]\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H}
  ```

  Se sabe que la DCT se aproxima a la transformada óptima (KLT) para un proceso de Markov de primer orden y que compacta bien la energía [1][2]. El coste es `O(W·H·(W+H))`: unas 340 000 multiplicaciones para una cuadrícula de 64 × 48 y unos 270 millones para 512 × 512.

- **Enfoque de ingeniería:** Ordenar por r, una frecuencia normalizada a un cuadrado, pone la misma frecuencia visible en la misma posición en fotos anchas y altas. El orden se guarda por tamaño de cuadrícula, y los empates en r se resuelven con los enteros v y u, así que navegadores y Node.js obtienen el mismo orden.
- **Por qué así (alternativas):** Con DCT por bloques de 8×8 al estilo JPEG [3][4], un presupuesto pequeño deja solo el DC de cada bloque y aparece un tablero de ajedrez. Las razones de usar bloques, el cálculo y la adaptación local, pesan poco en cuadrículas de 64–512, así que se eligió una transformada global. Las wavelets pueden conservar mejor los bordes con el mismo presupuesto, pero complican la implementación y el modelo de contexto.
- **Valor:** Aunque solo queden frecuencias bajas, la imagen se difumina con suavidad y sin bloques: el tipo de degradación adecuado para un presupuesto mínimo.
- **Limitaciones:** Al ser global, la textura fina de una zona compite por el presupuesto con toda la imagen. Una serie de cosenos truncada sobrepasa en torno a un 9 % cerca de los bordes (fenómeno de Gibbs) [19] y produce ondulaciones. Todavía no se usa una DCT rápida (basada en FFT) [24], así que una cuadrícula de 512 tarda casi un segundo.
- **Fuentes:** [1], [2], [3], [4], [19], [24]

#### Notas de diseño: cuantización

- **Qué:** La regla que convierte los coeficientes reales en niveles enteros: tamaño del paso, zona muerta, pesos de frecuencia y color, y la corrección de cuadrícula.
- **Por qué:** Aquí se decide la mayor parte de la compresión. Descartar con más fuerza lo menos visible hace que el resultado se vea mejor con la misma longitud.
- **Cómo (método y técnica):** `step = 0.0002 · 1.06^q · (1 + 0.11·r) · (1,7 en color) · (64/lado largo en la versión 1)`, `level = sign(x) · ⌊|x|/step + 0.5 − 0.1⌋`, reconstruido como `level · step`.
- **Enfoque matemático:** Se sabe que los coeficientes AC de la DCT siguen aproximadamente una distribución de Laplace [15], y para fuentes laplacianas un cuantizador uniforme con el intervalo del cero ensanchado (zona muerta) es favorable en términos de tasa-distorsión [14]. Aquí un coeficiente pasa a cero por debajo de `0.6 · step`. q es exponencial para repartir con 7 bits un rango amplio de `1.06^127 ≈ 1620` veces (6 % por paso). La corrección de la versión 1 sale de esta relación: una textura con varianza por píxel σ² tiene coeficientes de varianza σ² con una DCT ortonormal, y los coeficientes de amplitud usados aquí son los ortonormales multiplicados por `2/√(WH)`. Su desviación típica se reduce entonces como `1/√(WH)`, que con proporción fija es proporcional a `1/lado largo`. Multiplicar el paso por `64/lado largo` hace que el mismo q signifique la misma precisión relativa en cualquier cuadrícula.
- **Enfoque de ingeniería:** El peso de frecuencia `(1 + 0.11r)` es una aproximación lineal de la función de sensibilidad al contraste [16], y el factor 1,7 del color refleja la menor sensibilidad al contraste cromático [17]. Los valores se eligieron comparando el PSNR en las fotos de prueba (punto 4 de [Experimentos y decisiones](https://github.com/Velvitty/img-dev/blob/main/README.es.md#experimentos-y-decisiones)).
- **Por qué así (alternativas):** Las tablas de cuantización 8×8 al estilo JPEG están atadas al tamaño de bloque y no encajan con una DCT global ni con una cuadrícula variable. Acercar a cero los valores reconstruidos (shrink) es mejor en teoría para fuentes laplacianas, pero el 0 dio la mejor medida y se mantuvo.
- **Valor:** Un solo q ajusta la tasa con suavidad, lo que simplifica el ajuste al límite de caracteres. La corrección de cuadrícula permite que los presupuestos grandes usen todos sus caracteres.
- **Limitaciones:** Los pesos se eligieron por PSNR con ocho fotos de prueba y pueden no ser óptimos para la calidad percibida. En la versión 0, el DC usa 6 bits para el brillo y 5 para el color, así que el color medio puede desviarse hasta unos 2/255 (brillo) y 4/255 (color).
- **Fuentes:** [14], [15], [16], [17]

#### Notas de diseño: codificación entrópica

- **Qué:** El codificador aritmético binario (range coder) y el modelo de contexto que convierten los enteros cuantizados en bits.
- **Por qué:** La mayoría de los coeficientes son cero, y la probabilidad de serlo cambia mucho según la frecuencia y los vecinos. Codificar con esas probabilidades es lo que acerca la longitud a la entropía.
- **Cómo (método y técnica):** Cada coeficiente se convierte en una serie de decisiones binarias: cero o no → signo → mayor que 1 → mayor que 2 → Exp-Golomb (la misma binarización que CABAC [11][12]). Cada decisión se codifica con la probabilidad de su contexto (canal, banda de frecuencia, si los dos coeficientes anteriores eran cero). El codificador sigue la estructura del range coder del SDK de LZMA (rango de 32 bits, acarreo mediante un byte de caché) [9][10].
- **Enfoque matemático:** Un suceso de probabilidad p cuesta `−log2 p` bits, y la codificación aritmética deja la longitud total solo unos bytes por encima de esa suma [8]; los bytes sobrantes del final se eliminan con el recorte de caracteres finales de [Ajuste al límite de caracteres](https://github.com/Velvitty/img-dev/blob/main/README.es.md#ajuste-al-límite-de-caracteres). Huffman gasta al menos 1 bit por símbolo, incluso en un «cero» con probabilidad 0,95, donde la codificación aritmética gasta unos 0,074 bits. Las probabilidades son enteros de 11 bits que se actualizan en cada bit como `p ← p + (2048 − p)/16` con 0 y `p ← p − p/16` con 1: una media móvil exponencial con una memoria efectiva de unos 16 bits, que se adapta rápido incluso en un flujo de pocos cientos de bytes.
- **Enfoque de ingeniería:** El modelo se mantiene pequeño, con 216 contextos en total. Con pocos datos, demasiados contextos harían que el flujo terminara antes de que cada uno aprendiera algo. La cabecera pasa por el mismo codificador con probabilidad 0,5 y cuesta exactamente 1 bit por bit, sin código aparte para empaquetar bits.
- **Por qué así (alternativas):** SHIFT 4 (1/16) superó en 0,04 dB al valor por defecto de LZMA, 5 (1/32), en las fotos de validación. Unas probabilidades iniciales entrenadas con las fotos de prueba solo ganaron 0,03–0,07 dB, demasiado poco para el coste de incluir una tabla en el código.
- **Valor:** La misma información se escribe mucho más corta que con códigos de longitud fija, así que un código de 400 caracteres puede contener cientos de coeficientes.
- **Limitaciones:** El signo sigue costando casi un bit entero. No se usan vecinos bidimensionales (el coeficiente de arriba) como contexto, ni modelos más potentes como la mezcla de contextos. Un bit dañado estropea todo lo que viene después.
- **Fuentes:** [8], [9], [10], [11], [12], [13]

#### Notas de diseño: formato del flujo de bits

- **Qué:** La cabecera con versión, cuadrícula, proporción, paso de cuantización, color medio y número de coeficientes, seguida de los coeficientes.
- **Por qué:** El decodificador debe reconstruir la misma cuadrícula, el mismo orden de coeficientes y los mismos pasos solo a partir del código.
- **Cómo (método y técnica):** La versión 0 tiene una cabecera fija de 62 bits. La versión 1 añade un índice de cuadrícula de 3 bits tras el bit de versión, usa 8 bits por valor DC y `⌈log2(W·H)⌉` bits por número de coeficientes.
- **Enfoque matemático:** 62 bits son el 1,2 % de un presupuesto de 400 caracteres (unos 5200 bits). Guardar el número de coeficientes en la cabecera cuesta 10–16 bits fijos por canal, en lugar de preguntar en cada coeficiente si es el final del bloque.
- **Enfoque de ingeniería:** La cuadrícula de 64 píxeles se sigue escribiendo como versión 0, así que tras ampliar el formato, los códigos existentes y los de 400 caracteres no han cambiado ni un bit (comprobado con 16 códigos de las fotos de prueba). El índice de cuadrícula 7 queda libre como marca para la próxima ampliación del formato.
- **Por qué así (alternativas):** Un campo de versión de varios bits desde el principio habría sido más limpio, pero con 400 caracteres cada bit cuenta, así que empezó con 1 bit y las ampliaciones usan valores libres dentro de la versión 1.
- **Valor:** La sobrecarga es mínima con presupuestos pequeños y queda margen para ampliar el formato.
- **Limitaciones:** No hay suma de comprobación, así que un código cortado o alterado no se detecta con seguridad y puede decodificarse en una imagen equivocada.
- **Fuentes:** [11]

#### Notas de diseño: conversión a texto

- **Qué:** Escribir los bytes como sílabas hangul (13 bits) o caracteres base64url (6 bits).
- **Por qué:** Los comentarios se limitan por número de caracteres, así que cada carácter debe llevar el máximo de bits posible.
- **Cómo (método y técnica):** La cadena de bits se corta en grupos de 13 bits escritos como `U+AC00 + valor`, o en grupos de 6 bits escritos con el alfabeto base64url [25]. Al leer, se ignoran los caracteres fuera de esos rangos.
- **Enfoque matemático:** Unicode tiene 11 172 sílabas hangul en `U+AC00–U+D7A3` [29]. Usarlas todas daría `log2 11172 ≈ 13,447` bits, un 3,4 % más que 13, pero exigiría tratar todo el código como un número grande y cambiarlo de base, un cálculo `O(n²)`. Con una potencia de dos, 8192 caracteres, basta con cortar y unir bits.
- **Enfoque de ingeniería:** Los bits sobrantes del último carácter se rellenan con ceros, lo que encaja con la regla del decodificador de leer como cero los bytes que faltan y hace posible el recorte de caracteres finales.
- **Por qué así (alternativas):** base83 (BlurHash [33]) o base85 llevan algo más de 6 bits, pero incluyen caracteres especiales de las direcciones como `#`, `%` y `?`. Los emojis o los caracteres han podrían llevar más bits, pero su visualización y escritura son poco fiables. El hangul es natural en DCInside y está en casi todas las fuentes.
- **Valor:** Los mismos 400 caracteres guardan unas 2,2 veces más información que en ASCII.
- **Limitaciones:** Donde la longitud se cuenta en bytes UTF-8, el hangul lleva 4,33 bits por byte, peor que los 6 del ASCII. Una normalización que descomponga las sílabas en jamo (NFD) [29] rompe el código; el decodificador no convierte ahora el texto pegado de vuelta a NFC.
- **Fuentes:** [25], [29], [33]

#### Notas de diseño: control de tasa y elección de cuadrícula

- **Qué:** Cómo se eligen el q más fino y el tamaño de cuadrícula más adecuado dentro del límite de caracteres.
- **Por qué:** Solo se fija un número de caracteres, y la calidad debe ser la mejor posible dentro de él.
- **Cómo (método y técnica):** Se hace una búsqueda binaria de q entre 0 y 127 (7–8 codificaciones). Se quitan caracteres finales de uno en uno mientras el código se siga decodificando igual. Si la cuadrícula de 64 píxeles no llena el presupuesto ni con q = 0, la foto se vuelve a codificar en la siguiente cuadrícula, y los dos resultados se comparan por error cuadrático medio frente al original al tamaño de la cuadrícula mayor; se queda el mejor.
- **Enfoque matemático:** La longitud del código R(q) decrece casi monótonamente con q, así que la búsqueda binaria encuentra el menor q con `R(q) ≤ presupuesto` en unos `⌈log2 128⌉ = 7` pasos. Decidir si una tasa dada se gasta en más resolución con cuantización más gruesa o en menos resolución con cuantización más fina es un problema clásico de optimización tasa-distorsión [20]. Aquí se resuelve con una regla voraz, probar una cuadrícula mayor solo cuando la menor está saturada, más una comparación real del error.
- **Enfoque de ingeniería:** Una cuadrícula que cabe con q = 0 se salta la búsqueda binaria, lo que bajó una codificación de 7000 caracteres de unos 1,8 s a 1,4 s. El recorte de caracteres finales elimina los bytes sobrantes que el range coder emite al final, y como lo comprueba decodificando de verdad, no puede fallar.
- **Por qué así (alternativas):** Codificar todas las cuadrículas y compararlas podría dar mejores resultados, pero las grandes tardan segundos cada una. Se probó una cuadrícula de 768 píxeles y se descartó porque 20 000 caracteres tardaban 16 segundos.
- **Valor:** Con 400 caracteres sigue siendo igual de rápido, y con 7000 usa 6968 caracteres y llega a 28,0 dB.
- **Limitaciones:** R(q) no es estrictamente monótona, así que pueden sobrar algunos caracteres. La regla voraz no garantiza la cuadrícula óptima. El cálculo se ejecuta en el hilo principal, así que la página se detiene un momento con presupuestos grandes (sin Web Worker).
- **Fuentes:** [8], [20]

#### Notas de diseño: decodificación y ampliación

- **Qué:** Calcular directamente desde los coeficientes la imagen al tamaño de la pantalla.
- **Por qué:** Construir una imagen del tamaño de la cuadrícula y estirarla añade escalones de píxeles o el desenfoque de la interpolación.
- **Cómo (método y técnica):** Las coordenadas de pantalla, pasadas a unidades de cuadrícula, se sustituyen en x e y de la transformada inversa y la serie de cosenos se evalúa directamente. Primero se calculan las filas, y se saltan las filas cuyos coeficientes son todos cero.
- **Enfoque matemático:** La fórmula de la DCT inversa es una función suave definida también para x no enteros, así que muestrearla a cualquier resolución es una interpolación de banda limitada en la base de cosenos. El coste es `O(H·W·outW + filas·outH·outW)`.
- **Enfoque de ingeniería:** La misma función dibuja las vistas previas de las tarjetas (480 px en el lado largo) y la pestaña Ver (960 px). Termina con una conversión YCbCr → RGB recortada a 0–255.
- **Por qué así (alternativas):** Una DCT inversa al tamaño de la cuadrícula seguida del escalado de imagen del canvas sería más rápida, pero en cuadrículas pequeñas deja los patrones en cruz borrosos típicos de la interpolación bilineal.
- **Valor:** Con muy poca información se obtiene un resultado agradable, difuminado con suavidad.
- **Limitaciones:** Las ondulaciones de Gibbs [19] siguen viéndose al ampliar. `Math.cos` es, según la especificación, una aproximación de la implementación [31], así que los valores de píxel pueden diferir en ±1 entre navegadores.
- **Fuentes:** [19], [31]

#### Notas de diseño: diseño de la página

- **Qué:** Las pantallas, la detección del código, la restricción de dirección y los ajustes recordados dentro de un único archivo HTML.
- **Por qué:** El principio de diseño es un solo archivo HTML con JS y CSS incluidos que se ejecute solo en el navegador, y quien lo recibe debe poder abrirlo sin preparar nada.
- **Cómo (método y técnica):** El código del códec se inserta en la página al compilar. En el texto pegado, se toma como código el tramo más largo de hangul (al menos 4 caracteres); si no lo hay, el tramo base64url más largo después del último `?` o `#`. Fuera de g.yeon.at, localhost, archivos locales y la vista previa de claude.ai, la página solo dibuja un aviso y se detiene.
- **Enfoque de ingeniería:** Todos los accesos a localStorage están protegidos para que la página funcione con los valores por defecto si el almacenamiento está bloqueado. Si se rechaza la API del portapapeles, copiar recurre a seleccionar el texto. El canvas y las imágenes siguen el comportamiento del estándar HTML (una imagen animada dibuja su fotograma por defecto) [30].
- **Por qué así (alternativas):** Un framework acortaría el código de la interfaz, pero agrandaría el archivo y añadiría dependencias externas. Una restricción de dirección solo es fiable en un servidor, pero el alojamiento estático no tiene código de servidor.
- **Valor:** Publicar es copiar un archivo, y la página funciona sin conexión (salvo las fuentes).
- **Limitaciones:** La detección del código es heurística, así que una frase en hangul más larga que el código puede confundirse con él. La restricción de dirección es solo disuasoria y no detiene a quien modifique el código fuente.
- **Fuentes:** [27], [30]

#### Notas de diseño: compilación

- **Qué:** El script que combina el códec y la página de `src/` en el archivo publicable y la vista previa.
- **Por qué:** Las pruebas en Node.js y el navegador deben usar el mismo archivo del códec para que lo probado sea lo publicado.
- **Cómo (método y técnica):** Quita solo la línea `module.exports` de `src/codec.js` e inserta el resto en `/*CODEC*/`.
- **Enfoque de ingeniería:** Los resultados de la compilación (`dist/`, `build/`) también se guardan en el repositorio, para usar el archivo publicable donde no haya herramientas de compilación.
- **Por qué así (alternativas):** Los empaquetadores como webpack son pesados para esta escala y añaden código de ejecución al resultado.
- **Valor:** La compilación se reproduce con 30 líneas de Python sin dependencias.
- **Limitaciones:** Como los resultados se guardan en el repositorio, cada cambio debe compilarse y guardarse junto con ellos para que código y resultado no se desfasen.

#### Notas de diseño: método de evaluación

- **Qué:** Cómo se compara numéricamente la calidad (PSNR), más las fotos de prueba y las pruebas en navegador.
- **Por qué:** Para elegir parámetros y comprobar que un cambio no rompió resultados anteriores hacen falta números reproducibles.
- **Cómo (método y técnica):** Se compara el original reducido con filtro de caja a 160×120 con el resultado decodificado dibujado a 160×120. Las pruebas de cuadrículas grandes comparan a 512×512. Las fotos de prueba son 8 de entrenamiento y 6 de validación de picsum.photos (fotos de Unsplash) [35].
- **Enfoque matemático:** Las frecuencias altas del original no pueden transmitirse de todos modos, así que comparar cerca del tamaño que ve quien lo recibe es lo que revela las diferencias entre códecs. El PSNR es un logaritmo del error cuadrático medio, sencillo de calcular y fácil de reproducir.
- **Enfoque de ingeniería:** Tras cada cambio del códec, se comprueba que los resultados con 400 caracteres (PSNR medio y cadenas de código) coinciden con los anteriores, para mantener la compatibilidad. Las pruebas en navegador abren la página real con Playwright y comprueban la creación de códigos, la apertura de enlaces, el pegado y la restricción de dirección.
- **Por qué así (alternativas):** Las métricas perceptivas como SSIM se ajustan mejor al juicio humano [21], pero se empezó con PSNR por ser simple y fácil de interpretar.
- **Valor:** En segundos se sabe si un cambio mejoró la calidad o rompió códigos existentes.
- **Limitaciones:** Catorce fotos son una muestra pequeña, y el PSNR puede no coincidir con la calidad percibida [21][22]. La imagen de 1200×1200 usada en las pruebas de límites grandes no está en el repositorio.
- **Fuentes:** [21], [22], [35]

#### Notas de diseño: las matemáticas del PSNR

- **Qué:** La definición del PSNR y el recuento de casos del ejemplo de 4 píxeles.
- **Por qué:** Para responder con exactitud a preguntas como «¿cuántos dB si es idéntica?» y dar una escala para leer las cifras de este documento.
- **Cómo (método y técnica):** Todos los valores de canal de la imagen de 4 píxeles (rojo, amarillo, azul, verde) son 0 o 255, así que si los de la otra imagen recorren 0–255, la diferencia d de cada canal es uniforme en 0–255. La distribución de la suma de errores al cuadrado es la convolución de 12 veces la distribución de d².
- **Enfoque matemático:** `PSNR = 10·log10(255² / MSE)`, así que un MSE 10 veces mayor baja el PSNR exactamente 10 dB, y MSE = 0 da infinito por definición. Un solo canal con 1 de diferencia da `MSE = 1/12` y `PSNR = 10·log10(12 · 255²) ≈ 58,9 dB`. En el recuento, la convolución por FFT [24] tiene un error relativo de unos 10⁻¹⁶, que borró las probabilidades de la cola por debajo de 10⁻²⁰, y `np.histogram` obtiene los valores de cada intervalo como diferencias de una suma acumulada, lo que provoca cancelación al restar números diminutos de valores cercanos a 1 [23]. Pasar a una convolución directa, que solo suma términos positivos, y a `np.bincount` recuperó la cola exacta hasta 58,9 dB.
- **Enfoque de ingeniería:** La tabla de impresiones para cada valor de PSNR es una guía empírica de este proyecto, no un estándar de la literatura.
- **Valor:** El significado de las cifras se transmite sin malentendidos, y se corrigió la barra falsa causada por el error numérico.
- **Limitaciones:** El PSNR puede no coincidir con la calidad que percibe la gente [21][22].
- **Fuentes:** [21], [22], [23], [24]

### Referencias

Fuentes de los conceptos, estándares y trabajos relacionados usados en el diseño. Las mediciones proceden de los scripts de `lab/` de [img-dev](https://github.com/Velvitty/img-dev) y de la lista «Mediciones» de abajo.

#### Artículos y libros

1. N. Ahmed, T. Natarajan, K. R. Rao, "Discrete Cosine Transform," *IEEE Transactions on Computers*, vol. C-23, no. 1, pp. 90–93, 1974. doi:10.1109/T-C.1974.223784
2. K. R. Rao, P. Yip, *Discrete Cosine Transform: Algorithms, Advantages, Applications*, Academic Press, 1990.
3. G. K. Wallace, "The JPEG still picture compression standard," *IEEE Transactions on Consumer Electronics*, vol. 38, no. 1, pp. xviii–xxxiv, 1992. doi:10.1109/30.125072
4. ITU-T Recommendation T.81 | ISO/IEC 10918-1, *Digital compression and coding of continuous-tone still images – Requirements and guidelines*, 1992.
5. E. Hamilton, *JPEG File Interchange Format, Version 1.02*, C-Cube Microsystems, 1992. (Estandarizado como ITU-T T.871.)
6. ITU-R Recommendation BT.601-7, *Studio encoding parameters of digital television for standard 4:3 and wide-screen 16:9 aspect ratios*, 2011.
7. C. E. Shannon, "A Mathematical Theory of Communication," *Bell System Technical Journal*, vol. 27, pp. 379–423, 623–656, 1948.
8. I. H. Witten, R. M. Neal, J. G. Cleary, "Arithmetic coding for data compression," *Communications of the ACM*, vol. 30, no. 6, pp. 520–540, 1987. doi:10.1145/214762.214771
9. G. N. N. Martin, "Range encoding: an algorithm for removing redundancy from a digitised message," Video & Data Recording Conference, Southampton, UK, 1979.
10. I. Pavlov, *LZMA SDK* (range coder: `LzmaEnc.c`, `LzmaDec.c`), https://www.7-zip.org/sdk.html
11. D. Marpe, H. Schwarz, T. Wiegand, "Context-based adaptive binary arithmetic coding in the H.264/AVC video compression standard," *IEEE Transactions on Circuits and Systems for Video Technology*, vol. 13, no. 7, pp. 620–636, 2003. doi:10.1109/TCSVT.2003.815173
12. J. Teuhola, "A compression method for clustered bit-vectors," *Information Processing Letters*, vol. 7, no. 6, pp. 308–311, 1978. doi:10.1016/0020-0190(78)90024-8 (códigos Exp-Golomb)
13. S. W. Golomb, "Run-length encodings," *IEEE Transactions on Information Theory*, vol. 12, no. 3, pp. 399–401, 1966. doi:10.1109/TIT.1966.1053907
14. G. J. Sullivan, "Efficient scalar quantization of exponential and Laplacian random variables," *IEEE Transactions on Information Theory*, vol. 42, no. 5, pp. 1365–1374, 1996. doi:10.1109/18.532878
15. E. Y. Lam, J. W. Goodman, "A mathematical analysis of the DCT coefficient distributions for images," *IEEE Transactions on Image Processing*, vol. 9, no. 10, pp. 1661–1666, 2000. doi:10.1109/83.869177
16. J. L. Mannos, D. J. Sakrison, "The effects of a visual fidelity criterion on the encoding of images," *IEEE Transactions on Information Theory*, vol. 20, no. 4, pp. 525–536, 1974. doi:10.1109/TIT.1974.1055250
17. K. T. Mullen, "The contrast sensitivity of human colour vision to red-green and blue-yellow chromatic gratings," *The Journal of Physiology*, vol. 359, pp. 381–400, 1985. doi:10.1113/jphysiol.1985.sp015591
18. L. Williams, "Pyramidal parametrics," *Computer Graphics (SIGGRAPH '83)*, vol. 17, no. 3, pp. 1–11, 1983. doi:10.1145/964967.801126
19. J. W. Gibbs, "Fourier's Series," *Nature*, vol. 59, p. 606, 1899. doi:10.1038/059606a0; E. Hewitt, R. E. Hewitt, "The Gibbs-Wilbraham phenomenon: An episode in Fourier analysis," *Archive for History of Exact Sciences*, vol. 21, pp. 129–160, 1979. doi:10.1007/BF00330404
20. A. Ortega, K. Ramchandran, "Rate-distortion methods for image and video compression," *IEEE Signal Processing Magazine*, vol. 15, no. 6, pp. 23–50, 1998. doi:10.1109/79.733495
21. Z. Wang, A. C. Bovik, "Mean squared error: Love it or leave it? A new look at signal fidelity measures," *IEEE Signal Processing Magazine*, vol. 26, no. 1, pp. 98–117, 2009. doi:10.1109/MSP.2008.930649
22. Q. Huynh-Thu, M. Ghanbari, "Scope of validity of PSNR in image/video quality assessment," *Electronics Letters*, vol. 44, no. 13, pp. 800–801, 2008. doi:10.1049/el:20080522
23. D. Goldberg, "What every computer scientist should know about floating-point arithmetic," *ACM Computing Surveys*, vol. 23, no. 1, pp. 5–48, 1991. doi:10.1145/103162.103163
24. J. W. Cooley, J. W. Tukey, "An algorithm for the machine calculation of complex Fourier series," *Mathematics of Computation*, vol. 19, no. 90, pp. 297–301, 1965. doi:10.1090/S0025-5718-1965-0178586-1

#### Estándares y especificaciones

25. S. Josefsson, *RFC 4648: The Base16, Base32, and Base64 Data Encodings*, IETF, 2006. (Sección 5, base64url)
26. L. Masinter, *RFC 2397: The "data" URL scheme*, IETF, 1998.
27. T. Berners-Lee, R. Fielding, L. Masinter, *RFC 3986: Uniform Resource Identifier (URI): Generic Syntax*, IETF, 2005. (Consulta, fragmento y codificación por porcentajes)
28. R. Fielding, M. Nottingham, J. Reschke, *RFC 9110: HTTP Semantics*, IETF, 2022. (Sección 15.5.15, 414 URI Too Long)
29. The Unicode Consortium, *The Unicode Standard*, capítulo 3, "Conjoining Jamo Behavior" (sílabas hangul U+AC00–U+D7A3), y *Unicode Standard Annex #15: Unicode Normalization Forms*. https://www.unicode.org/
30. WHATWG, *HTML Living Standard*, contexto 2D de canvas (`drawImage`, `imageSmoothingQuality`). https://html.spec.whatwg.org/multipage/canvas.html
31. Ecma International, *ECMA-262: ECMAScript Language Specification* (`Math.cos` es una aproximación de la implementación). https://tc39.es/ecma262/

#### Documentación técnica

32. GitHub Docs, "Managing a custom domain for your GitHub Pages site." https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site

#### Trabajos relacionados

33. Wolt, *BlurHash*. https://github.com/woltapp/blurhash
34. E. Wallace, *ThumbHash*, 2023. https://github.com/evanw/thumbhash

#### Fuentes de datos

35. Picsum Photos (fotos de Unsplash). https://picsum.photos/ — img-dev `lab/imgs/` (ids 1015, 1025, 1062, 1084, 177, 237, 433, 64), img-dev `lab/val/` (ids 1003, 1035, 1074, 200, 287, 334)
36. C. E. Shannon, "Communication in the presence of noise," *Proceedings of the IRE*, vol. 37, no. 1, pp. 10–21, 1949. doi:10.1109/JRPROC.1949.232969 (teorema de muestreo)

#### Mediciones

- Límite de 400 caracteres en los comentarios de DCInside, imágenes y enlaces `data:` que no se muestran, y enlazado automático solo con `https://`: comprobado en un cuadro de comentarios real.
- Límite del 414 en GitHub Pages (g.yeon.at): el 2026-10-26, peticiones con consultas de 1000 a 60 000 bytes devolvieron 200 con 6000 bytes y 414 con 9000.
- Cifras de calidad y velocidad: medidas con los scripts de `lab/` de [img-dev](https://github.com/Velvitty/img-dev) (Node.js 22). La imagen de 1200×1200 usada en las pruebas de límites grandes no está en el repositorio.
