---
layout: post
title: "USB Rubber Ducky con Arduino UNO"
date:   2016-10-04
image: "/images/usb-rubber-ducky-arduino/arduino_flash.png"
description: "Creación de un teclado virtual malicioso utilizando una placa Arduino UNO."
tags: infosec things
---
La placa Arduino UNO, en teoría, no tiene la capacidad para actuar como un dispositivo HID, que es la función que necesitamos para poder crear con ella un USB Rubber Ducky. Sin embargo, esta placa tiene dos microcontroladores Arduino, el ATmega328 y el ATmega16u2, siendo este segundo el que nos interesa. Este microcontrolador se puede reprogramar para que funcione como un USB AVR (tal y como lo haría un Arduino Leonardo, por ejemplo), que es lo que nos permitiría emular un dispositivo HID y así crear nuestro propio USB Rubber Ducky.

Para conseguir esta función adicional tendremos que flashear un nuevo bootloader llamado HoodLoader2, creado por [NicoHood](http://www.nicohood.de/). El código fuente está disponible en su propio [Github](https://github.com/NicoHood/HoodLoader2).

Los pasos para flashear este bootloader empiezan por subir el sketch de instalación al Arduino con el Arduino IDE (podemos descargarlo desde [aquí](https://github.com/NicoHood/HoodLoader2/tree/master/avr/examples/Installation_Sketch)), de forma normal y corriente como haríamos con cualquier otro sketch programado por nososotrs mismos. Una vez hecho esto, necesitaremos hacer un pequeño cableado en la placa para así poder flashear el bootloader automáticamente.

El esquema de cableado que necesitamos para esto es el siguiente:

![Esquema Arduino](/images/usb-rubber-ducky-arduino/arduino_flash.png)

Una vez hecho esto, volvemos a conectar el Arduino a un puerto USB, y automáticamente iniciaría el flasheado del nuevo bootloader. Nada más conectarlo el led parpadeará lentamente durante 10 segundos, tras los cuales, empezaría el proceso de flasheo, que durará alrededor de unos 30 segundos, y cuando finalice el mismo led parpadeará rápidamente (cada 100ms).

Si el led sigue parpadeando lentamente (cada segundo) significará que la instalación ha fallado, y volverá a intentar autoflashearse en 10 segundos.

Todo el proceso de instalación de HoodLoader2 lo tenemos mucho más detallado en su propio Github: <https://github.com/NicoHood/HoodLoader2/wiki/Installation-sketch-(standalone-Arduino-Uno-Mega)>.

Ahora ya tenemos la placa con el nuevo bootloader, pero el propio Arduino IDE no lo reconocerá y no nos dejará subir nuestros sketches, por lo que necesitaremos instalar las definicones para las placas con HoodLoader2. Es un proceso realmente sencillo y podemos ver los pasos a seguir aquí: <https://github.com/NicoHood/HoodLoader2/wiki/Software-Installation>.

Una vez Arduino IDE nos reconocé la placa, podemos probar a subir un sketch de prueba y comprobar si lo ejecuta correctamente:

{% highlight CPP linenos %}
#include "HID-Project.h"

void setup() {
    Keyboard.begin();
    delay(500);

    Keyboard.press(KEY_LEFT_GUI);
    Keyboard.press(KEY_R);
    Keyboard.releaseAll();
    delay(500);
    Keyboard.println("cmd.exe");
    delay(500);
    Keyboard.println("calc.exe");
}

void loop() {}
{% endhighlight %}

Como se puede observar en la imagen, utilizaremos la librería "[HID-Project](https://github.com/NicoHood/HID)".
Esta librería nos permite utilizar Keyboard.println() para enviar un string seguido de un Enter. Si no quisieramos enviar el Enter final utilizaríamos Keyboard.print().
Si necesitamos enviar una combinación de teclas, podemos simular que estamos presionándolas con el comando Keyboard.press(), y una vez enviada la combinación soltarlas con el comando Keyboard.releaseAll().

Aquí dejo un .txt con todas las KEY_* disponibles con su respectivo código: [downloads/ducky_keys.txt](/downloads/ducky_keys.txt)

Solo nos queda conectar el Arduino al equipo en el que lo queremos comprobar y esperar. En este caso parece que funciona a la perfección abriendo la calculadora :P

![Demo](/images/usb-rubber-ducky-arduino/calc.gif)

Para poder utilizar los scripts que están diseñados para el USB Rubber Ducky original, he creado un pequeño script que convierte cada función a la función equivalente en Arduino, y genera automáticamente el scketch completo, listo para subirlo a la placa. Es posible que tenga algún fallo, pero los que he comprobado funcionaban a la perfección.

Ducky2Arduino: [https://github.com/kr0no/Ducky2Arduino](https://github.com/kr0no/Ducky2Arduino)

Como ejemplo sencillo, el script Hello World de Rubber Ducky tendría el siguiente formato en un sketch de Arduino:

{% highlight console %}
~$ ducky2arduino.py helloword_ducky.txt helloword_arduino.txt
{% endhighlight %}

![Ducky2Arduino](/images/usb-rubber-ducky-arduino/ducky2arduino.png)

Todo esto ya tiene sus años, pero tenía un Arduino UNO tirado por casa y quería darle algún uso útil. Si a día de hoy queremos hacer esto, tenemos una forma mucho más sencilla haciéndonos directamente con alternativas como Teensy o Arduino Zero, ya que permiten emular de forma nativa y sin ningún problema dispositivos HID.