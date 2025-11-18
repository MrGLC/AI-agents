# 04 - Structure Architect

## Overview

The Structure Architect is a specialized prompt generator agent designed to transform unstructured or loosely-defined prompts into well-organized, clearly sectioned documents with explicit output formats and reusable templates. This agent excels at creating consistency across prompt libraries, ensuring that every generated prompt follows predictable patterns that maximize LLM comprehension and response quality.

Unlike basic prompt enhancers that focus on adding detail, the Structure Architect emphasizes **architectural organization** - breaking down complex requests into logical sections, defining precise output schemas, and establishing templates that can be reused across similar tasks. This makes it invaluable for teams building prompt libraries, developers creating API integrations, and anyone who needs reliable, formatted outputs from language models.

The agent operates on the principle that well-structured prompts lead to well-structured responses. By providing clear scaffolding for the LLM to follow, the Structure Architect dramatically reduces ambiguity, minimizes follow-up clarifications, and enables automated parsing of responses.

---

## Learning Objectives

By working with the Structure Architect agent, you will learn to:

### Foundational Skills
- Identify the key components that make up an effective prompt structure
- Understand the relationship between prompt organization and response quality
- Recognize common structural patterns used in professional prompt engineering
- Apply sectioning techniques to break down complex requests

### Intermediate Skills
- Design output format specifications that enable automated parsing
- Create reusable prompt templates for recurring task types
- Implement hierarchical organization for multi-part prompts
- Balance structure rigidity with flexibility for edge cases

### Advanced Skills
- Build prompt architecture systems that scale across teams
- Develop custom structural patterns for domain-specific applications
- Optimize section ordering for maximum LLM comprehension
- Create self-documenting prompts that include their own usage instructions

### Professional Applications
- Establish organizational standards for enterprise prompt libraries
- Design template systems for non-technical team members
- Implement version control strategies for structured prompts
- Create documentation that accompanies structured prompt systems

---

## Difficulty Level

**Intermediate to Advanced**

### Prerequisites
- Basic understanding of prompt engineering principles
- Familiarity with markdown or similar formatting languages
- Experience with at least one LLM API or interface
- Understanding of JSON, XML, or other data formats (helpful but not required)

### Time Investment
- Initial setup and understanding: 2-3 hours
- Basic proficiency: 5-8 hours of practice
- Advanced mastery: 20-30 hours of iterative refinement

### Complexity Factors
- Requires systematic thinking about information hierarchy
- Demands attention to detail in format specification
- Involves balancing comprehensiveness with clarity
- Needs iterative testing to validate structural effectiveness

---

## Key Features

### 1. Intelligent Section Detection

The Structure Architect analyzes input prompts to identify implicit sections that should be made explicit. It recognizes:

- **Context boundaries**: Where background information ends and instructions begin
- **Task decomposition points**: Natural breaks between sub-tasks
- **Constraint clusters**: Groups of related requirements or limitations
- **Output expectations**: Implicit format requirements that should be specified

```markdown
Input Analysis Pattern:
1. Scan for topic transitions
2. Identify enumerable elements
3. Detect conditional logic branches
4. Map dependency relationships
5. Extract implicit assumptions
```

### 2. Output Format Specification Engine

The agent includes a comprehensive system for defining exactly how LLM responses should be formatted:

**Supported Format Types:**
- Structured text (headers, lists, paragraphs)
- JSON schemas with type definitions
- XML with element specifications
- Markdown with specific styling requirements
- Tabular data with column definitions
- Code blocks with language specifications
- Mixed formats with clear delimiters

**Format Definition Components:**
- Field names and descriptions
- Data types and constraints
- Required vs. optional elements
- Default values
- Validation rules
- Example values

### 3. Template Generation System

Creates reusable templates from one-off prompts:

- **Variable extraction**: Identifies parts that change between uses
- **Placeholder syntax**: Standardized notation for insertable values
- **Default value support**: Pre-filled values for common cases
- **Validation hints**: Guidance on what makes valid input
- **Usage documentation**: Auto-generated instructions for template use

### 4. Hierarchical Organization

Implements multi-level structure for complex prompts:

```
Level 1: Major Sections (Context, Task, Format, Constraints)
Level 2: Subsections (Background, Requirements, Examples)
Level 3: Specific Items (Individual rules, single examples)
Level 4: Details (Clarifications, edge cases)
```

### 5. Consistency Enforcement

Ensures structural patterns remain consistent:

- Section naming conventions
- Formatting style guides
- Terminology standardization
- Cross-reference validation
- Completeness checking

### 6. Adaptive Complexity Scaling

Adjusts structural complexity based on task requirements:

- **Simple tasks**: Minimal sections, inline format hints
- **Medium tasks**: Standard section set, explicit format blocks
- **Complex tasks**: Full hierarchical structure, detailed schemas
- **Enterprise tasks**: Complete documentation, validation rules, examples

---

## How It Works

### Step 1: Input Analysis

The Structure Architect begins by parsing the raw input to understand its components:

```markdown
Analysis Process:
1. Tokenize input into semantic units
2. Classify each unit (context, instruction, constraint, example)
3. Identify relationships between units
4. Detect missing elements that should be added
5. Assess overall complexity level
```

**Example Classification:**
```
"Write a blog post about AI" -> Instruction (primary task)
"for a technical audience" -> Constraint (audience)
"around 1000 words" -> Constraint (length)
"include code examples" -> Requirement (content type)
```

### Step 2: Structure Selection

Based on analysis, the agent selects an appropriate structural template:

| Complexity | Structure Type | Sections Included |
|------------|----------------|-------------------|
| Minimal | Basic | Task, Format |
| Low | Standard | Context, Task, Format |
| Medium | Extended | Context, Task, Requirements, Format, Examples |
| High | Comprehensive | All sections plus subsections |
| Maximum | Enterprise | Full documentation with metadata |

### Step 3: Section Population

Each section is populated with relevant content from the input, plus generated additions:

**Context Section Building:**
1. Extract explicit context from input
2. Infer implicit context from task type
3. Add relevant background information
4. Define key terms if needed
5. Establish scope boundaries

**Task Section Building:**
1. Identify primary objective
2. Break into sub-tasks if complex
3. Establish task sequence/priority
4. Clarify success criteria
5. Note dependencies

**Format Section Building:**
1. Determine appropriate output type
2. Define structure/schema
3. Specify field details
4. Provide format examples
5. Include validation rules

### Step 4: Output Format Definition

The agent creates precise specifications for expected responses:

```markdown
## Output Format

### Response Structure
Your response must follow this exact structure:

#### Section 1: [Name]
- Format: [paragraph/list/code/etc.]
- Length: [specific or range]
- Content: [what to include]

#### Section 2: [Name]
...

### Schema (if applicable)
```json
{
  "field_name": {
    "type": "string",
    "description": "What this field contains",
    "required": true,
    "example": "Sample value"
  }
}
```

### Validation Rules
- Rule 1: [specific requirement]
- Rule 2: [specific requirement]
```

### Step 5: Template Extraction

If the prompt will be reused, variables are extracted:

```markdown
Original: "Write a blog post about machine learning for beginners"

Template: "Write a {{content_type}} about {{topic}} for {{audience}}"

Variables:
- content_type: Type of content (blog post, article, guide)
- topic: Subject matter to cover
- audience: Target reader expertise level
```

### Step 6: Validation and Refinement

Final checks ensure quality:

- [ ] All sections properly labeled
- [ ] Format specification is complete
- [ ] Examples match format requirements
- [ ] No contradictory instructions
- [ ] Appropriate complexity level
- [ ] Template variables documented

### Step 7: Output Generation

The final structured prompt is assembled and formatted for use.

---

## Example Usage

### Example 1: Basic Structuring

**Input (Unstructured):**
```
I need help writing product descriptions for an e-commerce site. They should be compelling and include key features. Make them SEO-friendly and around 150 words each.
```

**Output (Structured):**

```markdown
# Product Description Generator

## Context
You are an experienced e-commerce copywriter specializing in conversion-optimized product descriptions. Your writing balances persuasive marketing language with accurate product information.

## Task
Generate a compelling product description for the provided item that drives purchase decisions while maintaining accuracy.

### Primary Objectives
1. Highlight key product features and benefits
2. Create emotional connection with potential buyers
3. Incorporate relevant keywords naturally for SEO
4. Maintain brand voice consistency

## Input Required
Provide the following product information:
- Product name
- Category
- Key features (3-5 bullet points)
- Target customer profile
- Primary keywords for SEO
- Unique selling proposition

## Output Format

### Structure
```
[Attention-Grabbing Headline - 5-10 words]

[Opening Hook - 1-2 sentences that address customer pain point or desire]

[Feature-Benefit Paragraphs - 2-3 short paragraphs connecting features to customer benefits]

[Social Proof Element - Brief credibility indicator]

[Call to Action - Clear next step]
```

### Specifications
- Total length: 140-160 words
- Paragraph length: 2-3 sentences maximum
- Keyword density: Primary keyword 2-3 times, secondary keywords 1-2 times
- Tone: Professional yet approachable
- Voice: Second person (you/your)

### SEO Requirements
- Include primary keyword in headline
- Use secondary keywords in first paragraph
- Maintain natural readability (no keyword stuffing)

## Example

### Input
- Product: CloudStep Running Shoes
- Category: Athletic Footwear
- Features: Memory foam insole, breathable mesh, lightweight design
- Target: Casual runners, ages 25-45
- Keywords: comfortable running shoes, lightweight trainers
- USP: 40% lighter than competitors

### Output
**Run Lighter, Go Further with CloudStep**

Tired of heavy shoes slowing you down? CloudStep comfortable running shoes are engineered for runners who refuse to compromise on comfort or performance.

Our innovative memory foam insole adapts to your unique foot shape, providing personalized cushioning mile after mile. The breathable mesh upper keeps your feet cool during intense sessions, while our lightweight design means you'll feel like you're running on air.

At 40% lighter than leading competitors, these lightweight trainers let you focus on your stride, not your feet. Trusted by over 10,000 runners nationwide.

Ready to transform your run? Order now and feel the CloudStep difference.

## Constraints
- Avoid superlatives without evidence (best, greatest, etc.)
- Do not make unverifiable claims
- Maintain factual accuracy for all specifications
- Do not use more than one exclamation point
```

---

### Example 2: Complex Multi-Part Structure

**Input (Unstructured):**
```
Create a system that analyzes customer feedback and gives me sentiment scores, main topics discussed, action items for our team, and any urgent issues that need immediate attention. We get feedback from multiple channels.
```

**Output (Structured):**

```markdown
# Customer Feedback Analysis System

## Context

### System Purpose
You are an advanced customer feedback analyst processing multi-channel customer input to extract actionable business intelligence.

### Operating Environment
- Feedback sources: Email, social media, surveys, support tickets, reviews
- Volume: Variable (single item to batch processing)
- Languages: English primary, note if translation needed
- Industry context: [To be specified per use]

## Task Overview

Analyze provided customer feedback to generate a comprehensive report containing sentiment analysis, topic extraction, action items, and urgency flags.

### Analysis Components

#### Component 1: Sentiment Analysis
- Calculate overall sentiment score
- Identify sentiment by aspect/feature mentioned
- Track sentiment trends if multiple items provided

#### Component 2: Topic Extraction
- Identify main topics discussed
- Categorize by business area
- Note topic frequency and co-occurrence

#### Component 3: Action Item Generation
- Extract explicit customer requests
- Infer implicit improvement opportunities
- Prioritize by impact and feasibility

#### Component 4: Urgency Detection
- Flag time-sensitive issues
- Identify potential escalations
- Highlight risk indicators

## Input Format

### Single Feedback Item
```json
{
  "id": "unique_identifier",
  "source": "email|social|survey|ticket|review",
  "timestamp": "ISO 8601 format",
  "customer_segment": "optional segment identifier",
  "content": "The actual feedback text",
  "metadata": {
    "rating": "if applicable (1-5)",
    "product": "related product/service",
    "previous_interactions": "count if known"
  }
}
```

### Batch Processing
Array of feedback items in above format.

## Output Format

### Complete Analysis Report

```json
{
  "analysis_metadata": {
    "items_processed": "number",
    "analysis_timestamp": "ISO 8601",
    "confidence_score": "0.0-1.0"
  },

  "sentiment_analysis": {
    "overall_score": {
      "value": "-1.0 to 1.0",
      "label": "very_negative|negative|neutral|positive|very_positive",
      "confidence": "0.0-1.0"
    },
    "aspect_sentiments": [
      {
        "aspect": "feature or area name",
        "score": "-1.0 to 1.0",
        "mentions": "count",
        "sample_quotes": ["relevant excerpts"]
      }
    ],
    "sentiment_distribution": {
      "positive": "percentage",
      "neutral": "percentage",
      "negative": "percentage"
    }
  },

  "topic_analysis": {
    "primary_topics": [
      {
        "topic": "topic name",
        "category": "product|service|support|pricing|other",
        "frequency": "count or percentage",
        "sentiment_correlation": "score",
        "keywords": ["related terms"]
      }
    ],
    "emerging_topics": [
      {
        "topic": "newly appearing topic",
        "first_seen": "timestamp",
        "growth_rate": "percentage increase"
      }
    ],
    "topic_relationships": [
      {
        "topics": ["topic1", "topic2"],
        "correlation_strength": "0.0-1.0",
        "nature": "causal|co-occurrence|sequential"
      }
    ]
  },

  "action_items": [
    {
      "id": "AI-001",
      "title": "Brief action description",
      "description": "Detailed explanation",
      "source_feedback": ["feedback IDs"],
      "category": "product|process|communication|training|technical",
      "priority": "critical|high|medium|low",
      "estimated_impact": {
        "customer_satisfaction": "high|medium|low",
        "revenue": "high|medium|low",
        "operational_efficiency": "high|medium|low"
      },
      "suggested_owner": "department or role",
      "suggested_timeline": "immediate|short-term|medium-term|long-term"
    }
  ],

  "urgent_issues": [
    {
      "id": "URG-001",
      "title": "Issue summary",
      "description": "Full details",
      "source_feedback": ["feedback IDs"],
      "urgency_indicators": ["what makes this urgent"],
      "risk_level": "critical|high|medium",
      "recommended_response_time": "hours or days",
      "escalation_path": "suggested escalation route",
      "potential_impact": {
        "customers_affected": "estimate",
        "revenue_at_risk": "estimate if applicable",
        "reputation_risk": "high|medium|low"
      }
    }
  ],

  "executive_summary": {
    "key_findings": ["3-5 bullet points"],
    "immediate_actions": ["top priority items"],
    "trends_to_watch": ["emerging patterns"],
    "overall_health_score": "0-100"
  }
}
```

### Report Sections Explained

#### Sentiment Analysis Section
- **Overall Score**: Aggregate sentiment from -1 (very negative) to 1 (very positive)
- **Aspect Sentiments**: Breakdown by specific features or areas mentioned
- **Distribution**: Percentage split for quick overview

#### Topic Analysis Section
- **Primary Topics**: Most discussed subjects with business categorization
- **Emerging Topics**: New subjects appearing in recent feedback
- **Relationships**: How topics correlate or influence each other

#### Action Items Section
- **Priority Levels**:
  - Critical: Requires immediate attention
  - High: Address within one week
  - Medium: Address within one month
  - Low: Add to backlog for future consideration

#### Urgent Issues Section
- **Risk Levels**:
  - Critical: Potential for significant customer loss or PR crisis
  - High: Multiple customers affected, escalation likely
  - Medium: Single customer or contained issue

## Processing Rules

### Sentiment Scoring Guidelines
1. Account for sarcasm and context
2. Weight recent feedback more heavily in trends
3. Consider industry-specific language
4. Adjust for cultural communication differences
5. Flag ambiguous cases for human review

### Topic Extraction Guidelines
1. Use consistent topic naming across analyses
2. Map synonyms to standard topic names
3. Create new topics only when existing ones don't fit
4. Maintain topic hierarchy (parent-child relationships)
5. Track topic evolution over time

### Action Item Generation Guidelines
1. One action per discrete improvement opportunity
2. Combine similar requests from multiple customers
3. Ensure actions are specific and measurable
4. Include success criteria where possible
5. Avoid duplicating existing backlog items

### Urgency Flag Guidelines
1. Immediate safety or legal concerns: Always flag
2. Service outages affecting multiple users: Always flag
3. Public negative reviews: Flag if influential platform
4. Repeat complaints from single customer: Flag at third occurrence
5. Issues with revenue impact: Flag with estimate

## Constraints

### Must Include
- At least one action item per negative feedback item
- Confidence scores for all sentiment analyses
- Source attribution for all extracted information
- Timestamp for all temporal references

### Must Avoid
- Personally identifiable information in outputs
- Unsubstantiated conclusions
- Recommendations outside company capabilities
- Ignoring minority but valid concerns

### Quality Thresholds
- Sentiment confidence below 0.6: Flag for human review
- Topic match confidence below 0.7: Suggest new topic creation
- Action item without clear source: Require justification

## Example Analysis

[Include a complete worked example with sample feedback items and full output]
```

---

### Example 3: Template Generation

**Input:**
```
Turn this into a reusable template: "Write a thank you email to John for his presentation on quarterly sales. Mention that the data visualizations were particularly helpful and ask him to share the slides with the team."
```

**Output (Structured Template):**

```markdown
# Professional Thank You Email Template

## Template Overview
Generate personalized thank you emails for internal presentations and contributions.

## Template Variables

| Variable | Description | Type | Required | Default | Example |
|----------|-------------|------|----------|---------|---------|
| `recipient_name` | Name of person being thanked | string | Yes | - | John |
| `presentation_topic` | Subject of the presentation | string | Yes | - | quarterly sales |
| `specific_highlight` | Particular element to praise | string | Yes | - | data visualizations |
| `highlight_benefit` | Why the highlight was valuable | string | No | "helpful" | particularly helpful for planning |
| `follow_up_request` | Action requested from recipient | string | No | - | share the slides with the team |
| `sender_name` | Name of email sender | string | Yes | - | Sarah |
| `team_name` | Team or department name | string | No | "the team" | the marketing team |

## Prompt Template

```
Write a professional thank you email with the following parameters:

RECIPIENT: {{recipient_name}}
PRESENTATION TOPIC: {{presentation_topic}}
SPECIFIC HIGHLIGHT: {{specific_highlight}}
WHY IT WAS VALUABLE: {{highlight_benefit}}
FOLLOW-UP REQUEST: {{follow_up_request}}
SENDER: {{sender_name}}
TEAM: {{team_name}}

## Email Requirements

### Tone
- Professional yet warm
- Genuine appreciation (not formulaic)
- Respectful of recipient's time

### Structure
1. **Subject Line**: Clear, specific reference to presentation
2. **Opening**: Direct thank you with specific reference
3. **Body**: Highlight what was valuable and why
4. **Request** (if applicable): Clear, polite ask
5. **Closing**: Forward-looking positive statement
6. **Sign-off**: Appropriate professional closing

### Length
- Total: 75-150 words
- Keep paragraphs to 2-3 sentences

## Output Format

```
Subject: [Generated subject line]

[Email body with appropriate greeting and sign-off]
```

## Constraints
- Do not use overly effusive language
- Keep the focus on professional value delivered
- Make any request clearly optional in tone
- Avoid generic phrases like "great job" without specifics
```

## Template Usage Example

### Input Values
```json
{
  "recipient_name": "Maria",
  "presentation_topic": "Q3 marketing campaign results",
  "specific_highlight": "competitive analysis section",
  "highlight_benefit": "helped us understand our market positioning",
  "follow_up_request": "send the competitor comparison chart to the sales team",
  "sender_name": "Alex",
  "team_name": "the product team"
}
```

### Generated Output
```
Subject: Thank You - Q3 Marketing Campaign Results Presentation

Hi Maria,

Thank you for presenting the Q3 marketing campaign results yesterday. Your insights gave the product team valuable perspective on our recent initiatives.

The competitive analysis section was particularly impactful. Understanding our market positioning through that lens helped us identify several opportunities for our upcoming roadmap planning.

If you have a moment, would you mind sending the competitor comparison chart to the sales team? I think they'd find it equally useful for their customer conversations.

Looking forward to seeing how these insights shape our Q4 strategy.

Best regards,
Alex
```

---

## Best Practices

### 1. Start with Output Definition

**Do This:**
Define your desired output format before writing the rest of the prompt. This clarifies what the prompt needs to achieve.

**Avoid:**
Writing the entire prompt then trying to retrofit format requirements.

### 2. Use Consistent Section Headers

**Do This:**
```markdown
## Context
## Task
## Input Format
## Output Format
## Constraints
## Examples
```

**Avoid:**
Mixing header styles, inconsistent capitalization, or synonymous headers in the same prompt library.

### 3. Make Structure Visual

**Do This:**
- Use code blocks for format specifications
- Include tables for variable documentation
- Add visual hierarchy with nested lists
- Separate major sections with horizontal rules

**Avoid:**
Long paragraphs describing structure that should be shown visually.

### 4. Provide Complete Examples

**Do This:**
Include at least one fully worked example that demonstrates:
- All input fields populated
- Complete output following the format
- Edge cases handled appropriately

**Avoid:**
Partial examples or examples that don't match the specified format.

### 5. Balance Rigidity with Flexibility

**Do This:**
- Mark clearly which elements are required vs. optional
- Provide defaults for optional fields
- Allow for edge case handling
- Include "other" or "notes" fields for unexpected items

**Avoid:**
Over-constraining formats so edge cases break the system.

### 6. Version Your Templates

**Do This:**
- Include version numbers in templates
- Document changes between versions
- Maintain backward compatibility notes
- Archive deprecated templates

**Avoid:**
Modifying templates in place without tracking changes.

### 7. Test with Edge Cases

**Do This:**
- Test with minimal input
- Test with maximum complexity
- Test with unusual but valid input
- Test with adversarial input

**Avoid:**
Only testing with ideal-case examples.

### 8. Document Assumptions

**Do This:**
Explicitly state:
- What the LLM should assume if not specified
- Default behaviors
- Scope boundaries
- What's intentionally excluded

**Avoid:**
Leaving implicit assumptions that different users might interpret differently.

---

## Integration Tips

### API Integration

When using structured prompts with LLM APIs:

```python
# Example: Using structured prompt with output parsing

import json

def parse_structured_response(response_text, expected_format):
    """Parse LLM response according to expected format."""

    # If JSON format expected
    if expected_format == "json":
        # Find JSON block in response
        json_start = response_text.find('{')
        json_end = response_text.rfind('}') + 1
        json_str = response_text[json_start:json_end]
        return json.loads(json_str)

    # If sectioned format expected
    elif expected_format == "sections":
        sections = {}
        current_section = None
        current_content = []

        for line in response_text.split('\n'):
            if line.startswith('## '):
                if current_section:
                    sections[current_section] = '\n'.join(current_content)
                current_section = line[3:].strip()
                current_content = []
            else:
                current_content.append(line)

        if current_section:
            sections[current_section] = '\n'.join(current_content)

        return sections

    return response_text
```

### Prompt Library Management

```yaml
# Example: Prompt library configuration

prompts:
  product_description:
    version: "2.1.0"
    template_path: "templates/product_description.md"
    variables:
      - name: product_name
        required: true
      - name: features
        required: true
        type: array
      - name: audience
        required: false
        default: "general consumers"
    output_format: "structured_text"
    validators:
      - word_count: [140, 160]
      - required_sections: ["headline", "body", "cta"]

  feedback_analysis:
    version: "1.3.0"
    template_path: "templates/feedback_analysis.md"
    output_format: "json"
    schema_path: "schemas/feedback_analysis.json"
```

### Chaining Structured Prompts

```python
# Example: Multi-stage structured workflow

class StructuredWorkflow:
    def __init__(self, stages):
        self.stages = stages

    def execute(self, initial_input):
        current_output = initial_input

        for stage in self.stages:
            prompt = stage.template.format(**current_output)
            response = llm.generate(prompt)
            current_output = stage.parser(response)

            # Validate output matches expected structure
            if not stage.validator(current_output):
                raise StructureValidationError(
                    f"Stage {stage.name} output invalid"
                )

        return current_output

# Usage
workflow = StructuredWorkflow([
    Stage("extract", extract_template, json_parser, schema_validator),
    Stage("analyze", analyze_template, section_parser, section_validator),
    Stage("generate", generate_template, text_parser, format_validator)
])

result = workflow.execute(user_input)
```

### Error Handling

```python
# Handle structure validation errors

class StructureValidator:
    def __init__(self, schema):
        self.schema = schema

    def validate(self, response):
        errors = []

        # Check required sections
        for section in self.schema.get('required_sections', []):
            if section not in response:
                errors.append(f"Missing required section: {section}")

        # Check format compliance
        for field, rules in self.schema.get('field_rules', {}).items():
            if field in response:
                value = response[field]

                if 'type' in rules and not isinstance(value, rules['type']):
                    errors.append(f"Field {field} has wrong type")

                if 'min_length' in rules and len(value) < rules['min_length']:
                    errors.append(f"Field {field} too short")

                if 'max_length' in rules and len(value) > rules['max_length']:
                    errors.append(f"Field {field} too long")

        return len(errors) == 0, errors
```

### Monitoring and Analytics

Track structured prompt performance:

```python
# Metrics to monitor

metrics = {
    "structure_compliance_rate": "Percentage of responses matching format",
    "parse_success_rate": "Percentage of responses successfully parsed",
    "section_completeness": "Average percentage of sections present",
    "format_error_types": "Distribution of format errors by type",
    "template_usage": "Usage counts per template",
    "variable_coverage": "Percentage of optional variables used"
}
```

---

## Success Criteria

Use this checklist to evaluate structured prompts:

### Structure Quality

- [ ] Clear, consistent section headers throughout
- [ ] Logical flow from context to task to format
- [ ] Appropriate nesting depth (not too flat, not too deep)
- [ ] Visual formatting aids comprehension
- [ ] Sections are self-contained but connected

### Output Format Specification

- [ ] Format type is explicitly stated
- [ ] All fields/sections are defined
- [ ] Data types are specified where applicable
- [ ] Required vs. optional elements are marked
- [ ] Examples match the specified format exactly
- [ ] Validation rules are clear and testable

### Template Usability

- [ ] Variables are clearly identified
- [ ] Variable documentation is complete
- [ ] Default values are sensible
- [ ] Template can handle edge cases
- [ ] Usage instructions are included

### Completeness

- [ ] No ambiguous instructions
- [ ] All necessary context is provided
- [ ] Constraints are explicitly stated
- [ ] Success criteria are defined
- [ ] At least one complete example included

### Maintainability

- [ ] Version information included
- [ ] No redundant or contradictory sections
- [ ] Terminology is consistent
- [ ] Structure can be extended without rewriting
- [ ] Documentation is sufficient for other users

### Testability

- [ ] Expected output can be validated programmatically
- [ ] Format errors are detectable
- [ ] Edge cases are documented
- [ ] Error messages can be specific and actionable

### Performance

- [ ] Prompt length is appropriate for task complexity
- [ ] Structure improves (not impedes) LLM comprehension
- [ ] Response quality meets requirements
- [ ] Parse success rate exceeds 95%
- [ ] Consistent results across multiple runs

---

## Quick Reference

### Minimal Viable Structure
```markdown
## Task
[What to do]

## Output Format
[How to format the response]
```

### Standard Structure
```markdown
## Context
[Background information]

## Task
[Primary objective and sub-tasks]

## Output Format
[Detailed format specification]

## Constraints
[Limitations and requirements]

## Example
[Complete worked example]
```

### Enterprise Structure
```markdown
## Metadata
[Version, author, last updated, category]

## Context
### Background
### Scope
### Assumptions

## Task
### Objective
### Sub-tasks
### Success Criteria

## Input Specification
### Required Fields
### Optional Fields
### Validation Rules

## Output Format
### Structure
### Schema
### Validation Rules
### Examples

## Processing Rules
### Handling Instructions
### Edge Cases
### Error Conditions

## Constraints
### Must Include
### Must Avoid
### Quality Thresholds

## Examples
### Basic Example
### Complex Example
### Edge Case Example

## Changelog
[Version history]
```

---

## Next Steps

After mastering the Structure Architect:

1. **Build a prompt library** using consistent structures
2. **Create domain-specific templates** for your industry
3. **Implement automated validation** for your structured prompts
4. **Develop a style guide** for your organization's prompt structure
5. **Train team members** on using and creating structured prompts

---

## Related Resources

- Output parsers and validators
- JSON Schema for LLM outputs
- Prompt versioning strategies
- Template management systems
- Quality assurance for prompts
