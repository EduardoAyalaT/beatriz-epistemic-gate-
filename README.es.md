Markdown
# Beatriz Epistemic Gate

**Autor:** Eduardo Ayala Tovar  
**Año:** 2026  
**Licencia:** [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)  
**Versión en inglés:** [README.md](README.md)

---

## ¿Qué es Beatriz?

Beatriz es un mecanismo de **compuerta epistémica** diseñado para mitigar el aprendizaje de desinformación en modelos generativos durante el fine-tuning. Durante el entrenamiento, una compuerta externa compara las muestras generadas contra un corpus ancla de hechos canónicos y decide si el modelo está siendo expuesto a información falsa.

Si se detecta una contradicción con el corpus, se aplica una **pérdida contrastiva** que penaliza la preferencia del modelo por la mentira y lo obliga a anclarse a la verdad de referencia.

## ¿Por qué existe este proyecto?

Los modelos de lenguaje pueden degradar su veracidad cuando se entrenan con datos sintéticos no validados o con información falsa. Beatriz explora una vía correctiva: **hacer explícito el anclaje a la verdad durante el entrenamiento**, no solo optimizar la probabilidad condicional de tokens.

## Trayectoria experimental

El desarrollo siguió un proceso iterativo:

- **EXP01–EXP04**: depuración y calibración del mecanismo.
- **EXP05–EXP08**: ajuste fino del método y validación de estabilidad.
- **EXP09–EXP14**: pruebas reales en cinco arquitecturas de código abierto.

En este repositorio se publican **dos implementaciones representativas**:

- **EXP08**: fine-tuning completo con pérdida contrastiva anclada a un modelo de referencia congelado (`pi_ref`).
- **EXP09**: adaptación LoRA con la compuerta epistémica integrada.

## Arquitecturas probadas

El mecanismo Beatriz fue validado experimentalmente en cinco arquitecturas:

| Arquitectura   | Organización      | Resultado   |
|----------------|-------------------|-------------|
| GPT-2          | OpenAI            | ✅ positivo |
| Qwen-2.5       | Alibaba           | ✅ positivo |
| TinyLlama      | Linaje Llama      | ✅ positivo |
| Phi-3          | Microsoft         | ✅ positivo |
| Pythia         | EleutherAI        | ✅ positivo |

Los registros detallados de las últimas cuatro arquitecturas se publicarán posteriormente. Este repositorio presenta dos implementaciones completas y reproducibles como base pública del método.

## Próximamente: adelanto de EXP14

Como parte de la validación en distintas arquitecturas, **EXP14** se ejecutó sobre **Pythia 1.4B** e incluyó un **protocolo de evaluación held-out**: el modelo se entrenó con un subconjunto del corpus ancla y se evaluó en dominios no vistos y paráfrasis.

Esta prueba mostró que el efecto Beatriz se generaliza más allá del corpus de entrenamiento. El código completo, los resultados y los detalles de reproducibilidad de EXP14 se publicarán en una próxima actualización.

## Resumen de resultados (datos reales medidos)

Los únicos resultados empíricos reales presentados en este proyecto son los de **EXP08** y **EXP09**. Todos los demás ejemplos numéricos del manual conceptual son ilustrativos e hipotéticos.

### EXP08 — Fine-tuning completo con anclaje a `pi_ref`

| Rama     | Margen final de veracidad          | PPL basal → PPL final          |
|----------|-----------------------------------|-------------------------------|
| NONE     | -0.224, -0.265, -0.322            | 102.51 → 541.70, 716.89, 566.67 |
| BEATRIZ  | +10.604, +11.008, +11.042         | 102.51 → 1098.49, 1121.18, 2081.75 |

*Observación:* BEATRIZ logra un margen de veracidad muy alto, pero a costa de un fuerte aumento de perplejidad (sobreajuste al corpus ancla).

### EXP09 — Adaptación LoRA

| Rama     | Margen final de veracidad          | PPL basal → PPL final          |
|----------|-----------------------------------|-------------------------------|
| NONE     | +0.129, +0.079, +0.201            | 102.51 → 76.97, 76.37, 72.99 |
| BEATRIZ  | +3.729, +3.464, +3.453            | 102.51 → 139.12, 131.80, 126.30 |

*Observación:* con LoRA, BEATRIZ logra un margen claramente positivo manteniendo la perplejidad mucho más controlada que en EXP08. Este es el resultado más relevante del conjunto publicado.

## Estructura del repositorio

```
beatriz-epistemic-gate/
├── README.md                  # Documentación en inglés
├── README.es.md               # Documentación en español
├── LICENSE                    # PolyForm Noncommercial 1.0.0
├── docs/
│   └── MANUAL_CORRECTIVO.md   # Manual conceptual (ejemplos hipotéticos)
├── exp08/
│   ├── exp08_beatriz_ref_constrained.ipynb
│   ├── exp08_beatriz_ref_constrained.py
│   └── exp08_results.json     # Resultados con hash SHA-256
└── exp09/
    ├── exp09_beatriz_lora.ipynb
    ├── exp09_beatriz_lora.py
    └── exp09_lora_results.json # Resultados con hash SHA-256
```

## Cómo reproducir

1. Clona el repositorio.
2. Instala dependencias:
   ```bash
   pip install torch transformers peft numpy jupyter
   ```
3. Abre el notebook `exp08/exp08_beatriz_ref_constrained.ipynb` o `exp09/exp09_beatriz_lora.ipynb` (están preparados para Kaggle con rutas locales de GPT-2; ajusta `MODEL_PATH` y `TOK_PATH` si es necesario).
4. Ejecuta todas las celdas. El script generará un reporte JSON con su hash SHA-256.

## Limitaciones

- El corpus ancla es pequeño (8 dominios). La generalización a dominios más amplios no está probada.
- El margen de veracidad se mide sobre el mismo corpus ancla usado en el entrenamiento.
- EXP08 aumenta la perplejidad de forma significativa, lo que sugiere sobreajuste.
- La comparación es solo NONE vs BEATRIZ; aún no se incluyen baselines externos.
- Los resultados en Qwen-2.5, TinyLlama, Phi-3 y Pythia aún no están publicados en este repositorio (EXP14 está anunciado como próxima publicación).

## Autoría

La compuerta epistémica Beatriz y los protocolos de entrenamiento correctivo presentados en este proyecto fueron creados por **Eduardo Ayala Tovar** en 2026. La inclusión de avisos de copyright, términos de licencia y hashes criptográficos de resultados busca establecer un registro público de autoría.

## Contacto

- **Comunidad y discusiones técnicas:** `danterunar@yahoo.com`  
- **Licencias comerciales y colaboraciones:** `danterunar@yahoo.com`

Para problemas con el código, usa la sección de Issues de GitHub en este repositorio.

## Licencia

Este proyecto está licenciado bajo [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/). Se permite uso no comercial, investigación y fines educativos. El uso comercial requiere autorización explícita del autor.
