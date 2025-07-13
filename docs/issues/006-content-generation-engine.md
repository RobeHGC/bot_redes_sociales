# Content Generation Engine

## Overview
Implement an adaptive content generation system using Gemini 2.0 Flash as primary LLM with OpenAI GPT-4 fallback, featuring automatic validation, personalization, and multi-platform optimization.

## Description
The content generation engine is the creative core of Nadia Bot, responsible for transforming medical research, trending topics, and user requests into engaging, accurate, and platform-optimized social media content. It maintains medical accuracy while ensuring content is accessible and engaging for healthcare professionals and patients.

## Acceptance Criteria

### Primary LLM Integration (Gemini 2.0 Flash)
- [ ] **Gemini Client**: Full integration with Google's Gemini 2.0 Flash API
- [ ] **Context Window**: Leverage 2M token context window for comprehensive analysis
- [ ] **Cost Optimization**: Implement cost-effective prompt strategies
- [ ] **Rate Limiting**: Respect API quotas and implement backoff strategies
- [ ] **Response Streaming**: Support streaming responses for better UX

### Fallback System (OpenAI GPT-4)
- [ ] **Automatic Fallback**: Switch to GPT-4 when Gemini fails or is unavailable
- [ ] **Function Calling**: Use OpenAI for complex function calling scenarios
- [ ] **Quality Comparison**: Monitor output quality between models
- [ ] **Cost Tracking**: Track usage and costs for both LLM providers
- [ ] **Seamless Integration**: Transparent fallback without user disruption

### Content Validation System
- [ ] **Medical Accuracy**: Verify medical information correctness
- [ ] **Tone Personalization**: Match user's preferred communication style
- [ ] **Disclaimer Integration**: Add appropriate medical disclaimers automatically
- [ ] **Reference Formatting**: Include scientific references when applicable
- [ ] **Length Optimization**: Generate appropriate length for each platform

### Multi-Platform Optimization
- [ ] **Twitter/X Format**: Concise, engaging posts under character limit
- [ ] **Instagram Content**: Visual-friendly content with hashtags
- [ ] **Facebook Posts**: Detailed, informative longer-form content
- [ ] **Story Versions**: Short, punchy content for Instagram Stories
- [ ] **Hashtag Generation**: Platform-appropriate and trending hashtags

## Technical Implementation

### Content Generation Pipeline
```python
class ContentGenerationEngine:
    def __init__(self):
        self.gemini_client = GeminiClient()
        self.openai_client = OpenAIClient()
        self.validator = ContentValidator()
        self.personalizer = ContentPersonalizer()
        self.platform_optimizer = PlatformOptimizer()
    
    async def generate_content(self, 
                             content_request: ContentRequest,
                             user_profile: UserProfile) -> GeneratedContent:
        """Main content generation pipeline"""
        
        # 1. Prepare context and prompts
        context = await self.prepare_context(content_request, user_profile)
        
        # 2. Generate with primary LLM (Gemini)
        try:
            raw_content = await self.generate_with_gemini(context)
        except Exception as e:
            # 3. Fallback to OpenAI if Gemini fails
            raw_content = await self.generate_with_openai(context)
        
        # 4. Validate content
        validated_content = await self.validator.validate(raw_content)
        
        # 5. Personalize for user
        personalized_content = await self.personalizer.personalize(
            validated_content, user_profile
        )
        
        # 6. Optimize for platforms
        platform_content = await self.platform_optimizer.optimize(
            personalized_content
        )
        
        return platform_content
```

### Gemini Integration
```python
class GeminiClient:
    def __init__(self):
        self.client = genai.GenerativeModel('gemini-2.0-flash-exp')
        self.max_tokens = 2000000  # 2M context window
        
    async def generate_medical_content(self, prompt: str, context: dict) -> str:
        """Generate medical content with Gemini"""
        
        system_prompt = """
        Eres Nadia, un asistente médico experto en crear contenido científicamente 
        preciso y atractivo para redes sociales. Tu especialidad es transformar 
        información médica compleja en contenido accesible y profesional.
        
        SIEMPRE:
        - Mantén precisión científica absoluta
        - Usa un tono profesional pero accesible
        - Incluye referencias cuando sea apropiado
        - Agrega disclaimers médicos cuando sea necesario
        - Optimiza para engagement profesional
        """
        
        full_prompt = f"{system_prompt}\n\nContexto: {context}\n\nSolicitud: {prompt}"
        
        response = await self.client.generate_content_async(
            full_prompt,
            generation_config=genai.types.GenerationConfig(
                temperature=0.7,
                max_output_tokens=2048,
                top_p=0.9
            )
        )
        
        return response.text
```

### Content Validation Framework
```python
class ContentValidator:
    def __init__(self):
        self.medical_fact_checker = MedicalFactChecker()
        self.tone_analyzer = ToneAnalyzer()
        self.disclaimer_engine = DisclaimerEngine()
    
    async def validate(self, content: str) -> ValidationResult:
        """Comprehensive content validation"""
        
        validations = await asyncio.gather(
            self.medical_fact_checker.verify(content),
            self.tone_analyzer.analyze(content),
            self.disclaimer_engine.check_requirements(content)
        )
        
        return ValidationResult(
            medical_accuracy=validations[0],
            tone_appropriateness=validations[1], 
            disclaimer_compliance=validations[2],
            overall_score=self.calculate_overall_score(validations)
        )

class MedicalFactChecker:
    async def verify(self, content: str) -> MedicalAccuracyScore:
        """Verify medical accuracy of content"""
        # Use specialized medical validation prompts
        # Cross-reference with medical databases
        # Flag potential inaccuracies or unsupported claims
```

### Platform Optimization System
```python
class PlatformOptimizer:
    def __init__(self):
        self.platform_configs = {
            "twitter": {"max_length": 280, "hashtag_count": 3},
            "instagram": {"max_length": 2200, "hashtag_count": 10},
            "facebook": {"max_length": 5000, "hashtag_count": 5}
        }
    
    async def optimize_for_platform(self, content: str, platform: str) -> OptimizedContent:
        """Optimize content for specific platform"""
        config = self.platform_configs[platform]
        
        if platform == "twitter":
            return await self.optimize_for_twitter(content, config)
        elif platform == "instagram":
            return await self.optimize_for_instagram(content, config)
        elif platform == "facebook":
            return await self.optimize_for_facebook(content, config)
    
    async def optimize_for_twitter(self, content: str, config: dict) -> TwitterContent:
        """Twitter-specific optimization"""
        # Shorten content while preserving key message
        # Add impactful hashtags
        # Ensure thread compatibility if needed
        
    async def optimize_for_instagram(self, content: str, config: dict) -> InstagramContent:
        """Instagram-specific optimization"""
        # Visual-friendly formatting
        # Emoji integration
        # Hashtag optimization
        # Story version creation
```

## Content Generation Scenarios

### Daily Content Generation
```python
daily_generation_prompt = """
Basándote en la investigación diaria y tendencias médicas, crea contenido para:

CONTEXTO:
- Hora: {current_time}
- Trending topics: {trending_topics}
- Artículos nuevos: {new_articles}
- Especialidad usuario: {user_specialty}
- Estilo preferido: {user_tone}

GENERAR:
Un post médico profesional que:
1. Aborde uno de los trending topics relevantes
2. Incluya evidencia científica reciente
3. Sea atractivo para colegas médicos
4. Tenga aplicación clínica práctica
5. Use el tono personalizado del usuario

FORMATO: Post listo para publicar con referencias y hashtags
"""
```

### Article-Based Content
```python
article_content_prompt = """
Transforma este artículo médico en contenido atractivo para redes sociales:

ARTÍCULO:
- Título: {article_title}
- Autores: {authors}
- Journal: {journal}
- Hallazgos principales: {key_findings}
- Relevancia clínica: {clinical_relevance}

CREAR:
1. Mensaje principal en lenguaje accesible
2. Destacar implicaciones prácticas
3. Incluir estadísticas relevantes
4. Agregar contexto para México (si aplica)
5. Referencias apropiadas

ESTILO: {user_tone} - {user_specialty}
"""
```

### Trending Topic Content
```python
trending_topic_prompt = """
Crea contenido sobre este trending topic médico:

TOPIC: {trending_topic}
CONTEXTO: {trend_context}
EVIDENCIA: {supporting_evidence}
CONTROVERSIAS: {controversies}

DESARROLLAR:
1. Posición balanceada basada en evidencia
2. Implicaciones para práctica clínica
3. Recomendaciones prácticas
4. Consideraciones para pacientes
5. Perspectiva actualizada del tema

AUDIENCIA: Profesionales de la salud
TONO: Profesional, informativo, equilibrado
"""
```

## Personalization Engine
```python
class ContentPersonalizer:
    def __init__(self):
        self.user_profiles = UserProfileManager()
        self.style_analyzer = StyleAnalyzer()
    
    async def personalize(self, content: str, user_profile: UserProfile) -> str:
        """Personalize content for specific user"""
        
        personalization_factors = {
            "specialty": user_profile.medical_specialty,
            "tone_preference": user_profile.communication_style,
            "audience_focus": user_profile.target_audience,
            "technical_level": user_profile.technical_depth,
            "regional_context": user_profile.geographic_context
        }
        
        return await self.apply_personalization(content, personalization_factors)
```

## Quality Assurance
- [ ] **Medical Accuracy**: 95% accuracy in medical statements
- [ ] **Source Verification**: All claims backed by credible sources
- [ ] **Tone Consistency**: Maintain user's preferred communication style
- [ ] **Platform Compliance**: Meet all platform-specific requirements
- [ ] **Engagement Optimization**: Generate content likely to engage target audience

## Performance Requirements
- [ ] **Generation Speed**: <3s for standard content generation
- [ ] **Gemini Response**: <2s average response time
- [ ] **Fallback Speed**: <5s total including OpenAI fallback
- [ ] **Validation Time**: <1s for content validation
- [ ] **Platform Optimization**: <500ms for all platform versions

## Dependencies
- **Before**: #004 (Medical Article Processor), #005 (Daily Research Automation)
- **After**: #007 (Approval Loop System)

## Testing Requirements
- [ ] **Unit Tests**: All generation and validation components
- [ ] **Integration Tests**: End-to-end content generation flows
- [ ] **LLM Tests**: Both Gemini and OpenAI integration reliability
- [ ] **Quality Tests**: Medical accuracy and tone consistency
- [ ] **Performance Tests**: Generation speed under various loads
- [ ] **A/B Tests**: Compare Gemini vs OpenAI output quality

## Error Handling
- [ ] **LLM Failures**: Seamless fallback between providers
- [ ] **Validation Failures**: Regeneration with adjusted prompts
- [ ] **Rate Limiting**: Queue management and backoff strategies
- [ ] **Content Rejection**: Alternative generation strategies
- [ ] **Platform Errors**: Graceful degradation for platform-specific issues

## Configuration
```yaml
content_generation:
  primary_llm: "gemini-2.0-flash"
  fallback_llm: "gpt-4"
  
  gemini:
    model: "gemini-2.0-flash-exp"
    temperature: 0.7
    max_tokens: 2048
    top_p: 0.9
    
  openai:
    model: "gpt-4"
    temperature: 0.7
    max_tokens: 1500
    
  validation:
    medical_accuracy_threshold: 0.95
    tone_consistency_threshold: 0.8
    
  personalization:
    style_adaptation: true
    regional_context: "Mexico"
    default_specialty: "family_medicine"
```

## Labels
- `core-system`
- `high-priority`
- `ai-integration`
- `content-creation`
- `medical-accuracy`

## Estimated Effort
**Story Points**: 13
**Timeline**: 3-4 weeks

## Definition of Done
- [ ] Gemini 2.0 Flash integration fully functional
- [ ] OpenAI fallback system working seamlessly
- [ ] Content validation achieving 95% medical accuracy
- [ ] Multi-platform optimization for all target platforms
- [ ] Personalization engine adapting to user preferences
- [ ] Performance targets met for generation speed
- [ ] Comprehensive test coverage (>90%)
- [ ] Error handling covering all failure scenarios
- [ ] Cost tracking and optimization implemented