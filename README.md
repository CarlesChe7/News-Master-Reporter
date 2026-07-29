# News Reporter Agent

Agente autónomo que selecciona, redacta e ilustra una noticia cada cuatro horas, y la
publica en Telegram y WhatsApp sin intervención humana. Construido sobre n8n autoalojado
con una arquitectura de dos agentes LLM especializados.

<p align="center">
  <img src="docs/ejemplo-telegram.jpg" alt="Ejemplo de publicación en Telegram" width="360">
  <br>
  <em>Publicación generada y enviada automáticamente al canal de Telegram</em>
</p>
  <em>Publicación generada y enviada automáticamente al canal de Telegram</em>
</p>

---

## El problema

Mantener un canal de noticias temático exige repetir el mismo ciclo varias veces al día:
revisar fuentes, descartar lo que no encaja con la línea editorial, elegir qué merece
publicarse, reescribirlo con un tono coherente, buscar una imagen que lo ilustre y
distribuirlo. Es trabajo de criterio, pero criterio repetitivo y acotado, justo el tipo de
tarea donde un pipeline con LLM aporta valor real.

Este agente automatiza el ciclo completo. La parte interesante no es llamar a un modelo,
sino **decidir dónde poner el juicio editorial y cómo evitar que el sistema publique
basura cuando algo va mal**.

---

## Arquitectura

```
      ┌──────────────┬──────────────┬──────────────┐
      │   Manual     │  Cron (4h)   │   Webhook    │
      └──────┬───────┴──────┬───────┴──────┬───────┘
             └──────────────┼──────────────┘
                            ▼
              RSS El País — Tecnología + Economía
                            │
                            ▼
                  Merge + Deduplicación
              (histórico en static data, tope 200)
                            │
                            ▼
              Agente EVALUADOR — Haiku 4.5, T=0.1
           rúbrica sobre 10 · umbral 6 · devuelve top 3
                            │
                            ▼
                 Lookup por índice del ganador
              (recupera el artículo original íntegro)
                            │
                            ▼
              Agente REDACTOR — Haiku 4.5, T=0.3
        título + resumen + prompt de imagen + campo error
                            │
                            ▼
               Validación del contrato JSON
            (corta el ciclo si el redactor rechazó)
                            │
                            ▼
        Replicate — Stable Diffusion 3.5 Large Turbo (16:9)
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
              Telegram            WhatsApp
          (imagen + texto)       (solo texto)
                  └─────────┬─────────┘
                            ▼
                  Registrar publicación
              (añade la URL al histórico)
```

16 nodos funcionales. Un workflow independiente con Error Trigger notifica cualquier
fallo a un chat privado de Telegram, con el nodo que falló y enlace directo a la ejecución.

---

## Decisiones técnicas

Las decisiones que definen el proyecto, y por qué se tomaron así.

### Dos agentes en lugar de uno

Seleccionar y redactar son tareas distintas y quieren temperaturas distintas. Un único
prompt que hiciera ambas obligaría a un compromiso: o el juicio editorial se vuelve
errático, o la redacción sale plana.

- **Evaluador (T=0.1).** Aplica una rúbrica numérica sobre 10 — relevancia temática (0-3),
  impacto editorial (0-3), idoneidad visual (0-2), calidad informativa (0-2), con umbral
  mínimo de 6. Devuelve las tres mejores con puntuación y justificación. Temperatura baja
  porque aquí se quiere consistencia, no creatividad: la misma noticia debería puntuar
  parecido en dos ejecuciones distintas.
- **Redactor (T=0.3).** Genera título (6-14 palabras), resumen (60-100 palabras) y prompt
  de imagen en inglés (25-45 palabras). Temperatura algo más alta para que el texto no
  suene mecánico, pero contenida para que no invente datos.

La justificación del evaluador se conserva en el output (`_evaluacion`) para poder auditar
después por qué se eligió cada noticia.

### El evaluador devuelve un índice, no el texto

El evaluador recibe los artículos numerados y responde con el índice del ganador. El
artículo completo se recupera después por lookup contra la lista original.

Dos ventajas: el modelo no reescribe ni trunca el contenido de origen al pasarlo al
siguiente paso, y la respuesta es corta y barata en tokens aunque haya cuarenta candidatos.

### Contrato JSON con campo de error explícito

El redactor devuelve siempre la misma estructura, y puede rechazar una noticia rellenando
`error` y dejando el resto a `null`:

```json
{ "titulo": null, "resumen": null, "prompt_imagen": null,
  "error": "La noticia no contiene información suficiente." }
```

El nodo siguiente valida el contrato antes de continuar: si hay `error`, o si falta algún
campo obligatorio, corta el ciclo y lo propaga al Error Trigger. Sin esa comprobación el
pipeline seguiría adelante y publicaría una imagen generada a partir de un prompt vacío.

### Defensa frente a prompt injection

Los feeds RSS son contenido de terceros y llegan directos al contexto del modelo. Ambos
system prompts declaran de forma explícita que el contenido de las noticias es **datos, no
instrucciones**, y el artículo va delimitado por etiquetas `<noticia></noticia>`. El prompt
del redactor incluye además un ejemplo few-shot con un intento de inyección y la respuesta
correcta: procesarlo como texto normal.

Las instrucciones viven en el `system` de la API, nunca en el turno de usuario ni en un
turno de assistant simulado.

### Deduplicación en el static data del workflow

El histórico de URLs publicadas vive en `$getWorkflowStaticData('global')`, con un tope de
200 entradas para que no crezca sin límite. Se descartó guardarlo en un fichero del
contenedor: habría exigido dar acceso a `fs` desde los nodos Code y el histórico se
perdería al recrear el contenedor.

### Distinguir "no hay nada nuevo" de "algo ha fallado"

Que no haya noticias nuevas es el caso normal en la mayoría de ejecuciones: el feed cambia
más despacio que el ciclo del agente. El pipeline termina en silencio devolviendo una lista
vacía. Solo se lanza error real cuando los feeds no devuelven nada en absoluto, que sí
indica un problema de conectividad o de URL.

La distinción importa: si cada ejecución sin novedades generase una alerta, el canal de
errores se volvería ruido y dejaría de leerse.

### Generación de imagen síncrona

La llamada a Replicate usa la cabecera `Prefer: wait`, que bloquea hasta que la predicción
termina y devuelve la URL directamente. Evita montar un bucle de polling y simplifica el
grafo a cambio de una llamada HTTP más larga.

---

## Métricas

| Métrica | Valor |
| --- | --- |
| Coste por ciclo | ~$0,045 (0,04 imagen + 2 llamadas LLM) |
| Tiempo de ejecución | 4-6 s |
| Nodos funcionales | 16 |
| Llamadas a LLM por ciclo | 2 |

Con el ciclo cada 4 horas son unas 6 publicaciones diarias, en torno a **8 €/mes**. El
coste dominante es la generación de imagen: las dos llamadas al LLM aportan céntimas
frente a los 0,04 $ de Replicate.

---

## Stack

| Capa | Tecnología |
| --- | --- |
| Orquestación | n8n autoalojado (Docker Compose) |
| LLM | Anthropic API, `claude-haiku-4-5-20251001` |
| Imagen | Replicate, `stability-ai/stable-diffusion-3.5-large-turbo` |
| Fuentes | RSS El País — Tecnología y Economía |
| Distribución | Telegram Bot API · WhatsApp vía Twilio |
| Persistencia | Static data del workflow |
| Observabilidad | Workflow independiente con Error Trigger |

---

## Puesta en marcha

**Requisitos:** Docker Desktop, cuentas con saldo en
[Anthropic](https://console.anthropic.com/) y [Replicate](https://replicate.com/), un bot
de Telegram creado con [@BotFather](https://t.me/BotFather) y una cuenta de
[Twilio](https://www.twilio.com/) con el sandbox de WhatsApp activo.

```bash
git clone https://github.com/CarlesEscrichAndreu/News-Master-Reporter.git
cd News-Master-Reporter

cp .env.example .env        # PowerShell: Copy-Item .env.example .env
# Edita .env con tus credenciales

docker compose up -d        # n8n queda en http://localhost:5678
```

Después, dentro de n8n:

1. Importa `workflow-export/news-reporter-main.json` y
   `workflow-export/news-reporter-error.json`.
2. Reconfigura las cuatro credenciales (Anthropic, Replicate, Telegram, Twilio). Las
   credenciales son locales a cada instancia y no viajan en los blueprints.
3. Ajusta el `chatId` de Telegram y el número de WhatsApp a los tuyos.
4. **Abre Settings del workflow principal y reasigna el workflow de errores.** El campo
   `errorWorkflow` guarda un ID interno de la instancia de origen, así que al importar
   apunta a un workflow que no existe en tu máquina y la gestión de errores queda muda.
5. Activa el workflow principal para que el cron empiece a ejecutarse.

Comandos útiles: `docker compose ps`, `docker compose logs -f n8n`,
`docker compose down` (parar), `docker compose down -v` (parar y borrar datos, irreversible).

---

## Limitaciones conocidas

- **Deduplicación y Error Trigger solo actúan en ejecuciones de producción** (cron o
  webhook). En ejecuciones manuales desde el editor no se activan, porque el Task Runner
  no persiste estado entre pruebas.
- **WhatsApp llega sin imagen.** El sandbox de Twilio restringe las URLs externas de
  medios. Con una cuenta de WhatsApp Business la imagen se entregaría.
- **Una sola fuente.** Añadir más medios exige revisar la rúbrica del evaluador, calibrada
  con el estilo y volumen de los feeds actuales.

---

## Estructura

```
News-Master-Reporter/
├── docker-compose.yml
├── .env.example
├── README.md
├── CLAUDE.md                        # Guía operativa para Claude Code
├── Prompts.txt                      # System prompts (evaluador + redactor)
└── workflow-export/
    ├── news-reporter-main.json      # Pipeline principal
    └── news-reporter-error.json     # Gestión de errores
```

---

## Autor

**Carles Escrich Andreu** — Ingeniero de Organización Industrial, consultor funcional de
ERP especializado en agentes de IA e hiperautomatización de procesos.

[LinkedIn](https://linkedin.com/in/CarlesEscrichAndreu) · [GitHub](https://github.com/CarlesEscrichAndreu)

Desarrollado durante el Máster en Agentes de IA e Hiperautomatización de Procesos
(EBIS Business Techschool + Universidad EUNEIZ).

## Licencia

MIT
