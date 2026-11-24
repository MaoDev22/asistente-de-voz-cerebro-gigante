---
title: Su asistente de voz no necesita un cerebro gigante
---

# Su asistente de voz no necesita un cerebro gigante

**Arquitectura, mediciones y errores de un agente de voz para comercio que corre con un modelo de 9B y otro de 2B en un servidor propio.**

Por: Marlon Torres Lozano · Zentria · noviembre de 2026

---

## Resumen

Construimos el modo llamada del asistente de una tienda en línea: el cliente habla, el sistema responde con voz, consulta precios reales y arma el carrito. Todo corre en un servidor propio con dos modelos cuantizados que suman 6,7 GB de pesos.

Tres resultados, medidos sobre 1.362 turnos etiquetados:

| Hallazgo | Cifra |
|---|---|
| Dejar que el modelo **decida** intención y producto triplica los turnos dañinos | de 9 a 29 |
| Pedirle solo que **reescriba** la frase y dejar la decisión a reglas sube el acierto sin ese costo | 1.214 → 1.251 turnos bien, 10 dañinos |
| Un filtro determinista sobre la reescritura no cuesta acierto y elimina 5 de los 6 daños que la reescritura añade | 15 → 10 dañinos, 1.248 → 1.251 bien |

El modelo de lenguaje interviene en cerca del 10 % de los turnos y tarda 0,61 s de mediana. El resto se resuelve en milisegundos.

La tesis: en un agente de voz de dominio cerrado, la variable que más pesa no es el tamaño del modelo sino **el tamaño de la tarea que se le asigna**.

## 1. El problema

Un agente de voz para una ferretería tiene tres restricciones que un chat escrito no tiene:

1. **Lo dicho no se retira.** Un precio equivocado dicho en voz alta ya llegó al cliente. No hay «editar mensaje».
2. **La entrada llega rota.** El texto que recibe el sistema es la salida de un reconocedor de voz. «Deme tres» llega como «Dermatrés»; «écheme diez pliegos», como «H me 10 pliegos».
3. **El silencio es un error.** Pasado un segundo sin respuesta, la persona repite o cuelga.

A eso se suma la restricción del negocio: el cliente es una pyme. Un costo por llamada atado a un modelo grande de terceros, y las conversaciones de sus clientes saliendo a otro proveedor, no eran aceptables.

## 2. Arquitectura

La llamada es una cadena de tres etapas con un plano de decisión determinista en el medio.

<pre class="mermaid">
flowchart TB
    N["Navegador: micrófono, PCM 16 kHz"]
    N -- "WebSocket" --> T
    subgraph S["Servidor de la llamada (Python, FastAPI)"]
        T["Cierre de turno: 600 ms de silencio"] --> O["Oír"]
        O --> G["Guion determinista: reglas + datos reales"]
        G -- "no entendió" --> W["Reescritura + filtro"]
        W -- "frase limpia" --> G
        G -- "entendió" --> R["Frase exacta, cifras en letras"]
        R --> H["Hablar"]
    end
    O <-.-> A[("LFM3-Audio-2B (llama.cpp)")]
    H <-.-> A
    W <-.-> M[("MiMo-V2.6 9B (llama.cpp)")]
    G <-.-> D[("PostgreSQL: catálogo, envíos, horario")]
    H -- "audio 24 kHz + texto" --> N
</pre>

### Modelos y servicio

| | Voz (oír y hablar) | Lenguaje (reescribir) |
|---|---|---|
| Modelo | LFM3-Audio-2B | MiMo-V2.6-Distill-Qwen-9B |
| Parámetros | 1,73 mil millones | 8,95 mil millones |
| Cuantización | Q4_K_M | Q4_K_M |
| Tamaño en disco | 1,09 GB | 5,62 GB |
| Contexto configurado | 8.192 fichas | 16.384 fichas |
| Ranuras paralelas | 4 | 2 |
| Servidor | llama.cpp (`llama-server`, compilación b12602) | igual |
| Interfaz | `/v1/audio/transcriptions`, `/v1/audio/speech`, `/v1/realtime` | `/v1/chat/completions` con `response_format: json_schema` |
| Razonamiento | no aplica | desactivado (`enable_thinking: false`) |
| Velocidad medida | transcribe un turno en 0,27 s; empieza a sonar en 0,24 s | 49 fichas por segundo |

Los dos modelos comparten una sola tarjeta gráfica. El navegador nunca habla con ellos: habla con el servidor de la llamada, que impone los topes (10 minutos por llamada, 30 s por turno, 2 llamadas simultáneas por defecto).

### Por qué una cadena y no un modelo de voz a voz

Lo probamos. El modelo de voz, conversando por su cuenta con el catálogo a la vista, inventó precios y productos («sí, tenemos taladros»), y usó la herramienta de agregar al carrito en 0 de 4 intentos. Lo que sí hace bien es repetir al pie de la letra una frase que se le entrega. La cadena permite comprobar cada cifra antes de pronunciarla.

## 3. El turno, paso a paso

<pre class="mermaid">
flowchart TD
    A["Turno cerrado: transcripción"] --> B{"¿Alguna regla<br/>la reconoce?"}
    B -- "sí (≈ 90 %)" --> C["Guion responde<br/>con datos reales"]
    B -- no --> P["En paralelo"]
    P --> E["Buscar en lo que la empresa<br/>le enseñó al asistente"]
    P --> F["Modelo: reescribir en limpio<br/>(flujo, JSON)"]
    F --> G{"¿El guion entiende<br/>la frase reescrita?"}
    G -- sí --> H{"¿Pasa el filtro?"}
    H -- sí --> C
    H -- no --> I["«No te entendí bien»"]
    G -- no --> J{"Clase que dio<br/>el modelo"}
    J -- no_claro --> I
    J -- fuera --> K["«Con eso no te puedo ayudar…»"]
    J -- pregunta --> L{"¿Lo enseñado<br/>la responde?"}
    E --> L
    L -- sí --> M2["Responde con lo enseñado"]
    L -- no --> N2["«Ese dato no lo tengo a la mano»<br/>queda en la lista de la dueña"]
</pre>

Dos propiedades de este flujo importan más que el resto:

- **El camino feliz no toca al modelo de lenguaje.** Nueve de cada diez turnos se deciden con reglas.
- **Todo desvío termina en el mismo guion.** El modelo no tiene un camino propio hacia la respuesta.

## 4. El guion determinista

El guion es código, no un modelo. Hace cuatro cosas:

**Corrige por sonido contra el catálogo.** Antes de interpretar, normaliza cada palabra a cómo suena en español y la compara con los productos y ciudades de la tienda:

```python
def _son(w: str) -> str:
    """«thinner» y «tiner», «nequi» y «neki», «cali» y «kali» quedan iguales."""
    w = w.replace("th", "t").replace("ph", "f").replace("sh", "ch")
    w = re.sub(r"g(?=[ei])", "j", w)
    w = re.sub(r"qu(?=[ei])", "k", w)
    w = re.sub(r"c(?=[ei])", "s", w)
    w = w.replace("c", "k").replace("z", "s").replace("v", "b").replace("ll", "y").replace("h", "")
    return re.sub(r"(.)\1+", r"\1", w)
```

**Clasifica entre 16 intenciones** con expresiones regulares: precio, disponibilidad, agregar al carrito, quitar, ver carrito, pagar, envío, horario, ubicación, medios de pago, hablar con una persona, saludo, gracias, despedida, afirmación y negación. Lo que no encaja queda como `otro`.

**Mantiene el estado de la conversación:** de qué producto se venía hablando, qué opciones se ofrecieron, qué quedó pendiente de confirmar. Eso resuelve «¿y cuánto vale?» o «mejor que sean cinco».

**Arma la frase exacta,** con precios y horas en letras («diecisiete mil ochocientos cincuenta pesos»), porque el modelo de voz lee mal las cifras. Y nunca agrega al carrito sin confirmación: pregunta «¿Agrego dos unidades de…?» y espera el sí.

El guion solo entiende 1.214 de 1.362 turnos (89 %). Ese 11 % restante es el problema de este artículo.

## 5. Tres maneras de usar el modelo

### A. El modelo decide

Primera versión: cuando el guion no entiende o no encuentra el producto, el modelo devuelve la interpretación.

```json
{"intencion": "precio", "producto": 14, "cantidad": null, "ciudad": null}
```

`producto` es el índice en el catálogo que se le pasa en el aviso. El servidor valida el índice y arma la respuesta.

### B. El modelo reescribe

Segunda versión: el modelo no interpreta. Devuelve la frase del cliente en limpio y una etiqueta gruesa.

```json
{"frase": "Deme tres", "clase": "tienda"}
```

`clase` es una de `tienda`, `pregunta`, `fuera`, `no_claro`. La frase vuelve a entrar al mismo guion de la sección 4. El aviso le fija el papel:

> Tu ÚNICO trabajo es reescribir lo que dijo ESA PERSONA (el cliente), en limpio, para que el vendedor lo entienda. NO eres el vendedor: nunca respondas, nunca saludes de vuelta, nunca digas si hay o no hay algo. […] Si pide algo que NO está en el catálogo, deja el nombre que dijo: no lo cambies por otro producto.

Como contexto recibe lo último que dijo el asistente, para resolver frases que siguen la conversación:

```json
{"vendedor_acababa_de_decir": "Sí, tomacorriente doble con polo a tierra cuesta…", "cliente_dijo": "Dermatrés."}
```

### C. Reescribe, con ejemplos

Tercera versión: el mismo aviso más veinte pares de entrada y salida. Algunos:

| Vendedor acababa de decir | Cliente dijo | Debe salir |
|---|---|---|
| — | «H me dos cajas de la piceros negros.» | «Agrégame dos cajas de lapiceros negros» |
| «Sí, cuaderno argollado de cien hojas cuesta ocho mil pesos…» | «Dermacinco.» | «Deme cinco» |
| — | «a ver, entonces cómo va quedando la cuenta» | «¿Qué llevo en el carrito?» |
| — | «Ustedes venden morrales.» | «¿Tienen morrales?» |
| — | «oiga y cómo quedó el partido anoche» | (igual), clase `fuera` |

Los ejemplos son de **otra tienda**, una papelería. Dos razones: que el modelo aprenda la forma sin tomar prestados productos, y no contaminar la evaluación con frases de los bancos de prueba.

## 6. El filtro

Reescribir tampoco es seguro por sí solo. Casos reales:

| Se oyó | El modelo escribió |
|---|---|
| «Deme tres bulticos pues» | «Deme tres bombillos LED» |
| «Kss prr tk» (ruido) | «¿Cuánto vale el alicate universal de ocho pulgadas?» |
| «ah ya y ustedes dan factura electrónica» | «¿Dónde quedan?» |
| «Yacajica» | «¿Cuánto cuesta el envío a Yacajica?» |

Cada reescritura que el guion entiende pasa por una función determinista que devuelve el motivo del rechazo, o vacío si se acepta:

```python
def objecion(dicho, frase, entendido, estado_antes, productos, entrega) -> str:
    # 1. Producto: tiene que sonar en lo dicho o venir de la conversación
    # 2. Cantidad > 1: tiene que haberse dicho, en cifras o en palabras
    # 3. Un número dicho que no es del producto («la de ochenta» ≠ lija nº 120)
    # 4. Ciudad: reconocida y sonando en lo dicho
    # 5. Envase dicho ≠ envase del producto («dos bolsas» ≠ tornillos por caja)
    # 6. Un «sí» nunca decide el carrito; un «no» exige que se haya dicho «no»
    # 7. Tema: lo dicho debe traer alguna seña de la intención resultante
```

La comparación es por sonido, tolerante a palabras pegadas o partidas:

```python
def suena_en(palabra: str, dicho: str) -> bool:
    s = _son(plano(palabra))
    sones = [_son(w) for w in plano(dicho).split()]
    sones += [a + b for a, b in zip(sones, sones[1:])]      # «der ma tres» → «dermatres»
    tope = 1 if len(s) <= 5 else 2
    return any(distancia(s, w) <= tope or (len(s) >= 4 and s in w) for w in sones)
```

La regla 7 merece una nota de diseño. La versión C produce fórmulas («¿Qué llevo en el carrito?») que no comparten palabras con lo dicho («cómo va quedando la cuenta»). Comparar palabra por palabra rechazaba reescrituras correctas. La solución fue pedir evidencia del tema: una tabla de señas por intención (`pedido`, `cuenta`, `llevo` para ver el carrito; `cóbreme`, `pagar` para pagar; `mandan`, `domicilio`, `llega` para envíos), comparadas también por sonido.

Si el guion entiende la frase pero el filtro la frena, la llamada dice «no te entendí bien». Es la respuesta honesta: el modelo puso algo que la persona no dijo.

## 7. Latencia

### Dónde se va el tiempo de la reescritura

Medido contra el servidor, con el aviso real:

| Componente | Fichas | Tiempo |
|---|---|---|
| Aviso completo (instrucciones + 20 ejemplos + catálogo de 30 productos) | 2.004 | — |
| Fichas nuevas por consulta (el resto está en la caché de prefijo de llama.cpp) | 36 | 111 ms |
| Salida (`frase` + `clase`) | 21 | 409 ms a 49 fichas/s |
| Red y sobrecarga | — | ≈ 200 ms |

Cuatro decisiones mantienen ese número bajo:

1. **Caché de prefijo.** Con un catálogo de hasta 60 productos, el aviso es idéntico entre consultas y siempre en el mismo orden. Agregar los 20 ejemplos lo llevó de unas 870 a 2.004 fichas sin costo perceptible: solo se evalúan las 36 fichas nuevas.
2. **Salida mínima.** La reescritura media tiene 5,2 palabras. A 49 fichas por segundo, cada palabra que el modelo no escribe son unos 30 ms.
3. **Corte temprano.** La respuesta se lee en flujo. `frase` va primero en el esquema: en cuanto se cierra su comilla, el guion la procesa. Si sirve, se cierra la conexión sin esperar `clase`.
4. **En paralelo.** La reescritura corre a la vez que la búsqueda en la base de conocimiento de la empresa.

Con catálogos de más de 60 productos, al aviso solo van los que suenan a lo dicho y los de la conversación. Ahí se pierde la caché de prefijo; no lo hemos medido a esa escala.

<pre class="mermaid">
sequenceDiagram
    participant C as Cliente
    participant S as Servidor
    participant V as LFM3-Audio
    participant G as Guion
    participant M as MiMo 9B
    C->>S: audio del turno
    Note over S: 600 ms de silencio cierran el turno
    S->>V: transcribir
    V-->>S: «Dermatrés.» (≈ 270 ms)
    S->>G: entender
    G-->>S: otro (≈ 1 ms)
    S->>M: reescribir (flujo)
    M-->>S: frase: «Deme tres» (≈ 600 ms)
    Note over S,M: se corta el flujo, no se espera «clase»
    S->>G: entender «Deme tres» + filtro
    G-->>S: agregar, 3 unidades
    S->>V: leer «¿Agrego tres unidades de…?»
    V-->>C: primer audio (≈ 240 ms)
</pre>

Sumando componentes medidos por separado, un turno normal empieza a sonar cerca de 1,1 s después de que la persona calla (0,6 de silencio, 0,27 de oír, 0,24 de voz) y uno reescrito, cerca de 1,7 s. **No es una medición de extremo a extremo**; es la suma de las partes. Si la reescritura pasa de 1,1 s, la llamada dice «Déjame revisar» para no dejar silencio.

### Concurrencia

Con el modelo de voz en una ranura, la respuesta de una llamada empieza a sonar en 0,3 s. Con dos llamadas, igual casi siempre (1,9 s en el peor 10 %). Con tres ya hay esperas de 4 s. Por eso el tope por defecto es dos, y a la tercera se le dice «todas las líneas están ocupadas» en vez de dejarla esperando.

## 8. Evaluación

### Método

- **Datos:** 14 bancos de frases, 1.362 turnos organizados en conversaciones. Cada turno tiene etiquetada la intención esperada, el producto, la cantidad y la ciudad.
- **Dos versiones de cada frase:** 764 turnos escritos y 598 **transcritos**, es decir, pronunciados y pasados por el reconocedor, con sus errores reales.
- **Bancos a ciegas:** la mayoría se escribieron sin mirar las reglas, para medir generalización del guion.
- **Dos métricas.** *Turno bien:* intención, producto, cantidad, ciudad y frase esperada coinciden. *Turno dañino:* el sistema haría o afirmaría algo equivocado: agregar lo que no se pidió, agregar otro producto u otra cantidad, hablar de otro producto, cotizar otra ciudad.
- **Reproducible sin GPU:** las salidas del modelo quedaron grabadas; la prueba automática las reproduce contra el guion y el filtro.

### Resultados

| Configuración | Turnos bien | % | Dañinos | Mediana del modelo |
|---|---|---|---|---|
| Solo guion | 1.214 | 89,1 | 9 | — |
| A. El modelo decide | 1.228 | 90,2 | **29** | 0,62 s |
| B. Reescribe + filtro | 1.232 | 90,5 | 10 | 0,74 s |
| C. Reescribe + ejemplos, **sin filtro** | 1.248 | 91,6 | 15 | 0,61 s |
| **C. Reescribe + ejemplos + filtro** | **1.251** | **91,9** | **10** | **0,61 s** (p90: 0,73 s) |

<pre class="mermaid">
xychart-beta
    title "Turnos dañinos por configuración (menos es mejor)"
    x-axis ["Solo guion", "A. Decide", "C. Sin filtro", "C. Con filtro"]
    y-axis "Turnos dañinos" 0 --> 30
    bar [9, 29, 15, 10]
</pre>

Por tipo de entrada, con la configuración final:

| | Turnos | Solo guion | Con reescritura | Dañinos |
|---|---|---|---|---|
| Escritos | 764 | 718 (94,0 %) | 730 (95,5 %) | 3 → 3 |
| Transcritos | 598 | 496 (82,9 %) | 521 (87,1 %) | 6 → 7 |

La mejora se concentra donde está el problema: en lo transcrito, 4,2 puntos.

### Qué pasó con los 142 turnos que el guion no entendía

| Desenlace | Turnos |
|---|---|
| Reescritura aceptada | 48 |
| El guion tampoco entiende la frase reescrita | 64 |
| El modelo no la cambió | 13 |
| El filtro la frenó | 16 |
| Sin respuesta del modelo | 1 |

De esos 142, **78 no tenían una intención que rescatar**: eran preguntas legítimas fuera del guion («¿dan factura electrónica?», «¿tienen parqueadero?») o temas ajenos. Ahí el trabajo de la reescritura no es acertar, es clasificar para dar la respuesta adecuada. Los otros 64 sí tenían una intención.

De las 48 aceptadas, 38 produjeron un turno completamente correcto y solo 1 correspondía a una frase que debía quedarse como `otro`.

### La ablación del filtro

Es el resultado que más nos sorprendió. De las 16 reescrituras frenadas:

- **0** habrían producido un turno correcto.
- **5** habrían producido un turno dañino.

En estos datos el filtro no tiene costo: no rechazó nada que sirviera. Hay que leerlo con cuidado (sección 10), pero la dirección es clara.

### El único daño nuevo

A quien dijo «Dímelo» queriendo decir «deme dos», el sistema reescribe «Deme el destornillador…» y pregunta «¿Agrego una unidad?». El medidor lo cuenta como dañino porque la cantidad no coincide. No se agrega nada: el cliente dice que no y sigue. Antes oía «no te entendí».

## 9. Análisis

### Por qué decidir falla

Un catálogo es un mundo cerrado: lo que no está, no existe. Un modelo de lenguaje opera con el supuesto contrario: siempre hay una mejor continuación. Ante «¿venden candados?» y una lista de 30 productos sin candados, la continuación más probable es el producto más cercano. En la configuración A, eso fue guantes.

No es un defecto de este modelo en particular. Es la consecuencia de pedirle una decisión de pertenencia a un sistema que está entrenado para completar. Las reglas, en cambio, codifican el mundo cerrado de forma nativa: buscan, no encuentran y lo dicen.

### Por qué reescribir funciona

La reescritura separa dos competencias que la configuración A mezclaba:

- **Normalización lingüística:** de «H me 10 pliegos» a «Écheme diez pliegos». Es una tarea de lenguaje, donde un modelo de 9B es más que suficiente.
- **Decisión sobre datos:** si hay pliegos, cuánto valen, si se agregan. Es una consulta, donde el código es exacto.

Cada parte hace lo que hace bien, y la interfaz entre ambas es una frase en español, que además es inspeccionable: se puede leer en un registro y entender qué pasó.

### El costo es asimétrico

Rechazar una reescritura correcta cuesta un «¿me lo repites?». Aceptar una incorrecta puede costar un producto equivocado en el carrito o un precio falso dicho en voz alta. Con esa asimetría, el filtro puede ser estricto. Y como verifica contra la entrada (¿esto se dijo?) y no contra el mundo (¿esto es cierto?), se resuelve con distancia de edición y sin otro modelo.

### Los ejemplos pesan más que las instrucciones

Pasar de B a C no cambió ninguna instrucción del aviso. Agregó veinte pares. Solo con eso el acierto subió de 1.232 a 1.240 y la latencia bajó 130 ms, porque el modelo dejó de producir frases largas y empezó a usar las fórmulas cortas que el guion espera. Los otros 11 turnos, hasta 1.251, vinieron de adaptar el filtro a esas fórmulas (la regla 7, por señas del tema). Para un modelo pequeño, mostrar el formato de salida resultó más efectivo que describirlo.

### Dónde no sirve

El 45 % de las reescrituras termina en «el guion tampoco la entiende». La reescritura no amplía lo que el sistema sabe hacer; solo acerca lo dicho a lo que ya sabe. Si el cliente pregunta por garantías y el guion no tiene garantías, ninguna reescritura lo arregla. El techo lo pone el guion.

## 10. Amenazas a la validez

- **No hay comparación con un modelo grande.** El título afirma que no hizo falta, no que uno grande lo haría peor. Esa prueba está pendiente.
- **El filtro se ajustó sobre los mismos bancos con los que se mide.** El resultado «0 rechazos de frases correctas» está inflado por eso. Una comprobación con 16 frases nuevas dio: 10 entendidas por el guion, 3 rescatadas, 1 ajena y 2 frenadas. Es una muestra pequeña.
- **Lo transcrito se generó con voz sintética,** no con personas. Faltan acentos reales, ruido de calle y micrófonos de celular.
- **Un solo dominio y un solo idioma:** ferretería, español de Colombia, catálogo de 30 productos.
- **La latencia de extremo a extremo no está medida,** solo sus componentes.
- **La configuración A se probó con un disparador más amplio** que las B y C (también cuando el guion no encontraba el producto), así que hizo más consultas. La comparación de daño sigue siendo válida en dirección, no en magnitud exacta.

## 11. Trabajo relacionado

Ninguna de las piezas es nueva por separado:

- Reescribir la consulta antes del módulo de comprensión para reducir errores de voz es una línea establecida en Alexa: [Chen y otros, 2020](https://arxiv.org/pdf/2002.05607.pdf); [reescritura personalizada](https://arxiv.org/pdf/2011.04748).
- Parafrasear hacia un sublenguaje controlado que un sistema determinista interpreta: [Shin y otros, EMNLP 2021](https://arxiv.org/abs/2104.08768).
- Corrección de transcripciones con modelos de lenguaje y verificación contra invenciones: [Fewer Hallucinations, More Verification](https://arxiv.org/pdf/2505.24347).
- Evidencia de que los modelos pequeños se degradan llevando solos una llamada: [CallScreenBench](https://arxiv.org/pdf/2608.01033).
- Presupuestos de latencia en agentes en cadena: [guía de Twilio](https://www.twilio.com/en-us/blog/developers/best-practices/guide-core-latency-ai-voice-agents).

Lo que aportamos es la combinación medida: activación solo ante fallo del guion, retorno al mismo guion, un filtro por evidencia fonética contra la entrada, y la comparación directa entre decidir y reescribir con una métrica de daño separada del acierto.

## 12. Conclusión

Antes de elegir un modelo más grande, vale la pena hacerse otra pregunta: **¿cuál es la tarea más pequeña que resuelve mi problema?**

En nuestro caso, la tarea pasó de «entiende al cliente y decide qué hacer» a «escribe bien esta frase». Con ese cambio, un modelo de 9B cuantizado a 4 bits, compartiendo tarjeta con un modelo de voz de 2B, alcanzó 91,9 % de turnos correctos con una latencia de 0,61 s en uno de cada diez turnos, y sin costo por llamada.

Cada responsabilidad que se le quita al modelo es un error que ya no puede cometer.

---

*Mediciones internas del 23 de noviembre de 2026. Este texto se redactó con ayuda de inteligencia artificial.*

<script type="module">
  import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs";
  mermaid.initialize({ startOnLoad: true, theme: "neutral" });
</script>
