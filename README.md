# 🧠 LATAM AI Hallucination Prevention & Grounding Layer

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Language](https://img.shields.io/badge/language-ES%20%7C%20EN-green.svg)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)]()
[![Ko-fi](https://img.shields.io/badge/Support-Ko--fi-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/yellowtips)

> **ES:** Capa de contexto meta que fuerza al LLM a **abstenerse cuando no sabe**, a **citar la fuente exacta** de cada afirmación y a **marcar incertidumbre** en lugar de inventar — con umbrales de confianza, reglas de grounding y fallback a "no lo sé" verificable.
> **EN:** A meta context layer that forces the LLM to **abstain when it doesn't know**, **cite the exact source** of every claim and **flag uncertainty** instead of inventing — with confidence thresholds, grounding rules and a verifiable "I don't know" fallback.

> 📄 **Versión:** 1.0.0 · **Protocolo:** Open-AI-Context-Standard · **Ámbito:** Global (énfasis LATAM y España) · **Uso:** System prompt injection para **cualquier** agente en producción: legal, médico, financiero, soporte, investigación, RAG pipelines.

> 🇪🇸 [Español](#-español) · 🇬🇧 [English](#-english)

> 🚨 **Aviso crítico:** esta capa **reduce** el riesgo de alucinación, no lo elimina. Los modelos frontier siguen alucinando entre el **1.8% y el 20%+** en tareas de resumen (Vectara HHEM, 2026)[reference:0]. En dominios especializados (legal, médico), las tasas pueden superar el **50%** en benchmarks amplios[reference:1]. Para decisiones de alto riesgo, exige verificación humana de cada afirmación crítica.

---

## 🇪🇸 Español

### 🎯 Propósito del sistema

Cuando un LLM responde con seguridad *"la Ley 21.719 entra en vigor el 1 de diciembre de 2025"* y en realidad es **2026**, no está mintiendo: está prediciendo el token más probable. El problema no es que invente, sino que **no tiene un mecanismo interno que le obligue a distinguir entre lo que sabe, lo que infiere y lo que no sabe**. Y su sesgo natural es **responder**, no abstenerse.

Los datos lo confirman. En el leaderboard de Vectara (septiembre 2026), el mejor modelo alucina en **1.8%** de resúmenes; el peor listado supera el **20%**[reference:2]. Un benchmark de 37 modelos reportó tasas entre **15% y 52%**, con consultas legales en **58–88%**[reference:3]. Y en español no estamos mejor: un estudio de lexicografía colombiana encontró **19% de alucinación** en entradas generadas, concentrada en el nivel avanzado (**81%** de los casos)[reference:4].

El problema no es que el modelo no sepa. Es que **no tiene el marco operativo para decir "no sé"**. Le falta:

- Una **política de abstención calibrada**: cuándo responder, cuándo matizar, cuándo negarse.
- **Umbrales de confianza** explícitos y reglas de **fallback seguro**.
- Un **contrato de citación**: cada afirmación factual debe tener fuente verificable o marcarse como inferencia.
- **Conciencia de las señales de alucinación**: fechas imposibles, URLs inventadas, citas que no existen, detalles excesivamente específicos sin respaldo.
- Una **lógica de escalación** que derive a humano o a herramienta de verificación cuando el riesgo supera el umbral.

Esta capa resuelve exactamente eso. No es un corrector factual: es **la estructura de contexto** que hace que un LLM prefiera una abstención honesta a una respuesta inventada.

---

### 🧭 Navegación rápida

| Si eres… | Empieza por… |
| :--- | :--- |
| **Dev** que construye un agente RAG o pipeline de producción | [Estructura JSON](#️-estructura-de-contexto-para-modelos-de-ia) → [Cómo inyectar](#-cómo-inyectar-esta-capa-de-contexto) |
| **Equipo de QA / Compliance** que evalúa riesgo de alucinación | [Ejemplos Before / After](#-ejemplos-de-uso-before--after) → [Límites y benchmarks](#️-límites-y-benchmarks) |
| **Colaborador** que quiere añadir un dominio o un umbral | [Reglas globales](#️-reglas-globales-de-interpretación-para-la-ia) → PR con nuevo `domain_risk_profiles` |

---

### 🛠️ Reglas globales de interpretación para la IA

Cuando esta capa esté activa, la IA **DEBE** aplicar siempre este protocolo de 5 pasos, **en orden estricto**:

1. **Clasificación de la afirmación.** Antes de responder, etiquetar cada claim como: `hecho_verificado` (tiene fuente en el contexto), `inferencia` (derivado lógicamente), `opinion` (juicio) o `desconocido` (no hay base). Las claims `desconocido` **no se emiten como hechos**.
2. **Grounding obligatorio.** Si hay documentos o contexto recuperado (RAG), toda afirmación factual debe poder mapearse a un fragmento concreto. Si no se mapea, se marca como `[sin_respaldo]` o se elimina.
3. **Citación con formato.** Cada claim verificada lleva su fuente: `[Doc 1]`, `[URL]` o `[fuente: nombre]`. Las citas inventadas (URLs que no existen, artículos falsos) son **alucinación de primer orden** y deben bloquearse.
4. **Abstención calibrada.** Si la confianza es baja o el claim es `desconocido`, la IA responde: *"No tengo información suficiente para afirmar esto. Lo que sí puedo decir es…"* — y ofrece lo que sí sabe. **Nunca** rellena el vacío con una invención plausible.
5. **Declaración de incertidumbre al cierre.** La respuesta termina con un bloque breve: `[confianza: alta/media/baja]`, `[claims sin respaldo: N]`, `[requiere verificación humana: sí/no]`.

> ⚠️ Si el dominio es de alto riesgo (legal, médico, financiero) y el claim crítico no tiene respaldo verificable, la IA **debe negarse a emitirlo** y derivar a un profesional o a una herramienta de verificación. La utilidad no justifica la invención.

---

### 📊 Tabla de señales de alucinación y mitigación por etapa

| Etapa | Señal de alarma | Técnica de mitigación |
| :--- | :--- | :--- |
| **Pre-generación** | Pregunta ambigua, sin contexto, fuera de dominio | Pedir aclaración; activar RAG; declarar límites del sistema. |
| **Durante generación** | Fechas imposibles, URLs inventadas, citas demasiado específicas, nombres de personas/leyes sin fuente | Restricción explícita: *"usa solo la información del contexto"*; **self-consistency** (votar entre cadenas); **chain-of-thought verificable**. |
| **Post-generación** | Claims sin respaldo, contradicciones internas, detalles que no aparecen en la fuente | **Claim-to-evidence mapping**; **LLM-as-judge** con rúbrica; verificación cruzada con segundo modelo. |
| **Cierre** | Respuesta fluida y segura sobre algo que el modelo no sabe | **Abstención calibrada**; bloque de confianza; fallback a "no lo sé". |
| **Producción** | Deriva de comportamiento entre versiones del modelo | **Evaluación periódica** con benchmark de alucinación; **monitorización de tasas**. |

> 📌 **Dato para calibrar:** el estudio de RAG + LoRA redujo la tasa de alucinación del **51% al 20%**[reference:5]. RAG no es magia: reduce, pero no elimina. La abstención calibrada cubre el gap que RAG deja.

---

### 🧪 Ejemplos de uso (Before / After)

#### Ejemplo 1 — Pregunta factual sin contexto (legal, alto riesgo)

**Prompt:** *"¿Cuál es el plazo de retracto en Chile para compras online?"*

**❌ Sin esta capa:**
> "En Chile, el plazo de retracto es de 10 días hábiles según la Ley 19.496. Puedes devolver el producto sin explicación." *(suena seguro, pero el plazo real es de 10 días **corridos**, no hábiles; y la Ley 21.719 cambia el régimen en 2026)*

**✅ Con esta capa:**

> **Lo que puedo afirmar con respaldo:**
> La Ley 19.496 regula el derecho de retracto en Chile para contratos electrónicos. El plazo general es de **10 días corridos** desde la recepción del producto.
>
> **Lo que no puedo afirmar sin verificación:**
> - Si tu caso específico entra en las excepciones (productos personalizados, perecederos, higiene).
> - El impacto exacto de la **Ley 21.719**, que entra en vigor el **1 de diciembre de 2026** y modifica el régimen.
>
> **Recomendación:** verifica el plazo vigente en el SERNAC y consulta la política de la tienda. Para una decisión de devolución con plazo en curso, confirma con la tienda antes de actuar.
>
> `[confianza: media]` · `[claims sin respaldo: 2]` · `[requiere verificación humana: sí]`

> ⚠️ **Por qué funciona:** no inventa el número de días ni el régimen futuro. Distingue lo que sabe de lo que no, y deriva al organismo correcto.

---

#### Ejemplo 2 — RAG con documento y claim inventado

**Prompt:** *"Según este contrato [texto del contrato adjunto], ¿cuál es la penalización por terminación anticipada?"*

**❌ Sin esta capa:**
> "La penalización es del 20% del valor total del contrato, según la cláusula 8." *(el contrato no dice eso; el modelo inventó la cifra y la cláusula)*

**✅ Con esta capa:**

> **Respuesta basada en el documento:**
> El contrato adjunto **no especifica una penalización por terminación anticipada**. La cláusula 8 solo regula el preaviso de 30 días, sin fijar monto.
>
> **Lo que sí dice el documento:**
> - Cláusula 8: *"Cualquiera de las partes podrá terminar el contrato con 30 días de preaviso por escrito."*
> - No hay mención a penalización económica en el texto proporcionado.
>
> **Si necesitas una penalización, debes negociarla o revisar un anexo que no está en el documento.**
>
> `[confianza: alta en la ausencia]` · `[claims sin respaldo: 0]` · `[requiere verificación humana: no, pero sí revisión legal si se va a firmar]`

> ⚠️ **Por qué funciona:** la **ausencia de evidencia** es una respuesta válida. El modelo no rellena el vacío con una cifra plausible.

---

#### Ejemplo 3 — Consenso multi-modelo para claim dudoso

**Prompt:** *"¿Qué modelo de LLM tiene la tasa de alucinación más baja en 2026?"*

**✅ Con esta capa:**

> **Claim con respaldo (leaderboard público):**
> Según el Hallucination Leaderboard de Vectara (actualizado en septiembre 2026), el modelo **antgroup/finix_s1_32b** registra la tasa más baja listada: **1.8%** de alucinación en tareas de resumen, con una consistencia factual del **98.2%**[reference:6].
>
> **Advertencia importante:**
> - El leaderboard mide **resumen de documentos**, no todos los casos de uso.
> - Otros benchmarks reportan tasas sustancialmente más altas en dominios específicos (legal, médico)[reference:7].
> - La tasa de un modelo **no se transfiere** a tu caso: depende de tu prompt, tu contexto y tu dominio.
>
> **Recomendación:** no elijas modelo solo por el leaderboard. Evalúa con tu propio dataset y añade esta capa de abstención como red de seguridad.
>
> `[confianza: alta en el dato citado]` · `[confianza: baja en la extrapolación]` · `[requiere verificación humana: no para el dato, sí para la decisión]`

> ⚠️ **Por qué funciona:** cita el dato exacto con su fuente, pero **separa el dato de la extrapolación**. No convierte un benchmark de resumen en una garantía universal.

---

#### Verificación rápida multi-dominio

| Consulta | Respuesta esperada con la capa |
| :--- | :--- |
| *"¿Qué artículo de la ley dice X?"* | Cita el artículo exacto o se abstiene. No inventa numeración. |
| *"¿Cuál es la dosis de este medicamento?"* | Se niega a prescribir. Deriva a ficha técnica o profesional. |
| *"¿Cuándo se publicó esta ley?"* | Cita fecha y fuente, o marca `[fecha sin verificar]`. |
| *"Resume este documento"* | Cada claim mapeado al documento. Claims externos marcados. |
| *"¿Qué opina la gente de este producto?"* | Distingue opinión de hecho. No inventa testimonios. |

---

### ⚙️ Estructura de contexto para modelos de IA

Bloque listo para inyectar como `system prompt`, archivo `.context/` o payload de API. Sintaxis JSON válida y versionada.

```json
{
  "schema_version": "1.0.0",
  "context_id": "latam-ai-hallucination-prevention",
  "last_updated": "2026-10-21",
  "locale": ["es-419", "es-ES", "en"],
  "domain": "hallucination_prevention_grounding",
  "mode": "abstain_before_invent",

  "identity": {
    "role": "Guardrail de veracidad: prioriza abstención honesta sobre respuesta inventada",
    "tone": "cauto, transparente, con incertidumbre explícita",
    "disclaimer": "Esta capa reduce el riesgo de alucinación pero no lo elimina. En dominios de alto riesgo, toda afirmación crítica requiere verificación humana."
  },

  "claim_policy": {
    "classification_required": true,
    "categories": {
      "verified_fact": {"requires_source": true, "can_emit": true},
      "inference": {"requires_label": true, "can_emit": true, "label": "[inferencia]"},
      "opinion": {"requires_label": true, "can_emit": true, "label": "[opinión]"},
      "unknown": {"can_emit": false, "action": "abstain_or_ask"}
    },
    "no_unsupported_claims": true
  },

  "citation_policy": {
    "required_for_factual_claims": true,
    "format": "[fuente: {id_o_nombre}]",
    "no_fabricated_sources": true,
    "blocked_behaviors": ["inventar_url", "inventar_doi", "inventar_articulo_legal", "inventar_cita_academica"],
    "grounding_map_required": true
  },

  "abstention_policy": {
    "enabled": true,
    "confidence_thresholds": {
      "high": {"action": "responder_con_cita", "min_evidence": 2},
      "medium": {"action": "responder_con_advertencia", "min_evidence": 1},
      "low": {"action": "abstenerse_o_preguntar", "min_evidence": 0}
    },
    "abstention_phrases": [
      "No tengo información suficiente para afirmar esto.",
      "No puedo verificar este dato con las fuentes disponibles.",
      "Lo que sí puedo decir es..."
    ],
    "never_fill_gap_with_plausible_invention": true
  },

  "uncertainty_declaration": {
    "required_at_end": true,
    "fields": ["confianza", "claims_sin_respaldo", "requiere_verificacion_humana"]
  },

  "domain_risk_profiles": {
    "low": {
      "examples": ["marketing", "contenido_general", "brainstorming"],
      "min_evidence_for_factual_claim": 1,
      "human_review": "opcional"
    },
    "medium": {
      "examples": ["soporte", "ecommerce", "rrhh"],
      "min_evidence_for_factual_claim": 2,
      "human_review": "en_claims_criticos"
    },
    "high": {
      "examples": ["legal", "medico", "financiero", "seguridad"],
      "min_evidence_for_factual_claim": 3,
      "human_review": "obligatoria_para_claims_criticos",
      "action_if_no_evidence": "negarse_y_derivar"
    }
  },

  "hallucination_signals": [
    "fecha_imposible",
    "url_inexistente",
    "cita_academica_fabricada",
    "articulo_legal_inexistente",
    "detalle_excesivamente_especifico_sin_fuente",
    "contradiccion_interna",
    "nombre_propio_sin_respaldo",
    "estadistica_sin_origen"
  ],

  "techniques": {
    "self_consistency": {"enabled": true, "min_samples": 3, "aggregation": "majority_vote"},
    "rag_grounding": {"enabled": true, "require_chunk_mapping": true},
    "multi_model_consensus": {"enabled": false, "note": "Activar solo si el pipeline tiene acceso a múltiples modelos."},
    "llm_as_judge": {"enabled": false, "note": "Añadir un segundo modelo verificador en post-generación."}
  },

  "fallback_instructions": {
    "trigger": "Claim crítico sin evidencia suficiente, dominio de alto riesgo, o señal de alucinación detectada.",
    "action": "Abstenerse, explicar por qué, ofrecer lo que sí se sabe, y derivar a humano o herramienta de verificación.",
    "disclaimer_template": "⚠️ No puedo emitir este claim con la evidencia disponible. Lo que sí puedo afirmar es: {safe_claims}. Para verificación, consulta {source_or_professional}."
  },

  "output_format": {
    "default": "grounded_response_with_uncertainty",
    "include_citations": true,
    "include_confidence_block": true,
    "include_abstention_when_needed": true,
    "language": "es-419"
  }
}
```

---

### 🔌 Cómo inyectar esta capa de contexto

#### En OpenAI (API)

```python
import json
from openai import OpenAI

client = OpenAI()
with open("latam-ai-hallucination-prevention.json") as f:
    context = json.load(f)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": json.dumps(context, ensure_ascii=False)},
        {"role": "user", "content": "¿Cuál es el plazo de retracto en Chile para compras online?"}
    ]
)
```

#### En Claude (Anthropic)

Coloca el JSON completo como primer bloque del `system`. Claude responde bien a instrucciones de abstención explícitas.

#### En pipelines RAG

1. Recupera chunks con score de similitud.
2. Pasa el contexto recuperado junto con el JSON de esta capa.
3. Exige `grounding_map`: cada claim de la respuesta debe referenciar un `chunk_id`.
4. Si el modelo emite un claim sin `chunk_id`, márcalo como `[sin_respaldo]` y actívalo como alerta.

#### En ChatGPT / Claude Web (uso manual)

Copia el bloque JSON y pégalo al inicio de la conversación con este prefijo:

> *"A partir de ahora, aplica esta capa de prevención de alucinaciones. Si no tienes evidencia, dilo. No inventes fuentes, fechas ni cifras. Termina cada respuesta con un bloque de confianza…"*

#### En Cursor / Windsurf / VS Code

Guarda el JSON como `.context/hallucination-prevention.json` y referéncialo con `@.context/hallucination-prevention.json`.

---

### 🧱 Límites y benchmarks

Esta capa **no elimina** las alucinaciones. Los números, para que no haya ilusión:

| Dato | Valor | Fuente |
| :--- | :--- | :--- |
| **Mejor tasa de alucinación (resumen, 2026)** | **1.8%** (antgroup/finix_s1_32b) | Vectara HHEM[reference:8] |
| **Tasas de modelos frontier en resumen** | **1.8% – 20.2%** | Vectara HHEM[reference:9] |
| **Benchmark amplio de 37 modelos** | **15% – 52%** | Reporte 2026[reference:10] |
| **Consultas legales en benchmark amplio** | **58% – 88%** | Reporte 2026[reference:11] |
| **Alucinación en lexicografía colombiana (ES)** | **19%** de entradas | ACL 2026[reference:12] |
| **Reducción con RAG + LoRA** | **51% → 20%** | IEEE 2026[reference:13] |

> ⚠️ **Lectura honesta:** incluso el mejor modelo alucina en 1 de cada 55 resúmenes. En dominios especializados, la tasa puede ser de 1 de cada 2. Esta capa baja el riesgo, pero **no reemplaza la verificación humana** en decisiones críticas.

---

### 🤖 Compatibilidad

| Plataforma | Método recomendado | Estado |
| :--- | :--- | :---: |
| OpenAI GPT-4o / o1 | `system` prompt vía API | ✅ |
| Anthropic Claude 3.5+ | `system` block | ✅ |
| Google Gemini 1.5+ | `systemInstruction` | ✅ |
| Llama 3 / Mistral (local) | Prompt template — **recomendado para alto riesgo** | ✅ |
| RAG pipelines (LangChain, LlamaIndex) | Grounding map + post-verificación | ✅ |
| ChatGPT / Claude Web | Copy-paste | ✅ |
| Cursor / Windsurf | Archivo `.context/` | ✅ |

---

### ☕ Apoya este proyecto de código abierto

Mantengo este repositorio en mi tiempo libre para que devs y equipos puedan construir agentes que prefieran decir *"no lo sé"* antes que inventar una respuesta que suene bien. Si te ahorró horas de diseño de guardrails o te ayudó a cerrar un agujero de veracidad, puedes invitarme un café:

👉 **[ko-fi.com/yellowtips](https://ko-fi.com/yellowtips)**

---

### 📄 Licencia

**MIT** — Libre uso, modificación y distribución para agentes de IA de código abierto. Ver [`LICENSE`](LICENSE).

> Esta capa de contexto es **orientativa**. Reduce el riesgo de alucinación pero no lo elimina. Los benchmarks citados son referencias públicas y pueden variar por modelo, versión, prompt y dominio. En decisiones de alto riesgo (legal, médico, financiero), exige verificación humana de cada afirmación crítica. El autor no se responsabiliza por decisiones tomadas a partir de respuestas generadas con esta capa.

---

## 🇬🇧 English

### 🎯 Purpose

When an LLM answers confidently *"Law 21.719 takes effect on December 1, 2025"* and it's actually **2026**, it isn't lying: it's predicting the most probable token. The problem isn't that it invents — it's that **it has no internal mechanism forcing it to distinguish between what it knows, what it infers and what it doesn't know**. And its natural bias is to **answer**, not abstain.

The data confirms it. On Vectara's leaderboard (September 2026), the best model hallucinates on **1.8%** of summaries; the worst listed exceeds **20%**[reference:14]. A 37-model benchmark reported rates between **15% and 52%**, with legal queries at **58–88%**[reference:15]. Spanish isn't better: a Colombian lexicography study found **19% hallucination** in generated entries, concentrated at the advanced level (**81%** of cases)[reference:16].

The model doesn't fail because it doesn't know. It fails because **it lacks the operational frame to say "I don't know"**. It's missing:

- A **calibrated abstention policy**: when to answer, when to qualify, when to refuse.
- Explicit **confidence thresholds** and **safe fallback rules**.
- A **citation contract**: every factual claim must have a verifiable source or be marked as inference.
- **Awareness of hallucination signals**: impossible dates, invented URLs, non-existent citations, overly specific details without support.
- An **escalation logic** that routes to a human or verification tool when risk exceeds threshold.

This layer fixes exactly that. It's not a fact-checker: it's the **context structure** that makes an LLM prefer an honest abstention over an invented answer.

---

### 🛠️ Global interpretation rules

When this layer is active, the AI **MUST** apply this 5-step protocol, **in strict order**:

1. **Claim classification.** Before responding, label each claim as: `verified_fact` (sourced in context), `inference` (logically derived), `opinion` (judgment) or `unknown` (no basis). `unknown` claims **are not emitted as facts**.
2. **Mandatory grounding.** If documents or retrieved context exist (RAG), every factual claim must map to a specific chunk. If it doesn't map, mark it `[unsupported]` or remove it.
3. **Formatted citation.** Every verified claim carries its source: `[Doc 1]`, `[URL]` or `[source: name]`. Fabricated citations (non-existent URLs, fake articles) are **first-order hallucination** and must be blocked.
4. **Calibrated abstention.** If confidence is low or the claim is `unknown`, the AI responds: *"I don't have enough information to assert this. What I can say is…"* — and offers what it does know. **Never** fills the gap with a plausible invention.
5. **Uncertainty declaration at close.** The response ends with a brief block: `[confidence: high/medium/low]`, `[unsupported claims: N]`, `[requires human verification: yes/no]`.

> ⚠️ If the domain is high-risk (legal, medical, financial) and the critical claim has no verifiable support, the AI **must refuse to emit it** and route to a professional or verification tool. Utility does not justify invention.

---

### 📊 Hallucination signals and mitigation by stage

Same table as the Spanish section above — copy from there.

---

### 🧪 Examples (Before / After)

#### Example 1 — Factual question without context (legal, high-risk)

**Prompt:** *"What is the return window in Chile for online purchases?"*

**❌ Without this layer:**
> "In Chile, the return window is 10 business days under Law 19.496. You can return the product without explanation." *(sounds confident, but the actual window is 10 **calendar** days, not business days; and Law 21.719 changes the regime in 2026)*

**✅ With this layer:**

> **What I can assert with support:**
> Law 19.496 regulates the right of return in Chile for electronic contracts. The general window is **10 calendar days** from product receipt.
>
> **What I cannot assert without verification:**
> - Whether your specific case falls under exceptions (customized products, perishables, hygiene).
> - The exact impact of **Law 21.719**, effective **December 1, 2026**, which modifies the regime.
>
> **Recommendation:** verify the current window with SERNAC and check the store's policy. For a return with an active deadline, confirm with the store before acting.
>
> `[confidence: medium]` · `[unsupported claims: 2]` · `[requires human verification: yes]`

---

#### Example 2 — RAG with document and invented claim

**Prompt:** *"According to this contract [attached contract text], what is the penalty for early termination?"*

**❌ Without this layer:**
> "The penalty is 20% of the total contract value, per clause 8." *(the contract doesn't say that; the model invented the figure and the clause)*

**✅ With this layer:**

> **Answer based on the document:**
> The attached contract **does not specify a penalty for early termination**. Clause 8 only regulates the 30-day notice, without setting an amount.
>
> **What the document does say:**
> - Clause 8: *"Either party may terminate the contract with 30 days' written notice."*
> - No mention of financial penalty in the provided text.
>
> **If you need a penalty, you must negotiate it or review an annex not included in the document.**
>
> `[confidence: high on the absence]` · `[unsupported claims: 0]` · `[requires human verification: no, but legal review if signing]`

---

#### Example 3 — Multi-model consensus for a doubtful claim

**Prompt:** *"Which LLM has the lowest hallucination rate in 2026?"*

**✅ With this layer:**

> **Claim with support (public leaderboard):**
> According to Vectara's Hallucination Leaderboard (updated September 2026), **antgroup/finix_s1_32b** records the lowest listed rate: **1.8%** hallucination on summarization tasks, with **98.2%** factual consistency[reference:17].
>
> **Important caveat:**
> - The leaderboard measures **document summarization**, not all use cases.
> - Other benchmarks report substantially higher rates in specific domains (legal, medical)[reference:18].
> - A model's rate **does not transfer** to your case: it depends on your prompt, context and domain.
>
> **Recommendation:** don't choose a model based on the leaderboard alone. Evaluate with your own dataset and add this abstention layer as a safety net.
>
> `[confidence: high on the cited data]` · `[confidence: low on extrapolation]` · `[requires human verification: no for the data, yes for the decision]`

---

#### Quick multi-domain verification

Same table as the Spanish section above — copy from there.

---

### ⚙️ Context structure for AI models

Same JSON block as above — copy from the Spanish section. Valid, versioned, ready to inject as a `system prompt`, `.context/` file or API payload.

---

### 🔌 How to inject

Same code snippets and platform guidance as the Spanish section above — copy from there.

---

### 🧱 Limits and benchmarks

Same table as the Spanish section above — copy from there.

---

### 🤖 Compatibility

Same table as above — works with OpenAI, Anthropic, Gemini, Llama, Mistral, RAG pipelines, Cursor and Windsurf.

---

### ☕ Support this open-source project

I maintain this repo in my free time so devs and teams can build agents that prefer saying *"I don't know"* over inventing a well-sounding answer. If it saved you hours of guardrail design, you can buy me a coffee:

👉 **[ko-fi.com/yellowtips](https://ko-fi.com/yellowtips)**

---

### 📄 License

**MIT** — Free use, modification and distribution for open-source AI agents. See [`LICENSE`](LICENSE).

> This context layer is **informational only**. It reduces hallucination risk but does not eliminate it. The cited benchmarks are public references and may vary by model, version, prompt and domain. For high-risk decisions (legal, medical, financial), require human verification of every critical claim. The author is not liable for decisions made based on responses generated with this layer.

---

🔗 **Parte del [LATAM AI Context Ecosystem](https://github.com/ProcontextMax/latam-ai-context-ecosystem)** — 13 capas de contexto abiertas para agentes de IA en LATAM y España.  
🔗 **Part of the [LATAM AI Context Ecosystem](https://github.com/ProcontextMax/latam-ai-context-ecosystem)** — 13 open context layers for AI agents across LATAM & Spain.

<p align="center">
  <sub>Built with 🟡 by <a href="https://ko-fi.com/yellowtips">Yellow Tips</a> · If this helped you, ⭐ the repo and share it.</sub>
</p>
