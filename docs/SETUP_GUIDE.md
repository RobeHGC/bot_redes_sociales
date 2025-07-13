# Guía de Configuración Detallada - Nadia Bot

## 1. Configuración de Telegram Bot con BotFather

### Paso 1: Crear el Bot
1. Abre Telegram y busca `@BotFather`
2. Envía el comando `/newbot`
3. Elige un nombre para tu bot (ej: "Nadia Medical Assistant")
4. Elige un username único que termine en `bot` (ej: `nadia_medical_bot`)
5. BotFather te dará un token como: `1234567890:ABCdefGHIjklmNOPqrstUVwxyz`

### Paso 2: Configurar el Bot
```
/setdescription - "Asistente IA para publicaciones médicas en redes sociales"
/setabouttext - "Bot que ayuda a médicos a crear y programar contenido"
/setuserpic - Sube una imagen de perfil
/setcommands - Define los comandos:
start - Iniciar el bot
generar - Crear nueva publicación
programar - Programar publicación
estado - Ver publicaciones pendientes
video - Generar video con Hedra
```

## 2. Configuración de Facebook API

### Paso 1: Crear App en Facebook Developers
1. Ve a https://developers.facebook.com/
2. Click en "My Apps" → "Create App"
3. Selecciona "Business" como tipo
4. Nombre: "Nadia Medical Publisher"

### Paso 2: Configurar Permisos
1. Ve a "Add Products" → "Facebook Login"
2. En Settings → Basic:
   - App ID: `1234567890`
   - App Secret: `abcdef1234567890`
3. Permisos necesarios:
   - `pages_manage_posts` - Publicar en páginas
   - `pages_read_engagement` - Leer métricas
   - `pages_show_list` - Listar páginas

### Paso 3: Obtener Page Access Token
1. Ve a Graph API Explorer
2. Selecciona tu app
3. Genera User Token con permisos
4. Intercambia por Page Token de larga duración

## 3. Sistema de Comandos Personalizados

### Arquitectura Extensible para Comandos

```python
# command_registry.py
class CommandRegistry:
    def __init__(self):
        self.commands = {}
        self.prompt_templates = {}
    
    def register_command(self, command: str, handler: callable, prompt_template: str = None):
        """Registra un nuevo comando con su handler y prompt opcional"""
        self.commands[command] = {
            'handler': handler,
            'prompt': prompt_template
        }
    
    def register_api_command(self, command: str, api_config: dict):
        """Registra comando que llama a API externa"""
        self.commands[command] = {
            'type': 'api',
            'config': api_config
        }

# Ejemplo de uso para Hedra
registry.register_api_command('video_hedra', {
    'api_endpoint': 'https://api.hedra.com/v1/generate',
    'auth_type': 'bearer',
    'prompt_template': """
    Genera un video médico con las siguientes características:
    - Character: {character_id}
    - Duración: {duration}
    - Script: {script}
    - Voz: Profesional médica, clara y empática
    """,
    'required_params': ['character_id', 'script', 'duration']
})
```

### Ejemplo de Comando Personalizado

```python
# commands/video_generator.py
async def handle_video_command(update, context):
    """Handler para generar videos con diferentes servicios"""
    args = context.args
    
    if args[0] == 'hedra':
        character = args[1] if len(args) > 1 else '3'
        
        # Buscar en caché o base de datos
        cached_prompt = await redis_client.get(f"video_prompt_{character}")
        
        if cached_prompt:
            # Usar prompt cacheado
            prompt = cached_prompt
        else:
            # Generar nuevo prompt
            prompt = prompt_templates.get('hedra_character_{}'.format(character))
        
        # Llamar a la API de Hedra
        video_url = await hedra_api.generate_video(
            character_id=character,
            prompt=prompt,
            voice_settings={...}
        )
        
        await update.message.reply_text(f"Video generado: {video_url}")
```

## 4. Configuración del Archivo .env

```bash
# Telegram
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklmNOPqrstUVwxyz

# Facebook
FACEBOOK_APP_ID=1234567890
FACEBOOK_APP_SECRET=abcdef1234567890
FACEBOOK_PAGE_ID=1234567890
FACEBOOK_PAGE_ACCESS_TOKEN=EAABsbCS...

# OpenAI/Gemini
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# PostgreSQL
DATABASE_URL=postgresql://user:password@localhost/nadia_db

# APIs Externas (Opcionales)
HEDRA_API_KEY=...
HEDRA_CHARACTER_3_ID=...

# Configuración
DAILY_POST_TIME=10:00
TIMEZONE=America/Mexico_City
```

## 5. Sistema de Plugins/Comandos Dinámicos

```python
# plugins/base.py
class PluginBase:
    """Clase base para todos los plugins"""
    
    def __init__(self, bot_instance):
        self.bot = bot_instance
        self.redis = bot_instance.redis
        self.db = bot_instance.db
    
    async def execute(self, update, context, **kwargs):
        raise NotImplementedError

# plugins/hedra_video.py
class HedraVideoPlugin(PluginBase):
    """Plugin para generar videos con Hedra"""
    
    command = "video"
    description = "Genera videos médicos con IA"
    
    async def execute(self, update, context, character_id=3):
        # Lógica para generar video
        pass

# main.py - Registro dinámico
plugin_manager = PluginManager()
plugin_manager.auto_discover('plugins/')  # Carga todos los plugins
```

## 6. Flujo de Comandos Personalizados

```yaml
# commands_config.yaml
commands:
  video:
    hedra:
      characters:
        1: "Doctor profesional, bata blanca"
        2: "Médica joven, consultorio moderno"
        3: "Especialista senior, hospital"
      prompt_template: |
        Crea un video médico educativo sobre {topic}
        Character: {character_description}
        Tono: Profesional pero accesible
        Duración: {duration} segundos
    
  infografia:
    canva:
      api_endpoint: "https://api.canva.com/v1/designs"
      templates:
        - "medical_facts"
        - "health_tips"
        - "symptom_checker"
```

¿Te gustaría que implemente el bot básico con esta arquitectura extensible para comandos personalizados?