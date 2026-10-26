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
| Letras del código: hangul | 13 bits por carácter, unos 630 bytes en 400 caracteres. La opción más nítida. |
| Letras del código: ASCII | Solo letras, dígitos, `-` y `_`, 6 bits por carácter. Para sitios que no aceptan hangul. |
| Dirección delante | Por defecto `https://g.yeon.at/?` (19 caracteres); el `https://` hace que se convierta en un enlace pulsable en los comentarios, que se descuentan del espacio para la foto. Déjala vacía para generar solo el código. |
| Límite de caracteres | Por defecto 400 (un comentario de DCInside). |

## Publicar en g.yeon.at (GitHub Pages + dominio propio)

1. Crea un repositorio público en GitHub y sube `index.html` y `CNAME` de esta carpeta. `CNAME` contiene la línea `g.yeon.at`.
2. En **Settings → Pages**, elige Source `Deploy from a branch` y Branch `main` / `/(root)`.
3. En la configuración DNS de `yeon.at`, añade un registro `CNAME` con nombre `img` y valor `tu-usuario.github.io`.
4. Cuando el DNS se propague, **Settings → Pages** mostrará `g.yeon.at` como dominio propio. Activa **Enforce HTTPS**.

GitHub Pages ignora todo lo que va después de `?` y sirve `index.html`, así que no hace falta configurar nada en el servidor. La página lee lo que va después de `?` y reconstruye la foto. El hangul se envía como `%EB%…` y la página lo convierte de nuevo en hangul.

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
