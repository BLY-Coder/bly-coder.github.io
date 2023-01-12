---
layout: single
title: eWPT - Review
excerpt: Esto es una review de la certificación de elearning security, donde os cuento mi experiencia.
date: 2023-01-12
classes: wide
header:
  teaser: /assets/images/ewpt-review/ewpt.png
  teaser_home_page: true
  icon: /assets/images/ewpt-review/ewpt.png
categories:
  - infosec
tags:  
  - ctf
  - certificates
---

## Mi Experiencia con el eWPT

Hola a todos! Como anuncié hace unos dias por [**Twitter**](https://twitter.com/bertrandlorente) he aprobado el examen de [**eLearnSecurity Web application Penetration Tester**](https://elearnsecurity.com/product/ewpt-certification/)(eWPT) y hoy os vengo a contar mi experiencia, no quiero alargar mucho esta review debido a que ya hay muchas. 
¡Vamos a comentar lo importante!

![](/assets/images/ewpt-review/ewpt.png)

#### Aspectos básicos del examen.
Lo primero que voy a comentar es... _"¿Cómo es el examen?"_ Bueno, esta certificación es de pentesting web por lo tanto en el examen tendremos que conseguir encontrar todas las vulnerabilidades posibles de una web más todos sus subdominios. El examen es totalmente práctico, te dan 7 días para encontrar todas las vulnerabilidades posibles y otros 7 días para crear un reporte que **recoja todo** lo encontrado anteriormente, es decir en total tienes **14 días**. 

Para sacarte esta cert tienes dos opciones:
- Pagar el examen + el curso de INE.
- Pagar únicamente el examen.

En mi caso la prueba **me la he preparado por mi cuenta** debido a que económicamente no me venía bien pagarme el entrenamiento de INE.

### Hablemos del examen.

Tardé al rededor de 5 días en acabar el examen, me organicé para ir haciendo el reporte e ir encontrando vulnerabilidades de forma simultamenea sin agobiarme y de forma muy tranquila. La verdad que me ha parecido un examen muy divertido.
Si te estás preguntando si merece la pena esta certificación... **¡Sin duda!**

La dificultad depende un poco de la experiencia. Yo me he estado preparando este examen alrededor de dos meses y medio. Normalmente los CTF con los que yo practico intento que toquen un poco de todo, pero esta vez me he centrado en **full pentesting web.**

La parte del reporte podemos considerarla "la más sencilla" dependiendo de la experiencia de cada uno. En mi caso he escrito pocos reportes pero tenia las ideas claras. 

### ¿Cómo es la parte de pentesting?

Para acceder al laboratorio te dan **acceso a una VPN** y la dirección IP del host que tiene el servicio web. 

Una vez conectado como he comentado anteriormente hay que encontrar todas las vulns posibles en 7 dias. **No te agobies, tienes tiempo**. Te recomiendo que te repases las vulnerabilidades basicas que te puedes encontrar, no te esperes encontrar un SSTI por ejemplo.

En este examen se pueden usar todo tipo de herramientas. Como consejo personal, tu mejor amigo va a ser Burp Suite. Esta herramientas son muy potentes y vienen super bien para el examen.

Fuera de esto... ¿Cuál creo yo que es la clave para aprobar esta parte? **La enumeración es fundamental**, hay cosas que se pueden pasar por alto si no enumeras bien. En serio no tengas prisa y tómatelo con calma.

> Para aprobar el examen hay una serie de objetivos que te dan en un a carta de presentación. Te recomiendo que la leeas muy bien. 

### Qué creo que es necesario saber.

- **Burpusite** es super importante como he dicho antes, considero que en este tipo de examenes tiene muchisimo alcance.
- **SQLMap** es una tool que también me ha servido en momentos en los que he dudado si algo era o no vulnerable.
- Saber cómo funcionan **las sesiones y las cookies**

### El reporte.

El reporte es a mi parecer la parte donde ya te puedes relajar un poco y simplemente contar lo que has descubierto. Este es en inglés y debe incluir la siguiente información.

- Vulnerabilidad
- Breve Descripción
- Activos Afectados 
- Descripción
- Impacto
- Recomendaciones

Yo en mi caso use una plantilla de un reporte que hice una vez y lo use para esto. La verdad que me vino perfecto.


### Conclusión final.

Recomiendo sin duda el eWPT esta certificación, me ha parecido muy divertida y didáctica. Si quieres enfocarte más en el hacking web. Después de todo este es un palo fundamental del pentesting.


