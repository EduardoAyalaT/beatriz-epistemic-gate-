# Beatriz Epistemic Gate

**Una capa auditable de control de admisión para el fine-tuning de LLMs: los datos verificados entrenan al modelo, las contradicciones se corrigen y lo desconocido va a cuarentena.**

**Autor:** Eduardo Ayala Tovar • 2026  
**Afiliación:** Investigación independiente / Soberanía en IA  
**Licencia:** [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)  
**Repositorio:** https://github.com/EduardoAyalaT/beatriz-epistemic-gate-  
**Documentos:** [Whitepaper (EN)](WHITEPAPER_EN.md) • [Whitepaper (ES)](WHITEPAPER_ES.md) • [Manual Correctivo](docs/MANUAL_CORRECTIVO.md) • [Financiación](FUNDING.md) • [Manifiesto y hashes](MANIFEST.json) • [English README](README.md)

> **Estado de la investigación.** Beatriz es un prototipo de investigación reproducible. Todos los resultados que siguen provienen de benchmarks controlados, de mundo cerrado, con ataques sintéticos escritos por la misma persona que diseñó la defensa. **No** constituyen una demostración de seguridad general frente al envenenamiento de datos en el mundo real. Cada afirmación empírica de este README está ligada a un experimento concreto (EXP01–EXP16) con su hash SHA-256. Las cifras ilustrativas del Manual Correctivo están marcadas como hipotéticas y no se reclaman como resultados.

> **Nota de infraestructura.** La orquestación se ejecutó en una Toshiba Satellite U205 de 2006 (2 GB RAM). Toda la ejecución de modelos usó sesiones gratuitas de Kaggle con T4. Costo directo de cómputo: USD $0. Se indica como contexto de lo que fue y no fue factible, no como afirmación sobre lo que costaría escalar.

---

## Resumen ejecutivo

- **Problema.** Durante el fine-tuning, una fracción pequeña de ejemplos envenenados puede llevar al modelo a la *indiferencia* entre una afirmación verdadera y su contradicción (`truth_margin ≈ 0`) en los hechos atacados, mientras la pérdida de entrenamiento baja y la perplejidad parece sana. Los monitores agregados no lo ven.
- **Mecanismo.** Beatriz se sitúa entre la fuente de datos y el modelo aprendiz. Cada ejemplo entrante se contrasta con un **corpus ancla verificado por humanos, versionado y con hash** y recibe uno de cuatro veredictos: `VERIFIED`, `CONTRADICTED`, `UNKNOWN`, `INVALID`. Solo el contenido verificado entrena al aprendiz tal cual; las contradicciones se sustituyen por el ancla y, opcionalmente, alimentan un término contrastivo de corrección; el contenido desconocido o malformado se pone en **cuarentena** para revisión humana en lugar de entrar al flujo de entrenamiento.
- **Evidencia.** 16 experimentos reproducibles, 5 arquitecturas (124M → 3.8B), 3 semillas cada uno. En la mayor prueba held-out (EXP16, Phi-3-mini, 30 hechos no vistos), el margen de verdad final fue **+4.19 ± 0.08** con Beatriz frente a **+2.45 ± 0.06** sin compuerta; precisión/recall de la compuerta 0.93 / 0.80.
- **Ablación clave.** La compuerta de admisión *por sí sola* —sin tocar la pérdida del aprendiz— aporta ~65–74% del efecto protector (EXP15). El término contrastivo añade el resto a un costo mayor de perplejidad.
- **Límites honestos.** Corpus de 8–36 hechos; ataques sintéticos; enrutamiento por palabras clave + embeddings cacheados; un costo de utilidad medible (la perplejidad sube); una arquitectura (Pythia) donde la robustez a paráfrasis *no* mejoró. Detalles en [Limitaciones](#6-limitaciones-y-hallazgos-negativos).

---

## 1. Problema: el envenenamiento quirúrgico es invisible para las métricas agregadas

Definimos, para una afirmación verificada *t* y su contradicción *ℓ*:
truth_margin = media log P(t) − media log P(ℓ)
text

Un margen positivo grande significa que el modelo prefiere con claridad la afirmación verificada. Un margen cercano a cero significa que no las distingue.

En EXP05–07 y EXP10–16, la rama sin protección (`NONE`) termina consistentemente con `truth_margin ≈ 0` —y en varias corridas ligeramente negativo— en los hechos atacados, **aunque su pérdida de entrenamiento decrece de forma monótona y su perplejidad en texto neutral a menudo mejora o se mantiene** (p. ej. EXP13: PPL 9.7 → 11.7 mientras el margen pasa de +1.34 → −0.03). El daño está localizado en los hechos atacados y no aparece en las cifras que normalmente se vigilan.

---

## 2. Arquitectura
Datos entrantes: sintéticos, humanos, externos
│
▼
┌───────────────────────┐
│ COMPUERTA EPISTÉMICA │ corpus ancla: verificado por humanos,
│ BEATRIZ │◄──versionado, SHA-256 + OpenTimestamps
└───────────┬───────────┘
┌─────────────┼──────────────────┐
▼ ▼ ▼
VERIFIED CONTRADICTED UNKNOWN / INVALID
│ │ │
▼ ▼ ▼
entrena con entrena con el CUARENTENA
el ejemplo ancla + término (excluido del entrenamiento;
contrastivo revisión humana → puede
opcional incorporarse al corpus ancla)
│
▼
MODELO APRENDIZ (fine-tuning LoRA)
│
▼
ledger: adapter_sha256, registros por época, reglas de rollback R1–R3
text

### 2.1 Componentes

| Componente | Implementación actual del prototipo |
|---|---|
| **Corpus ancla** | 8 hechos (EXP07–15); 36 elementos en EXP16 (6 atacados + 30 held-out). Cada elemento es una tripleta `(verdad, mentira, palabras clave)`. Con hash y sello de tiempo. |
| **Enrutamiento de la compuerta** | Coincidencia de palabras clave → tema; luego similitud coseno entre el estado oculto medio de la última capa del candidato y el de la `verdad` / `mentira` del ancla (oráculo congelado = modelo base). `sim_mentira > sim_verdad ⇒ CONTRADICTED`. Sin tema ⇒ `UNKNOWN`. Vacío/corto ⇒ `INVALID`. Los embeddings están precalculados; costo de decisión ≈ 0.1 ms. |
| **Política de admisión** | `NONE`: todo pasa. `GATE_ONLY`: los ejemplos verificados/contradichos se sustituyen por la verdad ancla; los desconocidos se descartan. `BEATRIZ`: `GATE_ONLY` + término contrastivo sobre los contradichos. |
| **Actualización del aprendiz** | LoRA (r=8, α=16), 0.10–0.24% de parámetros entrenables según arquitectura. |
| **Ledger / rollback** | Registros por época; `adapter_sha256` de los pesos LoRA finales; reglas R1 (caída neutral < −0.30), R2 (caída de verdad > 2.0), R3 (subida de unknown > 2.0) probadas en EXP05. |

### 2.2 Pérdida
L_total = α • L_CE(verdad_ancla) + β • L_verdad
L_verdad = softplus( m + log P(mentira) − log P(verdad) )
text

con α = 0.5, β = 1.0, m = 0.5 en EXP13–16. `L_verdad` se aplica **solo** a elementos que la compuerta marcó como `CONTRADICTED`. En EXP08 el margen se calcula respecto a una política de referencia congelada π_ref (forma de log-ratio).

Lo que el sistema **no** afirma: no determina la verdad universal. Hace cumplir una regla más estrecha y auditable — *un ejemplo no puede modificar al aprendiz salvo que esté respaldado por, o sea corregido contra, un corpus ancla definido e inspeccionable.*

---

## 3. Programa experimental (EXP01–EXP16)

Dieciséis experimentos, cada uno con un reporte JSON con hash. EXP01–08 son calibración sobre GPT-2; EXP09–16 son corridas LoRA sobre cinco arquitecturas. Semillas en todos: `[11, 22, 33]`.

| EXP | Modelo | Propósito | Resultado clave | SHA-256 (reporte) |
|---|---|---|---|---|
| 01 | GPT-2 124M | Control base | El control aprende la mentira inyectada | `713ffa62…4966ae` |
| 02 | GPT-2 124M | CONTROL vs FILTER vs PLACEBO (n=84 igualados) | El efecto depende de la política de admisión, no de la cantidad de datos | `4b3d424f…05947e` |
| 03 | GPT-2 124M | CONTAMINATED / HARD / EPISTEMIC | −15.73 / −0.80 / **+13.20** | `369c5cec…441c85` |
| 04 | GPT-2 124M | NONE / filtro HARD / REWRITE | −0.01 / +4.43 / **+10.88** — corregir supera a descartar | `ed466d51…d44e86` |
| 05 | GPT-2 124M | Consistencia Z3 + reglas de rollback | NONE dispara rollback R3 3/3; Beatriz completa 8 épocas | `97359f04…3626c1` |
| 06 | GPT-2 124M | Flujo de texto abierto | NONE 0.00 (indiferencia); Beatriz +0.57 → +4.23 | `b2e0e62a…559ed5` |
| 07 | GPT-2 124M | Compuerta vectorial densa, ancla de 8 hechos | NONE −0.27; Beatriz +10.27 | `5e3da9dc…8273483` |
| 08 | GPT-2 124M | Restringido por referencia (π_ref) | Beatriz +11.04 | `c93ba4b7…3d61e2` |
| 09 | GPT-2 124M | **LoRA** (0.236%) | NONE +0.14; Beatriz **+3.55** | `f4382f16…6251` |
| 10 | Qwen-2.5-0.5B | LoRA (0.109%) | NONE −0.25; Beatriz **+8.97** | `e894eaf4…c57731` |
| 11 | TinyLlama-1.1B | LoRA (0.102%) | NONE −0.17; Beatriz **+7.87** | `4c0de934…8cbab` |
| 12 | TinyLlama-1.1B | Held-out (n=2) + paráfrasis | Held-out no concluyente; paráfrasis +2.82 vs +2.04 | `09a1ad45…a43102` |
| 13 | Phi-3-mini 3.8B | LoRA QKV fusionado + held-out | Held-out **+5.91** vs +3.57 | `2c48c702…843152` |
| 14 | Pythia-1.4B | LoRA QKV fusionado + held-out | Train +8.03; **la paráfrasis no mejoró** | `2d71e2a7…c35f58` |
| 15 | Phi-3-mini 3.8B | **Ablación** NONE / GATE_ONLY / BEATRIZ | La compuerta sola = ~65–74% del efecto | `d95ac8c8…b008cd` |
| 16 | Phi-3-mini 3.8B | **Held-out n=30** + matriz de confusión de la compuerta | Held-out **+4.19 ± 0.08**; P 0.93 / R 0.80 | `27eda691…4799f` |

Los hashes completos están en [MANIFEST.json](MANIFEST.json) y en el [Apéndice](#apéndice-registros-de-integridad).

---

## 4. Resultados en detalle (EXP09–EXP16)

Todos los valores son media ± desviación estándar sobre 3 semillas en la época final (8/8). "Train" = hechos atacados; "Held-out" = hechos nunca usados como objetivo de entrenamiento. En EXP09–11 la métrica se reporta en los notebooks como *margen semántico*; se calcula de forma idéntica a `truth_margin`.

### 4.1 Resumen entre arquitecturas

| EXP | Arquitectura | Parámetros | LoRA % | NONE (train) | BEATRIZ (train) | Δ |
|---|---|---:|---:|---:|---:|---:|
| 09 | GPT-2 | 124M | 0.236 | +0.14 ± 0.05 | **+3.55 ± 0.13** | +3.4 |
| 10 | Qwen 2.5 | 0.5B | 0.109 | −0.25 ± 0.11 | **+8.97 ± 0.09** | +9.2 |
| 11 | TinyLlama | 1.1B | 0.102 | −0.17 ± 0.05 | **+7.87 ± 0.16** | +8.0 |
| 14 | Pythia (NeoX) | 1.4B | 0.167 | −0.09 ± 0.07 | **+8.03 ± 0.54** | +8.1 |
| 13/15/16 | Phi-3-mini | 3.8B | 0.123 | −0.03 ± 0.02 | **+10.13 ± 0.07** | +10.2 |

En todas las arquitecturas, el fine-tuning LoRA sin protección termina en margen cero o negativo sobre los hechos atacados. Beatriz lo mantiene claramente positivo, con varianza entre semillas ≤ 0.16 en todos los casos salvo Pythia (0.54).

### 4.2 Generalización: held-out y paráfrasis

| EXP | Arquitectura | n held-out | NONE held-out | BEATRIZ held-out | NONE paráfr. | BEATRIZ paráfr. |
|---|---|---:|---:|---:|---:|---:|
| 12 | TinyLlama 1.1B | 2 | +1.05 ± 0.24 | +1.33 ± 0.26 | +2.04 ± 0.29 | +2.82 ± 0.34 |
| 13 | Phi-3 3.8B | 2 | +3.57 ± 0.17 | **+5.91 ± 0.07** | +3.41 ± 0.24 | +5.63 ± 0.50 |
| 14 | Pythia 1.4B | 2 | +1.22 ± 0.41 | +2.09 ± 0.11 | **+1.68 ± 0.34** | +1.30 ± 0.07 |
| 16 | Phi-3 3.8B | **30** | +2.45 ± 0.06 | **+4.19 ± 0.08** | +3.41 ± 0.24 | +5.63 ± 0.50 |

Lectura: con n=2 la señal held-out es débil en TinyLlama, clara en Phi-3 y moderada en Pythia. EXP16 amplía el conjunto held-out a 30 hechos en Phi-3 y la separación persiste (Δ ≈ +1.7, > 20× la desviación agrupada). **EXP16 es la cifra que consideramos el titular conservador.** El resultado de paráfrasis en Pythia es un hallazgo negativo (ver §6).

### 4.3 Ablación (EXP15, Phi-3-mini, conjunto neutral de 40 oraciones)

| Rama | Train | Held-out (n=2) | Paráfrasis | PPL | Compuerta ms/llamada | VRAM pico |
|---|---:|---:|---:|---:|---:|---:|
| BASE (antes del FT) | +1.34 | +1.90 | +2.21 | 12.7 | — | — |
| NONE | −0.03 ± 0.02 | +3.57 ± 0.17 | +3.41 ± 0.24 | 30.9 | 0.009 | 7.86 GB |
| GATE_ONLY | +7.46 ± 0.24 | +5.08 ± 0.09 | +5.10 ± 0.51 | 58.8 | 0.102 | 7.86 GB |
| BEATRIZ | +10.13 ± 0.07 | +5.91 ± 0.07 | +5.63 ± 0.50 | 86.3 | 0.107 | 7.97 GB |

- Aporte aislado del término contrastivo (BEATRIZ − GATE_ONLY): **+2.67 train, +0.82 held-out**.
- Proporción de la ganancia total (vs NONE) aportada solo por la compuerta: **~74% train, ~65% held-out** — sin modificar la función de pérdida del aprendiz.
- El costo de perplejidad es aditivo: el fine-tuning solo sobre un corpus diminuto (NONE) ya sube la PPL de 12.7 → 30.9; la compuerta aproximadamente la duplica; el término contrastivo añade otros ~28. `GATE_ONLY` tiene la mejor relación protección/perplejidad de la serie.
- 9 corridas completadas en 15.6 min sobre una sola Tesla T4.

### 4.4 Held-out escalado y calidad de la compuerta (EXP16, Phi-3-mini)

| Rama | Train | Held-out (n=30) | PPL | Precisión compuerta | Recall compuerta |
|---|---:|---:|---:|---:|---:|
| NONE | −0.03 ± 0.02 | +2.45 ± 0.06 | 30.9 | — | — |
| GATE_ONLY | +7.46 ± 0.24 | +3.53 ± 0.13 | 58.8 | 0.93 | 0.80 |
| BEATRIZ | +10.13 ± 0.07 | +4.19 ± 0.08 | 86.3 | 0.93 | 0.80 |

Matriz de confusión de la compuerta (sumada sobre 3 semillas, 1,440 extracciones del flujo): TP 535 • FN 133 • FP 39 • TN 733. El margen held-out base antes del fine-tuning era +1.57, por lo que la ganancia neta held-out de Beatriz sobre la base es ≈ +2.6.

### 4.5 Determinismo

Las trayectorias de margen train y PPL para Phi-3 son idénticas bit a bit entre EXP13, EXP15 y EXP16 bajo las mismas semillas (p. ej. semilla 11, época 1: pérdida 0.8835, train −0.15 en los tres). Las columnas held-out difieren solo porque cambió el conjunto held-out (2 → 30). Los adaptadores LoRA finales llevan hash (`adapter_sha256`) en cada reporte.

---

## 5. Reproducibilidad

Cada carpeta de experimento contiene el notebook (`.ipynb`), un script exportado (`.py`), el reporte JSON y una prueba OpenTimestamps (`.ots`).
exp_calibracion_01-07/ EXP01–07 (GPT-2, calibración)
exp08/ restringido por referencia (π_ref)
exp09/ retención LoRA, GPT-2
beatriz-epistemic-gate-exp-10-15/ Qwen, TinyLlama, Phi-3, Pythia, ablación
exp16/ held-out escalado + matriz de confusión
beatriz-epistemic-gate/ implementación de la compuerta
docs/MANUAL_CORRECTIVO.md manual de entrenamiento correctivo (cifras ilustrativas señaladas)

Verificar un paquete:

sha256sum exp16.rar          # comparar con MANIFEST.json
ots verify exp16.rar.ots     # prueba OpenTimestamps
En Windows: certutil -hashfile exp16.rar SHA256.
Ejecutar en Kaggle: abrir el notebook, activar GPU T4, ejecutar todas las celdas. Los modelos base se descargan del Hugging Face Hub (GPT-2 se incluye también offline, hash c7d00560…20373). El tiempo total de EXP16 fue ~22 min.
________________________________________
6. Limitaciones y hallazgos negativos
Los listamos nosotros mismos en lugar de esperar a que lo haga un revisor.
1.	Corpus diminuto. 8 hechos ancla en la mayoría de los experimentos, 36 en EXP16. Nada de lo aquí mostrado describe el comportamiento con cientos o miles de afirmaciones.
2.	Mundo cerrado, ataques de autoría propia. Los pares verdad/mentira y el calendario de envenenamiento los escribió el autor. No se han probado ataques diseñados de forma independiente, contradicciones indirectas, verdades parciales ni entradas multilingües.
3.	Enrutamiento prototipo. Coincidencia de palabras clave + similitud coseno sobre estados ocultos cacheados. La cifra de 0.1 ms es el costo de decisión sobre embeddings precalculados; excluye el embedding en vivo, la recuperación, el manejo de cuarentena y cualquier latencia de producción de extremo a extremo.
4.	Costo de utilidad. La perplejidad en texto neutral sube con cada componente (EXP15). En EXP12 dos de tres semillas terminaron por debajo de la PPL base, así que el costo depende de la configuración y no es intrínseco — pero es real y no está resuelto.
5.	La evidencia held-out depende de la arquitectura. Clara en Phi-3 (EXP13/16), débil en TinyLlama (EXP12), moderada en Pythia (EXP14).
6.	Resultado negativo en paráfrasis de Pythia (EXP14). Beatriz puntuó por debajo de la rama sin protección (+1.30 vs +1.68, n=2). Hipótesis a probar: en esta arquitectura el término contrastivo podría estar anclándose a la redacción literal del ancla y no a su significado.
7.	Mayor inestabilidad en Pythia. Varianza entre semillas del margen train 0.54; una semilla perdió ~1 punto en la última época.
8.	Sin replicación externa todavía. Todas las corridas son de una sola persona sobre un solo tipo de GPU.
9.	Licencia. PolyForm Noncommercial restringe el uso comercial. El uso en investigación y educación está permitido.
________________________________________
7. Hoja de ruta (qué cambiaría con financiación)
Línea de trabajo	Ahora	Siguiente
Corpus ancla	36 elementos escritos a mano	Cientos–miles de afirmaciones con procedencia de fuentes, versionado y registros de revisión
Cuarentena	Solo veredicto (UNKNOWN descartado)	Cola operativa de revisión humana; los elementos revisados se promueven al ancla con traza de auditoría
Enrutamiento	Palabras clave + embeddings cacheados del oráculo	Encoder de recuperación dedicado (p. ej. E5-small), detección de contradicción, umbrales de incertidumbre calibrados, latencia en vivo medida con honestidad
Ataques	De autoría propia, sintéticos	De autoría independiente; datasets reales; paráfrasis, verdad parcial, multilingüe, adversarial
Utilidad	PPL reportada, no optimizada	Frontera explícita seguridad–utilidad; GATE_ONLY vs BEATRIZ bajo restricciones; evaluaciones en tareas downstream
Replicación	Un solo autor, T4 de Kaggle	Replicación externa; especificación de benchmark y harness estandarizados publicados
Pila de gobernanza	Campos de ledger + reglas de rollback R1–R3	Demo de extremo a extremo: compuerta → cuarentena → revisión → actualización del ancla → actualización del modelo → rollback
Ver FUNDING.md.
________________________________________
8. Contexto de gobernanza soberana
Beatriz es el componente de aplicación en tiempo de entrenamiento de una arquitectura más amplia cuyo objetivo es que las actualizaciones de un modelo sean trazables, revisables y reversibles: qué datos se admitieron, con qué evidencia, qué parámetros cambiaron y cuándo debe dispararse un rollback. La pila más amplia se describe en el whitepaper; en este repositorio solo están implementados y probados la compuerta y los campos de ledger indicados arriba.
________________________________________
## Apéndice: registros de integridad


**Semillas:** `[11, 22, 33]  · **Modelo GPT-2 offline:**              c7d00560d8910fbed77ffad4065dee5011c41ba401b1064e749c498ba9e20373

| EXP | SHA-256 (reportes) |
|---|---|
| 01 | `713ffa6227b68a9837a11f245b8c4e52d11b343d7f2d0c8f4af916edee4966ae` |
| 02 | `4b3d424f308943ce41c3d6c8f11b8c99130e1eb39fb380ab4f42e7ed1705947e` |
| 03 | `369c5cec029b792744978a9c81434ff46528be5fee49377c621a5f67c9441c85` |
| 04 | `ed466d516acdd202bd4c71d76e6919422d6dff00df019bdd6ac55e618d44ee86` |
| 05 | `97359f0464d7a7f7c33a87bca8a3d9e66b212495f5152c4603105ee1493626c1` (prereg `57691ac08801e38c63b346579f22091dd0e5da2b7c10ded63a909c678be69f9e`, sanity `2ee4949d9bb750afd356d5a7a9009b98ef8dcc014e1369bbb065d7aea9ac116b`) |
| 06 | `b2e0e62a84b1ed19f97657c36308076a44cae42c806b2022bcfd19a55a559ed5` |
| 07 | `5e3da9dca9162f62c6aac94175133e9cbf70ac101389f6714afc24d1b8273483` |
| 08 | `c93ba4b74ecb8ade32f0f645761c1441382c612105bf6b454d5fd9e62a3d61e2` |
| 09 | `f4382f16e5cd1a1877fbcefcd37d98d763b10e054020cb4233295940bd2b6251` |
| 10 | `e894eaf462ca00e40caea0ca13a8eb8df5445bf938d4e2b235bf844110c57731` |
| 11 | `4c0de9343412777ab592073aba999977be86cd23e0bd1bf4fac4cf89a9a8bcab` |
| 12 | `09a1ad451546493c6143788959acff580f47145b5f43a3f87d87e117c7a43102` |
| 13 | `2c48c7020a420ee6adc447efbf0bb668731f48e2bf2f1b51eecab921a9843152` |
| 14 | `2d71e2a7bf24e6132f2f2e7304ec1a91e38e113763bee3029c17a9ef34c35f58` |
| 15 | `d95ac8c87fe13068bb0ffa7c0699a27c8a9d7c915d42094ff57b3cb4b2b008cd` |
| 16 | `27eda691cad2b93c1181556eb2313f0d0f9b0d86e0cb03610989ccc507b4799f` |
| GUARD v0.2 | `7b112e9f16e33b31a94f4059f71ec5af06001be56ebe43fc0e4ec4d3396e0175` (smoke harness, 6 cases) |

**Paquetes**

| File | SHA-256 |
|---|---|
| `exp_calibracion_01-07.rar` | `7c0ba312ec1883b8aab3d54b0493fc0c9a7185087135fb80a6fc966d8b19543b` |
| `beatriz-epistemic-gate.rar` | `54fd6538619d516762ad8a9ab3028b9db0b651ae42216b6d131da358fcaf947a` |
| `beatriz-epistemic-gate-exp-10-15.rar` | `54e2338bce9a15ff8c2a1ef57500dfdcbc4149344749cf4ea3c1301748961018` |
| `exp16.rar` | `9958a3889dffa3dd11d220322da188fdc8347355ef219cd2a43b04932e43a327` |
| `beatriz guard 02-2/beatriz guard 02-2.rar` (BEATRIZ-GUARD v0.2) | `821448744a2e871283ec48d3f085e86c51e68dbca74ab3e2a4dd926700833a2e` |
________________________________________
Autoría, contacto y licencia
La compuerta epistémica Beatriz y los protocolos de entrenamiento correctivo fueron creados por Eduardo Ayala Tovar (2026). Los avisos de copyright, los términos de licencia y los hashes criptográficos se incluyen para establecer un registro público de autoría.
•	Discusión técnica y colaboraciones: danterunar@yahoo.com
•	Problemas con el código: sección de Issues de este repositorio
•	Hilo de evaluación pública: https://arena.ai/c/01a07f4c-3455-756f-ae3e-852f1b0e4804
Licenciado bajo PolyForm Noncommercial License 1.0.0. Se permite el uso no comercial, de investigación y educativo; el uso comercial requiere autorización escrita del autor.


