# Project 02: Context Synthesizer Agent

## Overview

The Context Synthesizer is an intelligent prompt generation agent that transforms minimal user input into comprehensive, context-rich prompts. Unlike traditional prompt templates that require extensive manual configuration, this agent automatically infers and generates relevant background context, domain-specific knowledge, applicable frameworks, and situational parameters.

Given just a topic, task description, or brief query, the Context Synthesizer:
- Identifies the domain and subdomain of the request
- Generates relevant background knowledge and terminology
- Suggests applicable methodologies and frameworks
- Provides historical context and current trends
- Anticipates edge cases and considerations
- Structures output for optimal LLM comprehension

This agent is particularly valuable for users who know what they want to accomplish but lack the expertise to provide the detailed context that produces high-quality AI responses.

---

## Learning Objectives

By implementing and using this agent, you will learn to:

1. **Context Inference Techniques**
   - Extract implicit requirements from minimal input
   - Map topics to knowledge domains automatically
   - Identify relevant adjacent concepts and dependencies

2. **Knowledge Graph Navigation**
   - Traverse conceptual relationships programmatically
   - Generate hierarchical context structures
   - Balance depth vs. breadth in context generation

3. **Dynamic Prompt Architecture**
   - Build prompts that adapt to input complexity
   - Layer context appropriately for different use cases
   - Optimize token usage while maximizing relevance

4. **Domain Modeling**
   - Create reusable domain knowledge templates
   - Implement domain detection algorithms
   - Handle cross-domain and interdisciplinary topics

5. **Quality Assurance in Generation**
   - Validate generated context for accuracy
   - Implement relevance scoring mechanisms
   - Create feedback loops for continuous improvement

---

## Difficulty Level

**Intermediate to Advanced**

### Prerequisites
- Understanding of prompt engineering fundamentals
- Familiarity with knowledge representation concepts
- Basic experience with NLP or text processing
- Comfort with structured data manipulation

### Time Investment
- Initial Implementation: 6-8 hours
- Domain Template Creation: 2-3 hours per domain
- Testing and Refinement: 4-6 hours

---

## Key Features

### 1. Intelligent Domain Detection

The agent automatically identifies the primary domain and relevant subdomains from user input.

```python
class DomainDetector:
    def __init__(self):
        self.domain_signatures = {
            "software_engineering": {
                "keywords": ["code", "api", "database", "function", "deploy"],
                "patterns": [r"\b(bug|feature|refactor)\b", r"\b(test|debug)\b"],
                "subdomains": ["web", "mobile", "backend", "devops", "security"]
            },
            "data_science": {
                "keywords": ["data", "model", "predict", "analyze", "dataset"],
                "patterns": [r"\b(train|validate|accuracy)\b"],
                "subdomains": ["ml", "statistics", "visualization", "etl"]
            },
            "business": {
                "keywords": ["revenue", "customer", "market", "strategy", "growth"],
                "patterns": [r"\b(roi|kpi|okr)\b"],
                "subdomains": ["marketing", "sales", "operations", "finance"]
            }
        }

    def detect(self, input_text: str) -> dict:
        scores = {}
        for domain, config in self.domain_signatures.items():
            score = self._calculate_domain_score(input_text, config)
            scores[domain] = score

        primary_domain = max(scores, key=scores.get)
        return {
            "primary": primary_domain,
            "confidence": scores[primary_domain],
            "secondary": self._get_secondary_domains(scores)
        }
```

### 2. Contextual Knowledge Generation

Automatically generates relevant background knowledge based on detected domains.

```python
class KnowledgeGenerator:
    def generate_context(self, domain: str, topic: str) -> dict:
        return {
            "background": self._generate_background(domain, topic),
            "terminology": self._extract_key_terms(domain, topic),
            "frameworks": self._suggest_frameworks(domain, topic),
            "considerations": self._identify_considerations(domain, topic),
            "related_concepts": self._find_related_concepts(domain, topic)
        }

    def _generate_background(self, domain: str, topic: str) -> str:
        """Generate domain-specific background context."""
        templates = {
            "software_engineering": """
                This task involves software development practices including
                {methodology} approaches, {architecture} patterns, and
                consideration of {quality_attributes} requirements.
            """,
            "data_science": """
                This analysis requires understanding of {data_types} data,
                {statistical_methods} for validation, and {ml_paradigm}
                learning approaches where applicable.
            """
        }
        return self._fill_template(templates.get(domain, ""), topic)
```

### 3. Framework Recommendation Engine

Suggests applicable methodologies and frameworks based on the task type.

```python
class FrameworkRecommender:
    def __init__(self):
        self.framework_database = {
            "problem_solving": ["First Principles", "5 Whys", "MECE"],
            "decision_making": ["Decision Matrix", "Cost-Benefit", "SWOT"],
            "analysis": ["PESTLE", "Porter's Five Forces", "Value Chain"],
            "development": ["Agile", "TDD", "DDD", "Clean Architecture"],
            "research": ["Scientific Method", "Grounded Theory", "Meta-analysis"]
        }

    def recommend(self, task_type: str, complexity: str) -> list:
        """Recommend frameworks based on task characteristics."""
        applicable = []
        for category, frameworks in self.framework_database.items():
            if self._is_relevant(category, task_type):
                applicable.extend(frameworks)

        return self._rank_by_complexity(applicable, complexity)
```

### 4. Adaptive Depth Control

Adjusts the level of detail based on input complexity and detected expertise level.

```python
class DepthController:
    def calculate_optimal_depth(self, input_analysis: dict) -> dict:
        return {
            "context_depth": self._assess_context_needs(input_analysis),
            "technical_level": self._detect_expertise(input_analysis),
            "explanation_style": self._determine_style(input_analysis),
            "example_complexity": self._set_example_level(input_analysis)
        }

    def _assess_context_needs(self, analysis: dict) -> str:
        """Determine how much background context to provide."""
        if analysis["ambiguity_score"] > 0.7:
            return "comprehensive"
        elif analysis["specificity_score"] > 0.8:
            return "minimal"
        else:
            return "moderate"
```

### 5. Edge Case Anticipation

Proactively identifies potential complications and special cases.

```python
class EdgeCaseAnalyzer:
    def anticipate_edge_cases(self, domain: str, task: str) -> list:
        """Identify potential edge cases and complications."""
        edge_cases = []

        # Domain-specific edge cases
        domain_edges = self._get_domain_edges(domain)
        edge_cases.extend(domain_edges)

        # Task-specific edge cases
        task_edges = self._analyze_task_edges(task)
        edge_cases.extend(task_edges)

        # Cross-cutting concerns
        cross_cutting = self._identify_cross_cutting(domain, task)
        edge_cases.extend(cross_cutting)

        return self._prioritize_by_likelihood(edge_cases)
```

### 6. Output Structure Optimization

Formats the generated context for maximum LLM comprehension and token efficiency.

```python
class OutputOptimizer:
    def structure_output(self, generated_context: dict) -> str:
        """Structure context for optimal LLM processing."""
        sections = [
            ("DOMAIN CONTEXT", generated_context["background"]),
            ("KEY TERMINOLOGY", self._format_terms(generated_context["terminology"])),
            ("APPLICABLE FRAMEWORKS", self._format_frameworks(generated_context["frameworks"])),
            ("IMPORTANT CONSIDERATIONS", generated_context["considerations"]),
            ("RELATED CONCEPTS", generated_context["related_concepts"]),
            ("EDGE CASES TO ADDRESS", generated_context["edge_cases"])
        ]

        return self._compile_sections(sections)
```

---

## How It Works

### Step-by-Step Workflow

#### Step 1: Input Reception and Preprocessing

```python
def preprocess_input(raw_input: str) -> dict:
    """Clean and analyze the raw user input."""
    return {
        "cleaned_text": normalize_text(raw_input),
        "tokens": tokenize(raw_input),
        "entities": extract_entities(raw_input),
        "intent_signals": detect_intent_signals(raw_input),
        "complexity_indicators": assess_complexity(raw_input)
    }
```

The agent receives minimal input and performs initial analysis:
- Text normalization and cleaning
- Entity extraction (names, tools, concepts)
- Intent signal detection (verbs, action words)
- Complexity assessment (technical terms, specificity)

#### Step 2: Domain Classification

```python
def classify_domain(preprocessed: dict) -> dict:
    """Determine the primary and secondary domains."""
    detector = DomainDetector()
    classification = detector.detect(preprocessed["cleaned_text"])

    # Enhance with entity-based refinement
    for entity in preprocessed["entities"]:
        classification = refine_with_entity(classification, entity)

    return classification
```

The agent maps the input to knowledge domains:
- Primary domain identification
- Secondary/adjacent domain detection
- Confidence scoring
- Subdomain specification

#### Step 3: Context Generation

```python
def generate_comprehensive_context(classification: dict, preprocessed: dict) -> dict:
    """Generate all context components."""
    generator = KnowledgeGenerator()

    context = {
        "background": generator.generate_background(
            classification["primary"],
            preprocessed["cleaned_text"]
        ),
        "terminology": generator.extract_terminology(
            classification["primary"],
            classification.get("subdomains", [])
        ),
        "historical_context": generator.generate_historical(
            classification["primary"]
        ),
        "current_trends": generator.identify_trends(
            classification["primary"]
        )
    }

    return context
```

Rich context is automatically generated:
- Background knowledge synthesis
- Technical terminology compilation
- Historical context gathering
- Current trends identification

#### Step 4: Framework Selection

```python
def select_frameworks(classification: dict, preprocessed: dict) -> list:
    """Choose applicable frameworks and methodologies."""
    recommender = FrameworkRecommender()

    task_type = infer_task_type(preprocessed["intent_signals"])
    complexity = assess_task_complexity(preprocessed)

    frameworks = recommender.recommend(task_type, complexity)

    return [
        {
            "name": fw,
            "relevance": calculate_relevance(fw, preprocessed),
            "application": describe_application(fw, classification)
        }
        for fw in frameworks
    ]
```

Relevant methodologies are suggested:
- Task type inference
- Complexity-appropriate selection
- Application guidance generation
- Relevance scoring

#### Step 5: Consideration Identification

```python
def identify_considerations(classification: dict, context: dict) -> dict:
    """Identify important factors and considerations."""
    return {
        "constraints": identify_constraints(classification),
        "assumptions": surface_assumptions(context),
        "risks": assess_risks(classification),
        "dependencies": map_dependencies(context),
        "success_factors": define_success_factors(classification)
    }
```

Critical factors are surfaced:
- Implicit constraints
- Hidden assumptions
- Potential risks
- Dependencies
- Success criteria

#### Step 6: Edge Case Analysis

```python
def analyze_edge_cases(classification: dict, considerations: dict) -> list:
    """Anticipate edge cases and special scenarios."""
    analyzer = EdgeCaseAnalyzer()

    edge_cases = analyzer.anticipate_edge_cases(
        classification["primary"],
        considerations
    )

    return [
        {
            "scenario": ec["description"],
            "likelihood": ec["probability"],
            "impact": ec["severity"],
            "mitigation": ec["suggested_handling"]
        }
        for ec in edge_cases
    ]
```

Potential complications are anticipated:
- Scenario identification
- Likelihood assessment
- Impact evaluation
- Mitigation suggestions

#### Step 7: Output Compilation

```python
def compile_final_output(all_components: dict) -> str:
    """Assemble all components into final prompt."""
    optimizer = OutputOptimizer()

    structured_output = optimizer.structure_output(all_components)

    # Add user's original request
    final_prompt = f"""
{structured_output}

---
ORIGINAL REQUEST:
{all_components['original_input']}

Please address this request using the context provided above.
"""

    return final_prompt
```

Everything is assembled optimally:
- Section organization
- Token optimization
- Readability enhancement
- Request integration

---

## Example Usage

### Example 1: Software Development Task

**Minimal Input:**
```
"optimize database queries"
```

**Generated Context:**

```markdown
# SYNTHESIZED CONTEXT

## DOMAIN CONTEXT
This task involves database performance optimization within software engineering.
Database query optimization focuses on reducing execution time, minimizing resource
consumption, and improving overall application responsiveness. This typically
involves analysis of query execution plans, index utilization, and data access patterns.

## KEY TERMINOLOGY
- **Query Execution Plan**: The sequence of operations the database performs
- **Index Scan vs. Table Scan**: Methods of data retrieval with different performance characteristics
- **Query Cost**: Estimated resources required to execute a query
- **Cardinality**: The number of unique values in a column
- **Selectivity**: The proportion of rows that match a predicate
- **N+1 Problem**: Anti-pattern where N additional queries are made for N results
- **Query Cache**: Storage of query results for reuse

## APPLICABLE FRAMEWORKS
1. **EXPLAIN Analysis**: Systematic examination of query execution plans
   - Application: Run EXPLAIN on problematic queries to identify bottlenecks

2. **Index Design Methodology**: Strategic index creation approach
   - Application: Analyze query patterns to design covering indexes

3. **Query Refactoring Patterns**: Common optimization transformations
   - Application: Apply patterns like query consolidation, predicate pushdown

## HISTORICAL CONTEXT
Query optimization has evolved from simple rule-based approaches to sophisticated
cost-based optimizers. Modern databases use statistics-driven optimization with
adaptive query processing capabilities.

## CURRENT TRENDS
- Automatic index recommendation systems
- Machine learning-based query optimization
- Adaptive query execution
- Columnar storage for analytical workloads

## IMPORTANT CONSIDERATIONS
- **Constraints**: Production database access limitations, downtime windows
- **Assumptions**: Queries are the bottleneck (not network/application logic)
- **Risks**: Index overhead on write operations, query plan regression
- **Dependencies**: Database version, existing indexes, data volume
- **Success Factors**: Measurable performance improvement, no regression

## EDGE CASES TO ADDRESS
1. **Parameter Sniffing Issues**
   - Likelihood: Medium
   - Impact: High
   - Mitigation: Use OPTIMIZE FOR UNKNOWN or query hints

2. **Statistics Staleness**
   - Likelihood: High
   - Impact: Medium
   - Mitigation: Schedule regular statistics updates

3. **Lock Contention**
   - Likelihood: Medium
   - Impact: High
   - Mitigation: Consider NOLOCK hints or isolation level adjustments

---
ORIGINAL REQUEST:
optimize database queries

Please address this request using the context provided above.
```

### Example 2: Business Strategy Task

**Minimal Input:**
```
"enter new market segment"
```

**Generated Context:**

```markdown
# SYNTHESIZED CONTEXT

## DOMAIN CONTEXT
This task involves market expansion strategy within business development. Entering
a new market segment requires analysis of market attractiveness, competitive
positioning, resource requirements, and go-to-market planning. Success depends on
understanding customer needs, competitive dynamics, and organizational capabilities.

## KEY TERMINOLOGY
- **Market Segment**: A distinct group of customers with similar needs
- **Total Addressable Market (TAM)**: Total market demand for a product
- **Serviceable Addressable Market (SAM)**: Portion of TAM targetable
- **Beachhead Strategy**: Focusing on a small market segment initially
- **Product-Market Fit**: Alignment between offering and market needs
- **Barriers to Entry**: Obstacles that make market entry difficult
- **Switching Costs**: Customer costs of changing providers

## APPLICABLE FRAMEWORKS
1. **Porter's Five Forces**: Industry attractiveness analysis
   - Application: Assess competitive intensity, supplier/buyer power, threats

2. **Ansoff Matrix**: Growth strategy classification
   - Application: Determine if this is market development or diversification

3. **STP Framework**: Segmentation, Targeting, Positioning
   - Application: Define segment boundaries, select targets, craft positioning

4. **Blue Ocean Strategy**: Uncontested market space creation
   - Application: Identify opportunities to create new demand

## HISTORICAL CONTEXT
Market entry strategies have shifted from large-scale launches to lean,
iterative approaches. The rise of digital channels has reduced entry barriers
but increased competitive intensity.

## CURRENT TRENDS
- Digital-first market entry strategies
- Platform and ecosystem approaches
- Data-driven customer segmentation
- Agile go-to-market methodologies
- Partnership-based market access

## IMPORTANT CONSIDERATIONS
- **Constraints**: Budget limitations, timeline pressures, regulatory requirements
- **Assumptions**: Market segment is accessible, capabilities are transferable
- **Risks**: Market timing, competitive response, cannibalization of existing business
- **Dependencies**: Sales capacity, product readiness, partner relationships
- **Success Factors**: Clear value proposition, adequate resources, executive support

## EDGE CASES TO ADDRESS
1. **Regulatory Barriers**
   - Likelihood: Variable by market
   - Impact: Critical
   - Mitigation: Early legal/compliance assessment

2. **Incumbent Retaliation**
   - Likelihood: High
   - Impact: High
   - Mitigation: Stealth entry or partnership approach

3. **Cultural Misalignment**
   - Likelihood: Medium
   - Impact: High
   - Mitigation: Local market research, cultural adaptation

---
ORIGINAL REQUEST:
enter new market segment

Please address this request using the context provided above.
```

### Example 3: Research Task

**Minimal Input:**
```
"analyze customer feedback"
```

**Generated Context:**

```markdown
# SYNTHESIZED CONTEXT

## DOMAIN CONTEXT
This task involves qualitative and quantitative analysis of customer feedback
within user research and product management. Customer feedback analysis aims to
extract actionable insights from various feedback channels to inform product
decisions, improve customer satisfaction, and identify emerging trends.

## KEY TERMINOLOGY
- **Sentiment Analysis**: Computational identification of emotional tone
- **Topic Modeling**: Automatic discovery of themes in text
- **NPS (Net Promoter Score)**: Metric measuring customer loyalty
- **CSAT (Customer Satisfaction)**: Direct satisfaction measurement
- **Voice of Customer (VoC)**: Systematic capture of customer expectations
- **Verbatim Analysis**: Examination of exact customer words
- **Closed-Loop Feedback**: Following up on customer input

## APPLICABLE FRAMEWORKS
1. **Thematic Analysis**: Systematic theme identification
   - Application: Code feedback into themes, analyze patterns

2. **Jobs-to-be-Done**: Outcome-focused analysis
   - Application: Map feedback to customer jobs and outcomes

3. **Kano Model**: Feature satisfaction classification
   - Application: Categorize feedback as basic, performance, or delighter

4. **Affinity Mapping**: Visual grouping of insights
   - Application: Cluster related feedback items into actionable groups

## HISTORICAL CONTEXT
Feedback analysis has evolved from manual survey review to automated NLP-powered
analysis. Modern approaches combine quantitative metrics with qualitative insights
for holistic understanding.

## CURRENT TRENDS
- AI-powered sentiment and intent detection
- Real-time feedback analysis
- Omnichannel feedback integration
- Predictive analytics on customer feedback
- Automated insight summarization

## IMPORTANT CONSIDERATIONS
- **Constraints**: Data privacy regulations, feedback volume, analysis timeframe
- **Assumptions**: Feedback is representative, honest responses
- **Risks**: Confirmation bias, over-indexing on vocal minority
- **Dependencies**: Feedback collection systems, analysis tools, domain expertise
- **Success Factors**: Actionable insights, clear prioritization, stakeholder buy-in

## EDGE CASES TO ADDRESS
1. **Conflicting Feedback**
   - Likelihood: High
   - Impact: Medium
   - Mitigation: Segment analysis, identify underlying needs

2. **Spam/Bot Contamination**
   - Likelihood: Medium
   - Impact: High
   - Mitigation: Implement filtering, validate data quality

3. **Contextual Ambiguity**
   - Likelihood: High
   - Impact: Medium
   - Mitigation: Cross-reference with metadata, follow-up interviews

---
ORIGINAL REQUEST:
analyze customer feedback

Please address this request using the context provided above.
```

---

## Best Practices

### 1. Input Quality Enhancement

Even with minimal input, small improvements yield better context:

```python
# Good: Include action verbs
"optimize database queries"  # Clear intent

# Better: Add a constraint or goal
"optimize slow database queries for e-commerce checkout"  # More specific context

# Best: Include domain signals
"optimize PostgreSQL queries causing timeout in checkout flow"  # Specific technology and scenario
```

### 2. Domain Template Maintenance

Keep domain knowledge current and accurate:

```python
class DomainTemplateManager:
    def update_domain_knowledge(self, domain: str, updates: dict):
        """Regularly update domain templates with new information."""
        current = self.load_domain_template(domain)

        # Merge updates while preserving structure
        updated = self.merge_knowledge(current, updates)

        # Validate for consistency
        self.validate_template(updated)

        # Version and save
        self.save_with_version(domain, updated)
```

### 3. Context Relevance Scoring

Implement feedback mechanisms to improve relevance:

```python
class RelevanceScorer:
    def score_generated_context(self, context: dict, user_feedback: dict) -> float:
        """Score context quality based on user feedback."""
        scores = {
            "usefulness": user_feedback.get("helpful", 0.5),
            "completeness": user_feedback.get("complete", 0.5),
            "accuracy": user_feedback.get("accurate", 0.5),
            "relevance": user_feedback.get("relevant", 0.5)
        }

        return sum(scores.values()) / len(scores)
```

### 4. Token Budget Management

Optimize context length for different models:

```python
class TokenBudgetManager:
    def __init__(self, model_context_window: int):
        self.max_tokens = model_context_window
        self.reserve_for_response = 0.4  # Reserve 40% for response

    def allocate_budget(self) -> dict:
        available = self.max_tokens * (1 - self.reserve_for_response)
        return {
            "background": int(available * 0.25),
            "terminology": int(available * 0.15),
            "frameworks": int(available * 0.20),
            "considerations": int(available * 0.20),
            "edge_cases": int(available * 0.15),
            "original_request": int(available * 0.05)
        }
```

### 5. Graceful Degradation

Handle uncertain or ambiguous inputs gracefully:

```python
def handle_ambiguous_input(input_text: str) -> dict:
    """Generate context even with ambiguous input."""
    analysis = analyze_ambiguity(input_text)

    if analysis["confidence"] < 0.3:
        # Generate clarifying questions
        return {
            "type": "clarification_needed",
            "questions": generate_clarifying_questions(input_text),
            "preliminary_context": generate_broad_context(input_text)
        }
    elif analysis["confidence"] < 0.6:
        # Generate context with caveats
        context = generate_context(input_text)
        context["caveats"] = identify_assumptions(input_text)
        return context
    else:
        # Full confidence generation
        return generate_full_context(input_text)
```

### 6. Multi-Domain Handling

Address inputs that span multiple domains:

```python
def handle_multi_domain_input(classification: dict) -> dict:
    """Generate context for cross-domain topics."""
    primary_context = generate_context(classification["primary"])

    # Add secondary domain context
    for secondary in classification["secondary"]:
        secondary_context = generate_context(secondary)
        primary_context = merge_contexts(
            primary_context,
            secondary_context,
            strategy="complement"  # Don't duplicate, complement
        )

    # Add cross-domain considerations
    primary_context["cross_domain"] = identify_integration_points(
        classification["primary"],
        classification["secondary"]
    )

    return primary_context
```

---

## Integration Tips

### 1. Pipeline Integration

```python
class ContextSynthesizerPipeline:
    def __init__(self):
        self.preprocessor = InputPreprocessor()
        self.detector = DomainDetector()
        self.generator = KnowledgeGenerator()
        self.recommender = FrameworkRecommender()
        self.analyzer = EdgeCaseAnalyzer()
        self.optimizer = OutputOptimizer()

    async def process(self, raw_input: str) -> str:
        # Execute pipeline stages
        preprocessed = await self.preprocessor.process(raw_input)
        classification = await self.detector.detect(preprocessed)
        context = await self.generator.generate(classification, preprocessed)
        frameworks = await self.recommender.recommend(classification)
        edge_cases = await self.analyzer.analyze(classification, context)

        # Compile output
        return await self.optimizer.compile({
            "original_input": raw_input,
            "context": context,
            "frameworks": frameworks,
            "edge_cases": edge_cases
        })
```

### 2. API Wrapper

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class SynthesisRequest(BaseModel):
    input_text: str
    depth: str = "moderate"  # minimal, moderate, comprehensive
    max_tokens: int = 2000

class SynthesisResponse(BaseModel):
    synthesized_prompt: str
    detected_domain: str
    confidence: float
    token_count: int

@app.post("/synthesize", response_model=SynthesisResponse)
async def synthesize_context(request: SynthesisRequest):
    pipeline = ContextSynthesizerPipeline()

    try:
        result = await pipeline.process(
            request.input_text,
            depth=request.depth,
            max_tokens=request.max_tokens
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### 3. LangChain Integration

```python
from langchain.prompts import BasePromptTemplate
from langchain.chains import LLMChain

class ContextSynthesizerPromptTemplate(BasePromptTemplate):
    """Custom LangChain prompt template with context synthesis."""

    synthesizer: ContextSynthesizerPipeline

    def format(self, **kwargs) -> str:
        raw_input = kwargs.get("input", "")

        # Synthesize context
        synthesized = self.synthesizer.process(raw_input)

        return synthesized

# Usage
from langchain.llms import OpenAI

synthesizer_template = ContextSynthesizerPromptTemplate(
    synthesizer=ContextSynthesizerPipeline(),
    input_variables=["input"]
)

chain = LLMChain(
    llm=OpenAI(),
    prompt=synthesizer_template
)

result = chain.run(input="optimize database queries")
```

### 4. Caching Strategy

```python
import hashlib
from functools import lru_cache

class CachedContextSynthesizer:
    def __init__(self):
        self.pipeline = ContextSynthesizerPipeline()
        self.cache = {}

    def _create_cache_key(self, input_text: str, depth: str) -> str:
        content = f"{input_text}:{depth}"
        return hashlib.md5(content.encode()).hexdigest()

    def synthesize(self, input_text: str, depth: str = "moderate") -> str:
        cache_key = self._create_cache_key(input_text, depth)

        if cache_key in self.cache:
            return self.cache[cache_key]

        result = self.pipeline.process(input_text, depth=depth)
        self.cache[cache_key] = result

        return result
```

### 5. Batch Processing

```python
import asyncio
from typing import List

class BatchContextSynthesizer:
    def __init__(self, concurrency: int = 5):
        self.pipeline = ContextSynthesizerPipeline()
        self.semaphore = asyncio.Semaphore(concurrency)

    async def synthesize_single(self, input_text: str) -> dict:
        async with self.semaphore:
            result = await self.pipeline.process(input_text)
            return {"input": input_text, "output": result}

    async def synthesize_batch(self, inputs: List[str]) -> List[dict]:
        tasks = [self.synthesize_single(text) for text in inputs]
        return await asyncio.gather(*tasks)
```

### 6. Monitoring and Logging

```python
import logging
import time
from dataclasses import dataclass

@dataclass
class SynthesisMetrics:
    input_length: int
    output_length: int
    processing_time: float
    domain_detected: str
    confidence: float

class MonitoredContextSynthesizer:
    def __init__(self):
        self.pipeline = ContextSynthesizerPipeline()
        self.logger = logging.getLogger(__name__)

    def synthesize_with_metrics(self, input_text: str) -> tuple:
        start_time = time.time()

        result = self.pipeline.process(input_text)

        metrics = SynthesisMetrics(
            input_length=len(input_text),
            output_length=len(result["synthesized_prompt"]),
            processing_time=time.time() - start_time,
            domain_detected=result["detected_domain"],
            confidence=result["confidence"]
        )

        self.logger.info(f"Synthesis completed: {metrics}")

        return result, metrics
```

---

## Success Criteria

Use this checklist to evaluate your Context Synthesizer implementation:

### Functionality

- [ ] Agent correctly identifies primary domain from minimal input
- [ ] Secondary/adjacent domains are detected when relevant
- [ ] Generated background context is accurate and relevant
- [ ] Key terminology is correctly identified and defined
- [ ] Suggested frameworks are applicable to the task type
- [ ] Historical context provides useful perspective
- [ ] Current trends are accurate and recent
- [ ] Edge cases are realistic and properly prioritized
- [ ] Output is well-structured and readable

### Quality

- [ ] Context depth matches input complexity appropriately
- [ ] No irrelevant or contradictory information included
- [ ] Terminology definitions are clear and accurate
- [ ] Framework applications are specific, not generic
- [ ] Considerations address real constraints and risks
- [ ] Edge case mitigations are actionable

### Performance

- [ ] Processing completes within acceptable time (< 5 seconds)
- [ ] Token usage is optimized for context window
- [ ] Caching reduces redundant processing
- [ ] Batch processing scales efficiently
- [ ] Memory usage is reasonable

### Usability

- [ ] Output format is consistent and predictable
- [ ] API is intuitive and well-documented
- [ ] Error messages are helpful and specific
- [ ] Clarifying questions are generated for ambiguous input
- [ ] Integration with existing tools is straightforward

### Maintainability

- [ ] Domain templates are easily updateable
- [ ] New domains can be added without code changes
- [ ] Logging provides useful debugging information
- [ ] Metrics enable performance monitoring
- [ ] Code is modular and testable

### Advanced Features

- [ ] Multi-domain topics are handled correctly
- [ ] Cross-domain integration points are identified
- [ ] Confidence scores accurately reflect certainty
- [ ] Feedback loop improves relevance over time
- [ ] Graceful degradation for edge cases works

---

## Conclusion

The Context Synthesizer agent transforms the prompt engineering process by automatically generating rich, relevant context from minimal input. By implementing intelligent domain detection, knowledge generation, and framework recommendation, this agent enables users to achieve high-quality LLM outputs without requiring deep expertise in prompt crafting.

Key implementation priorities:
1. Build comprehensive domain knowledge bases
2. Implement accurate domain detection
3. Design flexible context generation templates
4. Create robust edge case analysis
5. Optimize output structure for LLM consumption

This agent serves as a force multiplier for AI-assisted workflows, democratizing access to expert-level prompt engineering and enabling more effective human-AI collaboration across diverse domains and use cases.
