# 0014. Datos sensibles enmascarados antes del modelo

Estado: aceptado

## Contexto

El personal de RH puede escribir correos, teléfonos, CURP, RFC, tarjetas o NSS en la pregunta o el historial. Ese
texto llegaría al modelo externo (Cloudflare), al log, a LangSmith y a la caché. El producto no necesita esos datos
para responder: las herramientas buscan por id de empleado o departamento.

## Decisión

- `app/pii.py` detecta `email`, `phone`, `curp`, `rfc`, `card` (Luhn) y `nss` con expresiones regulares compiladas
  una vez. Los solapes se resuelven por prioridad (una CURP no es además RFC; una tarjeta no es teléfono ni NSS).
  `EMP-0004` y fechas no son PII.
- Con `PII_POLICY=mask` (por omisión) cada dato se reemplaza por `[EMAIL]`, `[TELEFONO]`, `[CURP]`, `[RFC]`,
  `[TARJETA]` o `[NSS]` en la ruta `/ask`, después del esquema y la inyección y antes del enrutador, la caché, el
  agente, el log y la traza. La respuesta también se enmascara. Se informa `pii_detected` con los tipos ordenados.
- Con `PII_POLICY=block`, `400 {"detail": "sensitive data not allowed"}`.
- El evento `audit pii_detected` lleva `types` y `where`, nunca el valor. Una pregunta con PII nunca se guarda en
  caché.

## Consecuencias

- Ningún sistema externo ni el log recibe el dato original; se cumple el mínimo de datos.
- Las regex tienen falsos positivos y negativos (formatos raros); se cubren con pruebas positivas y negativas.
- Si el modelo necesitara el dato enmascarado, la respuesta pierde ese detalle; para este producto no aplica.
- `block` es más estricto pero rechaza preguntas legítimas; por eso `mask` es la opción por omisión.
