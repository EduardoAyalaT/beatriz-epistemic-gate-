# Beatriz Epistemic Gate: Defensa contra Envenenamiento en Fine-Tuning

**Autor:** Eduardo Ayala Tovar - 2026
**Licencia:** PolyForm Noncommercial License 1.0.0
**Hardware:** Toshiba Satellite U205 (2006, 2GB RAM) + Kaggle T4 x2 - Costo $0 USD
**Afiliación:** Investigación Independiente / Soberanía en IA

> **Nota de honestidad:** Los porcentajes tipo "34% de mejora" del Manual son ilustrativos e hipotéticos con fines pedagógicos. Los únicos resultados empíricos verificados con hash SHA-256 son los de EXP01 a EXP16 de esta tabla.

### Resumen
La industria asume que defender un LLM requiere clusters millonarios. Este proyecto demuestra lo contrario. Presentamos **Beatriz**, un proxy epistémico no invasivo que evita que un modelo aprenda mentiras aunque el 70% del flujo esté envenenado.

Sin defensa, el margen de verdad colapsa a `≈0.0` - indiferencia exacta entre verdad y mentira - mientras la perplejidad parece mejorar. Con Beatriz, el margen se preserva y generaliza a hechos que nunca vio.

### El Problema: Envenenamiento Quirúrgico e Invisible
En EXP05, EXP06, EXP07, EXP10, EXP11, EXP13, EXP14, la rama `NONE` colapsa. Es el fenómeno del Manual. Los monitores basados en PPL son ciegos.

### La Solución: Beatriz
1.  **Corpus Ancla Inmutable:** 8 hechos en EXP07-14, 36 hechos en EXP16 [6 train + 30 held-out], con SHA-256 + OpenTimestamps .ots
2.  **Compuerta Densa Vectorial:** Oráculo congelado offline, `embedding = mean hidden_states[-1]`, similitud coseno → VERIFIED / CONTRADICTED / UNKNOWN / INVALID
3.  **Pérdida Compuesta:** `L_total = α•L_ce + β•L_verdad` donde `L_verdad = Softplus(MARGIN + logP(mentira) - logP(verdad))`

### Serie Experimental 01-16 - Solo lo que corrió de verdad

| EXP | Modelo | LoRA | Resultado Clave | SHA-256 del reporte |
|---|---|---|---|---|
| 01 | GPT-2 124M | - | Filtro vs sin filtro, CONTROL aprende mentira | `713ffa6227b68a...` |
| 02 | GPT-2 124M | - | CONTROL vs FILTER vs PLACEBO 84 igualados | `4b3d424f3089...` |
| 03 | GPT-2 124M | - | HARD -0.80 no alcanza, EPISTEMIC +13.20 | `369c5cec029b...` |
| 04 | GPT-2 124M | - | REWRITE +10.88 > HARD +4.43 > NONE -0.01 | `ed466d516acd...` |
| 05 | GPT-2 124M | - | Fire Test Z3 + rollback, NONE falla 3/3, BEATRIZ tm 26.75 | `97359f0464d7...` |
| 06 | GPT-2 124M | - | Texto abierto, NONE 0.00, BEATRIZ +4.23 | `b2e0e62a84b1...` |
| 07 | GPT-2 124M | - | Dense Vector Gate 8 hechos, NONE -0.27, BEATRIZ +10.27 | `5e3da9dca916...` |
| 08 | GPT-2 124M | - | Reference-Constrained + pi_ref, BEATRIZ +11.04 | `c93ba4b74ecb...` |
| 09 | GPT-2 124M | 0.23% | LoRA Retention, BEATRIZ +3.54 PPL 132 | `f4382f16e5cd...` |
| 10 | Qwen-2.5-0.5B | 0.10% | NONE -0.24 colapso, BEATRIZ +8.96 | `e894eaf462ca...` |
| 11 | TinyLlama-1.1B | 0.10% | NONE -0.17, BEATRIZ +7.86 | `4c0de9343412...` |
| 12 | TinyLlama-1.1B | 0.10% | Held-Out n=2 + Paraphrase, generaliza +1.33 | `09a1ad451546...` |
| 13 | Phi-3-mini 3.8B | 0.12% | NONE -0.03 / +3.57, BEATRIZ +10.13 / +5.91 | `2c48c7020a42...` |
| 14 | Pythia-1.4B | 0.16% | NONE -0.09, BEATRIZ +8.02 | `2d71e2a7bf24...` |
| 15 | Phi-3-mini 3.8B | 0.12% | Ablación NONE / GATE_ONLY / BEATRIZ | `d95ac8c87fe1...` |
| 16 | Phi-3-mini 3.8B | 0.12% | Held-Out n=30 Prec 0.93 Rec 0.80 | `27eda691cad2...` |

**5 arquitecturas:** GPT-2 124M, Qwen-2.5-0.5B, TinyLlama-1.1B, Pythia-1.4B, Phi-3-mini 3.8B

### Paquetes con Prueba de Tiempo - Tus 4 hashes de tu Toshiba
exp_calibracion_01-07.rar -> 7c0ba312ec1883b8aab3d54b0493fc0c9a7185087135fb80a6fc966d8b19543b
beatriz-epistemic-gate.rar -> 54fd6538619d516762ad8a9ab3028b9db0b651ae42216b6d131da358fcaf947a
beatriz-epistemic-gate-exp-10-15.rar -> 54e2338bce9a15ff8c2a1ef57500dfdcbc4149344749cf4ea3c1301748961018
exp16.rar -> 9958a3889dffa3dd11d220322da188fdc8347355ef219cd2a43b04932e43a327

### Resultados Clave - EXP15 Ablación Quirúrgica
BASE: +1.34 train / +1.90 held-out / PPL 12.7
NONE: -0.03±0.02 / +3.57±0.17 / PPL 30.9
GATE_ONLY: +7.46±0.24 / +5.08±0.09 / PPL 58.8 - Aporta 65% sin tocar el loop
BEATRIZ: +10.13±0.07 / +5.91±0.07 / PPL 86.3 - Añade 35% restante
Gate: 0.107 ms/llamada - VRAM 7.97 GB

### Puente: Del Manual Ideal al Prototipo Soberano $0
| Manual pide laboratorio | Beatriz con $0 |
|---|---|
| Corpus masivo con API | 36 hechos con hash + .ots |
| L_total con 4 términos | L_verdad como Softplus + L_divergencia como pi_ref y PPL |
| L_lógica con Z3 | EXP05: Z3 sat 24 axiomas, 0 mismatches en 672 claims, 7.9ms/claim |
| Ledger + rollback | model_hash, adapter_sha256, prereg_sha, sanity_sha, R1/R2/R3 |

### Verificación
certutil -hashfile exp_calibracion_01-07.rar SHA256
ots verify exp_calibracion_01-07.rar.ots

## Autoría

La compuerta epistémica Beatriz y los protocolos de entrenamiento correctivo presentados en este proyecto fueron creados por **Eduardo Ayala Tovar** en 2026. La inclusión de avisos de copyright, términos de licencia y hashes criptográficos de resultados busca establecer un registro público de autoría.

## Contacto

- **Comunidad y discusiones técnicas:** `danterunar@yahoo.com`  
- **Licencias comerciales y colaboraciones:** `danterunar@yahoo.com`

Para problemas con el código, usa la sección de Issues de GitHub en este repositorio.

## Licencia

Este proyecto está licenciado bajo [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/). Se permite uso no comercial, investigación y fines educativos. El uso comercial requiere autorización explícita del autor.

Evaluación pública https://arena.ai/c/01a07f4c-3455-756f-ae3e-852f1b0e4804

