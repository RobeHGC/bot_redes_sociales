# Approval Loop System

## Overview
Implement an infinite approval loop system with conversational states, iterative content refinement, and intelligent editing capabilities to ensure perfect content before publishing.

## Description
The approval loop system manages the iterative content refinement process, allowing users to review, edit, approve, or reject generated content. It maintains conversational state, handles multiple editing attempts, and provides intelligent suggestions when the editing process becomes extensive.

## Acceptance Criteria

### Conversational State Management
- [ ] **State Machine**: Implement FSM with defined states and transitions
- [ ] **State Persistence**: Maintain state across bot restarts and sessions
- [ ] **Context Preservation**: Keep content and editing history throughout loop
- [ ] **Timeout Handling**: Manage abandoned approval sessions gracefully
- [ ] **Multi-User Support**: Handle concurrent approval sessions for different users

### Core Approval States
- [ ] **IDLE**: Bot waiting for new content requests
- [ ] **GENERATING**: Content creation in progress
- [ ] **WAITING_APPROVAL**: Presenting content for user review
- [ ] **EDITING**: Processing user editing requests
- [ ] **APPROVED**: Content approved and ready for publishing
- [ ] **ARTICLE_PROCESSING**: Processing user-submitted articles

### Editing Intelligence
- [ ] **Natural Language Edits**: Process editing requests in natural language
- [ ] **Context-Aware Changes**: Apply edits while maintaining content coherence
- [ ] **Version Management**: Track all content versions and changes
- [ ] **Edit History**: Maintain log of all editing attempts and user feedback
- [ ] **Suggestion Intelligence**: Learn from edits to improve future generations

### Loop Management
- [ ] **Infinite Loop Support**: Allow unlimited editing attempts
- [ ] **5-Attempt Threshold**: Special handling after 5 editing attempts
- [ ] **User Options**: Provide alternatives when editing becomes extensive
- [ ] **Best Version Recovery**: Offer previous versions when current attempt fails
- [ ] **Session Management**: Handle long-running approval sessions

## Technical Implementation

### State Machine Architecture
```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional, List, Dict

class BotState(Enum):
    IDLE = "idle"
    GENERATING = "generating"
    WAITING_APPROVAL = "waiting_approval"
    EDITING = "editing"
    APPROVED = "approved"
    ARTICLE_PROCESSING = "article_processing"
    ERROR = "error"

@dataclass
class ApprovalSession:
    session_id: str
    user_id: str
    current_state: BotState
    content_versions: List[ContentVersion]
    edit_attempts: int
    created_at: datetime
    last_activity: datetime
    original_request: str
    user_feedback_history: List[str]

class ApprovalLoopFSM:
    def __init__(self):
        self.sessions: Dict[str, ApprovalSession] = {}
        self.state_handlers = {
            BotState.IDLE: self.handle_idle,
            BotState.GENERATING: self.handle_generating,
            BotState.WAITING_APPROVAL: self.handle_waiting_approval,
            BotState.EDITING: self.handle_editing,
            BotState.APPROVED: self.handle_approved,
            BotState.ARTICLE_PROCESSING: self.handle_article_processing
        }
    
    async def process_message(self, user_id: str, message: str) -> BotResponse:
        """Main state machine message processor"""
        session = self.get_or_create_session(user_id)
        handler = self.state_handlers[session.current_state]
        return await handler(session, message)
    
    async def transition_to(self, session: ApprovalSession, new_state: BotState, 
                          context: Optional[dict] = None):
        """Handle state transitions with proper cleanup and setup"""
        old_state = session.current_state
        session.current_state = new_state
        session.last_activity = datetime.now()
        
        # State transition logging and cleanup
        await self.log_state_transition(session, old_state, new_state, context)
        await self.save_session(session)
```

### Content Version Management
```python
@dataclass
class ContentVersion:
    version_id: str
    content: str
    platform_versions: Dict[str, str]  # twitter, instagram, facebook
    created_at: datetime
    edit_request: Optional[str]
    quality_score: float
    user_rating: Optional[int]

class ContentVersionManager:
    def __init__(self):
        self.versions: Dict[str, List[ContentVersion]] = {}
    
    def add_version(self, session_id: str, content: str, edit_request: str = None):
        """Add new content version to session"""
        version = ContentVersion(
            version_id=f"{session_id}_v{len(self.versions.get(session_id, [])) + 1}",
            content=content,
            platform_versions=self.generate_platform_versions(content),
            created_at=datetime.now(),
            edit_request=edit_request,
            quality_score=self.calculate_quality_score(content)
        )
        
        if session_id not in self.versions:
            self.versions[session_id] = []
        self.versions[session_id].append(version)
        return version
    
    def get_best_version(self, session_id: str) -> Optional[ContentVersion]:
        """Get highest quality version from session"""
        versions = self.versions.get(session_id, [])
        if not versions:
            return None
        return max(versions, key=lambda v: v.quality_score)
```

### State Handlers Implementation
```python
class ApprovalLoopFSM:
    async def handle_waiting_approval(self, session: ApprovalSession, message: str) -> BotResponse:
        """Handle user input when waiting for approval"""
        
        intent = await self.analyze_user_intent(message, session)
        
        if intent == "approve":
            await self.transition_to(session, BotState.APPROVED)
            return BotResponse(
                text="✅ ¡Perfecto! Publicando contenido en todas las plataformas...",
                action="publish_content"
            )
        
        elif intent == "reject":
            await self.transition_to(session, BotState.GENERATING)
            return BotResponse(
                text="🔄 Entendido, generando una alternativa...",
                action="regenerate_content"
            )
        
        elif intent == "edit":
            await self.transition_to(session, BotState.EDITING)
            return await self.process_edit_request(session, message)
        
        else:
            return BotResponse(
                text="No entendí tu respuesta. ¿Quieres aprobar (/aprobar), rechazar (/rechazar) o editar el contenido?",
                action="clarify_intent"
            )
    
    async def handle_editing(self, session: ApprovalSession, message: str) -> BotResponse:
        """Handle editing requests"""
        session.edit_attempts += 1
        
        # Check if we've hit the 5-attempt threshold
        if session.edit_attempts >= 5:
            return await self.handle_excessive_editing(session, message)
        
        # Process the edit request
        edited_content = await self.apply_edit_request(
            session.content_versions[-1].content, 
            message
        )
        
        # Add new version
        new_version = self.version_manager.add_version(
            session.session_id, 
            edited_content, 
            edit_request=message
        )
        
        await self.transition_to(session, BotState.WAITING_APPROVAL)
        
        return BotResponse(
            text=f"✏️ Contenido editado (Intento {session.edit_attempts}/∞):\n\n{edited_content}\n\n¿Te gusta esta versión?",
            content_version=new_version,
            action="present_edited_content"
        )
    
    async def handle_excessive_editing(self, session: ApprovalSession, message: str) -> BotResponse:
        """Handle when user has made 5+ editing attempts"""
        best_version = self.version_manager.get_best_version(session.session_id)
        
        options_text = f"""
🤔 Hemos intentado {session.edit_attempts} veces. ¿Qué prefieres?

1. /generar - Contenido completamente nuevo desde cero
2. /aprobar - Usar la mejor versión anterior (v{best_version.version_id if best_version else 'N/A'})
3. /continuar - Seguir editando esta versión
4. /programar - Dejarlo para más tarde

¿Qué eliges?
        """
        
        return BotResponse(
            text=options_text,
            action="offer_alternatives",
            best_version=best_version
        )
```

### Edit Processing Engine
```python
class EditProcessor:
    def __init__(self):
        self.gemini_client = GeminiClient()
        self.edit_analyzer = EditAnalyzer()
    
    async def apply_edit_request(self, original_content: str, edit_request: str) -> str:
        """Apply user edit request to content"""
        
        edit_prompt = f"""
        CONTENIDO ORIGINAL:
        {original_content}
        
        SOLICITUD DE EDICIÓN:
        {edit_request}
        
        INSTRUCCIONES:
        1. Aplica la edición solicitada manteniendo la calidad y coherencia
        2. Preserva la precisión médica
        3. Mantén el tono profesional apropiado
        4. Asegúrate de que el mensaje principal se mantenga claro
        5. Conserva referencias y disclaimers si los había
        
        CONTENIDO EDITADO:
        """
        
        return await self.gemini_client.generate(edit_prompt)
    
    async def analyze_edit_difficulty(self, edit_request: str) -> EditDifficulty:
        """Analyze how complex an edit request is"""
        # Implementation to categorize edit complexity
        # Simple: tone changes, small additions
        # Medium: structural changes, content additions
        # Complex: complete rewrites, conflicting requirements
```

## User Experience Flows

### Standard Approval Flow
```
Bot: 📝 Contenido generado sobre diabetes tipo 2:

     [Contenido mostrado]
     
     ¿Te gusta o necesitas ajustes?

Tú: "Más enfocado a adultos jóvenes"

Bot: ✏️ Ajustando para adultos jóvenes... (Intento 1/∞)
     
     [Contenido editado]
     
     ¿Mejor así?

Tú: "/aprobar"

Bot: ✅ ¡Perfecto! Publicando en Facebook, Instagram, Twitter...
     📅 Contenido programado para las próximas 2 horas
```

### Extensive Editing Flow
```
Bot: ✏️ Contenido editado (Intento 5/∞):
     
     [Contenido mostrado]
     
     🤔 Hemos intentado 5 veces. ¿Qué prefieres?
     
     1. /generar - Contenido completamente nuevo
     2. /aprobar - Usar la mejor versión anterior (v3)
     3. /continuar - Seguir editando esta versión
     4. /programar - Dejarlo para más tarde

Tú: "/aprobar"

Bot: ✅ Usando la mejor versión (v3) - Publicando...
```

### Context Preservation
```python
class ContextManager:
    def preserve_editing_context(self, session: ApprovalSession) -> dict:
        """Preserve context across editing attempts"""
        return {
            "original_request": session.original_request,
            "edit_history": [v.edit_request for v in session.content_versions if v.edit_request],
            "user_preferences": self.extract_preferences_from_history(session),
            "content_evolution": self.track_content_changes(session),
            "quality_progression": self.track_quality_changes(session)
        }
```

## Performance Requirements
- [ ] **State Transitions**: <100ms for state changes
- [ ] **Edit Processing**: <3s for content editing
- [ ] **Session Management**: Support 50+ concurrent sessions
- [ ] **Memory Efficiency**: Efficient storage of content versions
- [ ] **Response Time**: <500ms for approval/rejection responses

## Dependencies
- **Before**: #006 (Content Generation Engine)
- **After**: #010 (Multi-platform Publishing)

## Testing Requirements
- [ ] **Unit Tests**: All state machine transitions and handlers
- [ ] **Integration Tests**: End-to-end approval flows
- [ ] **State Tests**: All possible state combinations and edge cases
- [ ] **Performance Tests**: Concurrent session handling
- [ ] **User Experience Tests**: Complete approval scenarios

## Error Handling
- [ ] **Session Timeouts**: Clean up abandoned sessions after 24h
- [ ] **State Corruption**: Recovery mechanisms for invalid states
- [ ] **Edit Failures**: Fallback when edit processing fails
- [ ] **Version Conflicts**: Handle concurrent editing attempts
- [ ] **Memory Management**: Prevent memory leaks from long sessions

## Configuration
```yaml
approval_loop:
  max_edit_attempts_warning: 5
  session_timeout: 24h
  max_concurrent_sessions: 100
  version_retention: 30d
  
  state_timeouts:
    generating: 30s
    editing: 60s
    waiting_approval: 12h
    
  quality_thresholds:
    minimum_score: 0.7
    excellent_score: 0.9
```

## Labels
- `core-system`
- `high-priority`
- `user-experience`
- `state-management`
- `content-refinement`

## Estimated Effort
**Story Points**: 8
**Timeline**: 2-3 weeks

## Definition of Done
- [ ] All FSM states implemented and working correctly
- [ ] Infinite editing loop supporting 5+ attempts gracefully
- [ ] Content version management tracking all changes
- [ ] Natural language edit processing functional
- [ ] Session persistence across bot restarts
- [ ] Performance targets met for all operations
- [ ] Comprehensive test coverage (>95%)
- [ ] User experience flows validated
- [ ] Error recovery mechanisms operational