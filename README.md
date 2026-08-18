# Antonio-Morelli-UCEMA
Repositorio de entregas de la materia Creacion de Agentes de IA


# Bot de reservas de canchas — Club Harrods

## Qué construí

Un bot que reserva canchas de tenis en el sitio web de mi club (harrodstenis.com), un sistema PHP viejo sin API. El club habilita cada día a las 8:00 de dos días antes y los horarios buenos se agotan en segundos, así que reservar a mano implica estar frente a la computadora a esa hora exacta. El bot recibe un pedido ("reservá el jueves a las 19"), lo agenda, y una tarea de Windows lo dispara la mañana que corresponde: espera la apertura, elige la mejor cancha disponible según mis preferencias y completa la reserva con un segundo socio. Es para mí, aunque serviría para cualquier socio del club.

## Cómo se lo pedí

Trabajé con Claude Code en sesión interactiva, de a un paso por vez. Los prompts principales, en orden:

**1. Encuadre inicial y primer paso**

> Quiero construir un agente que automatice la reserva de canchas de tenis en el sitio web de mi club.
>
> Contexto del sistema de reservas del club:
> - Las reservas de un día se habilitan a las 8:00 AM, dos días antes (ej: las del miércoles se abren el lunes a las 8AM)
> - Los horarios más pedidos se agotan muy rápido, así que la velocidad de ejecución importa
> - URL del sitio: https://www.harrodstenis.com/sigmasports/TTlogin.php
> - Tipo de sitio: sitio tradicional sin API visible (a verificar porque no estoy seguro)
> - Requiere login: sí — usuario: 40495906 / contraseña: [redactada]
>
> Por ahora NO quiero que armes el proyecto completo. Quiero ir paso a paso, probando cada parte antes de seguir.
>
> Primer paso: ayudame a escribir un script en Python que solo haga login en el sitio y confirme que la sesión quedó autenticada (por ejemplo, accediendo a una página que requiera estar logueado y verificando la respuesta). Nada de reservas todavía.
>
> Antes de escribir código, hacéme las preguntas que necesites sobre cómo es el sitio.
>
> [Adjunté capturas del sitio y el flujo manual paso a paso: login, reservas de tenis, elegir día, elegir horario libre, agregar participante, buscar socio, aceptar, finalizar. Más las reglas: máximo 1 hora, hace falta un socio más, 14 canchas, las mejores son de la 3 a la 7, las 12 y 14 son de cemento y van últimas.]

**2. Reglas del club que faltaban**

> Los socios con los que podes reservar: 19982, 21955, 22467, 21424
>
> Las canchas bloqueadas en los dias de semana son la 8, 9, 10 y 11. Y las bloqueadas durante el fin de semana por interclubes son la 5, 6 y 7. Igual te das cuenta porque si esta en amarillo es porque esta reservada o bloqueada por interclubes, y si esta en gris es que esta bloqueada para clase.

**3. Provocar el caso de error a propósito**

> para que puedas hacer la prueba de lo que pasa cuando un compañero ya tiene reserva, intenta reservar para mañana martes a las 20hs la cancha 1 con el socio 19994 que ya se que tiene reserva. Asi podes ver lo que te salta cuando pasa eso.

**4. Reserva real**

> dale, proba reservar con alguien de la lista para el miercoles a las 17hs

**5. Cuestionar la arquitectura**

> espera, entonces como quedo el proceso de punta a punta? yo como te aviso que quiero una cancha? vos sabes en que dia estamos parados? por ejemplo, yo te puedo decir reserva una cancha para el viernes a las 19hs? sabes a que viernes me estoy refiriendo? donde dejas la tarea programada? por que via te mando el mensaje? tiene que estar la compu prendida?

**6. Cuestionar una decisión técnica**

> Con respecto a adelantar 2 minutos la coneccion, crees que es una buena idea? puede pasar que funcione rapido, no se peuda hacer la reserva y lo de de baja? si me confirmas que no, hago el cambio, pero si hay riesgo lo dejo asi como esta

Además, en varios momentos el agente me frenó para que yo decidiera: si probar sobre una cancha de cemento en horario muerto o sobre una que realmente quería, si guardaba las cookies entre corridas, y dónde iba a correr la tarea de las 8:00.

## Qué funciona

Probado contra el sitio real, no en simulación:

- **Login y sesión.** Detecta correctamente la sesión caída y falla ruidosamente con clave incorrecta. Reutiliza la sesión guardada en disco.
- **Lectura de la grilla.** Parsea las 167 celdas libres de un día y las ordena según mi preferencia de canchas. Lo verifiqué comparando contra el conteo de links del HTML crudo, no a ojo.
- **Búsqueda de socios.** Los 4 socios habilitados resuelven a su ID interno.
- **Reserva completa.** Creé una reserva real, la verifiqué en el sitio y la cancelé. Después hice una que quedó: miércoles 17:00 a 18:00 en la cancha 6.
- **Rotación de compañeros.** Cuando un socio ya tiene reserva ese día, el bot lo detecta y pasa al siguiente. Probado: rechazó a Robaldo y entró con Borysowski.
- **Agenda y disparo.** El pedido se guarda y una tarea de Windows lo ejecuta a las 8:00, despertando la PC si está suspendida.

Uso:

```bash
python agendar.py viernes 19:00      # agenda (imprime la fecha que interpretó)
python agendar.py --listar           # ver pendientes
python cancelar.py                   # ver y cancelar reservas hechas
python reservar.py --fecha viernes --hora 19:00 --confirm   # reservar ahora
```

## Qué falta o qué falló

**Fallas durante el desarrollo:**

- **Certificado TLS.** El primer intento de login murió con `SSLError: CERTIFICATE_VERIFY_FAILED: self-signed certificate in certificate chain`. El proxy de la red de mi trabajo intercepta HTTPS y Python no confía en esa CA porque usa su propio bundle de certificados en vez del almacén de Windows. Se resolvió con la librería `truststore`, sin desactivar la verificación de certificados, que era el atajo fácil y peligroso.
- **El parser devolvía 0 canchas libres.** El atributo `title` de las celdas contiene `=>`, y la expresión regular que buscaba el tag `<td>` cortaba en ese `>`, perdiendo el resto del tag. Apareció porque se comparó el resultado contra el conteo real de links del HTML, en vez de dar por bueno que "no explotó".
- **Reservó media hora en vez de una hora.** El sitio recorta la duración máxima según el hueco libre y acepta el pedido igual, sin avisar. Hubo que cancelar esa reserva y rehacerla fijando la hora de fin explícitamente.
- **Diagnóstico falso.** El agente reportó dos mensajes de error del sitio que resultaron ser texto estático presente en todas las respuestas, incluidas las exitosas. Se detectó comparando el HTML de un caso exitoso contra el de un rechazo.

**Lo que no está probado o no está hecho:**

- **El disparo de las 8:00 nunca corrió contra una apertura real.** El mecanismo de espera está verificado con una apertura simulada, pero la prueba de verdad es mañana a la mañana. Es la pieza más importante y la única que no se puede validar antes de que ocurra.
- **El reintento ante caída de red está escrito pero sin probar**, porque no hay forma de provocar una caída del servidor del club a voluntad.
- **Avisar desde el celular.** Falta un bot de Telegram. No es un problema de IA sino de transporte: el pedido tiene que llegar físicamente a la PC que corre la tarea.
- **Una optimización pendiente**: cachear los IDs de cancha para saltear la lectura de grilla en el momento pico y ahorrar unos 800 ms.

## Qué aprendí

Entendí qué es y qué no es un agente. Claude no es un servicio que queda corriendo esperando: una sesión desde el celular vive en la nube y no puede tocar mi PC, así que no sirve para disparar nada a las 8:00. MCP tampoco resuelve eso; es un estándar para darle herramientas al modelo mientras piensa, no algo que lo mantenga vivo.

Aprendí dónde conviene meter la IA y dónde no. Acá rindió muchísimo para investigar un sitio sin documentación y descubrir sus trampas, pero el producto final no tiene IA adentro: es Python haciendo pedidos HTTP. En una carrera de segundos, un modelo razonando en el medio es más lento y menos predecible que un script. La inteligencia va en la construcción, no en la ejecución.

Del método, dos cosas me sirvieron: pedirle que fuera paso a paso y que preguntara antes de codear, y cuestionarle lo que proponía. Cuando dudé de un cambio suyo, revisarlo destapó tres problemas que él no había visto.