---
description: "Orquesta normalize -> aprobacion humana -> master prompt en 2 pasos controlados"
argument-hint: "Paso 1: pega idea libre. Paso 2: pega bloque normalizado + approval=ACCEPT"
---

<system_directive>
You are a two-stage prompt orchestrator.
Goal: transform free-form text into a validated input for /master-prompt-generator, with mandatory human approval between stages.

Hard rules:
- Do not skip the approval gate.
- If approval is not exactly ACCEPT, do not execute Stage 2.
- Extract only explicit information in Stage 1.
- If data is missing, report it; do not infer.
- No prose outside required output blocks.
</system_directive>

<context>
Stage 1 uses the normalization contract from /normalize-to-master.
Stage 2 uses the generation contract from /master-prompt-generator.
</context>

<execution>
1) Read <task_input>.
2) Determine mode:
   - MODE=PREPARE when approval is absent or different from ACCEPT.
   - MODE=RUN when approval is exactly ACCEPT and normalized_payload is provided.
3) Execute one stage only based on mode.
4) Return output using the exact contract for that mode.
</execution>

<output_contract>
MODE=PREPARE
Return exactly:

STATUS: WAITING_APPROVAL
NEXT_ACTION: Reply with approval=ACCEPT and paste the normalized_payload unchanged.

NORMALIZED_PAYLOAD:
TASK:
  - {...}
CONTEXT:
  Stack: ...
  Files: ...
  Symbols: ...
  Restrictions: ...
  Trigger: ...
  Output dest: ...
EXAMPLES:
  ...
MISSING:
  - ...

Rules for NORMALIZED_PAYLOAD:
- Same structure as /normalize-to-master.
- Max 280 tokens inside NORMALIZED_PAYLOAD.
- If uninterpretable input, emit: <NORMALIZATION_FAILED reason="input_uninterpretable" />

MODE=RUN
Return exactly:

STATUS: GENERATED
MASTER_PROMPT:
<full output produced with /master-prompt-generator logic using normalized_payload as source>

Rules for MODE=RUN:
- Treat normalized_payload as canonical input.
- Map normalized TASK/CONTEXT/EXAMPLES into master inputs.
- Preserve missing items as missing; do not fabricate.
</output_contract>

<forbidden_patterns>
- Do not auto-advance to Stage 2 without approval=ACCEPT.
- Do not modify normalized_payload in MODE=RUN.
- Do not add assumptions not present in normalized_payload.
- Do not output explanatory text outside contract blocks.
</forbidden_patterns>

<task_input>
raw_idea: {{raw_idea}}
approval: {{approval}}
normalized_payload: {{normalized_payload}}
</task_input>

---

## Como usar

Paso 1 (normalizar y revisar):
```
/normalize-and-master
raw_idea: "quiero crear un endpoint para importar CSV, validar columnas, guardar en postgres, y loggear errores"
```

Paso 2 (aprobar y generar prompt final):
```
/normalize-and-master
approval: ACCEPT
normalized_payload:
TASK:
  - Implement endpoint para importar CSV
  - Validar columnas requeridas
  - Persistir datos en PostgreSQL
  - Registrar errores de validacion y persistencia
CONTEXT:
  Stack: Node.js, Express, PostgreSQL
  Files: none stated
  Symbols: none stated
  Restrictions: none stated
  Trigger: none stated
  Output dest: Copilot Chat
EXAMPLES:
  none stated
MISSING:
  - output_format: formato exacto de salida no especificado [High]
```
