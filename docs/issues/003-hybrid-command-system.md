# Hybrid Command System Development

## Overview
Implement a sophisticated command routing system that supports both structured slash commands and natural language input with intelligent intent detection.

## Description
The hybrid command system is the user interface backbone of Nadia Bot, allowing users to interact using either precise slash commands or natural language. The system intelligently routes messages to appropriate handlers while maintaining context and state awareness.

## Acceptance Criteria

### Core Commands Implementation
- [ ] **Essential Commands**: Implement all 8 core slash commands
  - `/start` - Initialize/reinitialize bot
  - `/generar` - Create specific content
  - `/aprobar` - Approve proposed content
  - `/rechazar` - Reject and request alternative
  - `/editar` - Modify current content
  - `/status` - Show bot status and statistics
  - `/historial` - View previous posts
  - `/config` - Configure preferences

### Social Interaction Commands
- [ ] **Interaction Management**: Implement social media interaction commands
  - `/revisar` - Overview of comments/messages
  - `/responder` - Generate specific response
  - `/ignorar` - Mark as not important

### Natural Language Processing
- [ ] **Intent Detection**: Recognize user intentions from natural language
- [ ] **Command Mapping**: Map natural language to equivalent commands
  - "Hazlo más empático" → `/editar más empático`
  - "Perfecto, publícalo" → `/aprobar`
  - "Crea contenido sobre diabetes" → `/generar diabetes`

### Routing Intelligence
- [ ] **Pattern Recognition**: Detect known patterns without LLM calls
- [ ] **Context Awareness**: Route based on current conversation state
- [ ] **Fallback Processing**: Use Gemini for ambiguous cases
- [ ] **Error Handling**: Graceful handling of unrecognized inputs

## Technical Implementation

### Hybrid Router Architecture
```python
class HybridRouter:
    def __init__(self):
        self.command_patterns = {}
        self.natural_language_patterns = {}
        self.gemini_fallback = GeminiFallback()
    
    async def route_message(self, message: str, context: dict):
        """Main routing logic"""
        # 1. Direct command routing
        if message.startswith("/"):
            return await self.handle_command(message)
        
        # 2. Known pattern detection
        if pattern := self.detect_known_pattern(message):
            return await self.handle_pattern(message, pattern)
        
        # 3. LLM analysis for ambiguous cases
        return await self.analyze_with_gemini(message, context)
    
    def detect_known_pattern(self, message: str) -> Optional[str]:
        """Pattern matching without LLM"""
        patterns = {
            r"(perfecto|bien|ok|sí|dale|publícalo)": "approve",
            r"(no|rechazar|otra|diferente)": "reject", 
            r"(más|menos|cambiar|editar)": "edit",
            r"(estado|status|cómo va)": "status",
            r"(contenido|post|generar).*(diabetes|hipertensión|covid)": "generate_topic"
        }
        # Implementation details...
```

### Command Handler System
```python
class CommandHandler:
    def __init__(self):
        self.handlers = {
            'start': self.handle_start,
            'generar': self.handle_generate,
            'aprobar': self.handle_approve,
            'rechazar': self.handle_reject,
            'editar': self.handle_edit,
            'status': self.handle_status,
            'historial': self.handle_history,
            'config': self.handle_config,
            'revisar': self.handle_review,
            'responder': self.handle_respond,
            'ignorar': self.handle_ignore
        }
    
    async def execute_command(self, command: str, args: list, context: dict):
        """Execute command with proper error handling"""
```

### Context Management
- [ ] **State Tracking**: Maintain conversation state across interactions
- [ ] **User Preferences**: Remember user configuration and preferences
- [ ] **Session Management**: Handle multiple concurrent conversations
- [ ] **History Tracking**: Keep command history for learning and debugging

## Command Specifications

### Content Management Commands
```bash
/start
# Initialize bot, show welcome message and current status

/generar [topic]
# Generate content about specific topic or use auto-research
# Examples: /generar diabetes, /generar, /generar trending

/aprobar
# Approve current content for publishing across all platforms

/rechazar [reason]
# Reject current content and request alternative
# Example: /rechazar demasiado técnico

/editar [instructions]
# Modify current content with specific instructions
# Example: /editar más empático para pacientes jóvenes
```

### System Commands
```bash
/status [component]
# Show overall status or specific component status
# Examples: /status, /status clock, /status social

/historial [count] [filter]
# Show previous posts with optional filtering
# Examples: /historial, /historial 10, /historial diabetes

/config [setting] [value]
# Configure bot settings
# Examples: /config, /config timezone America/Mexico_City
```

### Social Interaction Commands
```bash
/revisar [timeframe]
# Review social media interactions
# Examples: /revisar, /revisar 24h, /revisar urgentes

/responder [interaction_id] [message]
# Generate or send response to specific interaction
# Example: /responder urgent_1

/ignorar [interaction_id]
# Mark interaction as not important
# Example: /ignorar normal_5
```

## Natural Language Examples
```python
natural_language_mappings = {
    # Approval variations
    "Perfecto, publícalo": "/aprobar",
    "Está bien así": "/aprobar", 
    "Me gusta, dale": "/aprobar",
    
    # Editing requests
    "Hazlo más empático": "/editar más empático",
    "Más técnico por favor": "/editar más técnico",
    "Agrega estadísticas": "/editar incluir estadísticas",
    
    # Content generation
    "Crea algo sobre diabetes": "/generar diabetes",
    "Hazme un post": "/generar",
    "Qué hay de nuevo": "/generar trending",
    
    # Status requests
    "¿Cómo va todo?": "/status",
    "Qué has publicado": "/historial",
    "Revisa comentarios": "/revisar"
}
```

## Error Handling & Fallbacks
- [ ] **Unknown Commands**: Helpful suggestions for similar commands
- [ ] **Malformed Input**: Parse and suggest corrections
- [ ] **Context Errors**: Handle commands in wrong state gracefully
- [ ] **API Failures**: Fallback to cached responses when possible

## Performance Requirements
- [ ] **Response Time**: <500ms for command processing
- [ ] **Pattern Matching**: <100ms for known pattern detection
- [ ] **LLM Fallback**: <2s for Gemini analysis when needed
- [ ] **Memory Usage**: Efficient context storage and cleanup

## Dependencies
- **Before**: #002 (Internal Clock System)
- **After**: #006 (Content Generation Engine), #009 (Social Interactions)

## Testing Requirements
- [ ] **Unit Tests**: All command handlers and routing logic
- [ ] **Integration Tests**: End-to-end command execution flows
- [ ] **Natural Language Tests**: Intent detection accuracy testing
- [ ] **Performance Tests**: Response time under load
- [ ] **Error Handling**: All error scenarios and edge cases

## User Experience Design
```
Tú: "Crea contenido sobre hipertensión para adultos jóvenes"

Bot: 🎯 Entendido: generar contenido sobre hipertensión enfocado a adultos jóvenes
     📝 Generando contenido...
     
     [Contenido generado]
     
     ¿Te gusta o necesitas ajustes?

Tú: "Más casos prácticos"

Bot: ✏️ Agregando ejemplos prácticos...
     [Contenido editado]
     
     ¿Mejor así?

Tú: "/aprobar"

Bot: ✅ ¡Perfecto! Publicando en todas las plataformas...
```

## Labels
- `core-system`
- `high-priority`
- `user-interface`
- `nlp`
- `command-handling`

## Estimated Effort
**Story Points**: 8
**Timeline**: 2-3 weeks

## Definition of Done
- [ ] All 11 commands implemented and functional
- [ ] Natural language processing achieving 95% accuracy
- [ ] Response times meeting performance targets
- [ ] Context management working across sessions
- [ ] Error handling covering all edge cases
- [ ] Comprehensive test coverage (>95%)
- [ ] User experience flows validated
- [ ] Documentation complete with examples