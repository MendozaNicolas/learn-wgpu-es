# Actualización a wgpu 26.0.1 e inicio de la guía de canales de cómputo

Técnicamente tenía la actualización a 26.0 hecha hace un tiempo, pero me encontré con algunos
problemas.

1. Había un error en la implementación de Windows que causaba parpadeo de la vista.
Esto ha sido corregido en la versión 26.0.1
2. Hay un error en cómo wgpu y winit interactúan en WASM en Windows. Hay un bucle de retroalimentación donde
en sistemas donde [devicePixelRatio](https://developer.mozilla.org/en-US/docs/Web/API/Window/devicePixelRatio)
es mayor que `1`, lo que significa que cuando wgpu redimensiona el lienzo con `surface.configure()`,
winit emite un evento e informa que es más grande de lo que wgpu está usando. Esto causa
que el código de demostración llame a `surface.configure()` nuevamente. Este bucle continúa hasta que el tamaño de la superficie
excede los valores máximos que wgpu admite, lo que causa que wgpu entre en pánico. Aquí está el
[problema de seguimiento](https://github.com/gfx-rs/wgpu/issues/7938#issuecomment-3079523549)
por si estás interesado. Mientras tanto, los usuarios han encontrado una solución alternativa
para esto limitando el lienzo a un tamaño particular. Este bit de CSS es
todo lo que necesitas para prevenir el problema:

```css

        canvas {
            width: 100%;
            height: 100%;
        }
```

<div class="note">

Técnicamente solo necesitas la parte `width: 100%`. Básicamente solo necesitas hacer
que el navegador sea responsable del tamaño que debe tener el lienzo.

</div>

## Qué ha cambiado

Dado que `wasm-pack` [ya no está siendo mantenido](https://blog.rust-lang.org/inside-rust/2025/07/21/sunsetting-the-rustwasm-github-org/)
por la Fundación Rust, he optado por usar el fork de wasm-pack encontrado en:
<https://drager.github.io/wasm-pack/>. Funciona bien, aunque he
considerado cambiar a otras alternativas como trunk. Sin planes por el
momento, ya que tengo otros proyectos en este momento.

Revisando el `git diff` la mayoría de los otros cambios son cambios de `Cargo.toml`
y algunas correcciones de errores tipográficos. Para una lista completa de los cambios en wgpu,
consulta el [registro de cambios](https://github.com/gfx-rs/wgpu/releases). También si encuentras algún problema puedes
enviar un PR [aquí](https://github.com/sotrh/learn-wgpu/pulls)!

## Algo nuevo...

Durante un tiempo he estado debatiendo qué hacer a continuación con esta colección. Aunque
hay una gran cantidad de temas de gráficos que podría cubrir (mipmapping,
dibujo indirecto, trazado de rayos por hardware, sombras, animación esquelética, PBR, etc.), no he
decidido cómo quiero que fluya la guía de gráficos. Incluso he considerado reescribir esa
sección con un objetivo más específico en mente, como un juego o un visor de modelos.

Aún quería añadir más valor, especialmente para mis patrocinadores de apoyo. Entonces
decidí cubrir un tema que me ha interesado durante un tiempo: canales de cómputo.
En este momento solo tengo una introducción, pero he comenzado a trabajar en un
ejemplo sobre clasificación de datos en shaders de cómputo e estoy investigando métodos de filtrado
de datos también para técnicas como eliminación de objetos ocultos. También estoy probando un nuevo estilo de escritura con
menos volcados de código, así que déjame saber qué piensas.

Consulta [aquí](../../compute/introduction/)!

## Gracias a mis patrocinadores!

Si te gusta lo que hago y quieres apoyarme, consulta mi [patreon](https://patreon.com/sotrh)!
Un agradecimiento especial a estos miembros!

- Filip
- Lions Heart
- Jani Turkia
- Julius Liu
- 折登 樹
- Aron Granberg
- Ian Gowen
- Bernard Llanos
- David Laban
- Feng Liang
- papyDoctor
- dadofboi
- Davide Prati
- Andrea Postal
- 大典 加藤
- Tema
- Felix
- Mattia Samiolo
- Ken K
- Ryan
- Zeh Fernando
- Craft Links
- Ben Anderson
- Thunk
- Eliot Bolduc
- yutani
- charlesk
- Danny McGee
- Alexander Kabirov
- Youngsuk Kim
- Dude
- Nico Arbogast
- Gunstein Vatnar
- Paul E Hansen
- Joris Willems
- Jan Šipr
- Lennart
