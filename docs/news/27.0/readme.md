# Actualización a 27.0!

Una actualización bastante rápida con solo dos cambios principales

* `wgpu::DeviceDescriptor` ahora tiene un campo `experimental_features` que le
indica a WGPU si queremos usar características que aún no han sido estabilizadas.
Para el tutorial establecemos esto en `wgpu::ExperimentalFeatures::disabled()`
* `PollType::Wait` ahora tiene campos: `submission_index` y `timeout`. Podemos obtener
el índice de envío desde `Queue::submit`, pero solo usamos
esto en un par de lugares así que simplemente usaremos `PollType::wait_indefinitely()`.

## ¡Gracias a mis patrocinadores!

Si te gusta lo que hago y quieres apoyarme, visita mi [patreon](https://patreon.com/sotrh)!
¡Un agradecimiento especial a estos miembros!

* David Laban
* Bernard Llanos
* papyDoctor
* Ian Gowen
* Aron Granberg
* 折登 樹
* Julius Liu
* Lennart
* Jani Turkia
* Feng Liang
* Paul E Hansen
* Lions Heart
* Gunstein Vatnar
* Nico Arbogast
* Dude
* Youngsuk Kim
* Alexander Kabirov
* charlesk
* Danny McGee
* yutani
* Eliot Bolduc
* Filip
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
* IC
* Davide Prati
* dadofboi