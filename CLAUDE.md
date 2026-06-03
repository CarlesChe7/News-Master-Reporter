# CLAUDE.md

> **Proyecto:** News Reporter Agent
> **Versión:** 1.0
> **Mantenedor:** Carles Escrich
> **Contexto académico:** Máster en Agentes de IA e Hiperautomatización de Procesos — EBIS Business Techschool

Este archivo orienta a Claude Code cuando trabaja en este repositorio. Léelo siempre antes de empezar a operar.

---

## 1. Contexto del proyecto

Agente automatizado de generación de contenidos editoriales ("News Reporter"). Funciones:

1. Obtener noticias reales desde un feed RSS (El País — tecnología, economía, ciencia).
2. Procesarlas con un LLM (Claude Haiku 4.5): resumen + adaptación de tono + generación de prompt para imagen.
3. Generar una imagen ilustrativa con Replicate (Flux Schnell).
4. Publicar el resultado en canales de salida: **Telegram** (canal principal de demo) y **Outlook** (canal corporativo, justificación: empresa digital).

El proyecto debe demostrar: arquitectura de flujo limpia, manejo de errores, separación de responsabilidades, decisiones técnicas justificadas, código replicable y documentado.

---

## 2. Stack técnico

- **Orquestación:** n8n (autoalojado vía Docker Compose).
- **LLM (texto):** Anthropic API — modelo `claude-haiku-4-5` (5 USD prepago, ~$0.002/noticia).
- **Generación de imagen:** Replicate — `black-forest-labs/flux-schnell` (~$0.003/imagen).
- **Canal de salida 1:** Telegram (bot + grupo + chat privado para errores).
- **Canal de salida 2:** Microsoft Outlook (vía Graph API, OAuth2 con app registrada en Azure AD).
- **Fuente:** RSS de El País (feeds.elpais.com).
- **Versionado:** Git + GitHub (cuenta Student verificada).
- **Entorno:** Windows + Docker Desktop + VSCode + Claude Code.

---

## 3. Arquitectura del flujo en n8n

Disparadores (3, en paralelo confluyendo en flujo único):
- Cron programado.
- Manual trigger.
- Webhook.

Flujo principal:
```
[Disparador] → RSS Read → Limit 1 → Anthropic (Claude) → Replicate (Flux) → [Telegram + Outlook en paralelo]
```

Manejo de errores:
- Workflow separado activado por Error Trigger.
- Notifica fallos al chat privado de Telegram (no al grupo).

---

## 4. Convenciones de trabajo

- **Sistema operativo:** Windows. Terminal preferida: PowerShell (Warp). Si hay diferencias entre PowerShell y CMD, usa PowerShell.
- **Idioma del proyecto:** español neutro de España. Comentarios, README, sticky notes, mensajes de commit, documentación: todo en español. (Excepción: prompts dentro de Replicate van en inglés porque los modelos de imagen rinden mejor así.)
- **Mensajes de commit:** estilo Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`).
- **Docker:** orquestación con `docker compose` (no `docker run` suelto). Volúmenes nombrados o relativos para persistencia.
- **n8n:** todos los nodos llevan **sticky note** explicando qué hacen y por qué (es requisito del entregable).
- **Credenciales:** SIEMPRE en n8n Credentials (no hardcodeadas en nodos). En el sistema operativo, en `.env` (nunca commiteado).
- **Separación system/user prompt:** las llamadas a Claude usan system prompt fijo y user prompt variable. Nunca mezclar.

---

## 5. Restricciones críticas — leer antes de actuar

### Seguridad (bloqueantes)

- **NUNCA commitees `Credenciales.txt`, `.env`, ni ningún archivo con tokens/keys.** Verificar `.gitignore` antes de cualquier `git add`.
- **NUNCA pegues API keys, tokens o secretos en mensajes, logs, comentarios de código, o cualquier archivo que se vaya a versionar.**
- Las API keys de Anthropic, Replicate, Telegram, Azure AD están en `Credenciales.txt` (excluido de git). NO leas ese archivo salvo que sea estrictamente necesario.
- Si necesitas que el usuario aporte una credencial, pídela explícitamente; no asumas que está disponible.

### Operación

- Antes de ejecutar cualquier comando destructivo (`rm`, `Remove-Item`, `docker compose down -v`, `git reset --hard`, etc.), pide confirmación explícita.
- Antes de hacer cambios masivos a múltiples archivos, muestra plan y pide confirmación.
- Ante una duda real entre dos opciones, pregunta antes de actuar.

### Calidad

- No añadas dependencias innecesarias. Si propones usar una librería nueva, justifícalo.
- Mantén los commits pequeños y atómicos.
- No marques tareas como "completadas" sin haberlas probado.

---

## 6. Recursos clave del repositorio

- `Prompts.txt` — Prompt v2.0 para Claude (system + user). Versionado en este repo.
- `Credenciales.txt` — **NO commitear.** Listado de tokens y URLs sensibles para uso local.
- `docker-compose.yml` — Orquestación de n8n (pendiente de crear).
- `.env.example` — Plantilla de variables (pendiente de crear).
- `.gitignore` — Exclusiones de git (pendiente de crear, debe excluir `Credenciales.txt`, `.env`, `n8n_data/`).
- `README.md` — Documentación pública del proyecto (pendiente de crear).
- `workflow-export/` — JSONs exportados de n8n (pendiente).

---

## 7. Estado actual del proyecto

### ✅ Completado

- Servicios externos configurados:
  - Outlook personal (cuenta nueva).
  - Telegram (bot + token + chat ID privado + chat ID grupo).
  - Anthropic API (cuenta + 5 USD prepago + API key + Workbench probado).
  - Replicate (cuenta + 5 USD + API token + Flux Schnell probado).
- Repositorio creado en `D:\Master-IA\News-Master-Reporter`.
- Claude Code instalado y autenticado con cuenta Pro.
- Prompt v2.0 diseñado, revisado y guardado en `Prompts.txt`.
- Cuenta GitHub Student activa.

### 🔜 Pendiente inmediato

1. Vibe coding del entorno Docker: `docker-compose.yml`, `.env.example`, `.gitignore`, `README.md`.
2. Levantar n8n con `docker compose up -d`.
3. Crear usuario admin en n8n (localhost:5678).
4. Crear credenciales en n8n para Anthropic, Replicate, Telegram.
5. Construir flujo MVP: `RSS → Claude → Replicate → Telegram`.

### 📅 Pendiente posterior

- Añadir disparadores Manual y Webhook al flujo.
- Implementar workflow de manejo de errores.
- Configurar Azure AD + Microsoft Graph para integración Outlook.
- Añadir nodo Outlook al flujo principal.
- Documentar nodos con sticky notes.
- Exportar blueprint final.
- Grabar vídeo de 5 minutos.
- Redactar memoria escrita del entregable.

---

## 8. Comandos habituales

```powershell
# Levantar n8n
docker compose up -d

# Parar n8n
docker compose down

# Ver logs en directo
docker compose logs -f n8n

# Ver estado
docker compose ps

# Acceder a n8n en navegador
# → http://localhost:5678
```

---

## 9. Glosario y referencias rápidas

- **MVP:** Mínimo Producto Viable. El flujo base sin Outlook ni manejo de errores.
- **Sticky note:** etiqueta explicativa dentro de un nodo de n8n.
- **Blueprint:** archivo JSON exportado desde n8n con la configuración completa del workflow.
- **System prompt vs User prompt:** ver bloque 4 (separación obligatoria).
- **Modelo por defecto en Claude Code:** Sonnet 4.6 (suscripción Pro). Cambiar a Opus puntualmente con `/model opus` solo en tareas que lo justifiquen.

---

## 10. Cómo trabajar conmigo (Carles)

- Comunicación directa, sin halagos innecesarios. Si algo está mal, dilo.
- Prefiero entender antes que ejecutar a ciegas: explica el porqué de las decisiones.
- En Excel y similares, separador de argumentos = `;` (configuración española).
- En código, voy aprendiendo: si algo es no-obvio, deja un comentario o lo mencionas en el chat.
- Si una tarea va a llevar mucho tiempo o consumir muchos tokens, avisa antes.
