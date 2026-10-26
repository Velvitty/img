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

1. La foto se reduce a una cuadrícula de 64 píxeles en el lado largo y se convierte a YCbCr.
2. Se aplica una DCT 2D a toda la imagen y los coeficientes se ordenan de frecuencia baja a alta.
3. Las frecuencias altas y los canales de color (crominancia) se cuantizan de forma más gruesa.
4. Los niveles se comprimen con un codificador aritmético binario adaptativo al contexto (un range coder al estilo de LZMA).
5. Una búsqueda binaria encuentra el cuantizador más fino que cabe en el límite, y se recortan los caracteres finales mientras la decodificación no cambie.
6. El visor evalúa la serie de cosenos a la resolución de la pantalla, así que la imagen ampliada se ve suave y sin bloques.

## Pestaña de enlace JPG data

Donde el límite es grande (YouTube y Reddit permiten 10 000 caracteres), un JPG normal como enlace `data:` es más nítido. Solo la cabecera de un JPG ocupa varios cientos de caracteres, así que nunca cabe en 400.

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
- **Forma con `#`:** `https://g.yeon.at/#código` también funciona.

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
- **Subir el límite:** da un resultado más nítido, pero donde se permiten 1000 caracteres o más es mejor la pestaña de enlace JPG data.

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
foto → cuadrícula de 64 píxeles → YCbCr → DCT 2D de toda la imagen → cuantización → codificación aritmética → texto hangul o ASCII
```

1. **Reducir:** la foto se reduce a una cuadrícula con 64 píxeles en el lado largo. Se divide a la mitad varias veces para evitar bordes dentados, y las zonas transparentes se rellenan de blanco.
2. **Separar colores:** un canal de brillo (Y) y dos de diferencia de color (Cb, Cr). El ojo es más sensible al brillo, así que el color se trata de forma más gruesa.
3. **Frecuencias:** en lugar de los bloques 8×8 de JPEG, una sola DCT cubre toda la imagen. No hay bordes de bloque, y primero se guardan las frecuencias bajas (las formas grandes).
4. **Cuantización:** las frecuencias altas y los canales de color se redondean de forma más gruesa. El paso de compresión (0–127) fija el intervalo de redondeo.
5. **Codificación aritmética:** cada valor se divide en preguntas como «¿es cero?» y «¿es mayor que 1?», y se comprime con probabilidades que se adaptan a lo anterior (un range coder al estilo de LZMA).
6. **Escribir como texto:** los bytes se escriben de 13 en 13 bits como sílabas hangul, o de 6 en 6 bits como ASCII.
7. **Ajustar al límite:** una búsqueda binaria encuentra el paso de compresión más fino que cabe. Después se quitan caracteres finales mientras el código se siga decodificando igual.
8. **Decodificar:** el navegador de quien lo recibe suma las ondas coseno directamente a la resolución de la pantalla. Como no estira una imagen de 64 píxeles, la foto se difumina con suavidad a cualquier tamaño, sin escalones de píxeles.

Los primeros 62 bits de un código (unos 5 caracteres hangul) guardan la versión, la proporción, el paso de compresión, el color medio y el número de coeficientes. El formato a nivel de bits y los parámetros están en la [documentación detallada de img-dev](https://github.com/Velvitty/img-dev/blob/main/README.es.md#diseño-del-códec).

### Limitaciones

- **Calidad:** una vista previa borrosa. No se distinguen textos ni texturas finas.
- **Ondas:** aparecen ondulaciones junto a los bordes de brillo marcados, porque se recortaron frecuencias.
- **Techo:** la cuadrícula de 64 píxeles es un techo, así que a partir de cierto punto más caracteres no ayudan.
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
| La página da 404 | Revisa la configuración de Pages del repositorio, el archivo `CNAME` en la raíz y el CNAME del DNS (`g` → `velvitty.github.io`). La primera configuración tarda unos minutos en propagarse. |

### Preguntas frecuentes

**¿Se puede recuperar la foto original a partir de un código?**
No. Un código solo contiene la información de una vista previa borrosa.

**¿La misma foto da siempre el mismo código?**
En el mismo navegador, sí. Cada navegador reduce las imágenes de forma un poco distinta, así que el código puede variar ligeramente. Pero un código ya creado se decodifica en la misma foto en cualquier navegador.

**¿Y en sitios que permiten más de 400 caracteres?**
Subir el límite de caracteres da un resultado más nítido. Con 1000 caracteres o más, es mejor la pestaña de enlace JPG data.

**¿Por qué el código hangul es más nítido que el ASCII?**
Porque el mismo número de caracteres guarda unas 2,2 veces más información. Un carácter ASCII es uno entre 64 valores; uno hangul, uno entre 8192.

**El código hangul parece hecho de caracteres raros.**
Incluye sílabas poco usadas como `뗈` y `쯿`. Los caracteres no tienen significado a propósito, y la mayoría de las fuentes los incluyen todos.

### Material de desarrollo

El código fuente, el script de compilación, las fotos y scripts de prueba y el registro de experimentos están en [Velvitty/img-dev](https://github.com/Velvitty/img-dev). El `index.html` de aquí es una copia de la carpeta `dist/` que genera `python3 build.py` en img-dev.
