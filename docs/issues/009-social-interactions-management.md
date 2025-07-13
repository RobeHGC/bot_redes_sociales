# Social Interactions Management

## Overview
Implement an intelligent social media interaction management system that automatically classifies, prioritizes, and suggests responses to comments, messages, and mentions across all platforms.

## Description
The social interactions management system monitors social media activity, classifies interactions by urgency and importance, provides intelligent response suggestions, and enables efficient management of social engagement. It includes medical emergency detection and automated filtering of spam content.

## Acceptance Criteria

### Interaction Monitoring
- [ ] **Multi-Platform Monitoring**: Track comments, messages, mentions across Facebook, Instagram, Twitter/X
- [ ] **Real-Time Processing**: Process new interactions within 15 minutes of posting
- [ ] **Batch Processing**: Periodic review of missed interactions
- [ ] **Notification Integration**: Connect with platform notification APIs
- [ ] **Activity Filtering**: Filter relevant medical interactions from noise

### Intelligent Classification System
- [ ] **Urgency Detection**: Classify interactions by medical urgency
  - **Urgent**: Medical emergencies requiring immediate attention
  - **Important**: Medical questions needing professional response
  - **Normal**: General comments and social interactions
  - **Spam**: Automated filtering of irrelevant content
- [ ] **Context Analysis**: Understand interaction context and medical relevance
- [ ] **Sentiment Analysis**: Detect positive, negative, or concerning sentiment
- [ ] **Question Detection**: Identify direct medical questions vs. general comments

### Response Management
- [ ] **Response Suggestions**: Generate appropriate response suggestions
- [ ] **Medical Disclaimers**: Automatic inclusion of appropriate disclaimers
- [ ] **Escalation Triggers**: Flag content requiring immediate attention
- [ ] **Response Templates**: Pre-approved responses for common scenarios
- [ ] **Tone Matching**: Match response tone to user's communication style

### User Interface & Workflow
- [ ] **Interaction Dashboard**: Overview of all pending interactions
- [ ] **Urgency Sorting**: Prioritized display based on classification
- [ ] **Bulk Operations**: Handle multiple similar interactions efficiently
- [ ] **Response History**: Track all responses and their outcomes
- [ ] **Quick Actions**: Fast approve/ignore/respond workflow

## Technical Implementation

### Interaction Monitoring System
```python
class SocialInteractionMonitor:
    def __init__(self):
        self.platform_monitors = {
            'facebook': FacebookMonitor(),
            'instagram': InstagramMonitor(), 
            'twitter': TwitterMonitor()
        }
        self.classifier = InteractionClassifier()
        self.processor = InteractionProcessor()
    
    async def monitor_interactions(self):
        """Main monitoring loop - runs every 15 minutes"""
        for platform_name, monitor in self.platform_monitors.items():
            try:
                new_interactions = await monitor.fetch_new_interactions()
                for interaction in new_interactions:
                    await self.process_interaction(interaction, platform_name)
            except Exception as e:
                await self.handle_monitor_error(platform_name, e)
    
    async def process_interaction(self, interaction: Interaction, platform: str):
        """Process a single interaction through the classification pipeline"""
        # 1. Classify urgency and type
        classification = await self.classifier.classify_interaction(interaction)
        
        # 2. Store with classification
        stored_interaction = await self.store_interaction(interaction, classification, platform)
        
        # 3. Handle based on urgency
        if classification.urgency == UrgencyLevel.URGENT:
            await self.handle_urgent_interaction(stored_interaction)
        elif classification.urgency == UrgencyLevel.IMPORTANT:
            await self.handle_important_interaction(stored_interaction)
        
        # 4. Generate response suggestions if needed
        if classification.needs_response:
            await self.generate_response_suggestions(stored_interaction)
```

### Platform-Specific Monitors
```python
class FacebookMonitor:
    def __init__(self):
        self.graph_api = FacebookGraphAPI()
        self.last_check = self.load_last_check_time()
    
    async def fetch_new_interactions(self) -> List[Interaction]:
        """Fetch new Facebook interactions since last check"""
        interactions = []
        
        # Get comments on recent posts
        comments = await self.graph_api.get_comments_since(self.last_check)
        for comment in comments:
            interactions.append(Interaction(
                id=comment['id'],
                type=InteractionType.COMMENT,
                platform='facebook',
                author=comment['from']['name'],
                author_id=comment['from']['id'],
                content=comment['message'],
                timestamp=comment['created_time'],
                post_id=comment['post_id'],
                metadata=comment
            ))
        
        # Get private messages
        messages = await self.graph_api.get_messages_since(self.last_check)
        for message in messages:
            interactions.append(Interaction(
                id=message['id'],
                type=InteractionType.MESSAGE,
                platform='facebook',
                author=message['from']['name'],
                author_id=message['from']['id'],
                content=message['message'],
                timestamp=message['created_time'],
                metadata=message
            ))
        
        self.last_check = datetime.now()
        return interactions

class InstagramMonitor:
    def __init__(self):
        self.basic_display_api = InstagramBasicDisplayAPI()
        self.last_check = self.load_last_check_time()
    
    async def fetch_new_interactions(self) -> List[Interaction]:
        """Fetch new Instagram interactions"""
        # Instagram Basic Display API has limited capabilities
        # Focus on comments and mentions that are accessible
        
class TwitterMonitor:
    def __init__(self):
        self.twitter_api = TwitterAPIv2()
        self.last_check = self.load_last_check_time()
    
    async def fetch_new_interactions(self) -> List[Interaction]:
        """Fetch new Twitter interactions"""
        interactions = []
        
        # Get mentions
        mentions = await self.twitter_api.get_mentions_since(self.last_check)
        # Get replies to tweets
        replies = await self.twitter_api.get_replies_since(self.last_check)
        # Get DMs
        dms = await self.twitter_api.get_dms_since(self.last_check)
        
        # Process all interaction types...
        return interactions
```

### Interaction Classification Engine
```python
class InteractionClassifier:
    def __init__(self):
        self.urgency_detector = MedicalUrgencyDetector()
        self.sentiment_analyzer = SentimentAnalyzer()
        self.question_detector = QuestionDetector()
        self.spam_filter = SpamFilter()
        
    async def classify_interaction(self, interaction: Interaction) -> Classification:
        """Classify interaction across multiple dimensions"""
        
        # Check for spam first
        if await self.spam_filter.is_spam(interaction):
            return Classification(
                urgency=UrgencyLevel.SPAM,
                type=InteractionType.SPAM,
                needs_response=False,
                confidence=0.95
            )
        
        # Detect medical urgency
        urgency_result = await self.urgency_detector.analyze(interaction.content)
        
        # Analyze sentiment
        sentiment = await self.sentiment_analyzer.analyze(interaction.content)
        
        # Detect questions
        is_question = await self.question_detector.is_medical_question(interaction.content)
        
        # Combine results into classification
        classification = Classification(
            urgency=urgency_result.urgency_level,
            type=self.determine_interaction_type(interaction, is_question),
            sentiment=sentiment,
            needs_response=self.needs_response(urgency_result, is_question, sentiment),
            confidence=self.calculate_confidence([urgency_result, sentiment, is_question]),
            topics=urgency_result.detected_topics,
            keywords=urgency_result.medical_keywords
        )
        
        return classification

class MedicalUrgencyDetector:
    def __init__(self):
        self.gemini_client = GeminiClient()
        self.emergency_keywords = [
            'emergency', 'urgent', 'immediately', 'ambulance', 'hospital',
            'chest pain', 'difficulty breathing', 'severe pain', 'bleeding',
            'unconscious', 'seizure', 'stroke', 'heart attack', 'overdose'
        ]
        
    async def analyze(self, content: str) -> UrgencyAnalysis:
        """Analyze content for medical urgency"""
        
        # Quick keyword scan for obvious emergencies
        content_lower = content.lower()
        emergency_score = sum(1 for keyword in self.emergency_keywords if keyword in content_lower)
        
        if emergency_score > 0:
            # Use LLM for detailed analysis of potential emergency
            urgency_prompt = f"""
            Analiza este mensaje para determinar si describe una emergencia médica:
            
            Mensaje: "{content}"
            
            Clasifica como:
            - URGENT: Emergencia médica que requiere atención inmediata
            - IMPORTANT: Pregunta médica seria que requiere respuesta profesional
            - NORMAL: Comentario general o pregunta no urgente
            
            Incluye:
            1. Nivel de urgencia
            2. Razones específicas
            3. Temas médicos detectados
            4. Palabras clave relevantes
            """
            
            result = await self.gemini_client.analyze(urgency_prompt)
            return self.parse_urgency_result(result)
        
        else:
            # Standard classification for non-emergency content
            return await self.standard_classification(content)
```

### Response Generation System
```python
class ResponseGenerator:
    def __init__(self):
        self.gemini_client = GeminiClient()
        self.template_manager = ResponseTemplateManager()
        self.disclaimer_engine = DisclaimerEngine()
    
    async def generate_response_suggestions(self, interaction: StoredInteraction) -> List[ResponseSuggestion]:
        """Generate multiple response options for an interaction"""
        
        classification = interaction.classification
        suggestions = []
        
        if classification.urgency == UrgencyLevel.URGENT:
            suggestions.extend(await self.generate_urgent_responses(interaction))
        
        elif classification.urgency == UrgencyLevel.IMPORTANT:
            suggestions.extend(await self.generate_important_responses(interaction))
        
        elif classification.urgency == UrgencyLevel.NORMAL:
            suggestions.extend(await self.generate_normal_responses(interaction))
        
        # Add appropriate disclaimers
        for suggestion in suggestions:
            suggestion.response_text = await self.disclaimer_engine.add_disclaimer(
                suggestion.response_text, 
                classification.type
            )
        
        return suggestions
    
    async def generate_urgent_responses(self, interaction: StoredInteraction) -> List[ResponseSuggestion]:
        """Generate responses for urgent medical situations"""
        
        # Template-based urgent responses
        urgent_template = self.template_manager.get_template('urgent_medical')
        template_response = urgent_template.format(
            name=interaction.author,
            situation=interaction.content
        )
        
        # Custom LLM response
        custom_prompt = f"""
        Esta persona describe una posible emergencia médica:
        "{interaction.content}"
        
        Genera una respuesta profesional que:
        1. Recomiende buscar atención médica inmediata
        2. Sea empática pero firme
        3. No dé consejos médicos específicos
        4. Incluya información de emergencia relevante
        
        Respuesta:
        """
        
        custom_response = await self.gemini_client.generate(custom_prompt)
        
        return [
            ResponseSuggestion(
                type='template',
                response_text=template_response,
                confidence=0.9,
                reasoning="Respuesta estándar para emergencias médicas"
            ),
            ResponseSuggestion(
                type='custom',
                response_text=custom_response,
                confidence=0.8,
                reasoning="Respuesta personalizada para la situación específica"
            )
        ]
```

### User Interface Integration
```python
class InteractionsDashboard:
    async def get_interactions_overview(self, user_id: str, timeframe: str = '4h') -> InteractionsOverview:
        """Get overview of interactions for dashboard"""
        
        interactions = await self.db.get_interactions_since(
            user_id, 
            datetime.now() - timedelta(hours=4)
        )
        
        overview = InteractionsOverview(
            urgent_count=len([i for i in interactions if i.classification.urgency == UrgencyLevel.URGENT]),
            important_count=len([i for i in interactions if i.classification.urgency == UrgencyLevel.IMPORTANT]),
            normal_count=len([i for i in interactions if i.classification.urgency == UrgencyLevel.NORMAL]),
            spam_filtered=len([i for i in interactions if i.classification.urgency == UrgencyLevel.SPAM]),
            total_pending=len([i for i in interactions if not i.responded]),
            platforms_summary=self.generate_platform_summary(interactions)
        )
        
        return overview
```

## User Experience Flows

### Daily Interaction Review
```
Tú: /revisar

Bot: 📱 Actividad últimas 4 horas:
     
     🚨 URGENTES (2):
     • "Mi bebé tiene fiebre de 39°C desde anoche" - María G. (hace 30 min)
       [Facebook comment]
     • "Dolor de pecho fuerte, qué hago?" - Carlos M. (hace 1 hora)
       [Instagram DM]
     
     ⭐ IMPORTANTES (5):
     • "¿Es normal esta presión arterial?" - Ana L. (hace 2 horas)
     • "Dudas sobre medicamento para diabetes" - Pedro R. (hace 3 horas)
     • "Síntomas después de vacuna" - Laura S. (hace 3 horas)
     
     💬 NORMALES (23): /comentarios para ver todos
     🗑️ SPAM filtrado automáticamente: 12
     
     ¿Qué quieres revisar primero?

Tú: /responder urgente_1

Bot: 🚨 URGENTE - María G. (Facebook):
     "Hola doctor, mi bebé de 8 meses tiene fiebre de 39°C desde anoche. 
     Le di paracetamol pero sigue igual. ¿Debo llevarlo al hospital?"
     
     🤖 Respuestas sugeridas:
     
     1. INMEDIATA: "María, fiebre alta en bebés requiere evaluación médica 
        urgente. Te recomiendo acudir a urgencias pediátricas ahora mismo."
     
     2. DETALLADA: [Respuesta más extensa con información específica]
     
     ¿Cuál prefieres? ¿O quieres que genere otra opción?

Tú: /aprobar 1

Bot: ✅ Respuesta enviada a María G.
     📋 Caso marcado como resuelto - urgente
     
     ➡️ Siguiente urgente: Carlos M. - dolor de pecho
```

### Bulk Interaction Management
```
Tú: /comentarios normales

Bot: 💬 INTERACCIONES NORMALES (23):
     
     📊 Tipos más comunes:
     • Agradecimientos por posts (8)
     • Preguntas sobre contenido anterior (6)  
     • Comentarios generales (5)
     • Solicitudes de temas (4)
     
     🔧 Acciones rápidas:
     /agradecer - Responder a agradecimientos
     /explicar - Responder preguntas sobre contenido
     /ignorar_spam - Marcar spam restante
     /ver_individual - Revisar uno por uno
     
     ¿Qué prefieres hacer?

Tú: /agradecer

Bot: ✅ Respondiendo automáticamente a 8 agradecimientos...
     📝 Respuestas personalizadas enviadas
     
     ➡️ Quedan 15 interacciones normales pendientes
```

## Performance Requirements
- [ ] **Monitoring Frequency**: Check all platforms every 15 minutes
- [ ] **Classification Speed**: <2s per interaction classification
- [ ] **Response Generation**: <3s for response suggestions
- [ ] **Dashboard Load**: <1s for interactions overview
- [ ] **Bulk Operations**: Handle 50+ interactions simultaneously

## Dependencies
- **Before**: #003 (Hybrid Command System), #008 (Long-term Memory System)
- **After**: #010 (Multi-platform Publishing)

## Testing Requirements
- [ ] **Unit Tests**: All classification and response generation logic
- [ ] **Integration Tests**: End-to-end interaction processing flows
- [ ] **API Tests**: All social media platform integrations
- [ ] **Classification Tests**: Urgency detection accuracy testing
- [ ] **Performance Tests**: System under high interaction volume

## Platform API Configuration
```yaml
social_platforms:
  facebook:
    graph_api_version: "v18.0"
    permissions: ["pages_messaging", "pages_read_engagement"]
    webhook_fields: ["comments", "messages", "mentions"]
    
  instagram:
    api_type: "basic_display"
    permissions: ["user_profile", "user_media"]
    limitations: "Limited comment access"
    
  twitter:
    api_version: "v2"
    permissions: ["read", "write", "direct_messages"]
    endpoints: ["mentions", "replies", "dms"]

monitoring:
  check_interval: "15min"
  batch_size: 100
  retry_attempts: 3
  timeout: 30s
```

## Error Handling
- [ ] **API Failures**: Graceful degradation when platforms are unavailable
- [ ] **Rate Limiting**: Respect all platform rate limits with backoff
- [ ] **Classification Errors**: Fallback to manual review when AI fails
- [ ] **Network Issues**: Retry mechanisms with exponential backoff
- [ ] **Data Corruption**: Validation and recovery for malformed interactions

## Labels
- `social-media`
- `interaction-management`
- `ai-classification`
- `medical-urgency`
- `user-interface`

## Estimated Effort
**Story Points**: 13
**Timeline**: 3-4 weeks

## Definition of Done
- [ ] All social media platforms monitored every 15 minutes
- [ ] Interaction classification achieving 90% accuracy for urgency detection
- [ ] Response generation providing helpful suggestions
- [ ] User interface enabling efficient interaction management
- [ ] Medical emergency detection working reliably
- [ ] Spam filtering removing 95% of irrelevant content
- [ ] Performance targets met for all operations
- [ ] Comprehensive test coverage (>90%)
- [ ] Error handling covering all platform failure scenarios