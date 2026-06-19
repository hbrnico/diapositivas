---
name: reveal_slides_creator
description: Creación, estructuración y mantenimiento de diapositivas interactivas RevealJS con UTN branding y compiladores embebidos para AEDD (C++) y PBA (Java).
---

# RevealJS Slides Creator (UTN FRSF)

Esta skill define el estándar de diseño, estructura e interactividad para las diapositivas de las cátedras **AEDD (Algoritmos y Estructuras de Datos)** y **PBA (Programación Básica)** de la UTN Regional Santa Fe.

---

## 1. Estructura de Directorios

El proyecto organiza las diapositivas de la siguiente manera:
- `/deploy/aedd/claseX/index.html` - Diapositivas de la materia AEDD (C++).
- `/deploy/pba/nombre_clase/index.html` - Diapositivas de la materia PBA (Java).
- `/deploy/dist/`, `/deploy/plugin/` - Dependencias centrales de RevealJS.

---

## 2. Lineamientos Estéticos y de Diseño (Wow Factor)

Evita layouts monótonos o diapositivas llenas de capturas de pantalla de código (PNGs).

### Tipografía y Colores
Usa fuentes modernas de Google Fonts y respeta la paleta institucional UTN:
- **Títulos**: Poppins (negrita, sin mayúsculas forzadas, color UTN Blue).
- **Cuerpo**: Inter (peso 400 o 500, color Slate oscuro).
- **Código**: Fira Code (monoespaciada).

```css
:root {
    --utn-blue: #002B5B;      /* Color institucional UTN */
    --utn-gold: #FFC107;      /* Acentos y bordes decorativos */
    --bg-neutral: #F8FAFC;    /* Fondo de contenedores y bloques */
    --text-main: #1E293B;     /* Texto principal legible */
    --text-light: #64748B;    /* Descripciones y subtítulos */
}
```

### Cero Imágenes Estáticas para Conceptos de Código
Los diagramas de memoria, arreglos unidimensionales o matrices deben representarse con **HTML y CSS puro** (flexbox o tablas estilizadas), garantizando escalabilidad sin pixelación.

```html
<div class="array-diagram">
    <div class="array-cell filled">8<div class="array-index">0</div></div>
    <div class="array-cell filled">3<div class="array-index">1</div></div>
    <div class="array-cell empty"><div class="array-index">2</div></div>
</div>
```

---

## 3. Arquitectura del Compilador Vertical (OneCompiler)

Para evitar saturar la presentación con pantallas de ejecución obligatorias, usamos **diapositivas verticales (nested slides)** de RevealJS.

- **Eje Horizontal (Flecha Derecha):** Muestra el flujo teórico y la resolución modularizada del código usando el resaltado línea por línea nativo (`data-line-numbers`).
- **Eje Vertical (Flecha Abajo 👇):** Despliega un iframe embebido con OneCompiler pre-cargado con el código exacto de la diapositiva teórica.

```html
<!-- Grupo de Diapositivas de Solución + Compilador Vertical -->
<section>
    <!-- Diapositiva Teórica (Horizontal) -->
    <section>
        <h2 class="slide-title">Solución: Nombre del Ejercicio</h2>
        <pre><code class="cpp" data-trim data-noescape data-line-numbers="1-3|5-10">
        // Código fuente C++ o Java
        </code></pre>
    </section>

    <!-- Diapositiva Compilador (Vertical) -->
    <section id="slide-compiler-1">
        <h2 class="slide-title">Ejecución Interactiva</h2>
        <div class="compiler-window">
            <div class="header">
                <span><i class="fa-solid fa-code"></i> Compilador Online</span>
                <button onclick="sendCode('oc-editor-1', cppCodeExample)">Restaurar Código</button>
            </div>
            <iframe id="oc-editor-1" src="https://onecompiler.com/embed/cpp?listenToEvents=true&theme=dark&hideLanguageSelection=true&hideTitle=true"></iframe>
        </div>
    </section>
</section>
```

---

## 4. Dificultades Soportadas y Soluciones de Ingeniería

### A) Gestión de Gestos y Teclado en Móviles (Aplanado Dinámico)
Los iframes de compiladores externos causan secuestro de deslizamiento táctil y problemas para abrir teclados virtuales en pantallas chicas.
*   **Solución:** Detección de ancho de pantalla (`window.innerWidth < 768`). Si es móvil/tablet, el script elimina del DOM los compiladores verticales (`slide-compiler-X`) y "aplana" dinámicamente las diapositivas padres de RevealJS para evitar slides verticales vacíos.

```javascript
const isMobileOrTablet = window.innerWidth < 768;
if (isMobileOrTablet) {
    // Elimina compiladores interactivos en móviles
    document.querySelectorAll('[id^="slide-compiler-"]').forEach(el => el.remove());
    
    // Aplanar secciones anidadas vacías o con un solo hijo
    document.querySelectorAll('section > section').forEach(sec => {
        const parent = sec.parentNode;
        if (parent && parent.children.length === 1) {
            while (sec.firstChild) {
                parent.appendChild(sec.firstChild);
            }
            sec.remove();
        }
    });
}
```

### B) Sincronización Asíncrona del Iframe (Carga de Código)
El evento `slidechanged` de RevealJS puede dispararse antes de que el iframe de OneCompiler esté listo para recibir mensajes por `postMessage`.
*   **Solución:** Usar una función de reintentos progresivos (`sendCodeWithRetries`) que envía el código al iframe en intervalos crecientes para asegurar la inyección.

```javascript
function sendCode(iframeId, code) {
    const iframe = document.getElementById(iframeId);
    if (iframe && iframe.contentWindow) {
        iframe.contentWindow.postMessage({
            eventType: 'populateCode',
            language: 'cpp', // o 'java'
            files: [{ name: 'main.cpp', content: code }]
        }, '*');
    }
}

function sendCodeWithRetries(iframeId, code) {
    const send = () => sendCode(iframeId, code);
    send();
    setTimeout(send, 200);
    setTimeout(send, 500);
    setTimeout(send, 1000);
    setTimeout(send, 2000);
    setTimeout(send, 3500);
}

// Escuchar cambios de diapositiva
Reveal.on('slidechanged', event => {
    if (event.currentSlide.id === 'slide-compiler-1') {
        sendCodeWithRetries('oc-editor-1', cppCodeExample);
    }
});
```

### C) Prevención de Desbordamiento en Arreglos de Frecuencia/Presencia
Al simplificar búsquedas secuenciales a tiempo constante $O(1)$ usando arreglos booleanos de presencia/frecuencias con cotas superiores (ej: números menores a 30):
*   **Solución:** Utilizar cortocircuito lógico (`||`) en condiciones de bucle para evitar accesos fuera de rango (buffer overflow/segmentation fault).
*   **Ejemplo:** `while (suma >= 30 || !presencia[suma]);` evita evaluar `presencia[suma]` si la suma es inválida.

---

## 5. Buenas Prácticas Académicas
- **Entradas Directas:** Usa `cin >> p >> l;` o `Scanner` directo. Evita envolver la entrada dentro de condicionales de validación innecesarios (`if (cin >> ...)`) a menos que el flujo del algoritmo lo requiera explícitamente.
- **Mantener Código Real en Consola:** Las consolas interactivas OneCompiler deben contener el código fuente 100% real y fiel al de la diapositiva, sin inicializaciones mockearas, forzando a los estudiantes a proveer los datos reales en STDIN.
- **Único Retorno (Single Return):** Evitar múltiples retornos (`return`) en una misma función y, en particular, evitar el uso de `return` dentro de bucles o estructuras repetitivas. Toda función debe estructurarse con un único punto de retorno al final utilizando variables de resultado y banderas lógicas si es necesario.
- **Formateo de Negrita en HTML:** En diapositivas RevealJS escritas en HTML plano (sin `data-markdown`), evitar el uso de notación Markdown como `**texto**` para negritas, ya que se renderizarán literalmente en pantalla. Utilizar siempre etiquetas HTML tradicionales como `<strong>texto</strong>` o `<b>texto</b>`.
- **No usar operadores ternarios (AEDD):** En la materia AEDD no se enseña ni se permite el uso de operadores ternarios `? :`. Utilizar siempre estructuras `if-else` tradicionales para la asignación condicional de valores.
- **No usar break en bucles:** Evitar el uso de `break` dentro de bucles `for`, `while` o `do-while` para controlar la salida anticipada. Se debe estructurar la salida mediante condiciones de bucle compuestas y banderas lógicas. La sentencia `break` solo se permite dentro del bloque `switch`.



