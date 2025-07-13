# Long-term Memory System

## Overview
Implement an intelligent memory system that tracks content history, prevents repetition, manages topic lifecycle, and learns from user interactions to improve future content generation.

## Description
The long-term memory system is the learning brain of Nadia Bot, ensuring content diversity while building knowledge about user preferences, successful topics, and content patterns. It prevents repetitive posting while strategically reusing successful content with new perspectives.

## Acceptance Criteria

### Topic Tracking System
- [ ] **Topic Deduplication**: Avoid repeating similar topics within 30 days
- [ ] **Strategic Reuse**: Allow topic reuse after 90+ days with fresh perspective
- [ ] **Topic Lifecycle**: Track topic introduction, peak, and decline phases
- [ ] **Semantic Similarity**: Detect conceptually similar topics beyond keyword matching
- [ ] **Topic Evolution**: Track how topics develop and change over time

### Article Source Management
- [ ] **Source Tracking**: Never reuse article sources within configured timeframe
- [ ] **Source Quality Rating**: Track success rates of different source types
- [ ] **Source Diversity**: Ensure variety in content sources (PubMed, Medscape, WHO)
- [ ] **Reference History**: Maintain complete history of all referenced articles
- [ ] **Source Blacklisting**: Mark and avoid poor-performing sources

### Learning & Intelligence
- [ ] **User Preference Learning**: Adapt to user's successful content patterns
- [ ] **Engagement Analysis**: Track which topics generate most engagement
- [ ] **Edit Pattern Recognition**: Learn from user edits to improve generation
- [ ] **Success Rate Tracking**: Monitor approval rates by topic and style
- [ ] **Seasonal Awareness**: Recognize seasonal medical topics and trends

### Content Performance Analytics
- [ ] **Post Success Metrics**: Track likes, shares, comments, engagement
- [ ] **Topic Performance**: Analyze which medical topics perform best
- [ ] **Platform Optimization**: Learn platform-specific preferences
- [ ] **Time-based Analysis**: Identify optimal posting times and patterns
- [ ] **Audience Response**: Track audience preferences and feedback

## Technical Implementation

### Database Schema
```sql
-- Topic history and lifecycle management
CREATE TABLE topic_history (
    id UUID PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    topic_hash VARCHAR(64) NOT NULL,
    topic_title VARCHAR(500),
    topic_keywords TEXT[],
    last_used TIMESTAMP NOT NULL,
    usage_count INTEGER DEFAULT 1,
    success_rate FLOAT DEFAULT 0.0,
    avg_engagement FLOAT DEFAULT 0.0,
    seasonal_pattern JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(user_id, topic_hash),
    INDEX idx_topic_last_used (user_id, last_used),
    INDEX idx_topic_success (user_id, success_rate DESC)
);

-- Article source tracking
CREATE TABLE article_sources (
    id UUID PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    source_url TEXT NOT NULL,
    source_type VARCHAR(50) NOT NULL, -- pubmed, doi, medscape, text
    source_title TEXT,
    source_authors TEXT[],
    source_journal VARCHAR(200),
    used_date TIMESTAMP NOT NULL,
    success_rating INTEGER CHECK (success_rating BETWEEN 1 AND 5),
    engagement_score FLOAT DEFAULT 0.0,
    user_feedback TEXT,
    content_generated TEXT,
    
    UNIQUE(user_id, source_url),
    INDEX idx_source_used_date (user_id, source_type, used_date),
    INDEX idx_source_success (user_id, success_rating DESC)
);

-- Content performance tracking
CREATE TABLE content_performance (
    id UUID PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    content_id VARCHAR(100) NOT NULL,
    topic_hash VARCHAR(64),
    source_id UUID REFERENCES article_sources(id),
    platform VARCHAR(50) NOT NULL,
    published_at TIMESTAMP NOT NULL,
    
    -- Engagement metrics
    likes_count INTEGER DEFAULT 0,
    shares_count INTEGER DEFAULT 0,
    comments_count INTEGER DEFAULT 0,
    engagement_rate FLOAT DEFAULT 0.0,
    
    -- Content metadata
    content_length INTEGER,
    hashtag_count INTEGER,
    has_image BOOLEAN DEFAULT FALSE,
    has_reference BOOLEAN DEFAULT FALSE,
    
    -- User feedback
    user_rating INTEGER CHECK (user_rating BETWEEN 1 AND 5),
    edit_attempts INTEGER DEFAULT 0,
    approval_time INTERVAL,
    
    INDEX idx_performance_topic (user_id, topic_hash, published_at),
    INDEX idx_performance_platform (user_id, platform, engagement_rate DESC)
);

-- User learning profile
CREATE TABLE user_learning_profile (
    user_id VARCHAR(100) PRIMARY KEY,
    
    -- Content preferences learned from history
    preferred_content_length JSONB, -- {twitter: 200, instagram: 800, facebook: 1200}
    preferred_hashtag_count JSONB,
    preferred_tone VARCHAR(50),
    preferred_complexity_level VARCHAR(50),
    
    -- Successful patterns
    successful_topics TEXT[],
    successful_sources TEXT[],
    successful_posting_times TIME[],
    
    -- Engagement patterns
    avg_engagement_rate FLOAT DEFAULT 0.0,
    best_performing_platform VARCHAR(50),
    audience_demographics JSONB,
    
    -- Learning metadata
    total_posts INTEGER DEFAULT 0,
    learning_confidence FLOAT DEFAULT 0.0,
    last_updated TIMESTAMP DEFAULT NOW()
);
```

### Memory Management System
```python
class LongTermMemorySystem:
    def __init__(self):
        self.db = DatabaseManager()
        self.topic_analyzer = TopicAnalyzer()
        self.similarity_engine = SemanticSimilarity()
        self.learning_engine = LearningEngine()
    
    async def check_topic_availability(self, topic: str, user_id: str) -> TopicAvailability:
        """Check if a topic can be used based on history"""
        topic_hash = self.topic_analyzer.generate_topic_hash(topic)
        
        # Check recent usage
        recent_usage = await self.db.get_recent_topic_usage(user_id, topic_hash, days=30)
        if recent_usage:
            return TopicAvailability(
                available=False,
                reason="topic_recently_used",
                last_used=recent_usage.last_used,
                suggestion="Wait until " + str(recent_usage.last_used + timedelta(days=30))
            )
        
        # Check semantic similarity with recent topics
        similar_topics = await self.find_similar_recent_topics(topic, user_id, days=30)
        if similar_topics:
            return TopicAvailability(
                available=False,
                reason="similar_topic_recently_used",
                similar_topics=similar_topics,
                suggestion="Try a different angle or wait for cooldown"
            )
        
        return TopicAvailability(available=True)
    
    async def check_source_availability(self, source_url: str, user_id: str) -> SourceAvailability:
        """Check if an article source can be reused"""
        source_usage = await self.db.get_source_usage(user_id, source_url)
        
        if source_usage:
            days_since_use = (datetime.now() - source_usage.used_date).days
            if days_since_use < 90:  # 90-day cooldown for sources
                return SourceAvailability(
                    available=False,
                    reason="source_recently_used",
                    last_used=source_usage.used_date,
                    wait_until=source_usage.used_date + timedelta(days=90)
                )
        
        return SourceAvailability(available=True)
    
    async def record_content_generation(self, content_metadata: ContentMetadata, user_id: str):
        """Record successful content generation for learning"""
        # Record topic usage
        await self.record_topic_usage(content_metadata.topic, user_id)
        
        # Record source usage if applicable
        if content_metadata.source_url:
            await self.record_source_usage(content_metadata.source_url, user_id)
        
        # Update learning profile
        await self.update_learning_profile(content_metadata, user_id)
```

### Topic Analysis Engine
```python
class TopicAnalyzer:
    def __init__(self):
        self.embeddings_model = SentenceTransformer('all-MiniLM-L6-v2')
        self.medical_keywords = MedicalKeywordExtractor()
    
    def generate_topic_hash(self, topic: str) -> str:
        """Generate consistent hash for topic deduplication"""
        # Normalize topic text
        normalized = self.normalize_topic_text(topic)
        
        # Extract key medical concepts
        key_concepts = self.medical_keywords.extract_concepts(normalized)
        
        # Create hash from key concepts
        concept_string = "|".join(sorted(key_concepts))
        return hashlib.sha256(concept_string.encode()).hexdigest()[:16]
    
    def normalize_topic_text(self, topic: str) -> str:
        """Normalize topic text for consistent comparison"""
        # Remove common variations
        # diabetes type 2 -> diabetes type 2
        # hypertension/high blood pressure -> hypertension
        # COVID-19/coronavirus -> covid
        
        normalizations = {
            r"covid[-\s]?19": "covid",
            r"high blood pressure": "hypertension",
            r"diabetes type 2": "diabetes type 2",
            r"heart attack": "myocardial infarction"
        }
        
        normalized = topic.lower()
        for pattern, replacement in normalizations.items():
            normalized = re.sub(pattern, replacement, normalized)
        
        return normalized
    
    async def find_similar_topics(self, new_topic: str, recent_topics: List[str], 
                                threshold: float = 0.8) -> List[SimilarTopic]:
        """Find semantically similar topics using embeddings"""
        new_embedding = self.embeddings_model.encode([new_topic])
        recent_embeddings = self.embeddings_model.encode(recent_topics)
        
        similarities = cosine_similarity(new_embedding, recent_embeddings)[0]
        
        similar_topics = []
        for i, similarity in enumerate(similarities):
            if similarity > threshold:
                similar_topics.append(SimilarTopic(
                    topic=recent_topics[i],
                    similarity_score=similarity
                ))
        
        return sorted(similar_topics, key=lambda x: x.similarity_score, reverse=True)
```

### Learning Engine
```python
class LearningEngine:
    def __init__(self):
        self.performance_analyzer = PerformanceAnalyzer()
        self.pattern_detector = PatternDetector()
    
    async def update_user_learning_profile(self, user_id: str, new_content: ContentMetadata):
        """Update user learning profile based on new successful content"""
        profile = await self.db.get_user_learning_profile(user_id)
        
        # Learn from content patterns
        if new_content.user_rating >= 4:  # High-rated content
            await self.learn_successful_patterns(profile, new_content)
        
        # Learn from engagement data
        if new_content.engagement_data:
            await self.learn_engagement_patterns(profile, new_content)
        
        # Update confidence scores
        profile.learning_confidence = self.calculate_confidence_score(profile)
        
        await self.db.save_user_learning_profile(profile)
    
    async def learn_successful_patterns(self, profile: UserLearningProfile, 
                                      content: ContentMetadata):
        """Learn patterns from successful content"""
        # Content length preferences
        platform = content.platform
        length = content.content_length
        profile.preferred_content_length[platform] = self.update_preference(
            profile.preferred_content_length.get(platform, length),
            length,
            weight=0.2
        )
        
        # Hashtag count preferences
        profile.preferred_hashtag_count[platform] = self.update_preference(
            profile.preferred_hashtag_count.get(platform, content.hashtag_count),
            content.hashtag_count,
            weight=0.3
        )
        
        # Topic preferences
        if content.topic not in profile.successful_topics:
            profile.successful_topics.append(content.topic)
        
        # Keep only top 20 successful topics
        if len(profile.successful_topics) > 20:
            profile.successful_topics = profile.successful_topics[-20:]
    
    def calculate_confidence_score(self, profile: UserLearningProfile) -> float:
        """Calculate how confident we are in our learning about the user"""
        factors = {
            "post_count": min(profile.total_posts / 50, 1.0),  # Confidence increases with posts
            "consistency": self.calculate_consistency_score(profile),
            "engagement_stability": self.calculate_engagement_stability(profile),
            "time_range": min(profile.days_active / 90, 1.0)  # More confidence over time
        }
        
        weights = {"post_count": 0.4, "consistency": 0.3, "engagement_stability": 0.2, "time_range": 0.1}
        
        return sum(factors[k] * weights[k] for k in factors)
```

### Memory-Driven Content Suggestions
```python
class MemoryDrivenSuggestions:
    def __init__(self, memory_system: LongTermMemorySystem):
        self.memory = memory_system
    
    async def suggest_optimal_content(self, user_id: str, current_trends: List[str]) -> ContentSuggestions:
        """Suggest content based on memory and current trends"""
        profile = await self.memory.get_user_learning_profile(user_id)
        
        suggestions = []
        
        # Filter trends by user's successful topics
        for trend in current_trends:
            # Check availability
            availability = await self.memory.check_topic_availability(trend, user_id)
            if not availability.available:
                continue
            
            # Calculate suggestion score
            score = await self.calculate_suggestion_score(trend, profile)
            
            suggestions.append(ContentSuggestion(
                topic=trend,
                score=score,
                reason=self.explain_suggestion(trend, profile),
                optimal_platform=self.suggest_best_platform(trend, profile),
                estimated_engagement=self.predict_engagement(trend, profile)
            ))
        
        return sorted(suggestions, key=lambda x: x.score, reverse=True)[:5]
    
    async def suggest_content_improvements(self, content: str, user_id: str) -> List[Improvement]:
        """Suggest improvements based on learned preferences"""
        profile = await self.memory.get_user_learning_profile(user_id)
        
        improvements = []
        
        # Length optimization
        current_length = len(content)
        optimal_length = profile.preferred_content_length.get('facebook', 1200)
        if abs(current_length - optimal_length) > 200:
            improvements.append(Improvement(
                type="length",
                current=current_length,
                suggested=optimal_length,
                reason=f"Based on your successful posts, {optimal_length} characters work best"
            ))
        
        # Hashtag optimization
        current_hashtags = self.count_hashtags(content)
        optimal_hashtags = profile.preferred_hashtag_count.get('facebook', 5)
        if current_hashtags != optimal_hashtags:
            improvements.append(Improvement(
                type="hashtags",
                current=current_hashtags,
                suggested=optimal_hashtags,
                reason=f"Your most engaging posts use {optimal_hashtags} hashtags"
            ))
        
        return improvements
```

## Performance Requirements
- [ ] **Topic Lookup**: <100ms for topic availability checking
- [ ] **Similarity Analysis**: <500ms for semantic similarity comparison
- [ ] **Learning Updates**: <200ms for profile updates
- [ ] **Memory Queries**: <50ms for simple history lookups
- [ ] **Batch Processing**: Handle 1000+ memory operations per minute

## Dependencies
- **Before**: #006 (Content Generation Engine), #007 (Approval Loop System)
- **After**: #009 (Social Interactions Management)

## Testing Requirements
- [ ] **Unit Tests**: All memory operations and learning algorithms
- [ ] **Integration Tests**: End-to-end memory-driven content flows
- [ ] **Performance Tests**: Memory system under load
- [ ] **Learning Tests**: Validate learning accuracy over time
- [ ] **Data Tests**: Database operations and consistency

## Memory Lifecycle Management
```python
class MemoryLifecycleManager:
    async def cleanup_old_data(self, retention_days: int = 365):
        """Clean up old memory data while preserving learning"""
        # Archive old content performance data
        await self.archive_old_performance_data(retention_days)
        
        # Consolidate learning patterns
        await self.consolidate_learning_patterns()
        
        # Remove low-value memory entries
        await self.prune_low_value_memories()
    
    async def export_user_memory(self, user_id: str) -> MemoryExport:
        """Export user's complete memory profile"""
        return MemoryExport(
            topic_history=await self.db.get_topic_history(user_id),
            source_history=await self.db.get_source_history(user_id),
            learning_profile=await self.db.get_learning_profile(user_id),
            performance_data=await self.db.get_performance_data(user_id)
        )
```

## Configuration
```yaml
long_term_memory:
  topic_cooldown_days: 30
  source_cooldown_days: 90
  strategic_reuse_days: 90
  
  similarity_thresholds:
    topic_similarity: 0.8
    semantic_similarity: 0.75
    
  learning:
    confidence_threshold: 0.7
    max_successful_topics: 20
    pattern_weight: 0.3
    
  retention:
    performance_data: 365d
    topic_history: 720d
    learning_profile: "permanent"
    
  cleanup:
    run_schedule: "weekly"
    batch_size: 1000
```

## Labels
- `core-system`
- `intelligence`
- `learning`
- `memory-management`
- `content-optimization`

## Estimated Effort
**Story Points**: 13
**Timeline**: 3-4 weeks

## Definition of Done
- [ ] Topic deduplication preventing repetition within 30 days
- [ ] Source tracking preventing reuse within 90 days
- [ ] Learning engine improving content suggestions over time
- [ ] Performance analytics tracking engagement and success
- [ ] Memory-driven content optimization functional
- [ ] Database schema optimized for fast queries
- [ ] Comprehensive test coverage (>90%)
- [ ] Performance targets met for all memory operations
- [ ] Data lifecycle management operational