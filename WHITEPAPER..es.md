Beatriz Epistemic Gate: Especificaciòn Técnica y Arquitectura de defensa contra el Envenenamiento en Fine-Tuning. 
Autor: Eduardo Ayala Tovar
Año: 2026
Licencia: PolyForm Noncommercial License 1.0.0
Afiliación: Investigación Independiente / Soberanía en IA
Hardware de Orquestación: Toshiba Satellite U205 (2006, 2 GB RAM)
Motor de Cómputo Experimental: Kaggle T4 x2 (Costo total: $0 USD)
________________________________________
Resumen (Abstract)
La industria de la inteligencia artificial contemporánea sostiene el dogma de que la investigación avanzada en seguridad, alineación y defensa de modelos requiere infraestructura masiva y presupuestos multimillonarios. Este artículo demuestra lo contrario al presentar Beatriz, una arquitectura de compuerta epistémica ligera diseñada para prevenir el envenenamiento quirúrgico de datos durante el fine-tuning. A través de una serie de 16 experimentos sistemáticos (EXP08–EXP16) validados a lo largo de 5 arquitecturas (desde GPT-2 de 124M hasta Phi-3-mini-4k-instruct de 3.8B), demostramos que el envenenamiento epistémico es invisible a las métricas agregadas convencionales, pero puede contrarrestarse eficazmente introduciendo un proxy defensivo no invasivo entre la fuente generativa y el modelo en aprendizaje. Los resultados confirman que el filtrado basado en un corpus ancla estático aporta el 65% del beneficio defensivo sin alterar el bucle de entrenamiento del estudiante, mientras que un término de pérdida contrastiva adicional (Softplus) consolida un margen de veracidad robusto (+4.19 ± 0.08 en un conjunto held-out escalado de 30 hechos multidominio), manteniendo un determinismo bit-exacto y una latencia de decisión de ~0.1 ms por llamada.
________________________________________
1. Introducción: El Mito de la Infraestructura Masiva
El desarrollo actual de los Modelos de Lenguaje de Gran Escala (LLMs) está dominado por la concentración de recursos. Se asume comúnmente que auditar vulnerabilidades, entrenar contrafuegos o estudiar dinámicas de colapso representacional exige clústeres empresariales.
Esta investigación nace de una premisa contraria: el rigor metodológico, la claridad arquitectónica y la soberanía del código superan a la fuerza bruta de capital. Toda la infraestructura de orquestación, diseño y análisis de esta serie experimental se ejecutó en un ordenador portátil convencional de 2006 (Toshiba Satellite U205, 2 GB de RAM), utilizando recursos públicos gratuitos de GPU (Kaggle T4 x2) con un costo monetario total de cero dólares ($0).
El objetivo de este documento es exponer la arquitectura técnica y los resultados definitivos de la serie experimental, proveyendo a desarrolladores independientes y pequeñas startups de un mecanismo defensivo portable y de bajo costo para blindar sus pipelines de entrenamiento.
________________________________________
2. Marco Teórico: El Envenenamiento Quirúrgico e Invisible
En las iteraciones tempranas de la serie (EXP12–EXP14), se documentó un fenómeno crítico en la seguridad de los LLMs: el envenenamiento epistémico malicioso es quirúrgico e invisible por diseño.
Cuando un atacante inyecta desinformación específica en el flujo de entrenamiento (stream), el modelo destruye selectivamente los hechos atacados hasta llevarlos a una condición de indiferencia exacta (train margin ≈ 0.0), mientras que las métricas agregadas globales (como la perplejidad general o el rendimiento en otras tareas) mejoran o se mantienen estables debido al fine-tuning genérico sobre prosa fluida.
Los sistemas de monitoreo tradicionales basados en estadísticas agregadas son ciegos ante este tipo de ataques. Se requiere, por tanto, una intervención a nivel de arquitectura de entrenamiento: un mecanismo capaz de verificar la verdad objetiva de forma independiente al colapso de la función de pérdida tradicional (Cross-Entropy).
________________________________________
3. Arquitectura del Sistema: La Compuerta Epistémica
Para resolver este problema sin obligar a reestructurar los complejos bucles de optimización de los modelos base, se diseñó Beatriz, una compuerta epistémica estructurada bajo el patrón de un proxy defensivo de dos capas:
1.	El Corpus Ancla Inmutable: Un conjunto versionado de hechos canónicos, separado del flujo de entrenamiento del modelo, que actúa como fuente de verdad objetiva.
2.	La Compuerta Vectorial Denso (DenseVectorGateCached): Un componente de baja latencia que evalúa el texto candidato generado, lo compara contra el corpus mediante similitud de cosenos en espacios de embedding (calculados a través de un oráculo congelado) y emite un veredicto (VERIFIED, CONTRADICTED, UNKNOWN, INVALID).
3.1. Función de Pérdida Compuesta
En su rama avanzada (Beatriz), el sistema opera modificando la optimización mediante una pérdida compuesta:
Ltotal=α⋅Lce+β⋅LcontrastivaLtotal=α⋅Lce+β⋅Lcontrastiva
•	LceLce (α=0.5α=0.5): Pérdida de máxima verosimilitud estándar (Cross-Entropy) para preservar la fluidez lingüística.
•	LcontrastivaLcontrastiva (β=1.0β=1.0): Término basado en una función Softplus sobre el margen entre las probabilidades logarítmicas de la verdad y la falsedad detectada: Este termino no satura mientras el margen sea finito, manteniendo presión activa sobre el modelo incluso después de que la entropıa cruzada se agote. Este termino no satura mientras el margen sea finito, manteniendo presión activa sobre el modelo incluso después de que la entropıa cruzada se agote. 
________________________________________
4. Metodología Experimental
La serie experimental abarcó múltiples arquitecturas de código abierto para demostrar la universalidad del fenómeno:
•	Modelos evaluados (EXP01–EXP14): GPT-2 (124M), Qwen-2.5-0.5B, TinyLlama-1.1B, Pythia-1.4B y Phi-3-mini-4k-instruct (3.8B).
•	Configuración Estándar de Cierre (EXP15–EXP16): 
o	Modelo Base: microsoft/Phi-3-mini-4k-instruct (targets LoRA: qkv_proj, o_proj, r=8r=8, α=16α=16).
o	Hiperparámetros: 8 épocas, 60 muestras por época, semillas fijas deterministas (SEEDS = [11, 22, 33]).
o	Validación: Perplejidad ponderada por tokens sobre un corpus extendido de 40 oraciones neutrales multidominio.
________________________________________
5. Resultados Clave (EXP15 – EXP16)
5.1. Ablación Quirúrgica (EXP15)
Se compararon tres ramas de entrenamiento bajo idénticas condiciones: NONE (sin compuerta ni contraste), GATE_ONLY (filtro activo, β=0β=0) y BEATRIZ (filtro activo con término contrastivo, β=1.0β=1.0).
Rama	Margen de Entrenamiento	Margen Held-Out (n=2)	Perplejidad (PPL)
BASE (sin entrenar)	+1.34+1.34	+1.90+1.90	12.712.7
NONE	−0.03±0.02−0.03±0.02	+3.57±0.17+3.57±0.17	30.930.9
GATE_ONLY	+7.46±0.24+7.46±0.24	+5.08±0.09+5.08±0.09	58.858.8
BEATRIZ	+10.13±0.07+10.13±0.07	+5.91±0.07+5.91±0.07	86.386.3
Desglose del efecto en el held-out: El filtrado puro (GATE_ONLY) aporta el 65% del beneficio defensivo total; el término contrastivo (BEATRIZ) añade el 35% restante, aunque con un costo de perplejidad 1.8 veces mayor por unidad de margen. Esto valida una recomendación práctica para startups: GATE_ONLY es el punto de operación por defecto (no requiere modificar el loop del estudiante y satura en 2 épocas); BEATRIZ se reserva para escenarios que exigen la máxima pureza epistémica a cambio de un mayor costo de PPL.
5.2. Determinismo Bit-Exacto Verificado
Las corridas independientes ejecutadas en días distintos bajo el mismo protocolo semillero arrojaron resultados bit-idénticos en todas las épocas y variables (hashes SHA-256 idénticos en adaptadores LoRA). Esto demuestra que el protocolo experimental es rigurosamente reproducible, una garantía poco común en la investigación de LLMs.
5.3. Escalado del Held-Out y Matriz de Confusión del Gate (EXP16)
Para erradicar la debilidad estadística inicial (n=2n=2 hechos en held-out), el EXP16 escaló el banco de pruebas a 30 hechos multidominio no atacados (física, medicina, historia, biología, informática) e integró un registro estricto de la matriz de confusión del filtro.
•	Held-Out Escalado (n=30n=30): El margen positivo se consolida en +4.19±0.08+4.19±0.08 (BEATRIZ), probando de forma concluyente que la mejora en hechos no atacados es un fenómeno sistémico del fine-tuning guiado por prosa fluida y anclada, no una anomalía estadística.
•	Rendimiento del Gate (Matriz de Confusión Real): 
o	Precisión (Precision): 0.930.93
o	Exhaustividad (Recall): 0.800.80
o	Interpretación: El filtro acierta en 93 de cada 100 bloqueos de mentiras en el flujo, atrapando el 80% de los intentos maliciosos sin colapsar por falsos positivos masivos.
o	Latencia lógica: ∼0.1 ms∼0.1 ms por decisión en camino cacheado (∼10,000∼10,000 decisiones/segundo por hilo).
________________________________________
6. Limitaciones Declaradas
Con honestidad científica, se reconocen los siguientes límites en el estado actual del proyecto:
1.	Tamaño del Corpus Ancla: El corpus actual consta de 36 elementos totales (6 de entrenamiento, 30 de held-out). Es necesario escalar a cientos de anclas para evaluar el comportamiento en dominios abiertos masivos.
2.	Latencia del Camino Vivo: La medición de 0.1 ms0.1 ms corresponde a la lógica en memoria cacheada; en despliegue de producción, el embedding del texto candidato depende de un paso de inferencia (forward pass) del oráculo, lo que sugiere la necesidad futura de un encoder ligero especializado (ej. E5-small).
3.	Licenciamiento Comercial: El código actual opera bajo la licencia PolyForm Noncommercial 1.0.0 con fines de auditoría, investigación y defensa, requiriendo acuerdos específicos para explotación comercial directa.
________________________________________
7. Conclusión: La Soberanía Científica es Posible
La serie experimental de Beatriz demuestra que la seguridad y la gobernanza de modelos de lenguaje no son monopolio de los grandes conglomerados tecnológicos. Mediante un diseño arquitectónico limpio —un proxy epistémico no invasivo acoplado a un corpus ancla verificado— es posible neutralizar ataques quirúrgicos de envenenamiento de datos con recursos modestos.
Invitamos a la comunidad de desarrolladores independentistas, investigadores académicos y pequeñas startups a replicar estos experimentos utilizando el código disponible en el repositorio oficial y que poco a poco se iran publicando el resto de la serie periodicamente para su uso, demostrando que la verdadera innovación en IA nace de la libertad, el rigor y el código abierto.
