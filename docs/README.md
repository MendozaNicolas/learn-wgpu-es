# Introducción

## ¿Qué es wgpu?

[Wgpu](https://github.com/gfx-rs/wgpu) es una implementación en Rust de la [especificación de la API WebGPU](https://gpuweb.github.io/gpuweb/). WebGPU es una especificación publicada por el GPU for the Web Community Group. Su objetivo es permitir que el código web acceda a las funciones de la GPU de manera segura y confiable. Logra esto imitando la API de Vulkan y traduciéndola a la API que esté usando el hardware del host (por ejemplo, DirectX, Metal, Vulkan).

Wgpu todavía está en desarrollo, por lo que parte de esta documentación está sujeta a cambios.

## ¿Por qué Rust?

Wgpu en realidad tiene bindings de C que te permiten escribir código en C/C++, así como usar otros lenguajes que interactúan con C. Dicho esto, wgpu está escrito en Rust y tiene algunos bindings convenientes de Rust que no requieren rodeos adicionales. Además de eso, he disfrutado mucho escribir en Rust.

Deberías estar bastante familiarizado con Rust antes de usar este tutorial, ya que no entraré en muchos detalles sobre la sintaxis de Rust. Si no te sientes muy cómodo con Rust, puedes revisar el [tutorial de Rust](https://www.rust-lang.org/learn). También deberías estar familiarizado con [Cargo](https://doc.rust-lang.org/cargo/).

Estoy usando este proyecto para aprender wgpu yo mismo, así que podría pasar por alto algunos detalles importantes o explicar las cosas mal. Siempre estoy abierto a comentarios constructivos.

## Contribución y Soporte

* Acepto pull requests ([repositorio de GitHub](https://github.com/sotrh/learn-wgpu)) para corregir problemas en este tutorial, como errores tipográficos, información incorrecta y otras inconsistencias.
* Debido a la API de wgpu que cambia rápidamente, no estoy aceptando nuevos pull requests para demos de showcase.
* Si quieres apoyarme directamente, ¡visita mi [patreon](https://www.patreon.com/sotrh)!

## Traducciones

* [中文版: 增加了与 App 的集成与调试系列章节](https://jinleili.github.io/learn-wgpu-zh/)

## Agradecimientos especiales a estos patrocinadores

* David Laban
* Bernard Llanos
* Ian Gowen
* Aron Granberg
* 折登 樹
* Julius Liu
* Jani Turkia
* Lions Heart
* Filip
* IC
* papyDoctor
* Feng Liang
* Jan Šipr
* Joris Willems
* Mattia Samiolo
* Lennart
* Paul E Hansen
* Gunstein Vatnar
* Nico Arbogast
* Dude
* Youngsuk Kim
* Alexander Kabirov
* charlesk
* Danny McGee
* yutani
* Eliot Bolduc
* Ben Anderson
* Thunk
* Craft Links
* Zeh Fernando
* Ken K
* Ryan
* Felix
* Tema
* 大典 加藤
* Andrea Postal
* Davide Prati
* dadofboi
* Beryesa
* Dzianis Sheka
* George Offley
* Imbris
* Maximilian Temeschinko
* Michael Trainor
