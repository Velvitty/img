<p align="center"><a href="README.md">한국어</a> · <a href="README.en.md">English</a> · <a href="README.ja.md">日本語</a> · <a href="README.es.md">Español</a></p>

# Compresor de imágenes en 400 caracteres

Una página web que convierte una foto en un texto que cabe en una sola línea de comentario, y que vuelve a convertir ese texto en la foto.

**https://g.yeon.at/**

![El original y los resultados con 400, 1000, 4000 y 7000 caracteres](docs/example.jpg)

<sub>De izquierda a derecha: original, 400, 1000, 4000 y 7000 caracteres (código hangul, dirección incluida). Foto: Thomas Lefebvre / Unsplash</sub>

Se creó para compartir fotos en sitios con comentarios cortos donde no se pueden publicar imágenes, como DCInside. La foto se comprime en el navegador hasta unos cientos de bytes y se convierte en una línea como `https://g.yeon.at/?TqVTweO…`; quien la recibe abre el enlace y ve la foto.

- Sin servidor, sin subidas, sin almacenamiento. El código es la propia foto.
- Cada sílaba hangul lleva 13 bits, unas 2,2 veces más información que el ASCII con el mismo número de caracteres.
- Al subir el límite de caracteres, la cuadrícula que guarda la foto crece de 64 a 512 píxeles y la imagen gana nitidez.
- Todo funciona desde un único archivo HTML.

## Índice

1. [Uso](#uso)
2. [Códigos y enlaces](#códigos-y-enlaces)
3. [Calidad](#calidad)
4. [Privacidad y requisitos](#privacidad-y-requisitos)
5. [Limitaciones](#limitaciones)
6. [Solución de problemas](#solución-de-problemas)
7. [Preguntas frecuentes](#preguntas-frecuentes)
8. [Notas de implementación](#notas-de-implementación)
   - [Objetivos de diseño](#objetivos-de-diseño)
   - [Preprocesado](#preprocesado)
   - [Transformada](#transformada)
   - [Cuantización](#cuantización)
   - [Codificación entrópica](#codificación-entrópica)
   - [Flujo de bits](#flujo-de-bits)
   - [Conversión a texto](#conversión-a-texto)
   - [Control de tasa y elección de cuadrícula](#control-de-tasa-y-elección-de-cuadrícula)
   - [Decodificación y ampliación](#decodificación-y-ampliación)
   - [Direcciones y transporte](#direcciones-y-transporte)
   - [Diseño de la página](#diseño-de-la-página)
   - [Evaluación de la calidad](#evaluación-de-la-calidad)
   - [Las matemáticas del PSNR](#las-matemáticas-del-psnr)
9. [Referencias](#referencias)

## Uso

La página tiene tres pestañas: **만들기** (Crear), **보기** (Ver) y **JPG data 링크** (Enlace JPG data).

### Crear

1. Arrastra una foto, haz clic en el recuadro para elegirla o pégala con Ctrl/⌘+V. Se pueden añadir varias a la vez.
2. Cada foto genera una tarjeta: el original a la izquierda y, a la derecha, lo que verá quien la reciba.
3. Pulsa **복사** (Copiar) para copiar la dirección y el código en una sola línea, y pega esa línea tal cual en el comentario.

Bajo cada tarjeta aparecen el tamaño original, la longitud del código, la longitud total frente al límite, el tamaño de la cuadrícula y el paso de compresión (0–127; cuanto más bajo, más fino).

| Ajuste | Valor por defecto | Descripción |
|---|---|---|
| Letras del código | ASCII | El ASCII lleva 6 bits por carácter; el hangul, 13. Los comentarios de DCInside solo convierten en enlace las direcciones en ASCII, así que el ASCII es el valor por defecto; el hangul sirve para enviar solo el código y que quien lo reciba lo pegue en la pestaña Ver. |
| Límite de caracteres | 400 | Botones para un comentario de DCInside (400), un tuit (280) y 1000; también se puede escribir cualquier valor. |
| Dirección delante | `https://g.yeon.at/?` | Texto que va antes del código. Si se deja vacío, se genera solo el código. |

Al cambiar un ajuste se vuelven a comprimir todas las fotos de la página, y los ajustes se recuerdan en el navegador.

### Ver

Al abrir un enlace recibido se abre directamente esta pestaña y aparece la foto. También se puede pegar aquí un código copiado; el texto de alrededor se ignora y solo se lee el código.

### Enlace JPG data

Una herramienta para sitios que muestran los enlaces `data:` como imágenes (los que admiten imágenes en HTML o Markdown). Vuelve a codificar la foto como JPG y genera un enlace `data:image/jpeg;base64,…`, buscando la mejor calidad dentro de una longitud objetivo o usando una calidad elegida, y lo copia como enlace, etiqueta `<img>` o Markdown. Con el mismo número de caracteres, el código de la pestaña Crear es más nítido (consulta [Calidad](#calidad)).

## Códigos y enlaces

| Formato | Ejemplo | Caracteres para el código | Capacidad |
|---|---|---|---|
| ASCII + dirección (por defecto) | `https://g.yeon.at/?TqVTweO…` | 381 | unos 285 bytes |
| Hangul + dirección | `https://g.yeon.at/?뗈앏괟…` | 381 | unos 619 bytes |
| Solo código hangul | `뗈앏괟…` | 400 | 650 bytes |
| Solo código ASCII | `TqVTweO…` | 400 | 300 bytes |

- La dirección debe empezar por `https://` para convertirse en un enlace pulsable en los comentarios. Los comentarios de DCInside solo enlazan direcciones escritas en ASCII, así que una dirección con un código hangul no se puede pulsar.
- Cuando el código es tan largo que la dirección pasaría de unos 7 KB (códigos ASCII de unos 6980 caracteres o hangul de unos 770 o más), se usa `#` en lugar de `?`. `https://g.yeon.at/#código` abre la misma foto, pero los comentarios de DCInside dejan fuera del enlace lo que va después de `#`.
- `https://g.yeon.at/index.html?código` también funciona, pero usa 10 caracteres más.

## Calidad

Al subir el límite de caracteres, la cuadrícula que guarda la foto crece y la imagen gana nitidez. La foto del oso de la imagen de ejemplo (1280×960), comprimida como código hangul y medida a 640×480:

| Límite | Caracteres usados | Cuadrícula | PSNR | Enlace JPG `data:` de la misma longitud |
|---|---|---|---|---|
| 400 | 387 | 64 px | 23,9 dB | no cabe |
| 1000 | 955 | 96 px | 25,5 dB | no cabe |
| 4000 | 3810 | 256 px | 27,7 dB | 26,2 dB |
| 7000 | 6877 | 512 px | 28,8 dB | 27,4 dB |
| 10 000 | 8526 | 512 px | 29,2 dB | 28,2 dB |

La cuadrícula llega como máximo a 512 píxeles, así que con esta foto nada cambia por encima de unos 8500 caracteres. La compresión tarda menos de 0,1 s hasta 1000 caracteres, unos 0,4 s con 4000 y unos 3 s a partir de 7000.

Con 400 caracteres (dirección incluida), ocho fotos medidas a 160×120:

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

- Las fotos hechas de formas grandes y cambios de color suaves (niebla, cielo, primeros planos de caras o animales) salen bien.
- Las fotos cubiertas de textura fina, como vetas de madera, hierba o roca, quedan más borrosas con la misma longitud.
- Recortar antes la parte que se quiere mostrar la hace mucho más nítida con la misma longitud.
- Con 400 caracteres no se puede leer texto.

El PSNR mide cuánto se parece un resultado al original; cuanto más alto, más parecido. La guía siguiente es la escala empírica que usa este documento (consulta [Las matemáticas del PSNR](#las-matemáticas-del-psnr)).

| PSNR | Impresión |
|---|---|
| ∞ | Idéntico al original |
| 45 dB o más | Prácticamente indistinguible |
| 35–40 dB | JPG típico |
| Unos 30 dB | Se ve algo emborronado si se mira de cerca |
| Unos 25 dB | Borroso pero reconocible |
| Unos 20 dB | Solo quedan formas y colores |

## Privacidad y requisitos

- Las fotos nunca salen del navegador. La compresión y la decodificación las hace el propio script de la página.
- No se guarda nada en ningún servidor, así que cualquiera con el código puede ver la misma foto. Si se borra el código, la foto desaparece.
- El único recurso externo son las fuentes (Google Fonts); sin ellas se usan las fuentes por defecto.
- El almacenamiento del navegador (localStorage) solo guarda las letras del código, la dirección delante y la última pestaña. No se guardan fotos ni códigos.
- Funciona en versiones recientes de Chrome, Edge, Safari y Firefox, también en el móvil.
- La página solo funciona en g.yeon.at. Copiada en otra dirección, muestra un aviso y un botón para ir a g.yeon.at, que conserva el código de la dirección si lo hay.

## Limitaciones

- Con 400 caracteres el resultado es una vista previa borrosa; no se distinguen textos ni texturas finas.
- Aparecen ondulaciones junto a los bordes de brillo marcados.
- La cuadrícula llega como máximo a 512 píxeles, así que a partir de cierto punto más caracteres no ayudan.
- Los límites de 7000 caracteres o más tardan unos segundos en comprimirse, y la página se detiene mientras tanto.
- Las fotos más alargadas que unos 3,8:1 se aplastan a esa proporción, las zonas transparentes quedan en blanco y de los GIF animados solo se guarda el primer fotograma.
- Donde la longitud se cuenta en bytes UTF-8, los códigos hangul salen perdiendo; ahí conviene el código ASCII.
- En los comentarios de DCInside, los códigos hangul y lo que va después de `#` no entran en el enlace. Un código hangul ocupa menos de la mitad, pero quien lo recibe tiene que copiarlo y pegarlo en la pestaña Ver.

## Solución de problemas

| Síntoma | Solución |
|---|---|
| «이 글자로는 사진을 만들 수 없습니다» (no se puede crear una foto con este texto) | El código se cortó o se mezclaron otros caracteres. Copia de nuevo la línea entera. |
| El enlace abre la pestaña Crear sin foto | El enlace automático del comentario se cortó antes de `?` o `#`, algo que ocurre con los códigos hangul y los enlaces con `#`. Copia el código del comentario y pégalo en la pestaña Ver. Quien envía puede usar el código ASCII. |
| El botón de copiar no hace nada | El texto ya está seleccionado; pulsa Ctrl/⌘+C. |
| La foto no se parece en nada al original | El código puede venir de una versión con otro esquema de compresión. |
| El total queda un poco por debajo del límite | Es normal. Los pasos de compresión van a saltos, así que pueden sobrar algunos caracteres. |

## Preguntas frecuentes

**¿Se puede recuperar la foto original a partir de un código?**
No. Un código solo contiene la información de una vista previa borrosa.

**¿La misma foto da siempre el mismo código?**
En el mismo navegador, sí. Cada navegador reduce las imágenes de forma un poco distinta, así que el código puede variar ligeramente, pero un código ya creado se decodifica en la misma foto en cualquier navegador.

**¿Por qué el código hangul es más nítido que el ASCII?**
Porque el mismo número de caracteres guarda unas 2,2 veces más información. Un carácter ASCII es uno entre 64 valores; uno hangul, uno entre 8192.

**El código hangul parece hecho de caracteres raros.**
Incluye sílabas poco usadas como `뗈` y `쯿`. Los caracteres no tienen significado a propósito, y la mayoría de las fuentes los incluyen todos.

## Notas de implementación

La compresión sigue el orden de abajo. Este capítulo explica por qué cada etapa se construyó así, en qué argumentos matemáticos y de ingeniería se apoya, y qué se ganó y qué se dejó de lado.

```
foto → reducir a la cuadrícula (64–512 px) → YCbCr → DCT 2D de toda la imagen → cuantización → codificación aritmética binaria adaptativa al contexto → texto hangul o ASCII
```

### Objetivos de diseño

Un comentario de DCInside no puede superar los 400 caracteres, no admite imágenes adjuntas y no muestra los enlaces `data:` como imágenes. En estas condiciones, la única forma de transmitir una foto es escribir la foto misma como texto. Por eso se diseñaron a la vez un formato con pérdida que cabe en una línea de comentario y una página de navegador que lo crea y lo decodifica. La compresión sigue la cadena clásica de codificación por transformada, cuantización, codificación entrópica y conversión a texto, pero cada etapa se ha vuelto a ajustar para un presupuesto de apenas unos cientos de bytes.

El presupuesto empieza por cuánto puede transportar un carácter: `log2(tamaño del alfabeto)` bits. base64url transporta `log2 64 = 6` bits, y las 8192 sílabas hangul que se usan aquí, `log2 8192 = 13` bits. Los 381 caracteres que quedan tras los 19 de la dirección, a 13 bits cada uno, dan 4953 bits, unos 619 bytes, para toda la foto. Según el teorema de codificación de fuente de Shannon [1], la longitud media del código no puede bajar de la entropía del modelo. Una imagen reconocible solo cabe en este presupuesto si hay a la vez un modelo que describa las fotos con poca entropía (una transformada y un modelo de contexto) y un codificador que se acerque a esa entropía (codificación aritmética).

Se sopesaron otras vías. Un enlace a un alojamiento de imágenes es corto, pero depende de una subida y de la vida de un servidor, lo que choca con el objetivo de un código completo por sí mismo. Un enlace JPEG `data:` [2] gasta unos 600 bytes solo en tablas de cuantización y de Huffman, más de 800 caracteres en base64 [3][4]. BlurHash [5] y ThumbHash [6] están pensados para marcadores de unas pocas decenas de caracteres con un número fijo de componentes (BlurHash admite como máximo 9 por eje), así que no pueden aprovechar 400 caracteres. De ahí un códec propio cuya calidad sigue creciendo al crecer el presupuesto.

El diseño no tiene coste de servidor ni riesgo de privacidad, y mientras se copie el texto, se decodifica en la misma foto en cualquier sitio. A cambio, el techo de información es firme: con 400 caracteres el resultado no pasa de una vista previa borrosa.

### Preprocesado

El preprocesado reduce la foto al tamaño de la cuadrícula, la compone sobre blanco y la convierte a YCbCr. Ajustar la transformada al presupuesto mantiene bajo el cálculo, y separar brillo y color permite tratar el color de forma más gruesa. La conversión de color usa los coeficientes BT.601 de rango completo de JFIF (`Y = 0.299R + 0.587G + 0.114B`, etc.) [7][8]. Las zonas transparentes se rellenan de blanco.

La reducción está sujeta a la teoría del muestreo. Según el teorema de muestreo [9], reducir por un factor k pliega dentro de la imagen, como aliasing, las frecuencias superiores a `1/(2k)` de la resolución original. El filtro bilineal del navegador solo mira un entorno de 2×2 píxeles, así que una reducción grande de una vez equivale a muestrear puntos y no evita el problema. Por eso la foto se divide a la mitad en un canvas hasta quedar entre 2 y 4 veces el objetivo y luego se dibuja una última vez al tamaño de la cuadrícula [10]. Cada paso se acerca a una media de 2×2, con el mismo efecto paso bajo que una pirámide de mipmaps [11].

La proporción se guarda en 6 bits como `a = round(16 · log2(ancho/alto))`. Con 16 pasos por octava, el error de proporción es como mucho `2^(1/32) ≈ 2,2 %`, y ±31 pasos cubren hasta unos 3,83:1. Una cuadrícula de 64 píxeles tiene 64 × 48 = 3072 coeficientes por canal, suficientes para los pocos cientos distintos de cero que caben en un presupuesto de 400 caracteres, con una transformada que termina en menos de un milisegundo.

El color se separa porque el ojo es menos sensible al contraste de color que al de brillo [12]. Comprimir en RGB gasta bits en información repetida en los tres canales. En lugar de reducir el color en el espacio al estilo 4:2:0, los canales de color usan un paso de cuantización 1,7 veces mayor en la misma cuadrícula. Con una DCT de toda la imagen, los coeficientes de color de alta frecuencia acaban en cero de todos modos, así que el efecto es parecido y la implementación, más simple.

Así, fotos de cualquier tamaño siguen el mismo camino y los escalones de la reducción no se cuelan en el resultado. La reducción en canvas sí varía un poco entre navegadores, de modo que la misma foto puede dar códigos algo distintos en navegadores distintos; las fotos más alargadas que 3,8:1 se aplastan a esa proporción, y la transparencia se pierde en blanco.

### Transformada

Los píxeles vecinos de una imagen natural están muy correlacionados. Tras una DCT, la energía se reúne en unos pocos coeficientes de baja frecuencia, y el resto puede ponerse a cero con poca pérdida visible. Se sabe que la DCT se aproxima a la transformada óptima (KLT) para un proceso de Markov de primer orden y que compacta bien la energía [13][14]. Este códec aplica una única DCT-II 2D a toda la cuadrícula con la normalización siguiente.

```math
A[u,v] = \frac{\alpha_u \alpha_v}{WH} \sum_{x=0}^{W-1}\sum_{y=0}^{H-1} f(x,y)\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H},\quad \alpha_0=1,\ \alpha_{k>0}=2
```

```math
f(x,y) = \sum_{u,v} A[u,v]\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H}
```

Con esta normalización, `A[0,0]` es el brillo medio y cada otro coeficiente es la amplitud de su onda coseno, de modo que los pasos de cuantización se fijan directamente en unidades de brillo. La transformada se calcula de forma separable, primero en horizontal y luego en vertical, con un coste de `O(W·H·(W+H))`: unas 340 000 multiplicaciones para una cuadrícula de 64 × 48 y unos 270 millones para 512 × 512.

Los coeficientes se ordenan por `r = √((u·s/W)² + (v·s/H)²)` creciente, con `s = √(W·H)`. Usar una frecuencia normalizada a un cuadrado pone la misma frecuencia visible en la misma posición en fotos anchas y altas. El orden se guarda por tamaño de cuadrícula, y los empates en r se resuelven con los enteros v y u para que navegadores y Node.js obtengan el mismo orden.

Hay una razón para no dividir en bloques de 8×8 como JPEG [3]. Con un presupuesto pequeño solo sobrevive el DC de cada bloque y aparece un tablero de ajedrez. Los motivos originales de los bloques, el cálculo y la adaptación local, pesan poco en cuadrículas de 64–512. Las wavelets pueden conservar mejor los bordes con el mismo presupuesto, pero complican la implementación y el modelo de contexto. Una DCT de toda la imagen se difumina con suavidad y sin bloques aunque solo queden frecuencias bajas, que es el tipo de degradación adecuado para un presupuesto mínimo.

El precio es que, al ser global, la textura fina de una zona compite por el presupuesto con toda la imagen. Una serie de cosenos truncada sobrepasa en torno a un 9 % cerca de los bordes (fenómeno de Gibbs) [15] y deja ondulaciones. Todavía no se usa una DCT rápida basada en FFT [16], así que en una cuadrícula de 512 la transformada sola tarda casi un segundo.

### Cuantización

La mayor parte de la compresión se decide en la cuantización: cuanto más se descarta lo menos visible, mejor se ve el resultado con la misma longitud. El paso y la regla de redondeo son:

```math
\mathrm{step} = 0.0002 \cdot 1.06^{q} \cdot (1 + 0.11\,r) \cdot c_{\mathrm{chroma}} \cdot s_{\mathrm{grid}},\qquad \mathrm{level} = \mathrm{sign}(x)\left\lfloor \frac{|x|}{\mathrm{step}} + 0.5 - 0.1 \right\rfloor
```

Aquí `c = 1,7` en los canales de color y `s = 64/lado largo` en las cuadrículas de 96 píxeles o más; la reconstrucción es `level · step`.

Se sabe que los coeficientes AC de la DCT siguen aproximadamente una distribución de Laplace [17], y para fuentes laplacianas un cuantizador uniforme con el intervalo del cero ensanchado (zona muerta) es favorable en términos de tasa-distorsión [18]. En la regla anterior, un coeficiente pasa a cero por debajo de `0.6 · step`. El peso de frecuencia `1 + 0.11r` es una aproximación lineal de la función de sensibilidad al contraste [19], y el factor 1,7 del color refleja la menor sensibilidad al contraste cromático [12].

q es exponencial para repartir con 7 bits un rango de `1.06^127 ≈ 1620` veces. Cada paso cambia el tamaño del paso un 6 %, así que la tasa se ajusta con suavidad y el ajuste al límite de caracteres resulta sencillo. La primera versión usaba una base de 0,004, un factor de 1,09 y un q de 6 bits; era tan gruesa que incluso el paso más fino solo usaba 110–190 de 360 caracteres, así que el rango se amplió al actual.

Los pesos se eligieron cambiando un valor cada vez y comparando el PSNR medio de las ocho fotos de ajuste (media de 360 caracteres ASCII, 360 hangul y 150 ASCII). La referencia (beta 0,11, chroma 1,7, dz 0,22, shrink 0,08) daba 23,165 dB.

| Parámetro | Valor y PSNR medio |
|---|---|
| beta (peso de frecuencia) | 0,05 → 23,183; 0,08 → 23,180; 0,15 → 23,152; 0,2 → 23,106 |
| chroma (factor de color) | 1,3 → 23,125; 2,2 → 23,153; 3 → 23,119 |
| dz (zona muerta) | 0,1 → 23,253; 0,3 → 23,076; 0,4 → 22,965 |
| shrink (acercar a cero) | 0 → 23,213; 0,15 → 23,102 |

dz y shrink, con diferencias claras, pasaron a 0,1 y 0; beta y chroma, con diferencias de menos de 0,02 dB, se mantuvieron. Acercar a cero los valores reconstruidos es mejor en teoría para fuentes laplacianas, pero no desplazarlos dio la mejor medida.

El factor de cuadrícula `s = 64/lado largo` sale de esta relación. Una textura con varianza por píxel σ² tiene coeficientes de varianza σ² con una DCT ortonormal. Los coeficientes de amplitud usados aquí son los ortonormales multiplicados por `2/√(WH)`, así que su desviación típica se reduce como `1/√(WH)`, que con proporción fija es proporcional a `1/lado largo`. Multiplicar el paso por `64/lado largo` hace que el mismo q signifique la misma precisión relativa en cualquier cuadrícula. Sin este factor, la textura fina sigue redondeándose a cero en cuadrículas grandes: la foto del oso de la imagen de ejemplo se quedaba en 1303 caracteres (26,1 dB) incluso con la cuadrícula de 512 píxeles, mientras que con el factor un límite de 7000 caracteres usa 6877 y llega a 28,8 dB.

Las tablas de cuantización 8×8 al estilo JPEG están atadas al tamaño de bloque y no encajan con una DCT global ni con una cuadrícula variable. Los pesos se eligieron por PSNR con las fotos de prueba y pueden diferir del óptimo para la calidad percibida. En la cuadrícula de 64 píxeles, el DC usa 6 bits para el brillo y 5 para el color, así que el color medio puede desviarse hasta unos 2/255 (brillo) y 4/255 (color); las cuadrículas de 96 píxeles o más lo suben a 8 bits.

### Codificación entrópica

La mayoría de los coeficientes cuantizados son cero, y la probabilidad de que lo sean varía mucho según su frecuencia y sus vecinos. Solo cuando esa variación se refleja en las probabilidades se acerca la longitud a la entropía. Por eso los enteros no se escriben directamente, sino con un codificador aritmético binario que usa probabilidades según el contexto.

Un suceso de probabilidad p cuesta `−log2 p` bits, y la codificación aritmética deja el total solo unos bytes por encima de esa suma [20]. Huffman gasta al menos un bit por símbolo, incluso en un «cero» con probabilidad 0,95, donde la codificación aritmética gasta unos 0,074 bits. Los bytes sobrantes del final se eliminan con el recorte de caracteres finales descrito en [Control de tasa y elección de cuadrícula](#control-de-tasa-y-elección-de-cuadrícula).

Cada coeficiente se divide en decisiones binarias en este orden: cero o no, signo, mayor que 1, mayor que 2 y Exp-Golomb, la misma binarización que CABAC en H.264 [21][22][23]. Cada decisión se codifica con la probabilidad de un contexto definido por el canal, la banda de frecuencia y si los dos coeficientes anteriores eran cero. El codificador es un range coder [24] implementado según la estructura del SDK de LZMA, con un rango de 32 bits y acarreo mediante un byte de caché [25].

Las probabilidades son enteros de 11 bits que se actualizan en cada bit como `p ← p + (2048 − p)/16` con 0 y `p ← p − p/16` con 1. Esta media móvil exponencial tiene una memoria efectiva de unos 16 bits, así que se adapta rápido incluso en un flujo de unos cientos de bytes. En las seis fotos de validación, las velocidades de actualización 1/8, 1/16 y 1/32 dieron 25,372, 25,385 y 25,344 dB de media, así que se eligió 1/16. El modelo se mantiene pequeño, con 216 contextos, porque con pocos datos demasiados contextos harían terminar el flujo antes de que cada uno aprendiera algo. Unas probabilidades iniciales entrenadas con las fotos de prueba solo ganaron 0,03–0,07 dB, demasiado poco para justificar incluir una tabla en el código. La cabecera pasa por el mismo codificador con probabilidad 0,5 y cuesta exactamente un bit por bit, sin código aparte para empaquetar bits.

Escrita así, la misma información ocupa muchos menos bits que con códigos de longitud fija, de modo que un código de 400 caracteres contiene cientos de coeficientes. Quedan puntos débiles: el signo sigue costando casi un bit entero, no se usan vecinos bidimensionales como el coeficiente de arriba, no se usan modelos más potentes como la mezcla de contextos, y un daño a mitad del flujo estropea todo lo que viene después.

### Flujo de bits

El decodificador debe reconstruir la misma cuadrícula, el mismo orden de coeficientes y los mismos pasos solo a partir del código. Por eso el código empieza con una cabecera, seguida de los coeficientes de Y, Cb y Cr. La cabecera pasa por el mismo codificador con probabilidad 0,5. La cuadrícula de 64 píxeles se escribe en el formato 0.

| Campo | Bits | Valor |
|---|---|---|
| Formato | 1 | 0 |
| Código de proporción | 6 | a + 32 |
| Paso de compresión q | 7 | 0–127 |
| DC de Y | 6 | brillo medio × 63 |
| DC de Cb y Cr | 5, 5 | (media + 0,5) × 31 |
| Número de coeficientes de Y, Cb y Cr | 12, 10, 10 | 0–4095, 0–1023 |

Las cuadrículas de 96 píxeles o más se escriben en el formato 1: tras el bit de formato va un índice de cuadrícula de 3 bits (0–6 para 64, 96, 128, 192, 256, 384 y 512 píxeles), los valores DC ocupan 8 bits cada uno y los recuentos de coeficientes, `⌈log2(W·H)⌉` bits cada uno.

La cabecera del formato 0 ocupa 62 bits, el 1,2 % de un presupuesto de 400 caracteres de unos 5200 bits. Guardar el número de coeficientes en la cabecera en lugar de un marcador de fin de bloque cuesta 10–16 bits fijos por canal, sin preguntar en cada coeficiente si el bloque termina ahí [21]. Al añadir el formato de cuadrícula creciente, la cuadrícula de 64 píxeles se quedó en el formato 0, así que los códigos ya en circulación y los de 400 caracteres no cambiaron ni un bit (comprobado con 16 códigos de las fotos de prueba). Un campo de formato de varios bits desde el principio habría sido más limpio, pero con 400 caracteres cada bit cuenta, así que empezó con un bit. Como ya se usan los dos valores, la próxima ampliación usará como marca el índice de cuadrícula 7, que está libre. El punto débil que queda es la falta de una suma de comprobación: un código cortado o alterado no se detecta con seguridad y puede decodificarse en una imagen equivocada.

### Conversión a texto

Los comentarios se limitan por número de caracteres, así que cada carácter debe llevar el máximo de bits posible. Los bytes se unen en una cadena de bits y se cortan en grupos de 13 bits escritos como la sílaba hangul `U+AC00 + valor`, o en grupos de 6 bits escritos como caracteres base64url [26]. Al leer se ignoran los caracteres fuera de esos rangos.

Unicode tiene 11 172 sílabas hangul en `U+AC00–U+D7A3` [27]. Usarlas todas permitiría llevar `log2 11172 ≈ 13,447` bits, un 3,4 % más que 13. Sin embargo, eso exigiría tratar todo el código como un número grande y cambiarlo de base, un cálculo `O(n²)`, mientras que con una potencia de dos, 8192 caracteres, basta con cortar y unir bits. Los bits sobrantes del último carácter se rellenan con ceros, lo que encaja con la regla del decodificador de leer como cero los bytes que faltan y hace posible el recorte de caracteres finales.

Se consideraron otros alfabetos. base83 (BlurHash [5]) o base85 llevan algo más de 6 bits, pero incluyen caracteres especiales de las direcciones como `#`, `%` y `?`. Los emojis o los caracteres han podrían llevar más bits, pero su visualización y escritura son poco fiables. El hangul es natural en los comentarios de DCInside, está en casi todas las fuentes y guarda unas 2,2 veces más información que el ASCII en los mismos 400 caracteres.

Ahora bien, donde la longitud se cuenta en bytes UTF-8, el hangul lleva 4,33 bits por byte, peor que los 6 del ASCII. Además, una normalización que descomponga las sílabas en jamo (NFD) [27] rompe el código, y el decodificador no convierte ahora el texto pegado de vuelta a NFC. Y como el enlazado automático de los comentarios de DCInside solo abarca direcciones en ASCII, un código hangul solo puede transmitirse copiándolo y pegándolo, no como enlace. Por eso las letras del código por defecto de la página son ASCII.

### Control de tasa y elección de cuadrícula

Solo se fija un límite de caracteres, y dentro de él la calidad debe ser la mejor posible. Para ello se eligen a la vez el paso de compresión q y el tamaño de la cuadrícula.

La longitud del código R(q) decrece casi monótonamente con q, así que una búsqueda binaria entre 0 y 127 encuentra el menor q con `R(q) ≤ presupuesto` en unas `⌈log2 128⌉ = 7` codificaciones. Después se quitan caracteres finales de uno en uno mientras el resultado decodificado siga igual. Así desaparecen los bytes sobrantes que el range coder emite al final, y como se comprueba decodificando de verdad, no puede fallar [20].

Decidir si una tasa dada se gasta mejor en resolución o en precisión de cuantización es un problema clásico de optimización tasa-distorsión [28]. Aquí se resuelve con una regla voraz. La foto se codifica primero en la cuadrícula de 64 píxeles. Si incluso con q = 0 sobran caracteres, esa cuadrícula no puede llenar el presupuesto, así que la foto se vuelve a codificar en el siguiente tamaño. Los dos resultados se comparan por error cuadrático medio frente al original al tamaño de la cuadrícula mayor y se queda el mejor; la búsqueda se detiene cuando la cuadrícula mayor es peor o ya no llega a q = 0. Una cuadrícula que cabe con q = 0 se salta la búsqueda binaria, lo que bajó la compresión de 7000 caracteres de la foto del oso de 3,5 s a 2,9 s.

Codificar todas las cuadrículas y compararlas podría dar mejores resultados, pero las grandes tardan segundos cada una. También se probó una cuadrícula de 768 píxeles: comprimir la foto del oso con un límite de 20 000 caracteres usó 13 003 y llegó a 29,9 dB, pero tardó 7,5 s. Eso supera en solo 0,7 dB lo máximo que da la cuadrícula de 512 píxeles (8526 caracteres, 29,2 dB, 3,2 s), así que la cuadrícula se quedó en 512 píxeles. El resultado es que una compresión de 400 caracteres termina en 0,01 s, mientras que un límite de 7000 caracteres usa 6877 y llega a 28,8 dB (Node.js 22). R(q) no es estrictamente monótona, así que pueden sobrar algunos caracteres; la regla voraz no garantiza la cuadrícula óptima; y el cálculo se ejecuta en el hilo principal, de modo que la página se detiene un momento con presupuestos grandes.

### Decodificación y ampliación

Construir una imagen del tamaño de la cuadrícula y estirarla añadiría escalones de píxeles o el desenfoque de la interpolación, así que el decodificador calcula directamente desde los coeficientes la imagen al tamaño de la pantalla. Las coordenadas de pantalla, pasadas a unidades de cuadrícula, se sustituyen en x e y de la transformada inversa y la serie de cosenos se suma tal cual. La DCT inversa es una función suave definida también para x no enteros, así que muestrearla a cualquier resolución equivale a una interpolación de banda limitada en la base de cosenos.

Primero se calculan las filas, y se saltan las filas cuyos coeficientes son todos cero, con un coste de `O(H·W·outW + filas·outH·outW)`. La misma función dibuja las vistas previas de las tarjetas (480 px en el lado largo) y la pestaña Ver (960 px), y termina con una conversión de YCbCr a RGB recortada a 0–255. Una DCT inversa al tamaño de la cuadrícula seguida del escalado del canvas sería más rápida, pero en cuadrículas pequeñas deja los patrones en cruz borrosos típicos de la interpolación bilineal.

Este enfoque convierte muy poca información en una imagen agradable, difuminada con suavidad. Las ondulaciones de Gibbs [15] siguen viéndose al ampliar, y como `Math.cos` es, según la especificación, una aproximación de la implementación [29], los valores de píxel pueden diferir en ±1 entre navegadores.

### Direcciones y transporte

El enlace por defecto tiene la forma `https://g.yeon.at/?código`. DCInside solo convierte en enlace pulsable las direcciones en ASCII que empiezan por `https://`, y el alojamiento estático ignora todo lo que va después de `?` y sirve el mismo `index.html`. Así, cada dirección puede abrir una foto distinta sin configurar el servidor. Al cargarse, la página lee `location.search`, o `location.hash` si la consulta está vacía, para obtener el código.

En la sintaxis URI [30], la consulta tras `?` se envía al servidor, mientras que el fragmento tras `#` se queda en el navegador. Una sílaba hangul ocupa 3 bytes en UTF-8, y en una dirección cada byte se convierte en los tres caracteres `%XX`, así que una sílaba ocupa 9 bytes. Un código de 381 caracteres ocupa entonces 381 × 9 = 3429 bytes más los 19 de la dirección. Un servidor HTTP puede responder 414 a un destino demasiado largo [31], y las peticiones directas a g.yeon.at se aceptaron con 6000 bytes y se rechazaron con 414 con 9000. Para dejar margen al resto de la línea de petición, Copiar cambia `?` por `#` cuando la dirección codificada como URL pasa de 7000 bytes, lo que equivale a `(7000 − 19) / 9 ≈ 775` sílabas.

Usar siempre `#` eliminaría cualquier preocupación por el 414, pero `?` es la forma por defecto y basta para los enlaces cortos. Por eso los enlaces de 400 caracteres mantienen la forma con `?`, mientras que los de miles de caracteres pasan a `#` y se abren sin errores; lo que va después de `#` tampoco llega a los registros del servidor. El límite del 414 es una medición del alojamiento actual y puede cambiar, y como los comentarios de DCInside dejan fuera del enlace el hangul y lo que va después de `#`, allí los enlaces de miles de caracteres que pasan a `#` y los códigos hangul solo pueden transmitirse pegándolos.

### Diseño de la página

La compresión y la decodificación ocurren en el navegador, así que el servidor solo tiene que entregar un archivo. Por eso se eligió el alojamiento estático en GitHub Pages con el dominio propio `g.yeon.at` [32]. Una función sin servidor o un servidor propio podrían ocuparse de la restricción de dirección o de estadísticas de uso en el servidor, a costa de gastos de funcionamiento y de la responsabilidad de tratar datos; el alojamiento estático no tiene coste de funcionamiento y tiene un único punto de fallo.

La página es un solo archivo HTML con todo el JS y el CSS dentro. Quien la recibe debe poder abrirla sin preparar nada, y publicar debería consistir en copiar un archivo. El códec se inserta en la página al compilar y, salvo las fuentes, la página funciona sin conexión. Un framework acortaría el código de la interfaz, pero agrandaría el archivo y añadiría dependencias externas.

Para encontrar el código en un texto pegado, se elige el tramo más largo de al menos cuatro caracteres hangul o, si no lo hay, el tramo base64url más largo después del último `?` o `#` [30]. El código se lee aunque haya otras palabras alrededor, pero una frase en hangul más larga que el código puede elegirse por error. Todos los accesos a localStorage están protegidos para que la página vuelva a los valores por defecto si el almacenamiento está bloqueado, y si se rechaza la API del portapapeles, copiar recurre a seleccionar el texto. El canvas y las imágenes siguen el estándar HTML, según el cual una imagen animada dibuja su fotograma por defecto [10].

La restricción de dirección hace que la página solo dibuje un aviso y se detenga cuando se abre fuera de g.yeon.at, localhost, archivos locales y el entorno de vista previa. Una restricción fiable corresponde a un servidor, pero el alojamiento estático no tiene código de servidor, así que esta comprobación queda como disuasión y no detiene a quien modifique el código fuente.

### Evaluación de la calidad

Elegir parámetros y comprobar que un cambio no rompió resultados anteriores exige números reproducibles. El original se reduce con filtro de caja y se compara por PSNR con el resultado decodificado dibujado al mismo tamaño. Los resultados de 400 caracteres se midieron a 160×120, cerca del tamaño que ve quien los recibe, y el efecto del límite de caracteres a 640×480. Las frecuencias altas del original no caben en 400 caracteres de todos modos, así que solo una comparación cerca del tamaño de visualización revela las diferencias entre códecs. El PSNR, un logaritmo del error cuadrático medio, es sencillo de calcular y fácil de reproducir.

Las fotos de prueba son fotos de Unsplash de picsum.photos: ocho para ajustar parámetros y seis para validar los resultados [33]. Tras cada cambio del códec se comprobó que los resultados con 400 caracteres, tanto el PSNR medio como las cadenas de código, coincidían con los anteriores, para mantener la compatibilidad. Para la comparación con enlaces JPG `data:` se usó Pillow (libjpeg), probando distintos tamaños y calidades y quedándose con el mejor resultado que cabe en el número de caracteres. Las pruebas en navegador abren la página real con Playwright y comprueban la creación de códigos, la apertura de enlaces, el pegado y la restricción de dirección.

Las métricas perceptivas como SSIM se ajustan mejor al juicio humano [34], pero se eligió el PSNR como punto de partida simple y fácil de interpretar. Catorce fotos son una muestra pequeña, y el PSNR puede no coincidir con la calidad percibida [34][35].

### Las matemáticas del PSNR

El PSNR se define como

```math
\mathrm{PSNR} = 10 \log_{10} \frac{255^2}{\mathrm{MSE}}
```

así que un MSE 10 veces mayor baja el PSNR exactamente 10 dB, y MSE = 0 da infinito por definición. En una imagen de 4 píxeles (rojo, amarillo, azul, verde), un solo canal con 1 de diferencia da `MSE = 1/12` y por tanto `PSNR = 10·log10(12 · 255²) ≈ 58,9 dB`.

La distribución del PSNR sobre todas las imágenes posibles parte de una observación. Todos los valores de canal de la imagen de 4 píxeles son 0 o 255, así que si los de la otra imagen recorren 0–255, la diferencia d de cada canal es uniforme en 0–255, y la distribución de la suma de errores al cuadrado es la convolución de 12 veces la distribución de d². La primera versión usó convolución por FFT [16], cuyo error relativo de unos 10⁻¹⁶ borró las probabilidades de la cola por debajo de 10⁻²⁰. Además, `np.histogram` obtiene los valores de cada intervalo como diferencias de una suma acumulada, lo que provoca cancelación al restar números diminutos de valores cercanos a 1 [36]. Solo al pasar a una convolución directa, que suma únicamente términos positivos, y a `np.bincount` se obtuvo con exactitud la cola hasta 58,9 dB. Resulta que la mediana del PSNR de una imagen coloreada al azar es 4,8 dB, y una imagen idéntica es un caso entre unos 7,9 × 10²⁸.

La tabla de impresiones de PSNR de [Calidad](#calidad) es una guía empírica de este proyecto, no un estándar de la literatura. El PSNR puede no coincidir con la calidad que percibe la gente [35].

## Referencias

1. C. E. Shannon, "A Mathematical Theory of Communication," *Bell System Technical Journal*, vol. 27, pp. 379–423, 623–656, 1948.
2. L. Masinter, *RFC 2397: The "data" URL scheme*, IETF, 1998.
3. G. K. Wallace, "The JPEG still picture compression standard," *IEEE Transactions on Consumer Electronics*, vol. 38, no. 1, pp. xviii–xxxiv, 1992. doi:10.1109/30.125072
4. ITU-T Recommendation T.81 | ISO/IEC 10918-1, *Digital compression and coding of continuous-tone still images – Requirements and guidelines*, 1992.
5. Wolt, *BlurHash*. https://github.com/woltapp/blurhash
6. E. Wallace, *ThumbHash*, 2023. https://github.com/evanw/thumbhash
7. E. Hamilton, *JPEG File Interchange Format, Version 1.02*, C-Cube Microsystems, 1992. (estandarizado como ITU-T T.871)
8. ITU-R Recommendation BT.601-7, *Studio encoding parameters of digital television for standard 4:3 and wide-screen 16:9 aspect ratios*, 2011.
9. C. E. Shannon, "Communication in the presence of noise," *Proceedings of the IRE*, vol. 37, no. 1, pp. 10–21, 1949. doi:10.1109/JRPROC.1949.232969 (teorema de muestreo)
10. WHATWG, *HTML Living Standard*, the canvas 2D context (`drawImage`, `imageSmoothingQuality`). https://html.spec.whatwg.org/multipage/canvas.html
11. L. Williams, "Pyramidal parametrics," *Computer Graphics (SIGGRAPH '83)*, vol. 17, no. 3, pp. 1–11, 1983. doi:10.1145/964967.801126
12. K. T. Mullen, "The contrast sensitivity of human colour vision to red-green and blue-yellow chromatic gratings," *The Journal of Physiology*, vol. 359, pp. 381–400, 1985. doi:10.1113/jphysiol.1985.sp015591
13. N. Ahmed, T. Natarajan, K. R. Rao, "Discrete Cosine Transform," *IEEE Transactions on Computers*, vol. C-23, no. 1, pp. 90–93, 1974. doi:10.1109/T-C.1974.223784
14. K. R. Rao, P. Yip, *Discrete Cosine Transform: Algorithms, Advantages, Applications*, Academic Press, 1990.
15. J. W. Gibbs, "Fourier's Series," *Nature*, vol. 59, p. 606, 1899. doi:10.1038/059606a0; E. Hewitt, R. E. Hewitt, "The Gibbs-Wilbraham phenomenon: An episode in Fourier analysis," *Archive for History of Exact Sciences*, vol. 21, pp. 129–160, 1979. doi:10.1007/BF00330404
16. J. W. Cooley, J. W. Tukey, "An algorithm for the machine calculation of complex Fourier series," *Mathematics of Computation*, vol. 19, no. 90, pp. 297–301, 1965. doi:10.1090/S0025-5718-1965-0178586-1
17. E. Y. Lam, J. W. Goodman, "A mathematical analysis of the DCT coefficient distributions for images," *IEEE Transactions on Image Processing*, vol. 9, no. 10, pp. 1661–1666, 2000. doi:10.1109/83.869177
18. G. J. Sullivan, "Efficient scalar quantization of exponential and Laplacian random variables," *IEEE Transactions on Information Theory*, vol. 42, no. 5, pp. 1365–1374, 1996. doi:10.1109/18.532878
19. J. L. Mannos, D. J. Sakrison, "The effects of a visual fidelity criterion on the encoding of images," *IEEE Transactions on Information Theory*, vol. 20, no. 4, pp. 525–536, 1974. doi:10.1109/TIT.1974.1055250
20. I. H. Witten, R. M. Neal, J. G. Cleary, "Arithmetic coding for data compression," *Communications of the ACM*, vol. 30, no. 6, pp. 520–540, 1987. doi:10.1145/214762.214771
21. D. Marpe, H. Schwarz, T. Wiegand, "Context-based adaptive binary arithmetic coding in the H.264/AVC video compression standard," *IEEE Transactions on Circuits and Systems for Video Technology*, vol. 13, no. 7, pp. 620–636, 2003. doi:10.1109/TCSVT.2003.815173
22. J. Teuhola, "A compression method for clustered bit-vectors," *Information Processing Letters*, vol. 7, no. 6, pp. 308–311, 1978. doi:10.1016/0020-0190(78)90024-8 (códigos Exp-Golomb)
23. S. W. Golomb, "Run-length encodings," *IEEE Transactions on Information Theory*, vol. 12, no. 3, pp. 399–401, 1966. doi:10.1109/TIT.1966.1053907
24. G. N. N. Martin, "Range encoding: an algorithm for removing redundancy from a digitised message," Video & Data Recording Conference, Southampton, UK, 1979.
25. I. Pavlov, *LZMA SDK*. https://www.7-zip.org/sdk.html (range coder: `LzmaEnc.c`, `LzmaDec.c`)
26. S. Josefsson, *RFC 4648: The Base16, Base32, and Base64 Data Encodings*, IETF, 2006. (sección 5, base64url)
27. The Unicode Consortium, *The Unicode Standard*, Chapter 3, "Conjoining Jamo Behavior," and *Unicode Standard Annex #15: Unicode Normalization Forms*. https://www.unicode.org/ (sílabas hangul U+AC00–U+D7A3, normalización)
28. A. Ortega, K. Ramchandran, "Rate-distortion methods for image and video compression," *IEEE Signal Processing Magazine*, vol. 15, no. 6, pp. 23–50, 1998. doi:10.1109/79.733495
29. Ecma International, *ECMA-262: ECMAScript Language Specification*. https://tc39.es/ecma262/ (`Math.cos` es una aproximación de la implementación)
30. T. Berners-Lee, R. Fielding, L. Masinter, *RFC 3986: Uniform Resource Identifier (URI): Generic Syntax*, IETF, 2005. (consulta, fragmento y codificación por porcentajes)
31. R. Fielding, M. Nottingham, J. Reschke, *RFC 9110: HTTP Semantics*, IETF, 2022. (sección 15.5.15, 414 URI Too Long)
32. GitHub Docs, "Managing a custom domain for your GitHub Pages site." https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site
33. Picsum Photos. https://picsum.photos/ (fotos de Unsplash. Conjunto de ajuste: ids 1015, 1025, 1062, 1084, 177, 237, 433, 64; conjunto de validación: ids 1003, 1035, 1074, 200, 287, 334. El oso de la imagen de ejemplo es la id 433 (Thomas Lefebvre, https://unsplash.com/photos/aRXPJnXQ9lU))
34. Z. Wang, A. C. Bovik, "Mean squared error: Love it or leave it? A new look at signal fidelity measures," *IEEE Signal Processing Magazine*, vol. 26, no. 1, pp. 98–117, 2009. doi:10.1109/MSP.2008.930649
35. Q. Huynh-Thu, M. Ghanbari, "Scope of validity of PSNR in image/video quality assessment," *Electronics Letters*, vol. 44, no. 13, pp. 800–801, 2008. doi:10.1049/el:20080522
36. D. Goldberg, "What every computer scientist should know about floating-point arithmetic," *ACM Computing Surveys*, vol. 23, no. 1, pp. 5–48, 1991. doi:10.1145/103162.103163

### Mediciones

- Límite de 400 caracteres en los comentarios de DCInside, imágenes y enlaces `data:` que no se muestran, enlazado automático solo con `https://`, y que los códigos ASCII tras `?` entran en el enlace mientras que los códigos hangul y lo que va después de `#` no: comprobado en un cuadro de comentarios real.
- Límite del 414 en g.yeon.at: el 2026-10-26, peticiones con consultas de 1000 a 60 000 bytes devolvieron 200 con 6000 bytes y 414 con 9000.
- Cifras de calidad y velocidad: medidas con Node.js 22.
