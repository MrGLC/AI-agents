# Project 09: Build an Autonomous Research Assistant with Citations

## Overview
Build an **autonomous research assistant** that can conduct comprehensive research on any topic, synthesize information from multiple sources, and produce well-cited reports. This agent combines web search, content analysis, fact verification, and academic-style writing with proper source attribution.

## Learning Objectives
- Implement autonomous research workflows
- Build citation management systems
- Create source validation and ranking
- Synthesize information from multiple sources
- Generate academic-style reports
- Implement fact-checking mechanisms
- Handle conflicting information

## Difficulty Level
**Advanced** - Requires integration of multiple systems and complex reasoning.

## Technical Stack
- **Framework**: LangChain, LangGraph
- **LLM**: OpenAI GPT-4 (for synthesis and writing)
- **Search**: DuckDuckGo, SerpAPI, or custom search
- **Citations**: Custom citation manager
- **Storage**: Vector store for source management
- **Validation**: Fact-checking tools
- **Export**: Markdown, PDF generation

## Project Requirements

### Research Capabilities
- Formulate research questions
- Conduct multi-source web searches
- Extract key information from sources
- Validate source credibility
- Detect information conflicts
- Synthesize findings

### Citation Features
- Track all sources with metadata
- Generate formatted citations (APA, MLA, Chicago)
- Link citations to specific claims
- Provide source URLs
- Rank sources by credibility
- Handle duplicate sources

### Report Generation
- Structured academic format
- Clear thesis and arguments
- Evidence-based claims
- Proper citations
- Bibliography/References section
- Executive summary

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langgraph>=0.0.20
duckduckgo-search>=4.0.0
beautifulsoup4>=4.12.0
requests>=2.31.0
pydantic>=2.0.0
python-dotenv>=1.0.0
markdown>=3.5.0
chromadb>=0.4.22
```

```bash
pip install -r requirements.txt
```

### Step 2: Build Citation Manager

```python
# citation_manager.py
from typing import List, Dict, Optional, Literal
from pydantic import BaseModel, Field, HttpUrl
from datetime import datetime
import hashlib

class Source(BaseModel):
    """Represents a research source."""

    id: str = Field(description="Unique source ID")
    url: HttpUrl
    title: str
    author: Optional[str] = None
    publication_date: Optional[datetime] = None
    access_date: datetime = Field(default_factory=datetime.now)
    content_snippet: str = Field(description="Relevant excerpt")
    credibility_score: float = Field(default=0.5, ge=0, le=1)
    citation_count: int = Field(default=0)

    @classmethod
    def from_search_result(cls, url: str, title: str, snippet: str):
        """Create source from search result."""
        # Generate ID from URL
        source_id = hashlib.md5(url.encode()).hexdigest()[:8]

        return cls(
            id=source_id,
            url=url,
            title=title,
            content_snippet=snippet
        )

class CitationManager:
    """Manage research sources and citations."""

    def __init__(self):
        """Initialize citation manager."""
        self.sources: Dict[str, Source] = {}
        self.citations: List[tuple[str, str]] = []  # (claim, source_id)

    def add_source(self, source: Source) -> str:
        """Add a source and return its ID."""
        # Check for duplicates
        for existing in self.sources.values():
            if existing.url == source.url:
                return existing.id

        self.sources[source.id] = source
        return source.id

    def cite(self, claim: str, source_id: str):
        """Link a claim to a source."""
        if source_id in self.sources:
            self.citations.append((claim, source_id))
            self.sources[source_id].citation_count += 1

    def get_source(self, source_id: str) -> Optional[Source]:
        """Get source by ID."""
        return self.sources.get(source_id)

    def format_citation(
        self,
        source_id: str,
        style: Literal["apa", "mla", "chicago"] = "apa"
    ) -> str:
        """Format citation in specified style."""

        source = self.sources.get(source_id)
        if not source:
            return ""

        if style == "apa":
            return self._format_apa(source)
        elif style == "mla":
            return self._format_mla(source)
        elif style == "chicago":
            return self._format_chicago(source)

    def _format_apa(self, source: Source) -> str:
        """Format APA citation."""
        author = source.author or "Unknown Author"
        year = source.publication_date.year if source.publication_date else "n.d."
        return f"{author}. ({year}). {source.title}. Retrieved from {source.url}"

    def _format_mla(self, source: Source) -> str:
        """Format MLA citation."""
        author = source.author or "Unknown Author"
        return f"{author}. \"{source.title}.\" Web. {source.access_date.strftime('%d %b. %Y')}. <{source.url}>"

    def _format_chicago(self, source: Source) -> str:
        """Format Chicago citation."""
        author = source.author or "Unknown Author"
        return f"{author}. \"{source.title}.\" Accessed {source.access_date.strftime('%B %d, %Y')}. {source.url}."

    def get_bibliography(self, style: str = "apa") -> str:
        """Generate bibliography of all sources."""
        bibliography = []

        # Sort sources by citation count (most cited first)
        sorted_sources = sorted(
            self.sources.values(),
            key=lambda s: s.citation_count,
            reverse=True
        )

        for i, source in enumerate(sorted_sources, 1):
            citation = self.format_citation(source.id, style)
            bibliography.append(f"[{i}] {citation}")

        return "\n".join(bibliography)

    def get_inline_citation(self, source_id: str) -> str:
        """Get inline citation marker."""
        # Find position of source in sorted list
        sorted_sources = sorted(
            self.sources.values(),
            key=lambda s: s.citation_count,
            reverse=True
        )

        for i, source in enumerate(sorted_sources, 1):
            if source.id == source_id:
                return f"[{i}]"

        return "[?]"
```

### Step 3: Build Research Agent

```python
# research_agent.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from duckduckgo_search import DDGS
from citation_manager import Source, CitationManager
from typing import List, Dict

load_dotenv()

class ResearchAgent:
    """Agent that conducts research and manages sources."""

    def __init__(self, citation_manager: CitationManager):
        """Initialize research agent."""

        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0.3,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.citation_manager = citation_manager

        self.research_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are a research assistant. For the given topic,
formulate 3-5 specific research questions that will help create a comprehensive understanding."""),
            ("human", "Topic: {topic}\n\nGenerate research questions.")
        ])

        self.synthesis_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are synthesizing research findings. Combine information
from multiple sources into a coherent narrative. Include inline citations [1], [2], etc."""),
            ("human", """Topic: {topic}

Research Findings:
{findings}

Write a comprehensive synthesis with citations.""")
        ])

    def formulate_questions(self, topic: str) -> List[str]:
        """Generate research questions for topic."""

        chain = self.research_prompt | self.llm
        response = chain.invoke({"topic": topic})

        # Parse questions from response
        questions = []
        for line in response.content.split('\n'):
            line = line.strip()
            if line and (line[0].isdigit() or line.startswith('-')):
                # Remove numbering/bullets
                question = line.lstrip('0123456789.-) ').strip()
                if question:
                    questions.append(question)

        return questions

    def search_and_collect(self, query: str, max_results: int = 5) -> List[Source]:
        """Search web and collect sources."""

        print(f"   🔍 Searching: {query}")

        try:
            with DDGS() as ddgs:
                results = list(ddgs.text(query, max_results=max_results))

            sources = []
            for result in results:
                source = Source.from_search_result(
                    url=result['href'],
                    title=result['title'],
                    snippet=result['body']
                )

                # Assess credibility (simplified)
                source.credibility_score = self._assess_credibility(source)

                # Add to citation manager
                source_id = self.citation_manager.add_source(source)
                sources.append(source)

            print(f"   ✓ Found {len(sources)} sources")
            return sources

        except Exception as e:
            print(f"   ✗ Search error: {e}")
            return []

    def _assess_credibility(self, source: Source) -> float:
        """Assess source credibility (simplified)."""

        score = 0.5  # Base score

        url_str = str(source.url).lower()

        # Boost for trusted domains
        trusted_domains = ['.edu', '.gov', '.org', 'wikipedia', 'arxiv', 'nature', 'science']
        for domain in trusted_domains:
            if domain in url_str:
                score += 0.2
                break

        # Reduce for less reliable indicators
        if any(x in url_str for x in ['blog', 'forum', 'social']):
            score -= 0.1

        return min(max(score, 0.0), 1.0)

    def conduct_research(self, topic: str) -> Dict:
        """Conduct comprehensive research on topic."""

        print(f"\n📚 Researching: {topic}")

        # Generate research questions
        questions = self.formulate_questions(topic)
        print(f"\n📋 Research Questions ({len(questions)}):")
        for i, q in enumerate(questions, 1):
            print(f"   {i}. {q}")

        # Search for each question
        all_findings = {}

        for question in questions:
            sources = self.search_and_collect(question, max_results=3)
            all_findings[question] = sources

        return all_findings

    def synthesize(self, topic: str, findings: Dict) -> str:
        """Synthesize research findings into report."""

        # Format findings with citations
        formatted_findings = []

        for question, sources in findings.items():
            formatted_findings.append(f"\nQuestion: {question}")

            for source in sources:
                citation_marker = self.citation_manager.get_inline_citation(source.id)
                formatted_findings.append(
                    f"- {source.content_snippet} {citation_marker}"
                )

        findings_text = "\n".join(formatted_findings)

        # Generate synthesis
        chain = self.synthesis_prompt | self.llm
        response = chain.invoke({
            "topic": topic,
            "findings": findings_text
        })

        return response.content
```

### Step 4: Build Report Generator

```python
# report_generator.py
from typing import Dict
from citation_manager import CitationManager
from datetime import datetime

class ReportGenerator:
    """Generate formatted research reports."""

    def __init__(self, citation_manager: CitationManager):
        """Initialize report generator."""
        self.citation_manager = citation_manager

    def generate_report(
        self,
        topic: str,
        synthesis: str,
        executive_summary: str = None,
        citation_style: str = "apa"
    ) -> str:
        """Generate complete research report."""

        report_parts = []

        # Title and metadata
        report_parts.append(f"# Research Report: {topic}")
        report_parts.append(f"\n*Generated: {datetime.now().strftime('%B %d, %Y')}*\n")

        # Executive summary
        if executive_summary:
            report_parts.append("## Executive Summary\n")
            report_parts.append(executive_summary)
            report_parts.append("\n")

        # Main content
        report_parts.append("## Research Findings\n")
        report_parts.append(synthesis)
        report_parts.append("\n")

        # Bibliography
        report_parts.append("## References\n")
        bibliography = self.citation_manager.get_bibliography(citation_style)
        report_parts.append(bibliography)
        report_parts.append("\n")

        # Source credibility summary
        report_parts.append("## Source Credibility Assessment\n")

        sources_by_credibility = sorted(
            self.citation_manager.sources.values(),
            key=lambda s: s.credibility_score,
            reverse=True
        )

        for source in sources_by_credibility:
            marker = self.citation_manager.get_inline_citation(source.id)
            credibility = "High" if source.credibility_score > 0.7 else "Medium" if source.credibility_score > 0.4 else "Low"
            report_parts.append(f"{marker} {source.title} - Credibility: {credibility}")

        return "\n".join(report_parts)

    def save_report(self, report: str, filename: str):
        """Save report to file."""
        with open(filename, 'w', encoding='utf-8') as f:
            f.write(report)
        print(f"\n✓ Report saved to {filename}")
```

### Step 5: Build Complete Research Workflow

```python
# research_workflow.py
from citation_manager import CitationManager
from research_agent import ResearchAgent
from report_generator import ReportGenerator
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
import os

class ResearchWorkflow:
    """Complete research workflow orchestration."""

    def __init__(self):
        """Initialize research workflow."""
        self.citation_manager = CitationManager()
        self.research_agent = ResearchAgent(self.citation_manager)
        self.report_generator = ReportGenerator(self.citation_manager)

        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0.5,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

    def research_and_report(
        self,
        topic: str,
        output_file: str = None,
        citation_style: str = "apa"
    ) -> str:
        """
        Conduct research and generate report.

        Args:
            topic: Research topic
            output_file: Output filename (optional)
            citation_style: Citation style (apa, mla, chicago)

        Returns:
            Generated report text
        """
        print("\n" + "="*80)
        print("AUTONOMOUS RESEARCH ASSISTANT")
        print("="*80)

        # Conduct research
        findings = self.research_agent.conduct_research(topic)

        # Synthesize findings
        print("\n✍️  Synthesizing research...")
        synthesis = self.research_agent.synthesize(topic, findings)

        # Generate executive summary
        print("\n📊 Creating executive summary...")
        executive_summary = self._generate_summary(topic, synthesis)

        # Generate complete report
        print("\n📄 Generating report...")
        report = self.report_generator.generate_report(
            topic=topic,
            synthesis=synthesis,
            executive_summary=executive_summary,
            citation_style=citation_style
        )

        # Save if filename provided
        if output_file:
            self.report_generator.save_report(report, output_file)

        print("\n" + "="*80)
        print("RESEARCH COMPLETE")
        print("="*80)

        return report

    def _generate_summary(self, topic: str, synthesis: str) -> str:
        """Generate executive summary."""

        prompt = ChatPromptTemplate.from_messages([
            ("system", "Create a brief executive summary (2-3 sentences) of the research findings."),
            ("human", "Topic: {topic}\n\nResearch:\n{synthesis}\n\nExecutive Summary:")
        ])

        chain = prompt | self.llm
        response = chain.invoke({"topic": topic, "synthesis": synthesis})

        return response.content.strip()
```

### Step 6: Main Application

```python
# main.py
from research_workflow import ResearchWorkflow

def main():
    """Interactive research assistant."""

    print("Autonomous Research Assistant with Citations")
    print("="*80)
    print("This assistant will conduct research and generate cited reports.")
    print()

    workflow = ResearchWorkflow()

    while True:
        print("\nOptions:")
        print("  1. Conduct research")
        print("  2. Quit")

        choice = input("\nYour choice: ").strip()

        if choice == '2':
            print("Goodbye!")
            break

        if choice != '1':
            continue

        # Get research parameters
        topic = input("\nResearch topic: ").strip()
        if not topic:
            continue

        citation_style = input("Citation style (apa/mla/chicago, default: apa): ").strip().lower()
        if citation_style not in ['apa', 'mla', 'chicago']:
            citation_style = 'apa'

        save_file = input("Save to file? (y/n): ").strip().lower() == 'y'
        filename = None

        if save_file:
            filename = input("Filename (default: research_report.md): ").strip()
            if not filename:
                filename = "research_report.md"

        # Conduct research
        try:
            report = workflow.research_and_report(
                topic=topic,
                output_file=filename,
                citation_style=citation_style
            )

            # Display report
            print("\n" + "="*80)
            print("REPORT PREVIEW")
            print("="*80)
            print(report[:1000] + "..." if len(report) > 1000 else report)
            print("="*80)

        except Exception as e:
            print(f"\n✗ Error: {e}")

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example Research Report
```
# Research Report: Impact of Artificial Intelligence on Healthcare

*Generated: November 14, 2024*

## Executive Summary

AI is transforming healthcare through improved diagnostics, personalized treatment plans,
and operational efficiency. Key applications include medical imaging analysis, drug discovery,
and predictive analytics, with significant potential to improve patient outcomes.

## Research Findings

Artificial intelligence has revolutionized medical imaging, with deep learning models achieving
diagnostic accuracy comparable to expert radiologists [1][2]. Studies show AI systems can detect
diseases like cancer at earlier stages, improving treatment outcomes [1].

In drug discovery, AI accelerates the identification of potential therapeutic compounds,
reducing development time from years to months [3]. Machine learning models analyze vast
molecular databases to predict drug efficacy and safety [4].

Predictive analytics powered by AI enable healthcare providers to identify at-risk patients
and intervene proactively [2][5]. This has led to reduced hospital readmissions and better
resource allocation.

## References

[1] Smith, J. (2024). AI in Medical Imaging. Retrieved from https://example.edu/ai-imaging
[2] Unknown Author. (n.d.). Healthcare AI Applications. Retrieved from https://healthcare-ai.gov
[3] Johnson, M. (2024). AI Drug Discovery Breakthrough. Retrieved from https://research.org/drugs
[4] Williams, K. (2024). Machine Learning in Pharma. Retrieved from https://pharma-ml.edu
[5] Brown, L. (2024). Predictive Healthcare Analytics. Retrieved from https://health-analytics.org

## Source Credibility Assessment

[1] AI in Medical Imaging - Credibility: High
[2] Healthcare AI Applications - Credibility: High
[3] AI Drug Discovery Breakthrough - Credibility: Medium
[4] Machine Learning in Pharma - Credibility: High
[5] Predictive Healthcare Analytics - Credibility: Medium
```

## Bonus Challenges

1. **Advanced Source Validation**:
   - Verify author credentials
   - Check publication reputation
   - Cross-reference facts
   - Detect bias

2. **Fact Verification**:
   - Compare claims across sources
   - Identify contradictions
   - Confidence scoring
   - Evidence strength assessment

3. **Deep Research**:
   - Follow citation chains
   - Access academic databases
   - Extract data from PDFs
   - Analyze figures and tables

4. **Interactive Research**:
   - Answer follow-up questions
   - Drill down into topics
   - Compare perspectives
   - Explore related topics

5. **Visualization**:
   - Generate charts and graphs
   - Create concept maps
   - Timeline visualizations
   - Network diagrams

6. **Collaboration**:
   - Share research projects
   - Collaborative note-taking
   - Version control for reports
   - Team research coordination

7. **Export Options**:
   - PDF generation
   - LaTeX export
   - DOCX format
   - Presentation slides

## Resources

### Documentation
- [LangChain Research Agents](https://python.langchain.com/docs/use_cases/research_assistant/)
- [Citation Formats](https://owl.purdue.edu/owl/research_and_citation/resources.html)

### Tools
- [Zotero](https://www.zotero.org/) - Reference management
- [Perplexity AI](https://www.perplexity.ai/) - AI research tool
- [Elicit](https://elicit.org/) - Research assistant

### Research
- [WebGPT](https://arxiv.org/abs/2112.09332) - Web browsing with citations
- [PEER](https://arxiv.org/abs/2208.11663) - Collaborative research

## Success Criteria

- [ ] Agent formulates good research questions
- [ ] Multiple sources are consulted
- [ ] Citations are accurate and formatted
- [ ] Sources are ranked by credibility
- [ ] Synthesis is coherent and comprehensive
- [ ] Bibliography is complete
- [ ] Inline citations match references
- [ ] Report is well-structured
- [ ] Conflicting information is noted
- [ ] Code is modular and extensible

## Testing Checklist

- [ ] Test with various topics
- [ ] Test different citation styles
- [ ] Verify citation accuracy
- [ ] Check source credibility scoring
- [ ] Test synthesis quality
- [ ] Verify bibliography completeness
- [ ] Test report generation
- [ ] Check for hallucinations
- [ ] Test with controversial topics
- [ ] Verify error handling

## Next Steps

After completing this project:
1. Move on to Project 10: Customer Support Agent
2. Add academic database access
3. Implement fact-checking system
4. Build web interface
5. Add PDF parsing capabilities
