# News Reporter Agent

> **Máster en Agentes de IA e Hiperautomatización de Procesos — EBIS Business Techschool**

Agente automatizado que extrae noticias de feeds RSS en español, las procesa
editorialmente con Claude Haiku 4.5 (resumen + título adaptado + prompt de imagen),
genera una imagen ilustrativa con Flux Schnell y publica el resultado en Telegram
y Outlook. Diseñado como proyecto de demostración de arquitectura de agentes e
hiperautomatización con n8n.

---

## Arquitectura

```
[Cron / Manual / Webhook]
          │
          ▼
     RSS Read (El País)
          │
       Limit 1
          │
          ▼
  Anthropic Claude Haiku 4.5
  (título + resumen + prompt)
          │
          ▼
  Replicate Flux Schnell
     (imagen ilustrativa)
          │
    ┌─────┴─────┐
    ▼           ▼
 Telegram    Outlook
 (grupo)   (correo)
```

En caso de error, un workflow separado notifica al chat privado de Telegram.

---

## Stack

| Capa | Tecnología |
|---|---|
| Orquestación | n8n (Docker) |
| LLM | Anthropic API — `claude-haiku-4-5` |
| Generación de imagen | Replicate — `black-forest-labs/flux-schnell` |
| Fuente de noticias | RSS El País (tecnología / economía) |
| Canal de salida 1 | Telegram Bot API |
| Canal de salida 2 | Microsoft Outlook (Graph API + Azure AD) |
| Entorno | Docker Desktop + Windows |

---

## Requisitos previos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y en ejecución.
- Cuentas activas con saldo en:
  - [Anthropic](https://console.anthropic.com/) (API key)
  - [Replicate](https://replicate.com/) (API token)
- Bot de Telegram creado con [@BotFather](https://t.me/BotFather) (token + chat ID del grupo).
- App registrada en [Azure AD](https://portal.azure.com/) para acceso a Outlook vía Microsoft Graph.

---

## Instalación y arranque

```powershell
# 1. Clonar el repositorio
git clone https://github.com/tu-usuario/news-master-reporter.git
cd news-master-reporter

# 2. Copiar la plantilla de variables y rellenar los valores reales
Copy-Item .env.example .env
# Edita .env con tus credenciales antes de continuar

# 3. Levantar n8n en segundo plano
docker compose up -d

# 4. Abrir la interfaz de n8n en el navegador
# → http://localhost:5678
```

Al acceder por primera vez, n8n pedirá crear un usuario administrador.

---

## Comandos útiles

```powershell
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
news-master-reporter/
├── docker-compose.yml      # Definición del servicio n8n
├── .env.example            # Plantilla de variables de entorno
├── .env                    # Variables reales (NO versionado)
├── .gitignore
├── README.md
├── CLAUDE.md               # Guía para Claude Code
├── Prompts.txt             # System prompt v2.0 para Claude
├── Credenciales.txt        # Credenciales locales (NO versionado)
└── workflow-export/        # JSONs exportados de n8n (pendiente)
```

---

## Autor

**Carles Escrich** — Máster en Agentes de IA e Hiperautomatización de Procesos, EBIS Business Techschool.

---

## Licencia

MIT
