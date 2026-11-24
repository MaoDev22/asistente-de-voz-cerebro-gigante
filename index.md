---
title: Su asistente de voz no necesita un cerebro gigante
---

# Su asistente de voz no necesita un cerebro gigante

*Lo que aprendimos construyendo las llamadas de voz de Zentria para una ferretería, con un modelo pequeño, un servidor propio y 1.362 frases de prueba.*

Por: Marlon Torres Lozano

---

## «Dermatrés»

Un cliente llama a una ferretería. Acaba de preguntar el precio de un tomacorriente y, satisfecho, dice: «deme tres».

Lo que el sistema recibe, después de pasar por el micrófono de un celular, el ruido de la calle y un programa que convierte voz en texto, es esto:

> Dermatrés.

No existe esa palabra. Ninguna regla la reconoce. Y aquí aparece la tentación que hoy domina la industria: poner detrás un modelo de lenguaje enorme, darle todo el catálogo y dejar que él resuelva. Entenderá «Dermatrés», entenderá cualquier cosa.

Nosotros probamos el camino contrario. Usamos un modelo pequeño, que corre en nuestro propio servidor, y le quitamos casi todas las responsabilidades. El resultado fue una llamada que acierta más, se equivoca de forma menos grave y responde antes. Este artículo cuenta cómo, con las cifras, y también dice con claridad qué no hemos demostrado.

## El día que dejamos decidir al modelo

Nuestra llamada funciona en cadena: un modelo de voz oye, un guion de reglas decide qué responder con los datos reales de la tienda, y el modelo de voz lee la respuesta. Las reglas son rápidas y nunca inventan, pero tienen un límite: de 1.362 frases de prueba, entendían 1.214. En 142 casos ninguna regla reconocía lo dicho.

La solución obvia era pasarle esos casos al modelo y que decidiera él: qué quiere el cliente y de qué producto habla. Lo medimos.

Acertó un poco más: 1.228. Pero ocurrió algo que el número de aciertos escondía. Nuestro medidor cuenta aparte los turnos **dañinos**: aquellos donde el sistema haría o afirmaría algo equivocado, como hablar de otro producto o agregar al carrito lo que nadie pidió. Con las reglas solas había 9. Con el modelo decidiendo, **29**.

El patrón era siempre el mismo. Un cliente pregunta si venden candados. La tienda no vende candados. El modelo, servicial, escoge lo más parecido que encuentra en el catálogo y la llamada responde con el precio de unos guantes. Otro pide un taladro percutor y recibe la ficha de un alicate.

Un modelo de lenguaje casi nunca dice «no sé». Completa. Es su naturaleza, y en una conversación escrita se le perdona porque uno puede releer y corregir. En una llamada, lo dicho en voz alta ya se dijo.

## La idea: que aclare, no que decida

El cambio fue pequeño en código y grande en resultado. En lugar de pedirle al modelo que interpretara la frase, le pedimos una sola cosa: **reescribirla en limpio**. Nada más. Después, las mismas reglas de siempre leen la frase reescrita y responden con sus datos.

«Dermatrés» se convierte en «Deme tres». Las reglas, que sí entienden «Deme tres», saben de qué producto se venía hablando y preguntan: «¿Agrego tres unidades de tomacorriente doble a tu carrito?».

La diferencia con el experimento anterior está en quién tiene la última palabra. Si el cliente pregunta por candados, el modelo escribe «¿Tienen candados?» y son las reglas las que buscan en el catálogo, no encuentran nada y responden lo correcto: «Eso no lo encuentro en la tienda». El modelo ya no tiene oportunidad de ser servicial con datos que no existen.

Tres decisiones de diseño sostienen esto:

1. **El modelo solo entra cuando las reglas fallan.** Nueve de cada diez frases nunca pasan por él y se responden en milisegundos.
2. **El modelo no elige producto, precio ni respuesta.** Solo aclara qué se dijo.
3. **Nada de lo que escribe se usa sin revisar.** De eso se encarga un filtro.

## El filtro: prohibido lo que el cliente no dijo

Reescribir tampoco es inocente. A la frase «Deme tres bulticos pues», nuestro modelo le agregó de su cosecha: «Deme tres bombillos LED». A un simple ruido, «Kss prr tk», le inventó una pregunta por el precio de un alicate.

Por eso cada reescritura pasa un filtro de reglas fijas antes de aceptarse. La frase reescrita no puede traer:

- **Un producto que no suene en lo que se dijo.** «Bombillos» no aparece, ni de lejos, en «bulticos».
- **Una cantidad que no se dijo**, en cifras o en palabras.
- **Una ciudad que no se dijo** o que no se reconoce.
- **Un envase que no corresponde.** Si el cliente dice «écheme dos bolsas», no pueden ser los tornillos que vienen por caja.
- **Un «sí» que decida el carrito.** Agregar siempre exige que la persona confirme con su propia voz.
- **Un cambio de tema.** «¿Y dan factura electrónica?» no puede volverse «¿Dónde quedan?».

El filtro compara por sonido, no por letras. «Lievo» y «llevo» son lo mismo; «dermatrés» contiene «tres». Es la parte menos vistosa del sistema y la que más pesa: sin ella, la reescritura repetiría el problema que queríamos evitar.

Cuando el filtro frena una frase, la llamada no improvisa. Dice «No te entendí bien, ¿me lo repites?», que es la verdad.

## Enseñar con ejemplos, no con órdenes

La primera versión rescataba pocas frases. El modelo reescribía bien, pero no en la forma que las reglas esperan: dejaba «¿Cómo quedó el pedido?» cuando las reglas entienden «¿Qué llevo en el carrito?». En frases muy dañadas se desviaba: «Dermatrés» llegó a salir como «Deme uno».

La mejora vino de una sugerencia sencilla: en vez de explicarle más, mostrarle. Agregamos al aviso veinte ejemplos de lo que entra y lo que debe salir.

Los ejemplos son de otra tienda, una papelería con cuadernos y lapiceros, a propósito. Así el modelo aprende la forma sin tomar prestados productos, y no le enseñamos las respuestas del examen con el que lo medimos.

El efecto fue doble. Las reescrituras aprovechadas pasaron de 27 a 48, y el modelo se volvió más rápido, de 0,74 a 0,61 segundos, porque dejó de divagar.

## Las cifras

Todo se midió con bancos de frases etiquetadas, 1.362 turnos en total: 764 escritos y 598 tal como llegan transcritos por el modelo de voz, con sus errores.

| Quién decide | Turnos bien | Turnos dañinos | Tiempo del modelo |
|---|---|---|---|
| Solo las reglas | 1.214 (89 %) | 9 | — |
| El modelo decide intención y producto | 1.228 | 29 | 0,62 s |
| El modelo reescribe, deciden las reglas, con filtro | 1.232 | 10 | 0,74 s |
| Lo mismo, con ejemplos | 1.251 (92 %) | 10 | 0,61 s |

El turno dañino de más merece una nota. A quien quiso decir «deme dos», la llamada le pregunta «¿Agrego una unidad?». No agrega nada; el cliente responde que no y sigue. Antes, esa persona oía «no te entendí».

Y el tiempo: 0,61 segundos de mediana, 0,73 en el peor diez por ciento, y solo en los turnos que antes fallaban. Para comparar, las guías de la industria ponen como meta alrededor de un segundo para toda la respuesta de un agente de voz en cadena.

## Por qué un cerebro pequeño alcanza

La respuesta corta: porque el trabajo que le dejamos es pequeño.

Reescribir una frase de ocho palabras exige mucho menos que sostener una conversación, consultar un catálogo y decidir una venta. Un modelo pequeño hace bien lo primero. Lo segundo lo hace mal, y no solo el nuestro. En pruebas anteriores, cuando dejamos que el modelo respondiera libremente preguntas que no tenían respuesta en los datos, inventó en 29 de 48: «sí, tenemos parqueadero», «sí, hacemos factura electrónica». Un estudio reciente que puso modelos pequeños a atender llamadas completas encontró que los más pequeños se degradan hasta producir turnos interminables y sin sentido.

Tampoco los modelos grandes salen ilesos de la voz. Un banco de pruebas citado en ese mismo estudio reporta que los agentes de voz más avanzados conservan apenas entre 30 y 45 % del éxito que tienen por escrito. Son tareas más amplias que la nuestra y la comparación no es directa, pero apunta a lo mismo: en una llamada, el problema difícil no se resuelve con tamaño.

Se resuelve repartiendo bien. Las reglas hacen lo que exige exactitud: precios, existencias, carrito. El modelo hace lo que exige flexibilidad: entender a una persona que habla como habla la gente. Y un filtro vigila la frontera entre los dos.

Hay además razones prácticas que a una empresa pequeña le importan más que cualquier banco de pruebas. El modelo corre en un servidor propio: las conversaciones de los clientes no salen a un tercero, no hay un cobro por cada llamada y la velocidad no depende de la congestión de un servicio ajeno.

## Lo que este artículo no demuestra

Sería deshonesto terminar sin esto.

- **No comparamos contra un modelo grande.** El título dice que no hace falta un cerebro gigante porque con uno pequeño llegamos a 92 % en nuestras pruebas. No dice que uno gigante lo haría peor: esa prueba no la hemos hecho.
- **Los bancos de frases son nuestros**, de una ferretería, en español colombiano. En otro negocio o en otro idioma, las cifras serán otras.
- **El filtro se ajustó mirando esos mismos bancos.** Con frases nuevas rendirá algo menos. Hicimos una comprobación aparte con 16 frases que no estaban en ninguno: las reglas ya entendían 10; de las otras 6, la reescritura rescató 3, una no era de la tienda y el filtro frenó 2. Es una buena señal y una muestra pequeña.
- **Falta la prueba que importa:** llamadas de clientes reales, durante semanas.
- **Las ideas no son nuevas por separado.** Reescribir la frase antes de interpretarla lo publicó Amazon para Alexa hace años; parafrasear hacia una forma que un sistema determinista entiende es una técnica conocida; verificar lo que un modelo corrige es un área activa de investigación. Lo nuestro es la combinación, el filtro por sonido y haber medido el daño, no solo el acierto.

## La lección

Durante dos años la pregunta de moda fue cuál modelo es el más grande. Para quien construye un producto, la pregunta útil es otra: **¿cuál es la tarea más pequeña que puedo darle al modelo?**

Cada responsabilidad que se le quita es un error que ya no puede cometer. Al nuestro le quitamos el catálogo, los precios, el carrito y la última palabra. Le dejamos una frase mal oída y la instrucción de escribirla bien.

Con eso, «Dermatrés» volvió a ser tres tomacorrientes.

---

### Referencias

- Chen y otros, *Pre-Training for Query Rewriting in a Spoken Language Understanding System*: https://arxiv.org/pdf/2002.05607.pdf
- *Personalized Query Rewriting in Conversational AI Agents*: https://arxiv.org/pdf/2011.04748
- Shin y otros, *Constrained Language Models Yield Few-Shot Semantic Parsers* (EMNLP 2021): https://arxiv.org/abs/2104.08768
- *Fewer Hallucinations, More Verification: A Three-Stage LLM-Based Framework for ASR Error Correction*: https://arxiv.org/pdf/2505.24347
- *CallScreenBench: Benchmarking Small Language Models as Phone Secretaries*: https://arxiv.org/pdf/2608.01033
- Twilio, *A Guide to Core Latency in AI Voice Agents*: https://www.twilio.com/en-us/blog/developers/best-practices/guide-core-latency-ai-voice-agents

*Las cifras propias provienen de mediciones internas del 23 de noviembre de 2026 sobre 13 bancos de frases etiquetadas. Este texto se redactó con ayuda de inteligencia artificial.*
