# News Reporter Agent

> **Máster en Agentes de IA e Hiperautomatización de Procesos, EBIS Business Techschool**

Agente automatizado que extrae noticias de feeds RSS en español, las procesa
editorialmente con Claude Haiku 4.5 en una arquitectura de dos agentes (un evaluador
que filtra y selecciona, y un redactor que genera título, resumen y prompt de imagen),
crea una imagen ilustrativa con Stable Diffusion 3.5 Large Turbo y publica el resultado
en Telegram y WhatsApp. Diseñado como proyecto de demostración de arquitectura de
agentes e hiperautomatización con n8n autoalojado.

---

## Arquitectura

```
   [Manual / Cron (cada 4h) / Webhook]
                  │
                  ▼
     RSS El País (Tecnología + Economía)
                  │
                  ▼
       Merge + Filtro + Deduplicación
            (publicadas.json)
                  │
                  ▼
   Claude Haiku 4.5, Evaluador (T=0.1)
        relevancia y selección
                  │
                  ▼
   Claude Haiku 4.5, Redactor (T=0.3)
   título + resumen + prompt de imagen
                  │
                  ▼
   Replicate, Stable Diffusion 3.5 Large Turbo
            imagen ilustrativa
                  │
            ┌─────┴─────┐
            ▼           ▼
        Telegram     WhatsApp (Twilio)
     (texto+imagen)    (solo texto)
```

Un workflow separado de gestión de errores (Error Trigger) notifica al chat privado
de Telegram si algo falla durante la ejecución.

---

## Stack

| Capa | Tecnología |
| --- | --- |
| Orquestación | n8n (Docker, autoalojado) |
| LLM (texto) | Anthropic API, `claude-haiku-4-5-20251001` (evaluador + redactor) |
| Generación de imagen | Replicate, `stability-ai/stable-diffusion-3.5-large-turbo` |
| Fuente de noticias | RSS El País (Tecnología + Economía) |
| Canal de salida 1 | Telegram Bot API (texto + imagen) |
| Canal de salida 2 | WhatsApp vía Twilio (texto) |
| Gestión de errores | Workflow Error Trigger, notifica a Telegram privado |
| Deduplicación | `publicadas.json` (basada en fichero) |
| Entorno | Docker Desktop + Windows |

---

## Requisitos previos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y en ejecución.
- Cuentas activas con saldo en:
  * [Anthropic](https://console.anthropic.com/) (API key)
  * [Replicate](https://replicate.com/) (API token)
- Bot de Telegram creado con [@BotFather](https://t.me/BotFather), con token y chat ID del grupo.
- Cuenta de [Twilio](https://www.twilio.com/) con el sandbox de WhatsApp activado (Account SID, Auth Token y número del sandbox).

---

## Instalación y arranque

```bash
# 1. Clonar el repositorio
git clone https://github.com/CarlesEscrichAndreu/News-Master-Reporter.git
cd News-Master-Reporter

# 2. Copiar la plantilla de variables y rellenar los valores reales
Copy-Item .env.example .env
# Edita .env con tus credenciales antes de continuar

# 3. Levantar n8n en segundo plano
docker compose up -d

# 4. Abrir la interfaz de n8n en el navegador
# → http://localhost:5678
```

Al acceder por primera vez, n8n pedirá crear un usuario administrador.

> **Nota importante:** la deduplicación usa el sistema de ficheros desde un nodo Code,
> así que el `.env` debe incluir `NODE_FUNCTION_ALLOW_BUILTIN=fs`. Sin esa variable,
> n8n bloquea el acceso a `fs` y el nodo de deduplicación falla.

---

## Comandos útiles

```bash
# Ver estado del contenedor
docker compose ps

# Seguir los logs en directo
docker compose logs -f n8n

# Parar n8n (sin borrar datos)
docker compose down

# Parar y eliminar todos los datos persistentes (irreversible)
docker compose down -v
```

---

## Estructura del repositorio

```
News-Master-Reporter/
├── docker-compose.yml      # Definición del servicio n8n
├── .env.example            # Plantilla de variables de entorno
├── .env                    # Variables reales (NO versionado)
├── .gitignore
├── README.md
├── CLAUDE.md               # Guía operativa para Claude Code
├── Prompts.txt             # System prompts de Claude (evaluador + redactor)
├── Credenciales.txt        # Credenciales locales (NO versionado)
└── workflow-export/        # Blueprints JSON exportados de n8n
    ├── news-reporter-main.json      # Workflow principal
    └── news-reporter-error.json     # Workflow de gestión de errores
```

---

## Notas de funcionamiento

- La **deduplicación** y el **Error Trigger** solo funcionan en producción
  (Cron y Webhook). En ejecuciones manuales desde el editor de n8n no se activan,
  porque el Task Runner no persiste el estado entre ejecuciones de prueba.
- En el **sandbox de WhatsApp de Twilio** solo llega el texto, no la imagen, por las
  restricciones de URLs externas del sandbox. En un entorno de WhatsApp Business de
  producción las imágenes sí se entregarían.
- Las credenciales en n8n son locales a la instancia, así que al importar los blueprints
  JSON en otra máquina hay que reconfigurarlas manualmente.

---

## Autor

**Carles Escrich**, Máster en Agentes de IA e Hiperautomatización de Procesos, EBIS Business Techschool.

---

## Licencia

MIT
