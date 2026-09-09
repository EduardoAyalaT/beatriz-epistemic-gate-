# Beatriz Epistemic Gate: Especificación Técnica y Arquitectura de defensa contra el Envenenamiento en Fine-Tuning

**Autor:** Eduardo Ayala Tovar
**Año:** 2026
**Licencia:** PolyForm Noncommercial License 1.0.0
**Afiliación:** Investigación Independiente / Soberanía en IA
**Hardware de Orquestación:** Toshiba Satellite U205 (2006, 2 GB RAM)
**Motor de Cómputo:** Kaggle T4 x2 - Costo total: $0 USD

> **Nota sobre datos:** Los ejemplos con porcentajes tipo "34% de mejora" del Manual son ilustrativos e hipotéticos con fines pedagógicos. Los únicos resultados empíricos verificados con hash son EXP01 a EXP16 de este documento.

### Resumen
La industria sostiene que defender LLMs requiere infraestructura masiva. Demostramos lo contrario con Beatriz, una compuerta epistémica ligera que previene el envenenamiento quirúrgico durante el fine-tuning. A través de 16 experimentos sistemáticos validados en 5 arquitecturas desde 124M hasta 3.8B, demostramos que el envenenamiento es invisible a métricas agregadas pero se contrarresta con un proxy defensivo + corpus ancla + pérdida contrastiva. El filtrado puro aporta 65% del beneficio sin tocar el loop del estudiante, mientras el término Softplus consolida margen de veracidad +4.19 ±0.08 en held-out escalado de 30 hechos, con latencia ~0.1 ms y determinismo bit-exacto.

**Definición:** truth_margin = logP(verdad) - logP(mentira)

### 1. Introducción: El Mito de la Infraestructura Masiva
Toda la orquestación se ejecutó en portátil de 2006 + GPU gratuita.

### 2. Marco Teórico: Envenenamiento Quirúrgico e Invisible
Cuando se inyecta desinformación, el modelo destruye selectivamente los hechos atacados hasta indiferencia exacta train margin ≈ 0.0, mientras PPL mejora. Visto en EXP06, EXP07, EXP10, EXP11, EXP13, EXP14, EXP15. Los monitores agregados son ciegos.

### 3. Arquitectura: La Compuerta Epistémica
1. Corpus Ancla Inmutable versionado con SHA-256 + OpenTimestamps
2. DenseVectorGateCached: embedding = mean hidden_states[-1], similitud coseno -> VERIFIED, CONTRADICTED, UNKNOWN, INVALID
3. Función de Pérdida Compuesta

#### 3.1 Función de Pérdida
L_total = α•L_ce + β•L_verdad

Donde:
L_ce = Cross-Entropy estándar
L_verdad = Softplus(MARGIN + logP(mentira) - logP(verdad))
En EXP08 anclada a pi_ref: Softplus(MARGIN - (log_ratio_verdad - log_ratio_mentira)) donde log_ratio = logP_theta - logP_ref

Rangos Manual Ideal: α 0.4-0.6, β 0.8-1.2 obligatorio, γ 0.6-0.9, δ 0.2-0.4. β y γ nunca nulos.

### 4. Puente: Del Manual Ideal al Prototipo Soberano $0
| Manual pide laboratorio | Beatriz $0 implementa |
|---|---|
| Corpus masivo con API | 8 hechos EXP07-14, 36 hechos EXP16 [6 train + 30 held-out] hash + .ots |
| L_total 4 términos | L_verdad Softplus + L_divergencia como pi_ref y PPL |
| L_lógica Z3 y MMD | EXP05: Z3 verifica ancla sat 24 axiomas, 0 mismatches en 672 claims, 7.9 ms/claim, γ=0 medido no activo |
| Ledger + rollback | model_parameter_hash, adapter_sha256, prereg_sha, sanity_sha, epoch_records con R1 neutral<-0.30, R2 truth_drop>2.0, R3 unknown_rise>2.0 |

### 5. Metodología
Modelos: GPT-2 124M, Qwen-2.5-0.5B, TinyLlama-1.1B, Pythia-1.4B, Phi-3-mini-4k-instruct 3.8B
Config cierre: LoRA r=8 alpha=16, SEEDS [11,22,33], 8 épocas, 60 draws, P_LIE 0.50->0.70, ALPHA 0.5 BETA 1.0 MARGIN 0.5, LR 1e-4 a 2e-4

### 6. Resultados

**6.1 EXP01-07 Calibración anti-colapso**
EXP01: CONTROL aprende mentira
EXP02: CONTROL vs FILTER vs PLACEBO 84 igualados - No es cantidad, es verdad
EXP03: CONTAMINATED -15.73±0.96, HARD -0.80±0.27, EPISTEMIC +13.20±0.56
EXP04: NONE -0.01, HARD +4.43, REWRITE +10.88 - Corregir > filtrar
EXP05: NONE falla 3/3 por R3 unknown_delta 9.47, 8.73, 8.32 rollback a 0, BEATRIZ seed33 aguanta 8 épocas tm 26.75
EXP06: Texto abierto NONE 0.00 indiferencia, BEATRIZ +0.57 a +4.23
EXP07: NONE -0.27 colapso PPL 102->541, BEATRIZ +10.27 PPL 102->948
EXP08: Reference-Constrained BEATRIZ +11.04 PPL 2081
EXP09: LoRA 0.23% BEATRIZ +3.54 PPL 102->132 - Controla tax

**6.2 EXP10-14 Escalado 5 arquitecturas**
Qwen-2.5-0.5B: NONE -0.24 colapso, BEATRIZ +8.96 PPL 27->156
TinyLlama-1.1B: NONE -0.17, BEATRIZ +7.86 PPL 18->67
Pythia-1.4B: NONE -0.09 / +1.22, BEATRIZ +8.02 / +2.09 PPL 46->150
Phi-3-mini 3.8B: Ver EXP13 y 15

**6.3 EXP15 Ablación Quirúrgica - 40 textos neutrales**
BASE: +1.34 train / +1.90 held-out / +2.21 para / PPL 12.7
NONE: -0.03±0.02 / +3.57±0.17 / +3.41±0.24 / 30.9
GATE_ONLY: +7.46±0.24 / +5.08±0.09 / +5.10±0.51 / 58.8 - 65% sin tocar loop
BEATRIZ: +10.13±0.07 / +5.91±0.07 / +5.63±0.50 / 86.3 - Añade 35%
Aporte contraste: +2.67 train / +0.82 held-out. Gate 0.107 ms/call, VRAM 7.97 GB

**6.4 EXP16 Held-Out Escalado n=30**
BEATRIZ: Train +10.13 / Held-Out 30 +4.19±0.08 / Precisión 0.93 Recall 0.80 / 0.1 ms cacheado. Entrena en 6, generaliza a 30.

### 7. Limitaciones
1. Corpus 36 elementos, escalar a cientos
2. Latencia viva requiere forward del oráculo, futuro encoder E5-small
3. Licencia no comercial

### 8. Conclusión
La seguridad no es monopolio de grandes labs. Con proxy no invasivo + corpus ancla verificable se neutraliza envenenamiento con $0. Soberanía científica es posible.

### Apéndice Reproducibilidad - 16 SHA verificados
SEEDS [11,22,33], GPT-2 offline c7d00560d8910fbed77ffad4065dee5011c41ba401b1064e749c498ba9e20373
EXP01 713ffa6227b68a9837a11f245b8c4e52d11b343d7f2d0c8f4af916edee4966ae
EXP02 4b3d424f308943ce41c3d6c8f11b8c99130e1eb39fb380ab4f42e7ed1705947e
EXP03 369c5cec029b792744978a9c81434ff46528be5fee49377c621a5f67c9441c85
EXP04 ed466d516acdd202bd4c71d76e6919422d6dff00df019bdd6ac55e618d44ee86
EXP05 97359f0464d7a7f7c33a87bca8a3d9e66b212495f5152c4603105ee1493626c1 prereg 57691ac08801e38c63b346579f22091dd0e5da2b7c10ded63a909c678be69f9e sanity 2ee4949d9bb750afd356d5a7a9009b98ef8dcc014e1369bbb065d7aea9ac116b
EXP06 b2e0e62a84b1ed19f97657c36308076a44cae42c806b2022bcfd19a55a559ed5
EXP07 5e3da9dca9162f62c6aac94175133e9cbf70ac101389f6714afc24d1b8273483
EXP08 c93ba4b74ecb8ade32f0f645761c1441382c612105bf6b454d5fd9e62a3d61e2
EXP09 f4382f16e5cd1a1877fbcefcd37d98d763b10e054020cb4233295940bd2b6251
EXP10 e894eaf462ca00e40caea0ca13a8eb8df5445bf938d4e2b235bf844110c57731
EXP11 4c0de9343412777ab592073aba999977be86cd23e0bd1bf4fac4cf89a9a8bcab
EXP12 09a1ad451546493c6143788959acff580f47145b5f43a3f87d87e117c7a43102
EXP13 2c48c7020a420ee6adc447efbf0bb668731f48e2bf2f1b51eecab921a9843152
EXP14 2d71e2a7bf24e6132f2f2e7304ec1a91e38e113763bee3029c17a9ef34c35f58
EXP15 d95ac8c87fe13068bb0ffa7c0699a27c8a9d7c915d42094ff57b3cb4b2b008cd
EXP16 27eda691cad2b93c1181556eb2313f0d0f9b0d86e0cb03610989ccc507b4799f
Bundles: exp_calibracion_01-07.rar 7c0ba312ec1883b8aab3d54b0493fc0c9a7185087135fb80a6fc966d8b19543b, beatriz-epistemic-gate.rar 54fd6538619d516762ad8a9ab3028b9db0b651ae42216b6d131da358fcaf947a, beatriz-epistemic-gate-exp-10-15.rar 54e2338bce9a15ff8c2a1ef57500dfdcbc4149344749cf4ea3c1301748961018, exp16.rar 9958a3889dffa3dd11d220322da188fdc8347355ef219cd2a43b04932e43a327

## Autoría

La compuerta epistémica Beatriz y los protocolos de entrenamiento correctivo presentados en este proyecto fueron creados por **Eduardo Ayala Tovar** en 2026. La inclusión de avisos de copyright, términos de licencia y hashes criptográficos de resultados busca establecer un registro público de autoría.

## Contacto

- **Comunidad y discusiones técnicas:** `danterunar@yahoo.com`  
- **Licencias comerciales y colaboraciones:** `danterunar@yahoo.com`

Para problemas con el código, usa la sección de Issues de GitHub en este repositorio.

## Licencia

Este proyecto está licenciado bajo [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/). Se permite uso no comercial, investigación y fines educativos. El uso comercial requiere autorización explícita del autor.

Evaluación pública: https://arena.ai/c/01a07f4c-3455-756f-ae3e-852f1b0e4804
