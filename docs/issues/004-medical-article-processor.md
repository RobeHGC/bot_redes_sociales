# Medical Article Processor

## Overview
Implement an intelligent article processing system that can extract, analyze, and convert medical articles from various sources into social media content.

## Description
The medical article processor is a core component that enables Nadia Bot to work with scientific literature. It supports multiple input formats (PubMed URLs, DOIs, raw text, Medscape links) and automatically extracts relevant information to create engaging, scientifically accurate social media posts.

## Acceptance Criteria

### Source Detection & Processing
- [ ] **PubMed Integration**: Extract metadata from PubMed URLs automatically
- [ ] **DOI Resolution**: Process DOI links and resolve to full articles
- [ ] **Text Analysis**: Analyze pasted article text for key information
- [ ] **Medscape Scraping**: Extract content from Medscape article links
- [ ] **Auto-detection**: Intelligently identify source type from user input

### Content Extraction
- [ ] **Metadata Extraction**: Title, authors, journal, publication date, DOI
- [ ] **Abstract Processing**: Key findings and clinical relevance
- [ ] **Methodology Summary**: Study design and participant information
- [ ] **Results Highlighting**: Main outcomes and statistical significance
- [ ] **Clinical Implications**: Practical applications for healthcare providers

### Content Transformation
- [ ] **Social Media Optimization**: Convert academic content to engaging posts
- [ ] **Reference Formatting**: Proper citation formatting for social media
- [ ] **Disclaimer Addition**: Automatic medical disclaimers when appropriate
- [ ] **Hashtag Generation**: Relevant medical hashtags based on content
- [ ] **Multi-length Versions**: Different versions for various platforms

## Technical Implementation

### Article Processor Architecture
```python
class ArticleProcessor:
    def __init__(self):
        self.pubmed_api = PubMedAPI()
        self.doi_resolver = DOIResolver()
        self.web_scraper = MedicalWebScraper()
        self.content_analyzer = ContentAnalyzer()
    
    async def process_source(self, source: str) -> ProcessedArticle:
        """Main processing pipeline"""
        source_type = self.detect_source_type(source)
        
        if source_type == "pubmed":
            return await self.process_pubmed(source)
        elif source_type == "doi":
            return await self.process_doi(source)
        elif source_type == "medscape":
            return await self.process_medscape(source)
        elif source_type == "text":
            return await self.process_text(source)
        else:
            raise UnsupportedSourceError(f"Cannot process: {source}")
    
    def detect_source_type(self, source: str) -> str:
        """Intelligent source type detection"""
        patterns = {
            r"pubmed\.ncbi\.nlm\.nih\.gov": "pubmed",
            r"10\.\d{4,}/": "doi",
            r"medscape\.com": "medscape",
            r"doi\.org": "doi"
        }
        # Implementation details...
```

### PubMed Integration
```python
class PubMedAPI:
    async def fetch_article_metadata(self, pmid: str) -> dict:
        """Fetch complete article metadata from PubMed"""
        url = f"https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi"
        params = {
            "db": "pubmed",
            "id": pmid,
            "retmode": "xml",
            "api_key": self.api_key
        }
        # Implementation details...
    
    def extract_key_information(self, xml_data: str) -> dict:
        """Extract relevant fields from PubMed XML"""
        return {
            "title": "...",
            "authors": ["..."],
            "journal": "...",
            "pub_date": "...",
            "abstract": "...",
            "keywords": ["..."],
            "mesh_terms": ["..."]
        }
```

### Content Analysis Engine
```python
class ContentAnalyzer:
    def __init__(self):
        self.gemini_client = GeminiClient()
    
    async def analyze_medical_content(self, article_data: dict) -> AnalyzedContent:
        """Use Gemini to analyze and summarize medical content"""
        prompt = f"""
        Analiza este artículo médico y extrae:
        1. Hallazgos principales
        2. Relevancia clínica  
        3. Aplicaciones prácticas
        4. Limitaciones del estudio
        5. Implicaciones para pacientes
        
        Artículo: {article_data}
        """
        return await self.gemini_client.analyze(prompt)
    
    def generate_social_content(self, analyzed_content: AnalyzedContent) -> SocialContent:
        """Transform analysis into social media ready content"""
```

### Data Models
```python
@dataclass
class ProcessedArticle:
    title: str
    authors: List[str]
    journal: str
    publication_date: datetime
    doi: Optional[str]
    pmid: Optional[str]
    abstract: str
    key_findings: List[str]
    clinical_relevance: str
    methodology: str
    limitations: List[str]
    source_url: str
    processed_at: datetime

@dataclass  
class SocialContent:
    short_version: str  # Twitter/X
    medium_version: str  # Instagram
    long_version: str  # Facebook
    hashtags: List[str]
    references: str
    disclaimer: str
    images_suggested: List[str]
```

## Source-Specific Processing

### PubMed Articles
```python
# Input: https://pubmed.ncbi.nlm.nih.gov/38234567
# Process:
1. Extract PMID from URL
2. Fetch metadata via E-utilities API
3. Parse XML response for key fields
4. Analyze abstract and MeSH terms
5. Generate social content with proper citations
```

### DOI Links
```python
# Input: 10.1056/NEJMoa2024234 or https://doi.org/10.1056/NEJMoa2024234
# Process:
1. Resolve DOI to publisher page
2. Extract metadata from CrossRef API
3. Scrape additional content if available
4. Analyze and transform content
```

### Medscape Articles
```python
# Input: https://www.medscape.com/viewarticle/123456
# Process:
1. Web scraping with proper rate limiting
2. Extract structured content
3. Preserve clinical context
4. Generate engaging social versions
```

### Text Input
```python
# Input: Large block of pasted article text
# Process:
1. Text preprocessing and cleaning
2. Section identification (abstract, methods, results)
3. Key information extraction
4. Content analysis and summarization
```

## User Experience Flow
```
Tú: "Hazme un post de esto: https://pubmed.ncbi.nlm.nih.gov/38234567"

Bot: 📄 Analizando artículo de PubMed...
     ⏳ Extrayendo metadatos...
     ✅ "Mediterranean Diet and Type 2 Diabetes Prevention" - NEJM 2024
     
     📊 Analizando contenido científico...
     📝 Generando post optimizado...
     
     🎯 POST GENERADO:
     
     🥗 NUEVA EVIDENCIA: Dieta mediterránea reduce 23% el riesgo de diabetes tipo 2
     
     📊 Estudio con 15,000 participantes seguidos por 8 años muestra:
     ✅ 23% menos riesgo de diabetes
     ✅ Mejor control glucémico  
     ✅ Reducción de inflamación
     
     🔑 Componentes clave:
     • Aceite de oliva extra virgen
     • Nueces y pescado azul
     • Verduras y legumbres
     • Moderación en carbohidratos
     
     👨‍⚕️ "Los beneficios se observan desde los primeros 6 meses"
     
     📚 Fuente: New England Journal of Medicine, 2024
     DOI: 10.1056/NEJMoa2024234
     
     ⚠️ Consulta siempre con tu médico antes de cambios dietéticos importantes
     
     #DiabetesPrevención #DietaMediterránea #NEJM #MedicinaBasadaEnEvidencia
     
     ¿Te gusta o necesitas ajustes?

Tú: "Agrega estadísticas específicas de México"

Bot: ✏️ Incluyendo contexto mexicano...
     
     [Versión editada con estadísticas de ENSANUT México]
     
     ¿Mejor así?
```

## Error Handling
- [ ] **Invalid URLs**: Clear error messages with suggestions
- [ ] **Access Restrictions**: Handle paywalled or restricted content
- [ ] **Network Issues**: Retry logic with exponential backoff
- [ ] **Parsing Errors**: Graceful degradation with partial processing
- [ ] **Rate Limiting**: Respect API limits with queuing system

## Performance Requirements
- [ ] **Processing Speed**: <2s for metadata extraction
- [ ] **Content Analysis**: <5s for full article analysis
- [ ] **Concurrent Processing**: Handle 5+ articles simultaneously
- [ ] **Cache Efficiency**: Cache processed articles for 30 days
- [ ] **Memory Usage**: Efficient processing of large articles

## Dependencies
- **Before**: #003 (Hybrid Command System)
- **After**: #005 (Daily Research Automation), #006 (Content Generation)

## Testing Requirements
- [ ] **Unit Tests**: All processing functions and parsers
- [ ] **Integration Tests**: End-to-end article processing flows
- [ ] **API Tests**: PubMed and DOI resolution reliability
- [ ] **Content Quality**: Accuracy of extracted information
- [ ] **Performance Tests**: Processing speed under various loads

## External Dependencies
```yaml
APIs:
  - PubMed E-utilities API (free, rate limited)
  - CrossRef API (free, for DOI resolution)
  - Semantic Scholar API (backup for academic papers)

Libraries:
  - beautifulsoup4 (web scraping)
  - requests (HTTP client)
  - lxml (XML parsing)
  - python-dateutil (date parsing)
  - aiohttp (async HTTP)
```

## Configuration
```yaml
article_processor:
  pubmed_api_key: "${PUBMED_API_KEY}"
  rate_limits:
    pubmed: 10/second
    crossref: 50/second
    medscape: 1/second
  cache_duration: 30d
  max_article_size: 10MB
  timeout: 30s
```

## Labels
- `core-system`
- `high-priority`
- `medical-content`
- `api-integration`
- `content-processing`

## Estimated Effort
**Story Points**: 13
**Timeline**: 3-4 weeks

## Definition of Done
- [ ] All source types (PubMed, DOI, Medscape, text) processing correctly
- [ ] Content extraction achieving 90% accuracy
- [ ] Performance targets met for processing speed
- [ ] Error handling covering all failure scenarios
- [ ] Cache system operational and efficient
- [ ] Comprehensive test coverage (>90%)
- [ ] API rate limiting respected
- [ ] User experience flows validated with real articles