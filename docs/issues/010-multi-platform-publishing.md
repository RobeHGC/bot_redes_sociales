# Multi-Platform Publishing System

## Overview
Implement a robust multi-platform publishing system that simultaneously posts content to Facebook, Instagram, and Twitter/X with platform-specific optimization, error handling, and scheduling capabilities.

## Description
The multi-platform publishing system handles the final step of content delivery, adapting approved content for each platform's requirements, managing API interactions, handling publishing failures gracefully, and providing detailed publishing status feedback to users.

## Acceptance Criteria

### Platform Integration
- [ ] **Facebook Publishing**: Post to Facebook pages with images, links, and hashtags
- [ ] **Instagram Publishing**: Support feed posts and Instagram Stories
- [ ] **Twitter/X Publishing**: Handle character limits, threads, and media attachments
- [ ] **Cross-Platform Coordination**: Publish to all platforms simultaneously
- [ ] **Platform-Specific Optimization**: Adapt content format for each platform

### Content Adaptation
- [ ] **Automatic Formatting**: Adjust content length and structure per platform
- [ ] **Hashtag Optimization**: Platform-appropriate hashtag placement and counts
- [ ] **Image Handling**: Resize and optimize images for each platform
- [ ] **Link Management**: Handle link sharing with proper previews
- [ ] **Character Limit Compliance**: Ensure content fits platform constraints

### Publishing Management
- [ ] **Immediate Publishing**: Post content immediately after approval
- [ ] **Scheduled Publishing**: Support delayed posting for optimal timing
- [ ] **Batch Publishing**: Handle multiple posts across platforms efficiently
- [ ] **Publishing Queue**: Manage publishing queue with retry mechanisms
- [ ] **Status Tracking**: Real-time status updates for each publishing attempt

### Error Handling & Recovery
- [ ] **Platform Failures**: Continue publishing to available platforms when others fail
- [ ] **Rate Limiting**: Handle API rate limits with intelligent backoff
- [ ] **Authentication Issues**: Graceful handling of expired tokens
- [ ] **Content Rejection**: Handle platform-specific content policy violations
- [ ] **Retry Mechanisms**: Automatic retry for transient failures

## Technical Implementation

### Publishing Orchestrator
```python
class MultiPlatformPublisher:
    def __init__(self):
        self.publishers = {
            'facebook': FacebookPublisher(),
            'instagram': InstagramPublisher(),
            'twitter': TwitterPublisher()
        }
        self.content_adapter = ContentAdapter()
        self.publishing_queue = PublishingQueue()
        self.status_tracker = PublishingStatusTracker()
        
    async def publish_content(self, content: ApprovedContent, 
                            user_id: str, platforms: List[str] = None) -> PublishingResult:
        """Main publishing orchestration"""
        
        if platforms is None:
            platforms = ['facebook', 'instagram', 'twitter']
        
        # 1. Adapt content for each platform
        adapted_content = await self.content_adapter.adapt_for_platforms(content, platforms)
        
        # 2. Create publishing job
        job = PublishingJob(
            job_id=str(uuid.uuid4()),
            user_id=user_id,
            content=adapted_content,
            target_platforms=platforms,
            created_at=datetime.now(),
            status=PublishingStatus.PENDING
        )
        
        # 3. Add to queue and execute
        await self.publishing_queue.add_job(job)
        result = await self.execute_publishing_job(job)
        
        # 4. Update status and notify user
        await self.status_tracker.update_job_status(job.job_id, result)
        await self.notify_user_of_result(user_id, result)
        
        return result
    
    async def execute_publishing_job(self, job: PublishingJob) -> PublishingResult:
        """Execute publishing across all target platforms"""
        
        results = {}
        
        # Publish to all platforms concurrently
        tasks = []
        for platform in job.target_platforms:
            if platform in self.publishers:
                task = self.publish_to_platform(
                    platform, 
                    job.content.platform_versions[platform],
                    job.user_id
                )
                tasks.append((platform, task))
        
        # Execute all publishing tasks
        platform_results = await asyncio.gather(
            *[task for _, task in tasks], 
            return_exceptions=True
        )
        
        # Process results
        for i, (platform, _) in enumerate(tasks):
            result = platform_results[i]
            if isinstance(result, Exception):
                results[platform] = PlatformResult(
                    success=False,
                    error=str(result),
                    platform=platform
                )
            else:
                results[platform] = result
        
        return PublishingResult(
            job_id=job.job_id,
            platform_results=results,
            overall_success=any(r.success for r in results.values()),
            completed_at=datetime.now()
        )
```

### Platform-Specific Publishers
```python
class FacebookPublisher:
    def __init__(self):
        self.graph_api = FacebookGraphAPI()
        
    async def publish(self, content: PlatformContent, user_id: str) -> PlatformResult:
        """Publish content to Facebook page"""
        
        try:
            # Get user's Facebook page access token
            page_token = await self.get_page_access_token(user_id)
            
            # Prepare publishing payload
            payload = {
                'message': content.text,
                'access_token': page_token
            }
            
            # Add image if present
            if content.image_url:
                payload['link'] = content.image_url
            
            # Publish post
            response = await self.graph_api.post('/{page_id}/feed', payload)
            
            return PlatformResult(
                success=True,
                platform='facebook',
                post_id=response.get('id'),
                post_url=f"https://facebook.com/{response.get('id')}",
                published_at=datetime.now()
            )
            
        except FacebookAPIException as e:
            return PlatformResult(
                success=False,
                platform='facebook', 
                error=str(e),
                error_code=e.code,
                retry_possible=self.is_retryable_error(e)
            )

class InstagramPublisher:
    def __init__(self):
        self.basic_display_api = InstagramBasicDisplayAPI()
        
    async def publish(self, content: PlatformContent, user_id: str) -> PlatformResult:
        """Publish content to Instagram"""
        
        # Instagram requires different flows for feed vs stories
        if content.content_type == 'story':
            return await self.publish_story(content, user_id)
        else:
            return await self.publish_feed_post(content, user_id)
    
    async def publish_feed_post(self, content: PlatformContent, user_id: str) -> PlatformResult:
        """Publish to Instagram feed"""
        try:
            # Instagram publishing requires image for feed posts
            if not content.image_url:
                # Generate image from content or use default
                content.image_url = await self.generate_content_image(content.text)
            
            # Create media container
            container_response = await self.basic_display_api.create_media_container(
                image_url=content.image_url,
                caption=content.text,
                access_token=await self.get_access_token(user_id)
            )
            
            # Publish container
            publish_response = await self.basic_display_api.publish_media(
                creation_id=container_response['id'],
                access_token=await self.get_access_token(user_id)
            )
            
            return PlatformResult(
                success=True,
                platform='instagram',
                post_id=publish_response.get('id'),
                post_url=f"https://instagram.com/p/{publish_response.get('id')}",
                published_at=datetime.now()
            )
            
        except InstagramAPIException as e:
            return PlatformResult(
                success=False,
                platform='instagram',
                error=str(e),
                retry_possible=self.is_retryable_error(e)
            )

class TwitterPublisher:
    def __init__(self):
        self.twitter_api = TwitterAPIv2()
        
    async def publish(self, content: PlatformContent, user_id: str) -> PlatformResult:
        """Publish content to Twitter/X"""
        
        try:
            # Handle long content with threads
            if len(content.text) > 280:
                return await self.publish_thread(content, user_id)
            else:
                return await self.publish_single_tweet(content, user_id)
                
        except TwitterAPIException as e:
            return PlatformResult(
                success=False,
                platform='twitter',
                error=str(e),
                retry_possible=self.is_retryable_error(e)
            )
    
    async def publish_single_tweet(self, content: PlatformContent, user_id: str) -> PlatformResult:
        """Publish single tweet"""
        
        payload = {'text': content.text}
        
        # Add media if present
        if content.image_url:
            media_id = await self.upload_media(content.image_url, user_id)
            payload['media'] = {'media_ids': [media_id]}
        
        response = await self.twitter_api.create_tweet(
            payload,
            access_token=await self.get_access_token(user_id)
        )
        
        return PlatformResult(
            success=True,
            platform='twitter',
            post_id=response['data']['id'],
            post_url=f"https://twitter.com/user/status/{response['data']['id']}",
            published_at=datetime.now()
        )
    
    async def publish_thread(self, content: PlatformContent, user_id: str) -> PlatformResult:
        """Publish Twitter thread for long content"""
        
        # Split content into thread
        thread_parts = self.split_content_for_thread(content.text)
        
        # Publish thread
        tweet_ids = []
        reply_to_id = None
        
        for i, part in enumerate(thread_parts):
            payload = {
                'text': f"{part} ({i+1}/{len(thread_parts)})",
                'reply': {'in_reply_to_tweet_id': reply_to_id} if reply_to_id else None
            }
            
            response = await self.twitter_api.create_tweet(
                payload,
                access_token=await self.get_access_token(user_id)
            )
            
            tweet_id = response['data']['id']
            tweet_ids.append(tweet_id)
            reply_to_id = tweet_id
        
        return PlatformResult(
            success=True,
            platform='twitter',
            post_id=tweet_ids[0],  # First tweet ID
            post_url=f"https://twitter.com/user/status/{tweet_ids[0]}",
            thread_ids=tweet_ids,
            published_at=datetime.now()
        )
```

### Content Adaptation Engine
```python
class ContentAdapter:
    def __init__(self):
        self.platform_configs = {
            'facebook': {
                'max_length': 63206,
                'hashtag_limit': 30,
                'supports_images': True,
                'supports_links': True
            },
            'instagram': {
                'max_length': 2200,
                'hashtag_limit': 30,
                'supports_images': True,
                'supports_links': False  # Links in bio only
            },
            'twitter': {
                'max_length': 280,
                'hashtag_limit': 10,
                'supports_images': True,
                'supports_links': True,
                'thread_capable': True
            }
        }
    
    async def adapt_for_platforms(self, content: ApprovedContent, 
                                platforms: List[str]) -> AdaptedContent:
        """Adapt content for all target platforms"""
        
        platform_versions = {}
        
        for platform in platforms:
            if platform in self.platform_configs:
                adapted = await self.adapt_for_platform(content, platform)
                platform_versions[platform] = adapted
        
        return AdaptedContent(
            original=content,
            platform_versions=platform_versions
        )
    
    async def adapt_for_platform(self, content: ApprovedContent, platform: str) -> PlatformContent:
        """Adapt content for specific platform"""
        
        config = self.platform_configs[platform]
        adapted_text = content.text
        
        # Handle length constraints
        if len(adapted_text) > config['max_length']:
            if platform == 'twitter' and config.get('thread_capable'):
                # Keep full content for thread handling
                pass
            else:
                # Truncate content
                adapted_text = await self.intelligently_truncate(
                    adapted_text, 
                    config['max_length']
                )
        
        # Optimize hashtags
        hashtags = self.optimize_hashtags(content.hashtags, config['hashtag_limit'])
        
        # Handle platform-specific formatting
        if platform == 'instagram':
            adapted_text = self.format_for_instagram(adapted_text, hashtags)
        elif platform == 'twitter':
            adapted_text = self.format_for_twitter(adapted_text, hashtags)
        elif platform == 'facebook':
            adapted_text = self.format_for_facebook(adapted_text, hashtags)
        
        return PlatformContent(
            text=adapted_text,
            hashtags=hashtags,
            image_url=content.image_url,
            links=content.links if config.get('supports_links') else [],
            platform=platform,
            content_type='feed'  # or 'story' for Instagram
        )
```

### Publishing Queue & Retry Logic
```python
class PublishingQueue:
    def __init__(self):
        self.redis_client = RedisClient()
        self.celery_app = CeleryApp()
        
    async def add_job(self, job: PublishingJob):
        """Add publishing job to queue"""
        await self.redis_client.lpush('publishing_queue', job.to_json())
        
        # Schedule with Celery for reliability
        self.celery_app.send_task(
            'publish_content',
            args=[job.to_dict()],
            countdown=0  # Immediate execution
        )
    
    async def retry_failed_job(self, job: PublishingJob, delay_seconds: int = 300):
        """Retry failed publishing job with delay"""
        job.retry_count += 1
        job.next_retry = datetime.now() + timedelta(seconds=delay_seconds)
        
        if job.retry_count < MAX_RETRIES:
            self.celery_app.send_task(
                'publish_content',
                args=[job.to_dict()],
                countdown=delay_seconds
            )
        else:
            await self.mark_job_as_failed(job)

class PublishingStatusTracker:
    def __init__(self):
        self.db = DatabaseManager()
        
    async def track_publishing_attempt(self, job_id: str, platform: str, 
                                     result: PlatformResult):
        """Track individual platform publishing attempts"""
        await self.db.insert_publishing_attempt({
            'job_id': job_id,
            'platform': platform,
            'success': result.success,
            'post_id': result.post_id,
            'post_url': result.post_url,
            'error': result.error,
            'attempted_at': datetime.now()
        })
    
    async def get_publishing_status(self, job_id: str) -> PublishingStatus:
        """Get current status of publishing job"""
        attempts = await self.db.get_publishing_attempts(job_id)
        
        return PublishingStatus(
            job_id=job_id,
            total_platforms=len(set(a['platform'] for a in attempts)),
            successful_platforms=len([a for a in attempts if a['success']]),
            failed_platforms=len([a for a in attempts if not a['success']]),
            is_complete=self.is_job_complete(attempts),
            last_attempt=max(a['attempted_at'] for a in attempts) if attempts else None
        )
```

## User Experience Integration

### Publishing Feedback
```
Tú: /aprobar

Bot: ✅ ¡Perfecto! Publicando contenido...
     
     📤 Publicando en:
     • Facebook ⏳
     • Instagram ⏳  
     • Twitter ⏳
     
     [Tiempo real de actualización]
     
     ✅ Facebook - Publicado exitosamente
     ✅ Instagram - Publicado exitosamente
     ⚠️ Twitter - Error de API, reintentando...
     
     ✅ Twitter - Publicado exitosamente (reintento)
     
     🎉 ¡Contenido publicado en todas las plataformas!
     
     📊 Enlaces directos:
     • Facebook: https://facebook.com/post/123
     • Instagram: https://instagram.com/p/abc
     • Twitter: https://twitter.com/status/456
     
     📅 Próxima generación: Mañana 10:00 AM
```

### Publishing Error Handling
```
Bot: ⚠️ Publicación parcialmente exitosa:
     
     ✅ Facebook - Publicado
     ✅ Instagram - Publicado  
     ❌ Twitter - Error de autenticación
     
     🔧 ¿Qué quieres hacer con Twitter?
     
     1. /reintentar - Intentar publicar nuevamente
     2. /reautorizar - Renovar permisos de Twitter
     3. /omitir - Continuar sin publicar en Twitter
     
     El contenido ya está guardado y se puede republicar después.

Tú: /reautorizar

Bot: 🔗 Por favor autoriza nuevamente Twitter:
     [Enlace de autorización]
     
     Una vez autorizado, el contenido se publicará automáticamente.
```

## Performance Requirements
- [ ] **Publishing Speed**: Complete multi-platform publishing in <10s
- [ ] **Concurrent Jobs**: Handle 10+ publishing jobs simultaneously
- [ ] **Retry Logic**: Intelligent retry with exponential backoff
- [ ] **Queue Processing**: Process queue items within 5s of addition
- [ ] **Status Updates**: Real-time status updates to users

## Dependencies
- **Before**: #007 (Approval Loop System)
- **After**: None (final component)

## Testing Requirements
- [ ] **Unit Tests**: All publishing and adaptation logic
- [ ] **Integration Tests**: End-to-end publishing flows for all platforms
- [ ] **API Tests**: All social media platform integrations
- [ ] **Error Handling Tests**: All failure scenarios and recovery
- [ ] **Performance Tests**: Publishing under load and concurrent operations

## Platform Configuration
```yaml
publishing:
  platforms:
    facebook:
      api_version: "v18.0"
      permissions: ["pages_manage_posts", "pages_read_engagement"]
      rate_limits:
        posts_per_hour: 25
        
    instagram:
      api_type: "basic_display"
      permissions: ["user_profile", "user_media"]
      rate_limits:
        posts_per_hour: 25
        
    twitter:
      api_version: "v2"
      permissions: ["tweet.read", "tweet.write", "users.read"]
      rate_limits:
        tweets_per_15min: 300
        
  retry_policy:
    max_retries: 3
    initial_delay: 300s  # 5 minutes
    backoff_factor: 2
    max_delay: 3600s     # 1 hour
    
  queue:
    max_concurrent_jobs: 10
    job_timeout: 300s
    cleanup_interval: 1h
```

## Error Recovery Strategies
- [ ] **Token Refresh**: Automatic OAuth token refresh when possible
- [ ] **Graceful Degradation**: Continue with available platforms when others fail
- [ ] **User Notification**: Clear communication about publishing status
- [ ] **Manual Recovery**: Tools for users to retry failed publications
- [ ] **Content Preservation**: Save content for later retry attempts

## Labels
- `publishing`
- `social-media-apis`
- `multi-platform`
- `error-handling`
- `user-experience`

## Estimated Effort
**Story Points**: 13
**Timeline**: 3-4 weeks

## Definition of Done
- [ ] All three platforms (Facebook, Instagram, Twitter/X) publishing successfully
- [ ] Content adaptation working correctly for each platform's requirements
- [ ] Error handling and retry mechanisms functional
- [ ] Publishing queue processing jobs reliably
- [ ] Real-time status updates provided to users
- [ ] Performance targets met for publishing speed
- [ ] Comprehensive test coverage (>90%)
- [ ] Platform-specific edge cases handled
- [ ] User experience flows validated and polished