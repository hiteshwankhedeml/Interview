# 🟢 Code - BTP



* &#x20;

````python
"""
JD Generation — single-file FastAPI for SAP BTP (and local).

Copy this file anywhere with the same pip dependencies, set .env (or env vars), run:
  pip install python-dotenv tavily-python langgraph langchain langchain-core langchain-openai \
    fastapi uvicorn python-multipart httpx "PyJWT[cryptography]" certifi urllib3 requests pydantic
  python sap_btp.py
  # or: uvicorn sap_btp:app --host 0.0.0.0 --port 8000

Env: OPENAI_API_KEY, TAVILY_API_KEY, optional OPENAI_MODEL, MAX_ITERATIONS, ...
     IAS_ISSUER, IAS_AUDIENCE (or IAS_CLIENT_ID), BTP_SKIP_AUTH=true for local without JWT

Regenerate from modular `agents/` + `graph/`: run `python _assemble_sap_btp_monolith.py` in this repo.
"""
from __future__ import annotations

import json
import logging
import os
import re
import ssl
import time
import urllib3
from dataclasses import dataclass
from datetime import datetime
from typing import Annotated, Any, Dict, List, Literal, Optional, TypedDict

import certifi
import httpx
import jwt
import requests
from dotenv import load_dotenv
from fastapi import Depends, FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from jwt import PyJWKClient
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langgraph.graph import END, StateGraph
from pydantic import BaseModel, Field
from tavily import TavilyClient

load_dotenv()

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# =============================================================================
# CONFIG
# =============================================================================


class Config:
    OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
    OPENAI_MODEL = os.getenv("OPENAI_MODEL", "gpt-4o")
    TAVILY_API_KEY = os.getenv("TAVILY_API_KEY")
    MAX_JD_COUNT = int(os.getenv("MAX_JD_COUNT", "10"))
    DEFAULT_TEMPERATURE = float(os.getenv("DEFAULT_TEMPERATURE", "0.7"))
    REQUEST_TIMEOUT = int(os.getenv("REQUEST_TIMEOUT", "60"))
    TAVILY_TIMEOUT = int(os.getenv("TAVILY_TIMEOUT", "30"))
    MAX_ITERATIONS = int(os.getenv("MAX_ITERATIONS", "3"))


# =============================================================================
# AGENTS & GRAPH (inlined)
# =============================================================================


class ResearchAgent:
    """
    JD research via Tavily Search.

    Tavily optional domain filters (not passed below — open search across the web):
    - include_domains: list[str] — restrict results to these hosts only, e.g.
      ["naukri.com", "linkedin.com", "monster.com"]. Keep the list short for best results.
    - exclude_domains: list[str] — drop results from these hosts.

    Pass them as extra kwargs to TavilyClient.search(), e.g.:
        self.tavily_client.search(..., include_domains=["naukri.com", "linkedin.com"])
    See: https://docs.tavily.com/documentation/api-reference/endpoint/search
    """

    def __init__(self):
        # Fix SSL certificate verification issue on Windows
        self._setup_ssl()
        self.tavily_client = TavilyClient(api_key=Config.TAVILY_API_KEY)
        self.max_results = Config.MAX_JD_COUNT
    
    def _setup_ssl(self):
        """Setup SSL context to handle certificate verification"""
        try:
            # Try to use certifi certificates for proper SSL verification
            cert_path = certifi.where()
            os.environ['REQUESTS_CA_BUNDLE'] = cert_path
            os.environ['SSL_CERT_FILE'] = cert_path
            os.environ['CURL_CA_BUNDLE'] = cert_path
        except Exception:
            pass
        
        # Disable SSL verification warnings
        urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
        
        # For Windows SSL certificate issues, create unverified SSL context
        # WARNING: This disables SSL verification - use only for development
        try:
            ssl._create_default_https_context = ssl._create_unverified_context
        except Exception:
            pass
        
        # Also patch requests to not verify SSL and add timeout (for Tavily client)
        # This is a workaround for Windows SSL certificate verification issues
        original_request = requests.Session.request
        
        def patched_request(self, *args, **kwargs):
            kwargs.setdefault('verify', False)
            # Add timeout if not already specified
            if 'timeout' not in kwargs:
                kwargs['timeout'] = Config.TAVILY_TIMEOUT
            return original_request(self, *args, **kwargs)
        
        requests.Session.request = patched_request
    
    def build_search_query(self, user_inputs: Dict) -> str:
        """Build search query from user inputs"""
        query_parts = []
        
        if user_inputs.get("job_title"):
            query_parts.append(f"Job Description for {user_inputs['job_title']}")
        
        if user_inputs.get("functional_area"):
            query_parts.append(f"in {user_inputs['functional_area']}")
        
        if user_inputs.get("skills"):
            skills = user_inputs["skills"]
            if isinstance(skills, str):
                query_parts.append(f"skills: {skills}")
        
        if user_inputs.get("additional_input"):
            query_parts.append(user_inputs["additional_input"])
        
        query = " ".join(query_parts)
        return query if query else "job description"
    
    def search_jds(self, user_inputs: Dict, stream_callback=None) -> List[Dict]:
        """Search for JDs using Tavily (no include_domains / exclude_domains — see class docstring)."""
        query = self.build_search_query(user_inputs)
        
        # Get max_results from user_inputs if provided, otherwise use config default
        max_results = user_inputs.get("num_jds", self.max_results)
        
        if stream_callback:
            stream_callback(f"🔍 Research Agent: Searching for JDs with query: '{query}'")
            stream_callback(f"📊 Research Agent: Target: {max_results} JDs")
            #stream_callback(f"⏱️ Research Agent: Timeout set to {Config.TAVILY_TIMEOUT} seconds")
        
        try:
            # Tavily search — timeout via patched_request on requests.
            # Domain scoping: add include_domains=["..."] and/or exclude_domains=["..."] here
            # when you want only job boards (e.g. naukri.com, linkedin.com). Omit = web-wide.
            response = self.tavily_client.search(
                query=query,
                search_depth="advanced",
                max_results=max_results,
                include_answer=False,
                include_raw_content=True,
            )
            
            jds = []
            for result in response.get("results", []):
                jd_data = {
                    "title": result.get("title", ""),
                    "url": result.get("url", ""),
                    "content": result.get("content", ""),
                    "raw_content": result.get("raw_content", "")
                }
                jds.append(jd_data)
                
                if stream_callback:
                    stream_callback(f"✅ Research Agent: Found JD - {jd_data['title']}")
            
            if stream_callback:
                stream_callback(f"🎯 Research Agent: Retrieved {len(jds)} JDs successfully")
            
            return jds
            
        except requests.exceptions.Timeout as e:
            error_msg = f"❌ Research Agent: Request timed out after {Config.TAVILY_TIMEOUT} seconds. Please try again."
            if stream_callback:
                stream_callback(error_msg)
            raise Exception(error_msg)
        except requests.exceptions.RequestException as e:
            error_msg = f"❌ Research Agent: Network error - {str(e)}"
            if stream_callback:
                stream_callback(error_msg)
            raise Exception(error_msg)
        except Exception as e:
            error_msg = f"❌ Research Agent: Error searching JDs - {str(e)}"
            if stream_callback:
                stream_callback(error_msg)
            raise Exception(error_msg)


class AnalysisAgent:
    def __init__(self):
        # Validate API key
        if not Config.OPENAI_API_KEY or Config.OPENAI_API_KEY == "your-openai-api-key-here":
            raise ValueError(
                "OpenAI API key is not configured. Please set OPENAI_API_KEY in your .env file."
            )
        
        self.llm = ChatOpenAI(
            api_key=Config.OPENAI_API_KEY,
            model=Config.OPENAI_MODEL,
            temperature=0.3,  # Lower temperature for more focused analysis
            timeout=Config.REQUEST_TIMEOUT
        )
    
    def analyze_jds(self, retrieved_jds: List[Dict], user_inputs: Dict, stream_callback=None) -> Dict:
        """Analyze retrieved JDs to extract patterns and market standards"""
        
        if stream_callback:
            stream_callback("📊 Analysis Agent: Starting analysis of retrieved JDs...")
        
        if not retrieved_jds:
            if stream_callback:
                stream_callback("⚠️ Analysis Agent: No JDs to analyze")
            return {
                "education_patterns": "",
                "experience_patterns": "",
                "skills_patterns": "",
                "responsibilities_patterns": "",
                "market_insights": ""
            }
        
        # Prepare JD content for analysis
        jd_content = self._prepare_jd_content_for_analysis(retrieved_jds)
        
        if stream_callback:
            stream_callback(f"📊 Analysis Agent: Analyzing {len(retrieved_jds)} JDs for patterns...")
        
        # Build analysis prompt
        prompt = self._build_analysis_prompt(user_inputs, jd_content)
        
        try:
            if stream_callback:
                stream_callback("🔍 Analysis Agent: Extracting patterns (Education, Experience, Skills, Responsibilities)...")
            
            # Get analysis from LLM
            response = self.llm.invoke(prompt)
            analysis_content = response.content
            
            if stream_callback:
                stream_callback("✅ Analysis Agent: Pattern extraction completed")
            
            # Parse analysis into structured format
            parsed_analysis = self._parse_analysis(analysis_content)
            
            return parsed_analysis
            
        except Exception as e:
            error_msg = f"❌ Analysis Agent: Error analyzing JDs - {str(e)}"
            if stream_callback:
                stream_callback(error_msg)
            raise Exception(error_msg)
    
    def _prepare_jd_content_for_analysis(self, retrieved_jds: List[Dict]) -> str:
        """Prepare JD content for analysis"""
        content_parts = []
        for i, jd in enumerate(retrieved_jds, 1):
            content_parts.append(f"JD {i}:\n")
            content_parts.append(f"Title: {jd.get('title', 'N/A')}\n")
            # Use raw_content if available, otherwise content
            jd_text = jd.get('raw_content', '') or jd.get('content', '')
            # Limit to first 2000 characters per JD to avoid token limits
            content_parts.append(f"Content: {jd_text[:2000]}...\n")
            content_parts.append("---\n")
        return "\n".join(content_parts)
    
    def _build_analysis_prompt(self, user_inputs: Dict, jd_content: str) -> str:
        """Build the analysis prompt"""
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are an expert Job Market Analyst. Your task is to analyze multiple job descriptions 
            and extract common patterns, market standards, and trends.
            
            Analyze the provided JDs and extract:
            1. Education patterns - common degrees, certifications, qualifications
            2. Experience patterns - typical years of experience, level expectations
            3. Skills patterns - frequently mentioned skills (technical and soft skills)
            4. Responsibilities patterns - common duties and key responsibilities
            
            IMPORTANT: Provide a CLEAN, STRUCTURED output with NO DUPLICATES. Each section should appear only ONCE.
            Use clear section headers and concise bullet points. Avoid repeating the same information multiple times."""),
            ("human", """Job Title to Analyze: {job_title}
Functional Area: {functional_area}

Retrieved Job Descriptions:
{jd_content}

Analyze these JDs and extract patterns. Format your response with clear sections:

## Education Requirements
[List common degrees, certifications, qualifications - NO DUPLICATES]

## Experience Requirements
[Typical years, level expectations - NO DUPLICATES]

## Skills Requirements
[Technical and soft skills - NO DUPLICATES]

## Roles and Responsibilities
[Common duties and key responsibilities - NO DUPLICATES]

Provide a structured analysis that highlights market standards and common patterns. 
Each section should appear only once with clear, non-repetitive content."""),
        ])
        
        return prompt_template.format_messages(
            job_title=user_inputs.get("job_title", ""),
            functional_area=user_inputs.get("functional_area", ""),
            jd_content=jd_content
        )
    
    def _parse_analysis(self, analysis_content: str) -> Dict:
        """Parse analysis content into structured format"""
        analysis = {
            "education_patterns": "",
            "experience_patterns": "",
            "skills_patterns": "",
            "responsibilities_patterns": "",
            "market_insights": ""
        }
        
        # More robust parsing - look for clear section markers
        lines = analysis_content.split("\n")
        current_section = None
        seen_lines = set()  # Track seen lines to avoid duplicates
        
        # More specific section patterns (including markdown headers)
        section_patterns = {
            "education_patterns": [
                r"^##\s*education",
                r"^\d+\.?\s*education",
                r"education\s*requirements?",
                r"education\s*patterns?",
                r"common\s*degrees?",
                r"qualifications?",
                r"certifications?"
            ],
            "experience_patterns": [
                r"^##\s*experience",
                r"^\d+\.?\s*experience",
                r"experience\s*requirements?",
                r"experience\s*patterns?",
                r"typical\s*years?",
                r"level\s*expectations?"
            ],
            "skills_patterns": [
                r"^##\s*skills?",
                r"^\d+\.?\s*skills?",
                r"skills?\s*requirements?",
                r"skills?\s*patterns?",
                r"technical\s*skills?",
                r"soft\s*skills?",
                r"competencies?"
            ],
            "responsibilities_patterns": [
                r"^##\s*roles?\s*and\s*responsibilities?",
                r"^##\s*responsibilities?",
                r"^\d+\.?\s*roles?\s*and\s*responsibilities?",
                r"^\d+\.?\s*responsibilities?",
                r"responsibilities?\s*patterns?",
                r"common\s*duties?",
                r"key\s*responsibilities?"
            ]
        }
        
        for line in lines:
            line_stripped = line.strip()
            if not line_stripped:
                continue
            
            # Check if this line is a section header (must be at start of line or have markdown header)
            line_lower = line_stripped.lower()
            section_found = False
            
            # Check if this looks like a section header (short line, contains section keywords)
            is_likely_header = len(line_stripped) < 100 and (
                line_stripped.startswith("##") or 
                line_stripped.startswith("#") or
                re.match(r"^\d+\.?\s+", line_stripped) or
                any(keyword in line_lower for keyword in ["requirements", "patterns", "common", "typical", "key"])
            )
            
            if is_likely_header:
                for section_name, patterns in section_patterns.items():
                    for pattern in patterns:
                        if re.search(pattern, line_lower):
                            # Only switch section if it's different from current
                            if current_section != section_name:
                                current_section = section_name
                                # Add header only once
                                if line_stripped not in seen_lines:
                                    analysis[current_section] += line_stripped + "\n"
                                    seen_lines.add(line_stripped)
                                section_found = True
                                break
                    if section_found:
                        break
            
            # If not a section header, add to current section
            if not section_found and current_section:
                # Avoid adding duplicate lines (use normalized version)
                normalized_line = re.sub(r'\s+', ' ', line_stripped.lower())
                if normalized_line not in seen_lines:
                    analysis[current_section] += line_stripped + "\n"
                    seen_lines.add(normalized_line)
        
        # Clean up each section - remove duplicates, headers, and format properly
        for section_name in analysis:
            if analysis[section_name]:
                # Remove duplicate lines while preserving order
                lines = analysis[section_name].split("\n")
                seen = set()
                unique_lines = []
                for line in lines:
                    line_stripped = line.strip()
                    if not line_stripped:
                        continue
                    
                    # Skip markdown headers (##, ###, ####, #)
                    if re.match(r'^#{1,6}\s+', line_stripped):
                        continue
                    
                    # Skip section headers that are redundant (like "Education Requirements" when we already have "Education Patterns" as header)
                    line_lower = line_stripped.lower()
                    if any(keyword in line_lower for keyword in [
                        "education requirements", "education patterns",
                        "experience requirements", "experience patterns",
                        "skills requirements", "skills patterns",
                        "responsibilities requirements", "responsibilities patterns", "roles and responsibilities"
                    ]) and len(line_stripped) < 100:
                        # Skip if it's a short header line
                        continue
                    
                    if line_stripped not in seen:
                        unique_lines.append(line_stripped)
                        seen.add(line_stripped)
                
                # Join lines with proper spacing
                analysis[section_name] = "\n".join(unique_lines)
        
        # If no sections were parsed, use full content as market insights
        if not any(analysis[key] for key in ["education_patterns", "experience_patterns", "skills_patterns", "responsibilities_patterns"]):
            analysis["market_insights"] = analysis_content
        else:
            # Combine all sections for market insights
            insights_parts = []
            for key in ["education_patterns", "experience_patterns", "skills_patterns", "responsibilities_patterns"]:
                if analysis[key]:
                    insights_parts.append(analysis[key])
            analysis["market_insights"] = "\n\n".join(insights_parts)
        
        return analysis



class JDGenerationAgent:
    def __init__(self):
        # Validate API key
        if not Config.OPENAI_API_KEY or Config.OPENAI_API_KEY == "your-openai-api-key-here":
            raise ValueError(
                "OpenAI API key is not configured. Please set OPENAI_API_KEY in your .env file."
            )
        
        self.llm = ChatOpenAI(
            api_key=Config.OPENAI_API_KEY,
            model=Config.OPENAI_MODEL,
            temperature=Config.DEFAULT_TEMPERATURE,
            timeout=Config.REQUEST_TIMEOUT,
            streaming=True  # Enable streaming
        )
        self.streaming_callback = None
    
    def generate_jd(self, user_inputs: Dict, retrieved_jds: List[Dict], analysis_results: Dict = None, stream_callback=None, streaming_jd_callback=None) -> Dict:
        """Generate JD based on user inputs, retrieved JDs, and analysis results"""
        
        # Get temperature from user_inputs if provided
        temperature = user_inputs.get("temperature", Config.DEFAULT_TEMPERATURE)
        
        # Update LLM temperature if different from current
        if self.llm.temperature != temperature:
            self.llm.temperature = temperature
        
        if stream_callback:
            stream_callback("📝 JD Generation Agent: Starting JD generation...")
        
        # Prepare context from retrieved JDs
        jd_context = self._prepare_jd_context(retrieved_jds)
        
        # Build prompt with analysis results
        prompt = self._build_generation_prompt(user_inputs, jd_context, analysis_results)
        
        if stream_callback:
            stream_callback("🤖 JD Generation Agent: Analyzing market standards and generating JD...")
        
        try:
            # Generate JD sections with streaming
            if stream_callback:
                stream_callback("📝 JD Generation Agent: Generating JD (streaming)...")
            
            generated_content = ""
            full_response = ""
            token_buffer = ""  # Buffer to accumulate tokens
            buffer_size = 3  # Number of tokens to accumulate before displaying
            token_count = 0
            
            import time
            
            # Stream the response token by token
            for chunk in self.llm.stream(prompt):
                if hasattr(chunk, 'content'):
                    token = chunk.content
                    generated_content += token
                    full_response += token
                    token_buffer += token
                    token_count += 1
                    
                    # Display accumulated tokens with delay for typewriter effect
                    if streaming_jd_callback and (token_count % buffer_size == 0 or not token):
                        streaming_jd_callback(full_response)
                        time.sleep(0.05)  # Small delay (50ms) for smooth typewriter effect
                        token_buffer = ""
                    
                    # Update stream callback with progress updates
                    if stream_callback:
                        # Only show updates every few tokens to avoid too many updates
                        if len(generated_content) % 50 == 0 or len(generated_content) < 100:
                            stream_callback(f"✍️ Generating... ({len(generated_content)} characters)")
            
            # Display any remaining buffered content
            if streaming_jd_callback and token_buffer:
                streaming_jd_callback(full_response)
            
            if stream_callback:
                stream_callback("✅ JD Generation Agent: JD generation completed")
            
            # Parse the generated content into sections
            parsed_jd = self._parse_jd_sections(full_response, user_inputs)
            
            return parsed_jd
            
        except Exception as e:
            error_str = str(e)
            
            # Provide helpful error messages for common issues
            if "401" in error_str or "insufficient permissions" in error_str.lower():
                error_msg = (
                    f"❌ JD Generation Agent: Authentication Error\n"
                    f"Your OpenAI API key doesn't have the required permissions.\n"
                    f"Please check:\n"
                    f"1. Your API key is valid and active\n"
                    f"2. Your API key has 'model.request' scope enabled\n"
                    f"3. If using a restricted key, ensure it has necessary scopes\n"
                    f"4. Your account has access to {Config.OPENAI_MODEL}\n"
                    f"Original error: {error_str}"
                )
            elif "429" in error_str or "rate limit" in error_str.lower():
                error_msg = (
                    f"❌ JD Generation Agent: Rate Limit Error\n"
                    f"You've exceeded your OpenAI API rate limit. Please try again later.\n"
                    f"Original error: {error_str}"
                )
            elif "invalid_api_key" in error_str.lower() or "authentication" in error_str.lower():
                error_msg = (
                    f"❌ JD Generation Agent: Invalid API Key\n"
                    f"Please check your OPENAI_API_KEY in the .env file.\n"
                    f"Original error: {error_str}"
                )
            else:
                error_msg = f"❌ JD Generation Agent: Error generating JD - {error_str}"
            
            if stream_callback:
                stream_callback(error_msg)
            raise Exception(error_msg)
    
    def self_reflect(self, generated_jd: Dict, user_inputs: Dict, stream_callback=None) -> Dict:
        """Agent reflects on its own output before passing to validation"""
        if stream_callback:
            stream_callback("🤔 JD Generation Agent: Self-reflecting on generated JD...")
        
        jd_content = generated_jd.get("full_content", "")
        
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are the JD Generation Agent. You just generated a Job Description. 
            Before passing it to validation, reflect on your own work.
            
            Consider:
            1. Did you meet all user requirements?
            2. Are there any obvious issues you notice?
            3. What could be improved?
            4. Did you follow the instructions correctly?
            5. Is the JD complete and professional?
            
            Be honest and critical. This self-reflection helps improve quality."""),
            ("human", """User Requirements:
Job Title: {job_title}
Education: {education}
Experience: {experience}
Functional Area: {functional_area}
Skills: {skills}
Additional Input: {additional_input}

Generated JD:
{jd_content}

Reflect on your work. What did you do well? What could be better? Should you revise anything?"""),
        ])
        
        try:
            prompt = prompt_template.format_messages(
                job_title=user_inputs.get("job_title", ""),
                education=user_inputs.get("education", ""),
                experience=user_inputs.get("experience", ""),
                functional_area=user_inputs.get("functional_area", ""),
                skills=user_inputs.get("skills", ""),
                additional_input=user_inputs.get("additional_input", ""),
                jd_content=jd_content
            )
            
            reflection_response = self.llm.invoke(prompt)
            reflection_content = reflection_response.content
            
            if stream_callback:
                stream_callback(f"💭 Self-Reflection: {reflection_content[:150]}...")
            
            return {
                "self_assessment": reflection_content,
                "should_revise": self._parse_revision_decision(reflection_content)
            }
            
        except Exception as e:
            if stream_callback:
                stream_callback(f"⚠️ Self-reflection error: {str(e)}")
            return {
                "self_assessment": "Self-reflection unavailable",
                "should_revise": False
            }
    
    def _parse_revision_decision(self, reflection_content: str) -> bool:
        """Parse if agent thinks revision is needed"""
        reflection_lower = reflection_content.lower()
        revision_keywords = ["revise", "improve", "fix", "change", "better", "issue", "problem", "missing"]
        no_revision_keywords = ["good", "complete", "satisfactory", "meets", "adequate"]
        
        revision_score = sum(1 for keyword in revision_keywords if keyword in reflection_lower)
        no_revision_score = sum(1 for keyword in no_revision_keywords if keyword in reflection_lower)
        
        return revision_score > no_revision_score
    
    def _prepare_jd_context(self, retrieved_jds: List[Dict]) -> str:
        """Prepare context string from retrieved JDs"""
        context_parts = []
        for i, jd in enumerate(retrieved_jds[:5], 1):  # Use top 5 for context
            context_parts.append(f"JD {i}:\nTitle: {jd.get('title', 'N/A')}\n")
            context_parts.append(f"Content: {jd.get('content', '')[:1000]}...\n")
            context_parts.append("---\n")
        return "\n".join(context_parts)
    
    def _build_generation_prompt(self, user_inputs: Dict, jd_context: str, analysis_results: Dict = None) -> str:
        """Build the generation prompt"""
        # Prepare analysis context
        analysis_context = ""
        if analysis_results:
            analysis_context = f"""
Market Analysis Insights:
Education Patterns: {analysis_results.get('education_patterns', 'N/A')}
Experience Patterns: {analysis_results.get('experience_patterns', 'N/A')}
Skills Patterns: {analysis_results.get('skills_patterns', 'N/A')}
Responsibilities Patterns: {analysis_results.get('responsibilities_patterns', 'N/A')}
"""
        
        # Check if this is a regeneration with validation feedback
        validation_feedback = user_inputs.get("validation_feedback", {})
        adapted_goals = user_inputs.get("adapted_goals", {})
        
        feedback_context = ""
        if validation_feedback:
            suggestions = validation_feedback.get("suggestions", [])
            issues = validation_feedback.get("issues", [])
            quality_score = validation_feedback.get("quality_score", 0)
            
            feedback_context = f"""

IMPORTANT - Validation Feedback (Previous Attempt):
Quality Score: {quality_score}/100

Issues Found:
{chr(10).join(['- ' + issue for issue in issues]) if issues else 'None'}

Suggestions for Improvement:
{chr(10).join(['- ' + suggestion for suggestion in suggestions]) if suggestions else 'None'}

Please address these issues and incorporate the suggestions in this regeneration.
Make sure to improve the JD based on the feedback above.
"""
        
        # Add adapted goals context if available
        goals_context = ""
        if adapted_goals and adapted_goals.get("adapted_goals"):
            priorities = adapted_goals.get("priorities", [])
            focus_areas = adapted_goals.get("adapted_goals", {}).get("focus_areas", [])
            reasoning = adapted_goals.get("reasoning", "")
            
            goals_context = f"""

ADAPTED GOALS FOR THIS ITERATION:
Priorities: {', '.join(priorities) if priorities else 'Standard priorities'}
Focus Areas: {', '.join(focus_areas) if focus_areas else 'All areas'}

Goal Adaptation Reasoning:
{reasoning[:300] if reasoning else 'No specific adaptation'}

Focus on these priorities and areas when generating the JD.
"""
        
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are an expert Job Description writer. Your task is to create a comprehensive, 
            market-standard Job Description based on user inputs, company context, market analysis insights, and similar JDs from the market.
            
            CRITICAL INSTRUCTIONS:
            - Output ONLY the Job Description sections. Do NOT include any commentary, explanations, or meta-text.
            - Do NOT include phrases like "This job description...", "The following JD...", or any introductory/explanatory text.
            - Start directly with the section headers and content.
            - Generate ONLY these sections in order:
            1. Education
            2. Experience
            3. Roles and Responsibilities
            4. Skills
            
            - When company context is provided, tailor the JD to align with the company's culture, values, industry focus, and work environment.
            - Use market analysis insights to ensure the JD aligns with current market standards and common patterns.
            - Make sure the JD follows current market standards and best practices.
            - Expand on user inputs based on what you see in similar market JDs and analysis insights.
            - The JD should be professional, clear, and comprehensive.
            - Use clear section headers like "## Education", "## Experience", etc.
            - If validation feedback is provided, carefully address all issues and incorporate the suggestions to improve the JD."""),
            ("human", """User Requirements:
Job Title: {job_title}
Education: {education}
Experience: {experience}
Functional Area: {functional_area}
Skills: {skills}
Additional Input: {additional_input}

Company Context:
{company_context}

{analysis_context}

{feedback_context}

{goals_context}

Market JDs Reference:
{jd_context}

Generate a Job Description with ONLY these sections:
## Education
[Education requirements]

## Experience
[Experience requirements]

## Roles and Responsibilities
[Key responsibilities]

## Skills
[Required skills]

Output ONLY the JD sections. No additional text, commentary, or explanations."""),
        ])
        
        return prompt_template.format_messages(
            job_title=user_inputs.get("job_title", ""),
            education=user_inputs.get("education", ""),
            experience=user_inputs.get("experience", ""),
            functional_area=user_inputs.get("functional_area", ""),
            skills=user_inputs.get("skills", ""),
            additional_input=user_inputs.get("additional_input", ""),
            company_context=user_inputs.get("company_context", ""),
            analysis_context=analysis_context,
            feedback_context=feedback_context,
            goals_context=goals_context,
            jd_context=jd_context
        )
    
    def _parse_jd_sections(self, generated_content: str, user_inputs: Dict) -> Dict:
        """Parse generated content into structured sections"""
        # Remove meta-commentary and unwanted text
        content = self._clean_generated_content(generated_content)
        
        sections = {
            "Education": "",
            "Experience": "",
            "Roles and Responsibilities": "",
            "Skills": ""
        }
        
        content_lower = content.lower()
        
        # Section keywords for detection
        section_keywords = {
            "Education": ["education", "qualification", "degree", "educational"],
            "Experience": ["experience", "years of experience", "work experience", "professional experience"],
            "Roles and Responsibilities": ["roles", "responsibilities", "duties", "key responsibilities", "role and responsibilities"],
            "Skills": ["skills", "technical skills", "competencies", "required skills"]
        }
        
        lines = content.split("\n")
        current_section = None
        
        for line in lines:
            line_lower = line.lower().strip()
            
            # Skip meta-commentary lines
            if self._is_meta_commentary(line):
                continue
            
            # Check if this line is a section header
            for section_name, keywords in section_keywords.items():
                # Check for markdown headers (##, ###) or plain text headers
                is_header = (
                    line.strip().startswith("##") or 
                    line.strip().startswith("#") or
                    (any(keyword in line_lower for keyword in keywords) and len(line) < 100)
                )
                
                if is_header:
                    # Verify it's actually a section header by checking keywords
                    if any(keyword in line_lower for keyword in keywords):
                        current_section = section_name
                        # Add header without markdown if present
                        clean_header = line.replace("#", "").strip()
                        if clean_header:
                            sections[current_section] += clean_header + "\n"
                        break
            
            # Add content to current section
            if current_section and line.strip() and not line.strip().startswith("#"):
                sections[current_section] += line + "\n"
        
        # Clean up sections - remove empty lines at start/end
        for section_name in sections:
            sections[section_name] = sections[section_name].strip()
        
        # Fallback: if parsing failed, return cleaned raw content
        if not any(sections.values()):
            sections["Roles and Responsibilities"] = content
        
        return {
            "full_content": content,
            "sections": sections,
            "job_title": user_inputs.get("job_title", "")
        }
    
    def _clean_generated_content(self, content: str) -> str:
        """Remove meta-commentary and unwanted text from generated content"""
        lines = content.split("\n")
        cleaned_lines = []
        
        # Phrases to remove (meta-commentary)
        meta_phrases = [
            "this comprehensive job description",
            "this job description",
            "the following job description",
            "the following jd",
            "this jd",
            "reflects the current market standards",
            "is designed to attract",
            "note:",
            "important:",
            "please note:"
        ]
        
        for line in lines:
            line_lower = line.lower().strip()
            # Skip lines that are clearly meta-commentary
            if any(phrase in line_lower for phrase in meta_phrases):
                # Check if it's a standalone commentary line (not part of JD content)
                if len(line.strip()) < 200:  # Likely a commentary line
                    continue
            
            cleaned_lines.append(line)
        
        return "\n".join(cleaned_lines)
    
    def _is_meta_commentary(self, line: str) -> bool:
        """Check if a line is meta-commentary"""
        line_lower = line.lower().strip()
        meta_indicators = [
            "this comprehensive",
            "this job description",
            "reflects the current",
            "is designed to",
            "note:",
            "important:"
        ]
        return any(indicator in line_lower for indicator in meta_indicators) and len(line.strip()) < 200


class ValidationAgent:
    def __init__(self):
        # Validate API key
        if not Config.OPENAI_API_KEY or Config.OPENAI_API_KEY == "your-openai-api-key-here":
            raise ValueError(
                "OpenAI API key is not configured. Please set OPENAI_API_KEY in your .env file."
            )
        
        self.llm = ChatOpenAI(
            api_key=Config.OPENAI_API_KEY,
            model=Config.OPENAI_MODEL,
            temperature=0.1,  # Even lower temperature for more critical, consistent validation
            timeout=Config.REQUEST_TIMEOUT
        )
    
    def validate_jd(self, generated_jd: Dict, user_inputs: Dict, stream_callback=None) -> Dict:
        """Validate the generated JD for quality, completeness, and alignment"""
        
        if stream_callback:
            stream_callback("✅ Validation Agent: Starting quality check...")
        
        jd_content = generated_jd.get("full_content", "")
        
        if not jd_content:
            if stream_callback:
                stream_callback("❌ Validation Agent: No JD content to validate")
            return {
                "is_valid": False,
                "quality_score": 0,
                "issues": ["No JD content generated"],
                "suggestions": [],
                "approval_status": "REJECTED"
            }
        
        # Build validation prompt
        prompt = self._build_validation_prompt(jd_content, user_inputs)
        
        try:
            if stream_callback:
                stream_callback("🔍 Validation Agent: Checking completeness, quality, and alignment...")
            
            response = self.llm.invoke(prompt)
            validation_result = self._parse_validation(response.content)
            
            if stream_callback:
                score = validation_result.get("quality_score", 0)
                status = validation_result.get("approval_status", "UNKNOWN")
                stream_callback(f"📊 Validation Agent: Quality Score: {score}/100 | Status: {status}")
            
            return validation_result
            
        except Exception as e:
            error_msg = f"❌ Validation Agent: Error during validation - {str(e)}"
            if stream_callback:
                stream_callback(error_msg)
            return {
                "is_valid": False,
                "quality_score": 0,
                "issues": [f"Validation error: {str(e)}"],
                "suggestions": [],
                "approval_status": "ERROR"
            }
    
    def self_reflect(self, validation_results: Dict, generated_jd: Dict, stream_callback=None) -> Dict:
        """Agent reflects on its own validation work"""
        if stream_callback:
            stream_callback("🤔 Validation Agent: Self-reflecting on validation assessment...")
        
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are the Validation Agent. You just validated a Job Description. 
            Reflect on your own validation work.
            
            Consider:
            1. Was your assessment fair and accurate?
            2. Did you miss any issues?
            3. Were your suggestions helpful and actionable?
            4. Was your scoring consistent with your feedback?
            5. Should you reconsider your approval status?
            
            Be honest about your own work quality."""),
            ("human", """Your Validation Results:
Quality Score: {quality_score}/100
Approval Status: {approval_status}
Issues: {issues}
Suggestions: {suggestions}

Generated JD:
{jd_content}

Reflect on your validation. Was it thorough? Fair? Could you have been more helpful?"""),
        ])
        
        try:
            prompt = prompt_template.format_messages(
                quality_score=validation_results.get("quality_score", 0),
                approval_status=validation_results.get("approval_status", "UNKNOWN"),
                issues=", ".join(validation_results.get("issues", [])),
                suggestions=", ".join(validation_results.get("suggestions", [])),
                jd_content=generated_jd.get("full_content", "")[:1000]  # Limit content
            )
            
            reflection_response = self.llm.invoke(prompt)
            reflection_content = reflection_response.content
            
            if stream_callback:
                stream_callback(f"💭 Self-Reflection: {reflection_content[:150]}...")
            
            return {
                "self_assessment": reflection_content,
                "should_adjust": self._parse_adjustment_decision(reflection_content)
            }
            
        except Exception as e:
            if stream_callback:
                stream_callback(f"⚠️ Self-reflection error: {str(e)}")
            return {
                "self_assessment": "Self-reflection unavailable",
                "should_adjust": False
            }
    
    def _parse_adjustment_decision(self, reflection_content: str) -> bool:
        """Parse if agent thinks validation should be adjusted"""
        reflection_lower = reflection_content.lower()
        adjustment_keywords = ["adjust", "reconsider", "change", "revise", "missed", "wrong", "unfair"]
        no_adjustment_keywords = ["accurate", "fair", "correct", "thorough", "appropriate"]
        
        adjustment_score = sum(1 for keyword in adjustment_keywords if keyword in reflection_lower)
        no_adjustment_score = sum(1 for keyword in no_adjustment_keywords if keyword in reflection_lower)
        
        return adjustment_score > no_adjustment_score
    
    def _build_validation_prompt(self, jd_content: str, user_inputs: Dict) -> str:
        """Build the validation prompt"""
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are a STRICT Quality Assurance expert for Job Descriptions. Your task is to critically 
            evaluate the generated JD and provide a comprehensive quality assessment with actionable suggestions.
            
            CRITICAL EVALUATION APPROACH:
            - Be THOROUGH and CRITICAL - even good JDs can be improved
            - Look for enhancement opportunities, not just missing content
            - Score conservatively: 95+ only for exceptional JDs, 80-94 for good JDs, below 80 for JDs needing work
            - Always provide at least 2-3 constructive suggestions for improvement, even for good JDs
            
            EVALUATION CRITERIA (Be Strict):
            1. Completeness (20 points):
               - All required sections present and substantial
               - Each section has meaningful content (not just 1-2 lines)
               - Education section specifies degree types, fields, certifications
               - Experience section specifies years, level, industry context
               - Skills section lists both technical and soft skills with specificity
               - Responsibilities section has detailed, actionable bullet points (at least 5-7 points)
            
            2. Quality & Clarity (20 points):
               - Professional language and tone
               - Clear, concise, and well-structured
               - No vague or generic statements
               - Specific metrics, technologies, or tools mentioned where relevant
               - Grammar and formatting are perfect
            
            3. Alignment with User Requirements (20 points):
               - Job title matches user input
               - Education requirements align with user input
               - Experience requirements match user input
               - Skills listed match user input (check if all user skills are present)
               - Company context is incorporated if provided
            
            4. Market Standards & Best Practices (20 points):
               - Follows current industry standards
               - Includes modern tools/technologies if relevant
               - Competitive and attractive to candidates
               - Includes both required and preferred qualifications
               - Mentions growth opportunities, benefits, or company culture if appropriate
            
            5. Specificity & Detail (20 points):
               - Avoids generic statements like "strong communication skills"
               - Provides specific examples or contexts
               - Quantifies requirements where possible (years, team size, etc.)
               - Includes industry-specific terminology
               - Responsibilities are detailed and measurable
            
            SCORING GUIDELINES:
            - 90-100: Exceptional JD with minimal room for improvement (still provide 1-2 enhancement suggestions)
            - 80-89: Good JD but can be enhanced (provide 2-3 specific suggestions)
            - 70-79: Acceptable JD but needs improvement (provide 3-5 suggestions)
            - Below 70: JD needs significant revision (provide 5+ suggestions)
            
            SUGGESTIONS GUIDELINES:
            - Always provide at least 2-3 suggestions, even for high-scoring JDs
            - Focus on enhancements: "Consider adding...", "Could be strengthened by...", "Would benefit from..."
            - Be specific: Instead of "add more details", say "add specific metrics like 'manage team of 10+'"
            - Check for: missing specificity, generic statements, lack of quantifiable requirements, missing modern tools/technologies
            - Verify suggestions don't duplicate existing content, but look for enhancement opportunities
            
            Provide your response in this exact JSON format:
            {{
                "quality_score": <0-100>,
                "is_valid": <true/false>,
                "approval_status": "<APPROVED/NEEDS_REVISION/REJECTED>",
                "issues": ["issue1", "issue2", ...],
                "suggestions": ["suggestion1", "suggestion2", "suggestion3", ...],
                "completeness_check": {{
                    "has_education": <true/false>,
                    "has_experience": <true/false>,
                    "has_responsibilities": <true/false>,
                    "has_skills": <true/false>
                }},
                "quality_assessment": "Brief summary of quality assessment"
            }}
            
            IMPORTANT: Always provide at least 2-3 suggestions. Look for enhancement opportunities even in good JDs."""),
            ("human", """User Requirements:
Job Title: {job_title}
Education: {education}
Experience: {experience}
Functional Area: {functional_area}
Skills: {skills}
Additional Input: {additional_input}

Generated JD:
{jd_content}

CRITICAL VALIDATION TASK:
1. Score the JD conservatively (90+ only if truly exceptional)
2. Provide at least 2-3 specific enhancement suggestions
3. Check for:
   - Generic statements that need specificity
   - Missing quantifiable requirements
   - Opportunities to add more detail
   - Areas where the JD could be more competitive
   - Missing modern tools/technologies if relevant
   - Lack of specificity in responsibilities

Validate this JD critically and provide a comprehensive quality assessment with actionable suggestions in the specified JSON format."""),
        ])
        
        return prompt_template.format_messages(
            job_title=user_inputs.get("job_title", ""),
            education=user_inputs.get("education", ""),
            experience=user_inputs.get("experience", ""),
            functional_area=user_inputs.get("functional_area", ""),
            skills=user_inputs.get("skills", ""),
            additional_input=user_inputs.get("additional_input", ""),
            jd_content=jd_content
        )
    
    def _parse_validation(self, validation_content: str) -> Dict:
        """Parse validation response"""
        import json
        import re
        
        # Try to extract JSON from response - improved parsing for nested JSON
        try:
            # First, try to find JSON code blocks (```json ... ```)
            json_block_match = re.search(r'```(?:json)?\s*(\{.*?\})\s*```', validation_content, re.DOTALL)
            if json_block_match:
                json_str = json_block_match.group(1)
                parsed = json.loads(json_str)
                return {
                    "quality_score": parsed.get("quality_score", 0),
                    "is_valid": parsed.get("is_valid", False),
                    "approval_status": parsed.get("approval_status", "UNKNOWN"),
                    "issues": parsed.get("issues", []),
                    "suggestions": parsed.get("suggestions", []),
                    "completeness_check": parsed.get("completeness_check", {}),
                    "quality_assessment": parsed.get("quality_assessment", "")
                }
            
            # Try to find JSON object using balanced braces
            # Count braces to find complete JSON object
            start_idx = validation_content.find('{')
            if start_idx != -1:
                brace_count = 0
                end_idx = start_idx
                for i in range(start_idx, len(validation_content)):
                    if validation_content[i] == '{':
                        brace_count += 1
                    elif validation_content[i] == '}':
                        brace_count -= 1
                        if brace_count == 0:
                            end_idx = i + 1
                            break
                
                if end_idx > start_idx:
                    json_str = validation_content[start_idx:end_idx]
                    parsed = json.loads(json_str)
                    return {
                        "quality_score": parsed.get("quality_score", 0),
                        "is_valid": parsed.get("is_valid", False),
                        "approval_status": parsed.get("approval_status", "UNKNOWN"),
                        "issues": parsed.get("issues", []),
                        "suggestions": parsed.get("suggestions", []),
                        "completeness_check": parsed.get("completeness_check", {}),
                        "quality_assessment": parsed.get("quality_assessment", "")
                    }
        except Exception as e:
            # Log error for debugging (can be removed in production)
            print(f"JSON parsing error: {str(e)}")
            print(f"Content preview: {validation_content[:500]}")
        
        # Fallback: Basic validation
        return {
            "quality_score": 70,
            "is_valid": True,
            "approval_status": "APPROVED",
            "issues": [],
            "suggestions": [],
            "completeness_check": {},
            "quality_assessment": "Basic validation completed"
        }



class ReasoningAgent:
    """Agent that reasons about next steps using LLM-based decision making"""
    
    def __init__(self):
        # Validate API key
        if not Config.OPENAI_API_KEY or Config.OPENAI_API_KEY == "your-openai-api-key-here":
            raise ValueError(
                "OpenAI API key is not configured. Please set OPENAI_API_KEY in your .env file."
            )
        
        self.llm = ChatOpenAI(
            api_key=Config.OPENAI_API_KEY,
            model=Config.OPENAI_MODEL,
            temperature=0.3,  # Moderate temperature for reasoning
            timeout=Config.REQUEST_TIMEOUT
        )
        self.max_iterations = Config.MAX_ITERATIONS
    
    def reason_next_action_with_output(self, state: Dict, stream_callback=None) -> tuple[Literal["regenerate", "approve", "reject"], str]:
        """Use LLM to reason about what to do next and return both decision and reasoning content"""
        
        validation_results = state.get("validation_results", {})
        iteration_count = state.get("iteration_count", 0)
        quality_score = validation_results.get("quality_score", 0)
        approval_status = validation_results.get("approval_status", "UNKNOWN")
        issues = validation_results.get("issues", [])
        suggestions = validation_results.get("suggestions", [])
        agent_discussion = state.get("agent_discussion", {})
        adapted_goals = state.get("adapted_goals", {})
        
        if stream_callback:
            stream_callback("🧠 Reasoning Agent: Analyzing situation and reasoning about next action...")
        
        prompt = self._build_reasoning_prompt(
            quality_score, approval_status, iteration_count, issues, suggestions,
            agent_discussion, adapted_goals
        )
        
        try:
            response = self.llm.invoke(prompt)
            decision = self._parse_decision(response.content, stream_callback)
            reasoning_content = response.content
            
            if stream_callback:
                stream_callback(f"🤖 Reasoning Agent Decision: {decision.upper()}")
                stream_callback(f"💭 Reasoning: {reasoning_content[:200]}...")
            
            return decision, reasoning_content
            
        except Exception as e:
            # Fallback to rule-based if LLM fails
            if stream_callback:
                stream_callback(f"⚠️ Reasoning Agent: Error, using fallback logic - {str(e)}")
            fallback_decision = self._fallback_decision(approval_status, iteration_count, stream_callback)
            return fallback_decision, f"Fallback decision due to error: {str(e)}"
    
    def _build_reasoning_prompt(self, quality_score: int, approval_status: str, 
                               iteration_count: int, issues: list, suggestions: list,
                               agent_discussion: Dict = None, adapted_goals: Dict = None) -> str:
        """Build the reasoning prompt"""
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are a strategic reasoning agent for a Job Description generation system. 
            Your task is to analyze the current situation and decide the best next action.
            
            Your goal is to ensure the highest quality JD is generated while being efficient with resources.
            
            Consider:
            1. Quality Score: How good is the current JD? (0-100)
            2. Approval Status: What did validation say? (APPROVED/NEEDS_REVISION/REJECTED)
            3. Issues: What problems were identified?
            4. Suggestions: What improvements are recommended?
            5. Iteration Count: How many times have we tried? (out of max {max_iterations})
            6. Agent Discussion: What did the agents discuss? Any consensus?
            7. Adapted Goals: What are the current priorities?
            
            Decision Options:
            - "approve": JD is good enough, stop and approve
            - "regenerate": JD needs improvement, regenerate with feedback
            - "reject": JD is too poor or max iterations reached, stop and reject
            
            Think strategically:
            - If quality is high (80+) and approved, approve
            - If quality is moderate (60-79) and issues are fixable, regenerate
            - If quality is low (<60) and we have iterations left, regenerate
            - If max iterations reached, reject
            - Consider if suggestions are actionable and improvements are likely
            - Consider agent discussion consensus and adapted goals priorities
            
            Provide your reasoning first, then your decision."""),
            ("human", """Current Situation:
Quality Score: {quality_score}/100
Approval Status: {approval_status}
Iteration: {iteration_count}/{max_iterations}
Issues Found: {issues}
Suggestions: {suggestions}

Agent Discussion:
{discussion_summary}

Adapted Goals:
{goals_summary}

Reason through:
1. Is the JD good enough to approve?
2. Can we realistically improve it with another iteration?
3. Are the issues fixable?
4. What did the agents agree on?
5. What are the current priorities?
6. What's the best strategic decision?

Provide your reasoning and then your decision (one word: approve, regenerate, or reject)."""),
        ])
        
        # Prepare discussion summary
        discussion_summary = "No discussion yet"
        if agent_discussion and agent_discussion.get("discussion"):
            discussion_summary = agent_discussion.get("discussion", "")[:500]  # First 500 chars
        
        # Prepare goals summary
        goals_summary = "Standard goals"
        if adapted_goals and adapted_goals.get("priorities"):
            priorities = adapted_goals.get("priorities", [])
            reasoning = adapted_goals.get("reasoning", "")
            goals_summary = f"Priorities: {', '.join(priorities)}\nReasoning: {reasoning[:200]}"
        
        return prompt_template.format_messages(
            max_iterations=self.max_iterations,
            quality_score=quality_score,
            approval_status=approval_status,
            iteration_count=iteration_count,
            issues=", ".join(issues) if issues else "None",
            suggestions=", ".join(suggestions[:3]) if suggestions else "None",
            discussion_summary=discussion_summary,
            goals_summary=goals_summary
        )
    
    def _parse_decision(self, response_content: str, stream_callback=None) -> Literal["regenerate", "approve", "reject"]:
        """Parse decision from LLM response"""
        response_lower = response_content.lower()
        
        # Look for decision keywords
        if "approve" in response_lower and "regenerate" not in response_lower and "reject" not in response_lower:
            return "approve"
        elif "reject" in response_lower and "regenerate" not in response_lower:
            return "reject"
        elif "regenerate" in response_lower or "improve" in response_lower or "revise" in response_lower:
            return "regenerate"
        
        # Default fallback
        if stream_callback:
            stream_callback("⚠️ Reasoning Agent: Could not parse decision clearly, using fallback")
        return "regenerate"  # Safe default
    
    def _fallback_decision(self, approval_status: str, iteration_count: int, 
                          stream_callback=None) -> Literal["regenerate", "approve", "reject"]:
        """Fallback rule-based decision if LLM fails"""
        if approval_status == "APPROVED":
            return "approve"
        if iteration_count >= self.max_iterations:
            return "reject"
        if approval_status in ["NEEDS_REVISION", "REJECTED"]:
            return "regenerate"
        return "approve"



class CommunicationLayer:
    """Enables agents to communicate, discuss, and negotiate"""
    
    def __init__(self):
        # Validate API key
        if not Config.OPENAI_API_KEY or Config.OPENAI_API_KEY == "your-openai-api-key-here":
            raise ValueError(
                "OpenAI API key is not configured. Please set OPENAI_API_KEY in your .env file."
            )
        
        self.llm = ChatOpenAI(
            api_key=Config.OPENAI_API_KEY,
            model=Config.OPENAI_MODEL,
            temperature=0.4,  # Slightly higher for creative discussion
            timeout=Config.REQUEST_TIMEOUT
        )
    
    def agent_discussion(self, agent1_name: str, agent1_output: Dict, agent1_reflection: Dict,
                        agent2_name: str, agent2_output: Dict, agent2_reflection: Dict,
                        context: Dict, stream_callback=None) -> Dict:
        """Facilitate discussion between two agents"""
        
        if stream_callback:
            stream_callback(f"💬 Communication Layer: Facilitating discussion between {agent1_name} and {agent2_name}...")
        
        prompt = self._build_discussion_prompt(
            agent1_name, agent1_output, agent1_reflection,
            agent2_name, agent2_output, agent2_reflection,
            context
        )
        
        try:
            response = self.llm.invoke(prompt)
            discussion_content = response.content
            
            if stream_callback:
                stream_callback(f"💭 Discussion Summary: {discussion_content[:200]}...")
            
            return {
                "discussion": discussion_content,
                "consensus": self._extract_consensus(discussion_content),
                "action_items": self._extract_action_items(discussion_content)
            }
            
        except Exception as e:
            if stream_callback:
                stream_callback(f"⚠️ Communication error: {str(e)}")
            return {
                "discussion": "Communication unavailable",
                "consensus": {},
                "action_items": []
            }
    
    def _build_discussion_prompt(self, agent1_name: str, agent1_output: Dict, agent1_reflection: Dict,
                                 agent2_name: str, agent2_output: Dict, agent2_reflection: Dict,
                                 context: Dict) -> str:
        """Build discussion prompt"""
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are a facilitator for agent collaboration. Two AI agents are working together 
            on a Job Description generation task. Your role is to help them:
            
            1. Understand each other's perspectives
            2. Identify areas of agreement and disagreement
            3. Negotiate solutions to conflicts
            4. Reach consensus on next steps
            5. Identify action items for improvement
            
            Be constructive and help them collaborate effectively."""),
            ("human", """Context:
Task: Generate a high-quality Job Description
Current Iteration: {iteration_count}
User Requirements: {user_requirements}

{agent1_name} Output:
{agent1_output}

{agent1_name} Self-Reflection:
{agent1_reflection}

{agent2_name} Output:
{agent2_output}

{agent2_name} Self-Reflection:
{agent2_reflection}

Facilitate their discussion. Help them:
1. Understand each other's work
2. Identify what went well and what needs improvement
3. Agree on priorities for the next iteration
4. Reach consensus on action items

Provide a summary of their discussion, consensus reached, and action items."""),
        ])
        
        return prompt_template.format_messages(
            iteration_count=context.get("iteration_count", 0),
            user_requirements=str(context.get("user_inputs", {})),
            agent1_name=agent1_name,
            agent1_output=str(agent1_output),
            agent1_reflection=agent1_reflection.get("self_assessment", "No reflection"),
            agent2_name=agent2_name,
            agent2_output=str(agent2_output),
            agent2_reflection=agent2_reflection.get("self_assessment", "No reflection")
        )
    
    def _extract_consensus(self, discussion_content: str) -> Dict:
        """Extract consensus points from discussion"""
        # Simple extraction - could be enhanced with better parsing
        consensus_keywords = ["agree", "consensus", "both", "together", "aligned", "unified"]
        discussion_lower = discussion_content.lower()
        
        consensus_points = []
        lines = discussion_content.split("\n")
        for line in lines:
            if any(keyword in line.lower() for keyword in consensus_keywords):
                consensus_points.append(line.strip())
        
        return {
            "points": consensus_points[:5],  # Limit to top 5
            "summary": discussion_content[:300] if len(discussion_content) > 300 else discussion_content
        }
    
    def _extract_action_items(self, discussion_content: str) -> list:
        """Extract action items from discussion"""
        action_keywords = ["should", "need to", "must", "will", "action", "improve", "fix", "add"]
        action_items = []
        lines = discussion_content.split("\n")
        
        for line in lines:
            line_lower = line.lower()
            if any(keyword in line_lower for keyword in action_keywords):
                # Clean up the line
                clean_line = line.strip().lstrip("- ").lstrip("* ").strip()
                if len(clean_line) > 10:  # Only meaningful items
                    action_items.append(clean_line)
        
        return action_items[:5]  # Limit to top 5



class GoalManager:
    """Manages and adapts goals dynamically based on current progress"""
    
    def __init__(self):
        # Validate API key
        if not Config.OPENAI_API_KEY or Config.OPENAI_API_KEY == "your-openai-api-key-here":
            raise ValueError(
                "OpenAI API key is not configured. Please set OPENAI_API_KEY in your .env file."
            )
        
        self.llm = ChatOpenAI(
            api_key=Config.OPENAI_API_KEY,
            model=Config.OPENAI_MODEL,
            temperature=0.3,  # Lower temperature for focused goal setting
            timeout=Config.REQUEST_TIMEOUT
        )
        self.original_goals = {
            "primary": "Generate high-quality Job Description",
            "secondary": ["Meet user requirements", "Follow market standards", "Ensure completeness"]
        }
    
    def adapt_goals(self, current_state: Dict, user_inputs: Dict, stream_callback=None) -> Dict:
        """Adapt goals based on current progress and context"""
        
        if stream_callback:
            stream_callback("🎯 Goal Manager: Analyzing current state and adapting goals...")
        
        # Only adapt if we're in a regeneration cycle
        iteration_count = current_state.get("iteration_count", 0)
        if iteration_count == 0:
            # First iteration - use original goals
            return {
                "adapted_goals": self.original_goals,
                "reasoning": "First iteration - using original goals",
                "priorities": ["quality", "completeness", "alignment"]
            }
        
        validation_results = current_state.get("validation_results", {})
        quality_score = validation_results.get("quality_score", 0)
        issues = validation_results.get("issues", [])
        suggestions = validation_results.get("suggestions", [])
        
        prompt = self._build_goal_adaptation_prompt(
            iteration_count, quality_score, issues, suggestions, user_inputs
        )
        
        try:
            response = self.llm.invoke(prompt)
            adapted_goals = self._parse_adapted_goals(response.content)
            
            if stream_callback:
                stream_callback(f"🎯 Adapted Goals: {adapted_goals.get('priorities', [])}")
                stream_callback(f"💭 Reasoning: {adapted_goals.get('reasoning', '')[:150]}...")
            
            return adapted_goals
            
        except Exception as e:
            if stream_callback:
                stream_callback(f"⚠️ Goal adaptation error: {str(e)}")
            # Fallback to original goals
            return {
                "adapted_goals": self.original_goals,
                "reasoning": f"Goal adaptation failed: {str(e)}",
                "priorities": ["quality", "completeness", "alignment"]
            }
    
    def _build_goal_adaptation_prompt(self, iteration_count: int, quality_score: int,
                                     issues: list, suggestions: list, user_inputs: Dict) -> str:
        """Build goal adaptation prompt"""
        prompt_template = ChatPromptTemplate.from_messages([
            ("system", """You are a strategic Goal Manager for a Job Description generation system. 
            Your task is to adapt goals based on current progress.
            
            Original Goals:
            - Primary: Generate high-quality Job Description
            - Secondary: Meet user requirements, Follow market standards, Ensure completeness
            
            Based on current state, adapt these goals to:
            1. Focus on areas that need improvement
            2. Prioritize based on what's missing or problematic
            3. Adjust strategy for better outcomes
            4. Consider iteration count and remaining attempts
            
            Provide adapted goals with clear priorities and reasoning."""),
            ("human", """Current State:
Iteration: {iteration_count}
Quality Score: {quality_score}/100
Issues Found: {issues}
Suggestions: {suggestions}
User Requirements: {user_requirements}

Adapt the goals based on:
1. What needs the most attention?
2. What should be prioritized in the next iteration?
3. What strategy changes are needed?

Provide adapted goals with priorities and reasoning."""),
        ])
        
        return prompt_template.format_messages(
            iteration_count=iteration_count,
            quality_score=quality_score,
            issues=", ".join(issues) if issues else "None",
            suggestions=", ".join(suggestions[:3]) if suggestions else "None",
            user_requirements=str(user_inputs)
        )
    
    def _parse_adapted_goals(self, response_content: str) -> Dict:
        """Parse adapted goals from LLM response"""
        # Extract priorities (simple keyword-based extraction)
        priorities = []
        priority_keywords = {
            "quality": ["quality", "professional", "polish"],
            "completeness": ["complete", "missing", "sections", "all"],
            "alignment": ["align", "match", "requirements", "user"],
            "specificity": ["specific", "detail", "vague", "generic"],
            "market": ["market", "standards", "competitive", "industry"]
        }
        
        response_lower = response_content.lower()
        for priority, keywords in priority_keywords.items():
            if any(keyword in response_lower for keyword in keywords):
                priorities.append(priority)
        
        # If no priorities found, use defaults
        if not priorities:
            priorities = ["quality", "completeness", "alignment"]
        
        return {
            "adapted_goals": {
                "primary": "Generate improved Job Description",
                "secondary": priorities,
                "focus_areas": priorities[:3]  # Top 3 focus areas
            },
            "priorities": priorities,
            "reasoning": response_content[:500]  # First 500 chars as reasoning
        }



def reduce_messages(left: list, right: list) -> list:
    """Reducer function for stream_messages"""
    return left + right

class JDGenerationState(TypedDict):
    user_inputs: dict
    retrieved_jds: list
    analysis_results: dict
    generated_jd: dict
    validation_results: dict
    validation_feedback: dict  # Store feedback for regeneration
    generation_reflection: dict  # Self-reflection from generation agent
    validation_reflection: dict  # Self-reflection from validation agent
    agent_discussion: dict  # Communication between agents
    adapted_goals: dict  # Dynamically adapted goals
    reasoning_output: dict  # Reasoning agent output (decision + reasoning)
    iteration_count: int  # Track regeneration iterations
    stream_messages: Annotated[list, reduce_messages]  # For streaming updates
    execution_log: Annotated[list, reduce_messages]  # Execution log for workflow trace

class JDGenerationGraph:
    def __init__(self):
        self.research_agent = ResearchAgent()
        self.analysis_agent = AnalysisAgent()
        self.jd_generation_agent = JDGenerationAgent()
        self.validation_agent = ValidationAgent()
        self.reasoning_agent = ReasoningAgent()
        self.communication_layer = CommunicationLayer()  # Agent communication
        self.goal_manager = GoalManager()  # Dynamic goal management
        self.max_iterations = Config.MAX_ITERATIONS
        self.graph = self._build_graph()
        self.stream_callback = None
        self.streaming_jd_callback = None
        self.update_session_callback = None
        self.current_node = None  # Track currently executing node
        self.execution_log = []  # Track execution flow
    
    def _log_execution(self, node_name: str, action: str = "executed", details: str = "", state: JDGenerationState = None):
        """Log execution step"""
        import time
        timestamp = datetime.now().strftime("%H:%M:%S")
        log_entry = {
            "timestamp": timestamp,
            "node": node_name,
            "action": action,
            "details": details,
            "iteration": state.get("iteration_count", 0) if state else 0
        }
        self.execution_log.append(log_entry)
        
        # Update session state if callback available
        if self.update_session_callback:
            self.update_session_callback("execution_log", self.execution_log)
        
        return log_entry
    
    def _build_graph(self) -> StateGraph:
        """Build the LangGraph workflow with agentic reasoning and self-reflection"""
        workflow = StateGraph(JDGenerationState)
        
        # Add nodes
        workflow.add_node("research", self._research_node)
        workflow.add_node("analyze", self._analyze_node)
        workflow.add_node("generate", self._generate_node)
        workflow.add_node("self_reflect_generate", self._self_reflect_generate_node)
        workflow.add_node("validate", self._validate_node)
        workflow.add_node("self_reflect_validate", self._self_reflect_validate_node)
        workflow.add_node("agent_discussion", self._agent_discussion_node)  # Agent communication
        workflow.add_node("adapt_goals", self._adapt_goals_node)  # Goal adaptation
        workflow.add_node("reason", self._reason_node)  # Reasoning (placeholder)
        
        # Set entry point
        workflow.set_entry_point("research")
        
        # Add edges
        workflow.add_edge("research", "analyze")
        workflow.add_edge("analyze", "generate")
        workflow.add_edge("generate", "self_reflect_generate")
        workflow.add_edge("self_reflect_generate", "validate")
        workflow.add_edge("validate", "self_reflect_validate")
        workflow.add_edge("self_reflect_validate", "agent_discussion")  # Self-reflect → Discussion
        workflow.add_edge("agent_discussion", "reason")  # Discussion → Reason
        
        # Conditional edge: Reasoning agent decides next step using LLM
        workflow.add_conditional_edges(
            "reason",
            self._reasoning_decision,
            {
                "regenerate": "adapt_goals",  # Regenerate → Adapt Goals → Generate
                "approve": END,
                "reject": END
            }
        )
        workflow.add_edge("adapt_goals", "generate")  # Adapt Goals → Generate
        
        return workflow.compile()
    
    def _reasoning_decision(self, state: JDGenerationState) -> Literal["regenerate", "approve", "reject"]:
        """Use reasoning agent to make LLM-based decision"""
        self._log_execution("reasoning", "started", "Analyzing situation and making decision", state)
        
        # Get reasoning output (decision + reasoning content)
        decision, reasoning_content = self.reasoning_agent.reason_next_action_with_output(
            state, stream_callback=self.stream_callback
        )
        
        # Store reasoning output in state
        reasoning_output = {
            "decision": decision,
            "reasoning": reasoning_content,
            "quality_score": state.get("validation_results", {}).get("quality_score", 0),
            "approval_status": state.get("validation_results", {}).get("approval_status", "UNKNOWN"),
            "iteration_count": state.get("iteration_count", 0)
        }
        
        # Log the decision
        quality_score = reasoning_output["quality_score"]
        approval_status = reasoning_output["approval_status"]
        self._log_execution("reasoning", "decision", 
                          f"Decision: {decision.upper()} | Quality: {quality_score}/100 | Status: {approval_status} | Iteration: {state.get('iteration_count', 0)}", 
                          state)
        
        # Update session state immediately
        if self.update_session_callback:
            self.update_session_callback("reasoning_output", reasoning_output)
        
        # Store in state for return (though this is a conditional edge, state update happens via callback)
        # Note: Conditional edges don't modify state directly, so we rely on callback
        return decision
    
    def _research_node(self, state: JDGenerationState) -> JDGenerationState:
        """Research Agent node"""
        self.current_node = "research"
        if self.update_session_callback:
            self.update_session_callback("current_node", "research")
        
        # Only research once (skip on regeneration)
        if state.get("retrieved_jds"):
            self._log_execution("research", "skipped", "Already retrieved JDs, skipping on regeneration", state)
            return state
        
        self._log_execution("research", "started", "Searching for JDs from internet", state)
        
        retrieved_jds = self.research_agent.search_jds(
            state["user_inputs"],
            stream_callback=self.stream_callback
        )
        
        # Update session state immediately after research completes
        if self.update_session_callback:
            self.update_session_callback("retrieved_jds", retrieved_jds)
        
        self._log_execution("research", "completed", f"Retrieved {len(retrieved_jds)} JDs", state)
        
        log_entry = self.execution_log[-1] if self.execution_log else None
        return {
            **state,
            "retrieved_jds": retrieved_jds,
            "execution_log": state.get("execution_log", []) + ([log_entry] if log_entry else []),
            "stream_messages": state.get("stream_messages", []) + [f"✅ Research completed: {len(retrieved_jds)} JDs retrieved"]
        }
    
    def _analyze_node(self, state: JDGenerationState) -> JDGenerationState:
        """Analysis Agent node"""
        self.current_node = "analyze"
        if self.update_session_callback:
            self.update_session_callback("current_node", "analyze")
        
        # Only analyze once (skip on regeneration)
        if state.get("analysis_results"):
            self._log_execution("analyze", "skipped", "Already analyzed, skipping on regeneration", state)
            return state
        
        self._log_execution("analyze", "started", "Extracting market patterns from retrieved JDs", state)
        
        analysis_results = self.analysis_agent.analyze_jds(
            state["retrieved_jds"],
            state["user_inputs"],
            stream_callback=self.stream_callback
        )
        
        # Update session state immediately after analysis completes
        if self.update_session_callback:
            self.update_session_callback("analysis_results", analysis_results)
        
        self._log_execution("analyze", "completed", "Market patterns extracted successfully", state)
        
        log_entry = self.execution_log[-1] if self.execution_log else None
        return {
            **state,
            "analysis_results": analysis_results,
            "execution_log": state.get("execution_log", []) + ([log_entry] if log_entry else []),
            "stream_messages": state.get("stream_messages", []) + [f"✅ Analysis completed: Patterns extracted"]
        }
    
    def _generate_node(self, state: JDGenerationState) -> JDGenerationState:
        """JD Generation Agent node"""
        self.current_node = "generate"
        if self.update_session_callback:
            self.update_session_callback("current_node", "generate")
        
        iteration_count = state.get("iteration_count", 0)
        
        if iteration_count > 0:
            # Regeneration: Include validation feedback and adapted goals
            self._log_execution("generate", "started", f"Regenerating JD with improvements (Attempt {iteration_count + 1})", state)
            if self.stream_callback:
                self.stream_callback(f"🔄 Regenerating JD with improvements (Attempt {iteration_count + 1})")
            
            # Pass validation feedback and adapted goals to generation agent
            validation_feedback = state.get("validation_feedback", {})
            adapted_goals = state.get("adapted_goals", {})
            user_inputs_with_feedback = {
                **state["user_inputs"],
                "validation_feedback": validation_feedback,
                "adapted_goals": adapted_goals
            }
        else:
            self._log_execution("generate", "started", "Generating initial JD", state)
            user_inputs_with_feedback = state["user_inputs"]
        
        generated_jd = self.jd_generation_agent.generate_jd(
            user_inputs_with_feedback,
            state["retrieved_jds"],
            state.get("analysis_results", {}),
            stream_callback=self.stream_callback,
            streaming_jd_callback=self.streaming_jd_callback
        )
        
        self._log_execution("generate", "completed", f"JD generated successfully (Iteration {iteration_count + 1})", state)
        
        log_entry = self.execution_log[-1] if self.execution_log else None
        return {
            **state,
            "generated_jd": generated_jd,
            "iteration_count": iteration_count + 1,
            "execution_log": state.get("execution_log", []) + ([log_entry] if log_entry else []),
            "stream_messages": state.get("stream_messages", []) + [f"✅ JD generation completed (Iteration {iteration_count + 1})"]
        }
    
    def _self_reflect_generate_node(self, state: JDGenerationState) -> JDGenerationState:
        """Self-reflection node for Generation Agent"""
        self.current_node = "self_reflect_generate"
        if self.update_session_callback:
            self.update_session_callback("current_node", "self_reflect_generate")
        
        generation_reflection = self.jd_generation_agent.self_reflect(
            state.get("generated_jd", {}),
            state["user_inputs"],
            stream_callback=self.stream_callback
        )
        
        if self.update_session_callback:
            self.update_session_callback("generation_reflection", generation_reflection)
        
        return {
            **state,
            "generation_reflection": generation_reflection,
            "stream_messages": state.get("stream_messages", []) + ["🤔 Generation Agent self-reflection completed"]
        }
    
    def _validate_node(self, state: JDGenerationState) -> JDGenerationState:
        """Validation Agent node"""
        self.current_node = "validate"
        if self.update_session_callback:
            self.update_session_callback("current_node", "validate")
        
        self._log_execution("validate", "started", "Validating JD quality and completeness", state)
        
        validation_results = self.validation_agent.validate_jd(
            state.get("generated_jd", {}),
            state["user_inputs"],
            stream_callback=self.stream_callback
        )
        
        quality_score = validation_results.get("quality_score", 0)
        approval_status = validation_results.get("approval_status", "UNKNOWN")
        self._log_execution("validate", "completed", f"Quality Score: {quality_score}/100 | Status: {approval_status}", state)
        
        # Store feedback for regeneration
        validation_feedback = {
            "suggestions": validation_results.get("suggestions", []),
            "issues": validation_results.get("issues", []),
            "quality_score": validation_results.get("quality_score", 0),
            "quality_assessment": validation_results.get("quality_assessment", "")
        }
        
        # Update session state immediately after validation completes
        if self.update_session_callback:
            self.update_session_callback("validation_results", validation_results)
        
        log_entry = self.execution_log[-1] if self.execution_log else None
        return {
            **state,
            "validation_results": validation_results,
            "validation_feedback": validation_feedback,
            "execution_log": state.get("execution_log", []) + ([log_entry] if log_entry else []),
            "stream_messages": state.get("stream_messages", []) + [
                f"✅ Validation completed: Quality Score {validation_results.get('quality_score', 0)}/100 | Status: {validation_results.get('approval_status', 'UNKNOWN')}"
            ]
        }
    
    def _self_reflect_validate_node(self, state: JDGenerationState) -> JDGenerationState:
        """Self-reflection node for Validation Agent"""
        self.current_node = "self_reflect_validate"
        if self.update_session_callback:
            self.update_session_callback("current_node", "self_reflect_validate")
        
        validation_reflection = self.validation_agent.self_reflect(
            state.get("validation_results", {}),
            state.get("generated_jd", {}),
            stream_callback=self.stream_callback
        )
        
        if self.update_session_callback:
            self.update_session_callback("validation_reflection", validation_reflection)
        
        return {
            **state,
            "validation_reflection": validation_reflection,
            "stream_messages": state.get("stream_messages", []) + ["🤔 Validation Agent self-reflection completed"]
        }
    
    def _agent_discussion_node(self, state: JDGenerationState) -> JDGenerationState:
        """Agent discussion node - facilitates communication between Generation and Validation agents"""
        self.current_node = "agent_discussion"
        if self.update_session_callback:
            self.update_session_callback("current_node", "agent_discussion")
        
        discussion = self.communication_layer.agent_discussion(
            agent1_name="Generation Agent",
            agent1_output=state.get("generated_jd", {}),
            agent1_reflection=state.get("generation_reflection", {}),
            agent2_name="Validation Agent",
            agent2_output=state.get("validation_results", {}),
            agent2_reflection=state.get("validation_reflection", {}),
            context={
                "iteration_count": state.get("iteration_count", 0),
                "user_inputs": state.get("user_inputs", {})
            },
            stream_callback=self.stream_callback
        )
        
        if self.update_session_callback:
            self.update_session_callback("agent_discussion", discussion)
        
        return {
            **state,
            "agent_discussion": discussion,
            "stream_messages": state.get("stream_messages", []) + ["💬 Agent discussion completed"]
        }
    
    def _adapt_goals_node(self, state: JDGenerationState) -> JDGenerationState:
        """Goal adaptation node - adapts goals based on current progress"""
        self.current_node = "adapt_goals"
        if self.update_session_callback:
            self.update_session_callback("current_node", "adapt_goals")
        
        adapted_goals = self.goal_manager.adapt_goals(
            current_state={
                "iteration_count": state.get("iteration_count", 0),
                "validation_results": state.get("validation_results", {}),
                "agent_discussion": state.get("agent_discussion", {})
            },
            user_inputs=state.get("user_inputs", {}),
            stream_callback=self.stream_callback
        )
        
        if self.update_session_callback:
            self.update_session_callback("adapted_goals", adapted_goals)
        
        return {
            **state,
            "adapted_goals": adapted_goals,
            "stream_messages": state.get("stream_messages", []) + ["🎯 Goals adapted for next iteration"]
        }
    
    def _reason_node(self, state: JDGenerationState) -> JDGenerationState:
        """Reasoning node - placeholder that passes through state"""
        self.current_node = "reason"
        if self.update_session_callback:
            self.update_session_callback("current_node", "reason")
        # Actual reasoning happens in conditional edge function
        return state
    
    def run(self, user_inputs: dict, stream_callback=None, streaming_jd_callback=None, update_session_callback=None):
        """Run the graph with streaming"""
        # Store callbacks for use in nodes
        self.stream_callback = stream_callback
        self.streaming_jd_callback = streaming_jd_callback
        self.update_session_callback = update_session_callback
        
        # Reset execution log for new run
        self.execution_log = []
        self._log_execution("workflow", "started", "JD Generation workflow initiated", {})
        
        initial_state = {
            "user_inputs": user_inputs,
            "retrieved_jds": [],
            "analysis_results": {},
            "generated_jd": {},
            "validation_results": {},
            "validation_feedback": {},
            "generation_reflection": {},
            "validation_reflection": {},
            "agent_discussion": {},
            "adapted_goals": {},
            "reasoning_output": {},
            "iteration_count": 0,
            "stream_messages": [],
            "execution_log": self.execution_log.copy()
        }
        
        # Run graph
        result = self.graph.invoke(initial_state)
        
        # Log workflow completion
        self._log_execution("workflow", "completed", "JD Generation workflow completed", result)
        if self.execution_log:
            result["execution_log"] = self.execution_log.copy()
        
        return result


# =============================================================================
# SAP IAS JWT + FastAPI
# =============================================================================


@dataclass(frozen=True)
class BtpAuthSettings:
    ias_issuer: str
    ias_audience: str
    verify_ias_audience: bool
    btp_skip_auth: bool
    cors_origins: str


def _load_btp_settings() -> BtpAuthSettings:
    return BtpAuthSettings(
        ias_issuer=os.getenv("IAS_ISSUER", "").strip(),
        ias_audience=os.getenv("IAS_AUDIENCE", os.getenv("IAS_CLIENT_ID", "")).strip(),
        verify_ias_audience=os.getenv("VERIFY_IAS_AUDIENCE", "true").lower()
        not in ("0", "false", "no"),
        btp_skip_auth=os.getenv("BTP_SKIP_AUTH", "").lower() in ("1", "true", "yes"),
        cors_origins=(os.getenv("CORS_ORIGINS", "*").strip() or "*"),
    )


btp_settings = _load_btp_settings()
_jwks_client: PyJWKClient | None = None
security = HTTPBearer(auto_error=False)


def _normalize_issuer(issuer: str) -> str:
    return issuer.rstrip("/")


def _get_jwks_client(issuer: str) -> PyJWKClient:
    global _jwks_client
    issuer = _normalize_issuer(issuer)
    well_known = f"{issuer}/.well-known/openid-configuration"
    if _jwks_client is None:
        with httpx.Client(timeout=15.0) as client:
            resp = client.get(well_known)
            resp.raise_for_status()
            jwks_uri = resp.json().get("jwks_uri")
            if not jwks_uri:
                raise RuntimeError("OIDC discovery missing jwks_uri")
        _jwks_client = PyJWKClient(jwks_uri)
    return _jwks_client


def verify_ias_jwt(token: str) -> dict[str, Any]:
    issuer = _normalize_issuer(btp_settings.ias_issuer)
    if not issuer:
        raise HTTPException(status_code=500, detail="IAS_ISSUER is not configured")

    jwks_client = _get_jwks_client(issuer)
    signing_key = jwks_client.get_signing_key_from_jwt(token)

    decode_kwargs: dict[str, Any] = {
        "algorithms": ["RS256"],
        "issuer": issuer,
        "leeway": 60,
        "options": {"verify_aud": btp_settings.verify_ias_audience and bool(btp_settings.ias_audience)},
    }
    if btp_settings.verify_ias_audience and btp_settings.ias_audience:
        decode_kwargs["audience"] = btp_settings.ias_audience

    try:
        return jwt.decode(token, signing_key.key, **decode_kwargs)
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired") from None
    except jwt.InvalidAudienceError:
        raise HTTPException(status_code=401, detail="Invalid token audience") from None
    except jwt.InvalidIssuerError:
        raise HTTPException(status_code=401, detail="Invalid token issuer") from None
    except jwt.PyJWTError as e:
        logger.warning("JWT validation failed: %s", e)
        raise HTTPException(status_code=401, detail="Invalid or unreadable token") from None


def auth_enabled() -> bool:
    if btp_settings.btp_skip_auth:
        logger.warning("BTP_SKIP_AUTH is enabled — not for production")
        return False
    return bool(btp_settings.ias_issuer.strip())


async def require_bearer_jwt(
    credentials: Annotated[Optional[HTTPAuthorizationCredentials], Depends(security)],
) -> dict[str, Any]:
    if not auth_enabled():
        return {"sub": "anonymous", "auth_mode": "disabled"}
    if credentials is None or not credentials.credentials:
        raise HTTPException(
            status_code=401,
            detail="Missing Authorization: Bearer token",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return verify_ias_jwt(credentials.credentials)


app = FastAPI(
    title="JD Generation API (BTP monolith)",
    description="Single-file JD pipeline + optional SAP IAS JWT.",
    version="1.0.0-btp-monolith",
    docs_url="/docs",
    redoc_url="/redoc",
)

_origins = (
    ["*"]
    if btp_settings.cors_origins.strip() == "*"
    else [o.strip() for o in btp_settings.cors_origins.split(",") if o.strip()]
)
app.add_middleware(
    CORSMiddleware,
    allow_origins=_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


class JDRequest(BaseModel):
    job_title: str = Field(..., min_length=2, max_length=200)
    education: Optional[str] = Field(None, max_length=500)
    min_experience: Optional[int] = Field(None, ge=0, le=50)
    max_experience: Optional[int] = Field(None, ge=0, le=50)
    experience: Optional[str] = Field(None, max_length=200)
    functional_area: Optional[str] = Field(None, max_length=200)
    skills: Optional[str] = Field(None, max_length=1000)
    additional_input: Optional[str] = Field(None, max_length=2000)
    num_jds: Optional[int] = Field(5, ge=1, le=20)
    temperature: Optional[float] = Field(0.2, ge=0.0, le=1.0)
    company_context: Optional[str] = Field(None, max_length=5000)

    model_config = {
        "json_schema_extra": {
            "example": {
                "job_title": "Senior Software Engineer",
                "education": "Bachelor's in Computer Science",
                "min_experience": 5,
                "max_experience": 7,
                "skills": "Python, React, AWS, Docker",
                "num_jds": 5,
                "temperature": 0.2,
            }
        }
    }


class JDResponse(BaseModel):
    success: bool
    job_title: str
    generated_jd: dict
    quality_score: int
    approval_status: str
    iteration_count: int
    execution_time: float
    timestamp: datetime


def _build_experience_string(request: JDRequest) -> str:
    if request.min_experience is not None and request.max_experience is not None:
        if request.min_experience == request.max_experience:
            return f"{request.min_experience} years"
        return f"{request.min_experience}-{request.max_experience} years"
    if request.experience:
        return request.experience
    if request.min_experience is not None:
        return f"{request.min_experience}+ years"
    if request.max_experience is not None:
        return f"up to {request.max_experience} years"
    return ""


def _request_to_inputs(request: JDRequest) -> dict:
    return {
        "job_title": request.job_title,
        "education": request.education or "",
        "experience": _build_experience_string(request),
        "functional_area": request.functional_area or "",
        "skills": request.skills or "",
        "additional_input": request.additional_input or "",
        "num_jds": request.num_jds,
        "temperature": request.temperature,
        "company_context": request.company_context or "",
    }


@app.get("/")
async def root():
    return {
        "service": "JD Generation API (BTP monolith)",
        "status": "running",
        "version": "1.0.0-btp-monolith",
        "docs": "/docs",
        "auth": "enabled" if auth_enabled() else "disabled",
    }


@app.get("/health")
async def health():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}


@app.get("/auth/status")
async def auth_status(_claims: Annotated[dict, Depends(require_bearer_jwt)]):
    safe = {k: v for k, v in _claims.items() if k in ("sub", "client_id", "azp", "iss", "exp", "scope")}
    return {"valid": True, "claims": safe}


@app.post("/generate", response_model=JDResponse)
async def generate_jd(
    request: JDRequest,
    _claims: Annotated[dict, Depends(require_bearer_jwt)],
):
    logger.info("Generating JD for: %s", request.job_title)
    inputs = _request_to_inputs(request)
    try:
        graph = JDGenerationGraph()
        start_time = time.time()
        result = graph.run(inputs)
        execution_time = time.time() - start_time
        return JDResponse(
            success=True,
            job_title=request.job_title,
            generated_jd=result.get("generated_jd", {}),
            quality_score=result.get("validation_results", {}).get("quality_score", 0),
            approval_status=result.get("validation_results", {}).get("approval_status", "UNKNOWN"),
            iteration_count=result.get("iteration_count", 0),
            execution_time=round(execution_time, 2),
            timestamp=datetime.now(),
        )
    except Exception as e:
        logger.error("Generation failed: %s", e, exc_info=True)
        raise HTTPException(status_code=500, detail=str(e)) from e


@app.post("/generate/simple")
async def generate_simple(
    request: JDRequest,
    _claims: Annotated[dict, Depends(require_bearer_jwt)],
):
    logger.info("Simple JD generation for: %s", request.job_title)
    inputs = _request_to_inputs(request)
    try:
        graph = JDGenerationGraph()
        result = graph.run(inputs)
        return {
            "success": True,
            "jd_content": result.get("generated_jd", {}).get("full_content", ""),
            "quality_score": result.get("validation_results", {}).get("quality_score", 0),
        }
    except Exception as e:
        logger.error("Generation failed: %s", e)
        raise HTTPException(status_code=500, detail=str(e)) from e


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(
        "sap_btp:app",
        host="0.0.0.0",
        port=int(os.getenv("PORT", "8000")),
        reload=False,
    )

````
