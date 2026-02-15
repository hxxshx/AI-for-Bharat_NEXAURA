# Nexura Platform - Design Document

**Project Name**: Nexura  
**Tagline**: From Learning to Earning  

---

## 1. System Architecture Overview

### 1.1 Architectural Style

Nexura employs a **serverless-first microservices architecture** with event-driven patterns, deployed entirely on AWS. The architecture combines:

- **Serverless compute** (AWS Lambda) for stateless business logic
- **Managed AI/ML services** (SageMaker, Bedrock) for model inference
- **Event-driven orchestration** (EventBridge, SQS) for asynchronous workflows
- **Polyglot persistence** (DynamoDB, S3, RDS Aurora) for data storage
- **API-first design** (API Gateway) for service communication

### 1.2 Layered Architecture

**Layer 1: Presentation Layer**
- React SPA hosted on S3, distributed via CloudFront
- WebRTC for real-time video/audio capture
- i18next for multilingual UI rendering

**Layer 2: API Gateway Layer**
- REST API orchestration via API Gateway
- Request validation, throttling, caching
- Cognito authorizers for authentication
- API versioning (v1, v2)

**Layer 3: Business Logic Layer**
- Lambda functions for core services (resume parsing, skill gap analysis, interview orchestration)
- Step Functions for complex workflows (multi-stage interview, roadmap generation)
- ECS Fargate for long-running ML inference (optional for cost optimization)

**Layer 4: AI/ML Layer**
- SageMaker endpoints for custom models (skill extraction, emotion detection)
- Bedrock for foundation model access (GPT-4, Claude for interview Q&A)
- Comprehend for NLP tasks (entity extraction, sentiment)
- Translate for multilingual support
- Rekognition for facial emotion detection

**Layer 5: Data Layer**
- DynamoDB for transactional data (users, sessions, scores)
- S3 for object storage (resumes, videos, model artifacts)
- RDS Aurora Serverless for relational analytics (optional)
- ElastiCache Redis for session caching

**Layer 6: Infrastructure Layer**
- CloudWatch for logging and metrics
- X-Ray for distributed tracing
- EventBridge for event routing
- SNS/SQS for pub/sub messaging
- Secrets Manager for credential storage

### 1.3 Architectural Rationale

**Why Serverless-First?**
- Cost efficiency: Pay-per-use for variable workloads (hackathon → production)
- Auto-scaling: Handle traffic spikes without manual intervention
- Reduced operational overhead: No server management
- Fast iteration: Deploy individual functions independently

**Why Event-Driven?**
- Decoupling: Services communicate via events, not direct calls
- Asynchronous processing: Long-running tasks (video analysis) don't block API responses
- Resilience: Failed events can be retried automatically
- Scalability: Event sources scale independently

**Why Microservices?**
- Independent deployment: Update resume parser without affecting interview engine
- Technology flexibility: Use Python for ML, Node.js for API logic
- Team autonomy: Different teams own different services
- Fault isolation: One service failure doesn't crash entire system

---

## 2. End-to-End System Flow (Step-by-Step Execution Flow)

### 2.1 Resume + Job Description Skill Gap Analysis Flow


1. **User uploads resume** via frontend (React component)
2. **Frontend sends POST request** to `/api/v1/resumes/upload` with multipart/form-data
3. **API Gateway** validates request, checks JWT token via Cognito authorizer
4. **Lambda: Upload Handler** generates pre-signed S3 URL, returns to frontend
5. **Frontend uploads file directly to S3** using pre-signed URL (bypasses API Gateway for large files)
6. **S3 triggers EventBridge event** on object creation (`s3:ObjectCreated:*`)
7. **EventBridge routes event** to Lambda: Resume Parser
8. **Lambda: Resume Parser** downloads file from S3, invokes parsing logic:
   - PDF/DOCX → text extraction (PyPDF2, python-docx)
   - Text → structured data extraction (regex + NLP)
9. **Lambda invokes SageMaker endpoint** (Skill Extraction Model) with resume text
10. **SageMaker returns** extracted skills as JSON array
11. **Lambda normalizes skills** against skill taxonomy (DynamoDB lookup)
12. **Lambda stores parsed resume** in DynamoDB (`Resumes` table) with metadata
13. **Lambda publishes event** to EventBridge: `resume.parsed`
14. **User provides job description** via frontend (URL or text)
15. **Frontend sends POST request** to `/api/v1/analysis/skill-gap`
16. **API Gateway routes** to Lambda: Skill Gap Analyzer
17. **Lambda: Skill Gap Analyzer** fetches job description:
    - If URL: scrape content (Puppeteer/Playwright in Lambda layer)
    - If text: use directly
18. **Lambda invokes SageMaker endpoint** (Skill Extraction Model) with JD text
19. **SageMaker returns** required skills as JSON array
20. **Lambda fetches user skills** from DynamoDB (`UserSkills` table)
21. **Lambda performs skill gap calculation**:
    - Matched skills: intersection of user skills and JD skills
    - Missing skills: JD skills - user skills
    - Skill similarity: cosine similarity of skill embeddings
22. **Lambda calculates match percentage**: (matched skills / total JD skills) × 100
23. **Lambda stores analysis results** in DynamoDB (`SkillGapAnalyses` table)
24. **Lambda publishes event** to EventBridge: `skillgap.analyzed`
25. **EventBridge triggers** Lambda: Roadmap Generator (asynchronous)
26. **Lambda returns response** to API Gateway with analysis results
27. **API Gateway returns JSON** to frontend
28. **Frontend renders** skill gap visualization (charts, missing skills list)


### 2.2 Mock Interview with Emotion Analysis Flow

1. **User starts interview session** via frontend (clicks "Start Interview")
2. **Frontend sends POST request** to `/api/v1/interviews/start` with `jobId` and `userId`
3. **API Gateway routes** to Lambda: Interview Orchestrator
4. **Lambda: Interview Orchestrator** creates interview session:
   - Generates `sessionId` (UUID)
   - Fetches job description from DynamoDB
   - Fetches user resume from DynamoDB
5. **Lambda invokes Bedrock** (GPT-4/Claude) with prompt:
   - Context: job description + user resume
   - Task: Generate 5 contextually relevant interview questions
   - Output format: JSON array of questions
6. **Bedrock returns** generated questions
7. **Lambda stores session** in DynamoDB (`InterviewSessions` table) with questions
8. **Lambda returns** first question to frontend
9. **Frontend displays question** and starts video/audio recording (WebRTC)
10. **User answers question** (video/audio captured in browser)
11. **Frontend sends video/audio chunks** to Kinesis Video Streams (real-time streaming)
12. **Kinesis Video Streams triggers** Lambda: Stream Processor (per chunk)
13. **Lambda: Stream Processor** processes video frames:
    - Extracts frames (1 frame per second)
    - Invokes Rekognition `DetectFaces` API for emotion detection
14. **Rekognition returns** emotion labels (HAPPY, SAD, CALM, CONFUSED, etc.) with confidence scores
15. **Lambda aggregates emotions** over 5-second windows, stores in DynamoDB
16. **Lambda processes audio**:
    - Sends audio to Transcribe for speech-to-text
    - Sends audio to SageMaker endpoint (Audio Emotion Model) for voice sentiment
17. **Transcribe returns** text transcript
18. **SageMaker returns** audio emotion scores (confidence, nervousness, enthusiasm)
19. **Lambda stores transcript and emotions** in DynamoDB (`InterviewSessions` table, nested in answers array)
20. **Frontend sends POST request** to `/api/v1/interviews/{sessionId}/submit-answer` with answer text
21. **API Gateway routes** to Lambda: Answer Evaluator
22. **Lambda: Answer Evaluator** invokes Bedrock with prompt:
    - Context: question + user answer + job description
    - Task: Evaluate answer quality, relevance, completeness
    - Output format: JSON with score (0-100), feedback, improvement suggestions
23. **Bedrock returns** evaluation results
24. **Lambda stores evaluation** in DynamoDB
25. **Lambda invokes Bedrock** to generate next question (adaptive based on previous answer)
26. **Lambda returns** next question + feedback on previous answer to frontend
27. **Repeat steps 9-26** for remaining questions
28. **User completes interview** (all questions answered)
29. **Frontend sends POST request** to `/api/v1/interviews/{sessionId}/complete`
30. **Lambda: Interview Orchestrator** calculates overall interview score:
    - Average answer quality score
    - Communication score (from transcript analysis)
    - Confidence score (from emotion analysis)
31. **Lambda updates session** in DynamoDB with final score
32. **Lambda publishes event** to EventBridge: `interview.completed`
33. **EventBridge triggers** Lambda: Employability Score Updater (asynchronous)
34. **Lambda returns** interview summary to frontend
35. **Frontend renders** interview results (score, emotion timeline, feedback per question)


### 2.3 Multilingual Code Debugging Flow

1. **User submits code snippet** via frontend (code editor component)
2. **User selects programming language** (Python, JavaScript, Java, C++, SQL)
3. **User selects preferred language** for explanation (English, Hindi, Tamil, etc.)
4. **Frontend sends POST request** to `/api/v1/debug/analyze` with:
   - `code`: string (code snippet)
   - `language`: string (programming language)
   - `outputLanguage`: string (explanation language)
5. **API Gateway routes** to Lambda: Code Debugger
6. **Lambda: Code Debugger** performs syntax validation:
   - Python: `ast.parse()` to detect syntax errors
   - JavaScript: `esprima.parse()` to detect syntax errors
   - Other languages: language-specific parsers
7. **If syntax errors found**, Lambda formats error messages
8. **Lambda invokes Bedrock** (GPT-4/Claude) with prompt:
   - Context: code snippet + programming language
   - Task: Identify bugs, explain issues, suggest fixes
   - Output format: JSON with `errors`, `explanations`, `fixedCode`
   - Language: English (initial response)
9. **Bedrock returns** debugging analysis in English
10. **Lambda checks if outputLanguage != English**
11. **If translation needed**, Lambda invokes AWS Translate:
    - Source: English
    - Target: user's preferred language
    - Text: explanations and suggestions
12. **AWS Translate returns** translated text
13. **Lambda constructs response** with:
    - Original code
    - Identified errors
    - Explanations (in user's language)
    - Fixed code
    - Confidence score
14. **Lambda stores debugging session** in DynamoDB (`DebuggingSessions` table) for analytics
15. **Lambda returns response** to API Gateway
16. **API Gateway returns JSON** to frontend
17. **Frontend renders** debugging results (side-by-side code comparison, explanations)


### 2.4 Employability Score Calculation Flow

1. **EventBridge scheduled rule** triggers daily at 00:00 UTC
2. **EventBridge invokes** Lambda: Employability Score Calculator
3. **Lambda fetches all active users** from DynamoDB (`Users` table)
4. **For each user**, Lambda aggregates data:
   - **Resume Quality**: Fetch from `Resumes` table
     - Completeness score: (filled sections / total sections) × 100
     - Formatting score: ATS-friendly check (keyword density, structure)
   - **Skill Proficiency**: Fetch from `UserSkills` table
     - Number of skills (normalized to 0-100)
     - Skill relevance: match to target job roles
     - Skill depth: certifications, projects
   - **Interview Performance**: Fetch from `InterviewSessions` table
     - Average answer quality score
     - Average communication score
     - Average confidence score
   - **Learning Progress**: Fetch from `LearningPaths` table
     - Course completion rate
     - Skill acquisition rate (new skills / time)
     - Consistency score (daily activity streak)
   - **Market Demand**: Fetch from external API (job board API)
     - Job availability for user's skills
     - Salary potential (normalized)
     - Industry growth trend
5. **Lambda calculates component scores**:
   - Resume Quality Score = (Completeness × 0.5) + (Formatting × 0.5)
   - Skill Proficiency Score = (Number × 0.3) + (Relevance × 0.4) + (Depth × 0.3)
   - Interview Performance Score = (Answer Quality × 0.5) + (Communication × 0.3) + (Confidence × 0.2)
   - Learning Progress Score = (Completion Rate × 0.4) + (Acquisition Rate × 0.4) + (Consistency × 0.2)
   - Market Demand Score = (Job Availability × 0.5) + (Salary Potential × 0.3) + (Growth Trend × 0.2)
6. **Lambda calculates overall Employability Index**:
   - EI = (Resume × 0.20) + (Skills × 0.30) + (Interview × 0.25) + (Learning × 0.15) + (Market × 0.10)
7. **Lambda generates improvement insights** using rule-based logic:
   - If Resume Quality < 60: "Complete missing resume sections"
   - If Skill Proficiency < 70: "Focus on acquiring high-demand skills"
   - If Interview Performance < 65: "Practice more mock interviews"
   - If Learning Progress < 50: "Maintain consistent learning schedule"
8. **Lambda stores score** in DynamoDB (`EmployabilityScores` table) with timestamp
9. **Lambda checks if score changed significantly** (>5 points from last score)
10. **If significant change**, Lambda publishes event to SNS topic: `employability.score.updated`
11. **SNS triggers** Lambda: Notification Service
12. **Lambda: Notification Service** sends email to user with score update and insights
13. **Lambda logs calculation** to CloudWatch for monitoring


---

## 3. Detailed Component Breakdown

### 3.1 Frontend Web Layer

**Purpose**: User interface for all platform features

**Technology Stack**:
- React 18 with TypeScript
- Redux Toolkit for state management
- Material-UI (MUI) for components
- i18next for internationalization
- WebRTC for video/audio capture
- Recharts for data visualization

**Internal Logic**:
- SPA with client-side routing (React Router)
- JWT token management (stored in httpOnly cookies)
- Real-time video/audio streaming to Kinesis
- Responsive design (mobile-first)

**Input**: User interactions (clicks, form submissions, file uploads)

**Output**: API requests to backend, rendered UI

**Dependencies**: API Gateway, CloudFront, S3

**AWS Service Mapping**:
- Hosted on S3 (static website hosting)
- Distributed via CloudFront (CDN)
- Route 53 for DNS


### 3.2 API Gateway Layer

**Purpose**: Single entry point for all API requests, handles routing, authentication, throttling

**Internal Logic**:
- REST API with resource-based routing
- Request validation (JSON schema)
- Response transformation
- CORS configuration
- API key management for external integrations

**Input**: HTTP requests from frontend

**Output**: Routed requests to Lambda functions, HTTP responses

**Dependencies**: Lambda functions, Cognito

**AWS Service Mapping**:
- API Gateway (REST API)
- Cognito authorizers for JWT validation
- CloudWatch for access logs

**Configuration**:
- Rate limiting: 100 requests/minute per user
- Burst limit: 200 requests
- Caching: 5-minute TTL for GET requests
- API versioning: `/v1/`, `/v2/` prefixes


### 3.3 Authentication Service

**Purpose**: User authentication, authorization, session management

**Internal Logic**:
- User registration with email verification
- Password hashing (bcrypt)
- JWT token generation and validation
- OAuth integration (Google, LinkedIn)
- MFA support (TOTP, SMS)
- Role-based access control (RBAC)

**Input**: Login credentials, OAuth tokens, MFA codes

**Output**: JWT tokens, user session data

**Dependencies**: Cognito User Pools, DynamoDB

**AWS Service Mapping**:
- Cognito User Pools for user management
- Cognito Identity Pools for AWS resource access
- Lambda triggers for custom authentication logic
- SNS for SMS-based MFA

**Token Structure**:
```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "role": "student",
  "language": "hi",
  "exp": 1234567890
}
```


### 3.4 Resume Parsing Service

**Purpose**: Extract structured data from resume files (PDF, DOCX, TXT)

**Technology**: Python 3.11, Lambda

**Internal Logic**:
1. Download file from S3
2. Detect file type (MIME type)
3. Extract text:
   - PDF: PyPDF2, pdfplumber
   - DOCX: python-docx
   - TXT: direct read
4. Parse sections using regex patterns:
   - Contact info: email, phone (regex)
   - Education: degree, institution, year (NER)
   - Experience: company, role, duration (NER)
   - Skills: keyword extraction
5. Normalize data (dates, phone numbers)
6. Validate extracted data (completeness check)

**Input**: S3 object key (resume file path)

**Output**: Structured JSON with parsed resume data

**Dependencies**: S3, SageMaker (Skill Extraction Model), DynamoDB

**AWS Service Mapping**:
- Lambda function (Python 3.11, 512MB memory, 60s timeout)
- S3 for file storage
- EventBridge for triggering

**Error Handling**:
- Malformed PDF: Return partial data + error flag
- Unsupported format: Return error message
- Timeout: Store partial results, retry with increased timeout


### 3.5 Skill Extraction Engine

**Purpose**: Extract skills from unstructured text (resumes, job descriptions)

**Technology**: SageMaker endpoint hosting fine-tuned BERT model

**Model Architecture**:
- Base: `bert-base-uncased` (Hugging Face)
- Fine-tuned on: LinkedIn job postings + resume datasets
- Task: Named Entity Recognition (NER) for skills
- Labels: B-SKILL, I-SKILL, O (BIO tagging)

**Internal Logic**:
1. Tokenize input text (BERT tokenizer)
2. Generate token embeddings
3. Pass through fine-tuned BERT model
4. Extract entities tagged as skills
5. Post-process: merge multi-word skills, remove duplicates
6. Normalize skills against taxonomy (fuzzy matching)

**Input**: Text string (resume or job description)

**Output**: JSON array of skills with confidence scores

**Example Output**:
```json
{
  "skills": [
    {"name": "Python", "confidence": 0.95, "category": "programming"},
    {"name": "Machine Learning", "confidence": 0.89, "category": "technical"},
    {"name": "Communication", "confidence": 0.76, "category": "soft"}
  ]
}
```

**Dependencies**: SageMaker, S3 (model artifacts), DynamoDB (skill taxonomy)

**AWS Service Mapping**:
- SageMaker endpoint (ml.t3.medium instance)
- S3 for model storage
- CloudWatch for endpoint metrics

**Performance**:
- Inference latency: <500ms
- Throughput: 100 requests/second
- Auto-scaling: 1-5 instances based on invocations


### 3.6 Skill Gap Analyzer

**Purpose**: Compare user skills against job requirements, identify gaps

**Technology**: Python 3.11, Lambda

**Internal Logic**:
1. Fetch user skills from DynamoDB
2. Fetch job description skills (from Skill Extraction Engine)
3. Generate skill embeddings:
   - Use sentence-transformers (`all-MiniLM-L6-v2`)
   - Embed each skill as 384-dimensional vector
4. Calculate similarity matrix:
   - For each user skill, compute cosine similarity with each JD skill
   - Threshold: 0.7 (skills with similarity >0.7 are considered matched)
5. Identify matched skills (user has, JD requires)
6. Identify missing skills (JD requires, user doesn't have)
7. Identify proficiency gaps (user has skill but lower proficiency than required)
8. Rank missing skills by importance:
   - Frequency in JD (TF-IDF)
   - Market demand (from job board API)
   - Skill category weight (technical > soft skills)
9. Calculate match percentage: (matched skills / total JD skills) × 100

**Input**: 
- `userId`: string
- `jobDescription`: string or URL

**Output**: 
```json
{
  "matchPercentage": 75,
  "matchedSkills": ["Python", "SQL", "Git"],
  "missingSkills": [
    {"name": "Docker", "importance": 0.9, "category": "devops"},
    {"name": "AWS", "importance": 0.85, "category": "cloud"}
  ],
  "proficiencyGaps": [
    {"name": "Machine Learning", "userLevel": "beginner", "requiredLevel": "intermediate"}
  ]
}
```

**Dependencies**: SageMaker (Skill Extraction), DynamoDB, sentence-transformers library

**AWS Service Mapping**:
- Lambda function (Python 3.11, 1GB memory, 30s timeout)
- DynamoDB for data retrieval
- EventBridge for triggering downstream services


### 3.7 Roadmap Personalization Engine

**Purpose**: Generate personalized learning paths based on skill gaps

**Technology**: Python 3.11, Lambda

**Approach**: Hybrid (rule-based + ML-based recommendation)

**Internal Logic**:

**Phase 1: Skill Prioritization**
1. Fetch missing skills from Skill Gap Analyzer
2. Apply prioritization rules:
   - High importance skills first (from JD analysis)
   - Foundational skills before advanced (dependency graph)
   - High-demand skills prioritized (market data)
3. Generate skill learning sequence

**Phase 2: Resource Recommendation**
1. For each skill, query resource database (DynamoDB):
   - Courses (Coursera, Udemy, YouTube)
   - Tutorials (freeCodeCamp, W3Schools)
   - Projects (GitHub, Kaggle)
   - Government programs (NSDC, Skill India)
2. Filter by:
   - User language preference
   - Cost (prioritize free resources)
   - Duration (match user's time availability)
   - Difficulty level (match user's current proficiency)
3. Rank resources using scoring function:
   - Score = (Relevance × 0.4) + (Rating × 0.3) + (Completion Rate × 0.2) + (Recency × 0.1)

**Phase 3: Adaptive Adjustment**
1. Track user progress (courses completed, time spent)
2. Adjust difficulty based on performance
3. Re-prioritize skills based on user feedback
4. Update recommendations weekly

**Input**:
- `userId`: string
- `analysisId`: string (skill gap analysis ID)

**Output**:
```json
{
  "pathId": "uuid",
  "estimatedDuration": "3 months",
  "skills": [
    {
      "name": "Docker",
      "priority": 1,
      "resources": [
        {
          "title": "Docker for Beginners",
          "provider": "Udemy",
          "url": "https://...",
          "duration": "4 hours",
          "cost": "Free",
          "language": "English",
          "rating": 4.7
        }
      ]
    }
  ]
}
```

**Dependencies**: DynamoDB (resources, user progress), external APIs (course providers)

**AWS Service Mapping**:
- Lambda function (Python 3.11, 512MB memory, 60s timeout)
- DynamoDB for resource catalog and user progress
- EventBridge for scheduled updates


### 3.8 Mock Interview Engine

**Purpose**: Conduct AI-powered mock interviews with adaptive questioning

**Technology**: Node.js 20, Lambda, Step Functions

**Architecture**: Step Functions orchestrate multi-stage interview workflow

**Internal Logic**:

**Stage 1: Interview Initialization**
1. Create session in DynamoDB
2. Fetch job description and user resume
3. Generate initial question set (5-10 questions)

**Stage 2: Question Generation**
1. Invoke Bedrock (GPT-4 or Claude) with prompt:
```
You are an experienced technical interviewer.
Job Description: {jd}
Candidate Resume: {resume}
Previous Q&A: {history}

Generate the next interview question that:
- Tests relevant skills for the role
- Adapts to candidate's previous answers
- Varies between technical and behavioral
- Is clear and unambiguous

Output format: JSON with question, type, expectedTopics
```
2. Parse Bedrock response
3. Store question in session

**Stage 3: Answer Collection**
1. Present question to user
2. Capture response (text, audio, or video)
3. Transcribe audio (if applicable) using Transcribe
4. Store answer in session

**Stage 4: Answer Evaluation**
1. Invoke Bedrock with evaluation prompt:
```
Question: {question}
Candidate Answer: {answer}
Job Requirements: {jd}

Evaluate the answer on:
- Relevance (0-100)
- Completeness (0-100)
- Technical accuracy (0-100)
- Communication clarity (0-100)

Provide:
- Overall score (0-100)
- Strengths (list)
- Weaknesses (list)
- Improvement suggestions (list)
```
2. Parse evaluation results
3. Store in session

**Stage 5: Adaptive Decision**
1. Analyze answer quality
2. Decide next action:
   - If answer weak: Generate follow-up clarification question
   - If answer strong: Move to next topic
   - If all questions answered: End interview
3. Loop back to Stage 2 or proceed to Stage 6

**Stage 6: Interview Completion**
1. Calculate overall interview score
2. Generate summary report
3. Trigger emotion analysis aggregation
4. Store final results in DynamoDB
5. Publish completion event

**Input**: `userId`, `jobId`

**Output**: Interview session with questions, answers, evaluations, scores

**Dependencies**: Bedrock, Transcribe, Rekognition, DynamoDB, Step Functions

**AWS Service Mapping**:
- Step Functions for workflow orchestration
- Lambda functions for each stage
- Bedrock for question generation and evaluation
- DynamoDB for session storage


### 3.9 Emotion Detection Module

**Purpose**: Analyze facial expressions and voice tone during mock interviews

**Technology**: Python 3.11, SageMaker, Rekognition

**Architecture**: Multi-modal emotion detection (visual + audio)

**Visual Emotion Detection**:

**Pipeline**:
1. Extract video frames from Kinesis Video Streams (1 fps)
2. Invoke Rekognition `DetectFaces` API for each frame
3. Extract emotion labels and confidence scores
4. Aggregate emotions over 5-second windows
5. Apply temporal smoothing (moving average)

**Rekognition Emotions**:
- HAPPY, SAD, ANGRY, CONFUSED, DISGUSTED, SURPRISED, CALM, FEAR

**Output per frame**:
```json
{
  "timestamp": 5.2,
  "emotions": [
    {"type": "CALM", "confidence": 0.87},
    {"type": "HAPPY", "confidence": 0.12}
  ],
  "dominantEmotion": "CALM"
}
```

**Audio Emotion Detection**:

**Model**: Custom SageMaker endpoint (Wav2Vec 2.0 fine-tuned)

**Pipeline**:
1. Extract audio from video stream
2. Preprocess audio (16kHz sampling, noise reduction)
3. Extract features (MFCCs, pitch, energy)
4. Invoke SageMaker endpoint
5. Get emotion predictions (confidence, nervousness, enthusiasm)

**Output per segment**:
```json
{
  "timestamp": 5.2,
  "confidence": 0.72,
  "nervousness": 0.15,
  "enthusiasm": 0.68,
  "pace": "moderate",
  "pitch": "normal"
}
```

**Emotion Fusion**:
1. Combine visual and audio emotions
2. Weighted average (visual: 0.6, audio: 0.4)
3. Generate overall confidence score
4. Identify concerning patterns (excessive nervousness, low confidence)

**Output**:
```json
{
  "overallConfidence": 0.68,
  "emotionTimeline": [...],
  "insights": [
    "Confidence decreased during technical questions",
    "Strong enthusiasm when discussing projects"
  ],
  "recommendations": [
    "Practice technical explanations to build confidence",
    "Maintain eye contact throughout the interview"
  ]
}
```

**Dependencies**: Kinesis Video Streams, Rekognition, SageMaker, DynamoDB

**AWS Service Mapping**:
- Kinesis Video Streams for real-time video ingestion
- Rekognition for facial emotion detection
- SageMaker endpoint for audio emotion detection
- Lambda for processing and aggregation


### 3.10 Code Debugging AI Service

**Purpose**: Provide multilingual code debugging explanations

**Technology**: Node.js 20, Lambda, Bedrock

**Internal Logic**:

**Phase 1: Syntax Validation**
1. Detect programming language (user-provided or auto-detect)
2. Parse code using language-specific parser:
   - Python: `ast.parse()`
   - JavaScript: `esprima.parse()`
   - Java: `java-parser` library
   - C++: `tree-sitter-cpp`
   - SQL: `sql-parser`
3. Identify syntax errors (line number, error type)

**Phase 2: Semantic Analysis**
1. Invoke Bedrock (GPT-4 or Claude) with prompt:
```
You are an expert programmer and debugging assistant.

Code:
```{language}
{code}
```

Task:
1. Identify all bugs (syntax, logical, runtime)
2. Explain each bug clearly
3. Suggest fixes with explanations
4. Provide corrected code

Output format: JSON with bugs[], explanations[], fixedCode
```
2. Parse Bedrock response
3. Extract bugs, explanations, fixed code

**Phase 3: Translation (if needed)**
1. Check user's preferred language
2. If not English, invoke AWS Translate:
   - Translate explanations
   - Translate suggestions
   - Keep code and technical terms unchanged
3. Construct multilingual response

**Phase 4: Response Construction**
1. Format response with:
   - Original code (with line numbers)
   - Identified bugs (highlighted)
   - Explanations (in user's language)
   - Fixed code (with diff highlighting)
   - Confidence score
2. Store session for learning analytics

**Input**:
```json
{
  "code": "def add(a, b)\n  return a + b",
  "language": "python",
  "outputLanguage": "hi"
}
```

**Output**:
```json
{
  "bugs": [
    {
      "line": 1,
      "type": "syntax",
      "message": "Missing colon after function definition"
    }
  ],
  "explanations": [
    "पायथन में फ़ंक्शन परिभाषा के बाद कोलन (:) आवश्यक है"
  ],
  "fixedCode": "def add(a, b):\n  return a + b",
  "confidence": 0.95
}
```

**Dependencies**: Bedrock, AWS Translate, DynamoDB

**AWS Service Mapping**:
- Lambda function (Node.js 20, 512MB memory, 30s timeout)
- Bedrock for code analysis
- AWS Translate for multilingual support
- DynamoDB for session storage


### 3.11 Employability Index Engine

**Purpose**: Calculate quantifiable employability score (0-100)

**Technology**: Python 3.11, Lambda

**Scoring Model**:

**Component 1: Resume Quality (20% weight)**
- Completeness: (filled sections / total sections) × 100
  - Sections: contact, summary, education, experience, skills, projects, certifications
- Formatting: ATS-friendly score
  - Keyword density (0-100)
  - Structure clarity (0-100)
  - Length appropriateness (0-100)
- Formula: `(Completeness × 0.6) + (Formatting × 0.4)`

**Component 2: Skill Proficiency (30% weight)**
- Number of skills: Normalized to 0-100 (sigmoid function)
  - 0 skills = 0, 20+ skills = 100
- Skill relevance: Average match percentage across target jobs
- Skill depth: Certifications + projects score
- Formula: `(Number × 0.3) + (Relevance × 0.4) + (Depth × 0.3)`

**Component 3: Interview Performance (25% weight)**
- Answer quality: Average score across all mock interviews
- Communication: Clarity and coherence score
- Confidence: Emotion analysis score
- Formula: `(Answer Quality × 0.5) + (Communication × 0.3) + (Confidence × 0.2)`

**Component 4: Learning Progress (15% weight)**
- Completion rate: (completed courses / enrolled courses) × 100
- Acquisition rate: New skills acquired per month
- Consistency: Daily activity streak (normalized)
- Formula: `(Completion × 0.4) + (Acquisition × 0.4) + (Consistency × 0.2)`

**Component 5: Market Demand (10% weight)**
- Job availability: Number of relevant job postings (normalized)
- Salary potential: Average salary for user's skills (normalized)
- Growth trend: Industry growth rate (0-100)
- Formula: `(Availability × 0.5) + (Salary × 0.3) + (Growth × 0.2)`

**Overall Formula**:
```
EI = (Resume × 0.20) + (Skills × 0.30) + (Interview × 0.25) + (Learning × 0.15) + (Market × 0.10)
```

**Score Interpretation**:
- 0-40: Needs Significant Improvement
- 41-60: Developing
- 61-75: Competitive
- 76-90: Strong Candidate
- 91-100: Exceptional

**Insights Generation**:
Rule-based logic generates actionable recommendations:
```python
if resume_quality < 60:
    insights.append("Complete missing resume sections")
if skill_proficiency < 70:
    insights.append("Focus on acquiring high-demand skills")
if interview_performance < 65:
    insights.append("Practice more mock interviews")
if learning_progress < 50:
    insights.append("Maintain consistent learning schedule")
```

**Dependencies**: DynamoDB (all user data), external job board API

**AWS Service Mapping**:
- Lambda function (Python 3.11, 512MB memory, 60s timeout)
- EventBridge for scheduled daily calculation
- DynamoDB for data retrieval and storage
- SNS for notifications


### 3.12 Responsible AI Monitoring Module

**Purpose**: Detect and mitigate bias in AI recommendations

**Technology**: Python 3.11, Lambda

**Bias Detection Checkpoints**:

**Checkpoint 1: Resume Analysis**
- Remove PII before processing:
  - Name, age, gender, photo, address
  - Use regex + NER to detect and redact
- Monitor skill extraction for demographic bias:
  - Compare skill extraction accuracy across demographic groups
  - Flag if accuracy variance >10%

**Checkpoint 2: Skill Gap Analysis**
- Monitor recommendation diversity:
  - Ensure recommendations span multiple skill categories
  - Flag if >80% recommendations are in single category
- Check for keyword bias:
  - Detect gendered language ("aggressive" vs "assertive")
  - Replace biased terms with neutral alternatives

**Checkpoint 3: Interview Evaluation**
- Monitor evaluation consistency:
  - Compare scores across demographic groups (if data available)
  - Flag if mean score difference >15 points
- Check for sentiment bias:
  - Analyze feedback language for bias indicators
  - Flag negative sentiment patterns

**Checkpoint 4: Learning Recommendations**
- Ensure resource diversity:
  - Recommendations should include free and paid options
  - Recommendations should span multiple providers
  - Flag if >70% recommendations from single source

**Fairness Metrics**:

**Demographic Parity**:
- P(recommendation | group A) ≈ P(recommendation | group B)
- Threshold: difference <0.1

**Equalized Odds**:
- P(high score | qualified, group A) ≈ P(high score | qualified, group B)
- Threshold: difference <0.15

**Monitoring Pipeline**:
1. Log all AI decisions to DynamoDB (`BiasAuditLogs` table)
2. Aggregate metrics daily (Lambda triggered by EventBridge)
3. Calculate fairness metrics
4. Flag violations (store in `BiasAlerts` table)
5. Generate weekly bias report
6. Notify admin if critical violations detected

**Mitigation Strategies**:
- Adversarial debiasing during model training
- Post-processing fairness constraints
- Human review for flagged decisions
- User feedback loop (report bias button)

**Explainability**:
- Use SHAP values for model explanations
- Provide feature importance for each decision
- Allow users to understand why recommendations were made

**Dependencies**: DynamoDB, CloudWatch, SNS

**AWS Service Mapping**:
- Lambda function (Python 3.11, 512MB memory, 60s timeout)
- DynamoDB for audit logs
- CloudWatch for metrics and dashboards
- SNS for admin alerts


### 3.13 Data Storage Layer

**Purpose**: Persistent storage for all platform data

**Storage Strategy**: Polyglot persistence (right tool for right data)

**DynamoDB Tables**:

**Users Table**
- PK: `userId` (UUID)
- Attributes: email, name, language, role, createdAt, lastLogin
- GSI: `email-index` (for login lookup)

**Resumes Table**
- PK: `resumeId` (UUID)
- SK: `userId`
- Attributes: s3Key, parsedData (JSON), uploadedAt, version
- GSI: `userId-uploadedAt-index`

**Skills Table** (Taxonomy)
- PK: `skillId` (UUID)
- Attributes: name, category, synonyms (list), demandScore, embedding (vector)
- GSI: `category-index`

**UserSkills Table**
- PK: `userId`
- SK: `skillId`
- Attributes: proficiencyLevel, acquiredAt, source, certifications (list)

**JobDescriptions Table**
- PK: `jobId` (UUID)
- Attributes: title, company, skills (list), url, cachedAt, expiresAt (TTL)

**SkillGapAnalyses Table**
- PK: `analysisId` (UUID)
- SK: `userId`
- Attributes: jobId, matchPercentage, matchedSkills, missingSkills, createdAt
- GSI: `userId-createdAt-index`

**LearningPaths Table**
- PK: `pathId` (UUID)
- SK: `userId`
- Attributes: targetJobId, skills (list), resources (list), progress, createdAt, updatedAt

**InterviewSessions Table**
- PK: `sessionId` (UUID)
- SK: `userId`
- Attributes: jobId, questions (list), answers (list), emotions (list), score, createdAt

**EmployabilityScores Table**
- PK: `userId`
- SK: `timestamp` (ISO 8601)
- Attributes: overallScore, componentScores (map), insights (list)

**BiasAuditLogs Table**
- PK: `logId` (UUID)
- SK: `timestamp`
- Attributes: eventType, userId, decision, fairnessMetrics (map), flagged (boolean)

**S3 Buckets**:

**nexura-resumes**
- Structure: `{userId}/{resumeId}.pdf`
- Lifecycle: Move to Glacier after 90 days
- Encryption: SSE-KMS

**nexura-videos**
- Structure: `{userId}/{sessionId}/video.webm`
- Lifecycle: Delete after 30 days
- Encryption: SSE-S3

**nexura-models**
- Structure: `{modelName}/{version}/model.tar.gz`
- Versioning: Enabled
- Encryption: SSE-KMS

**nexura-static**
- Structure: Frontend assets (HTML, JS, CSS)
- CloudFront distribution
- Encryption: SSE-S3

**Dependencies**: DynamoDB, S3, KMS

**AWS Service Mapping**:
- DynamoDB (on-demand capacity mode)
- S3 (standard storage class)
- KMS for encryption keys


### 3.14 Analytics Layer

**Purpose**: Track user behavior, platform performance, business metrics

**Technology**: CloudWatch, Athena, QuickSight (future)

**Metrics Tracked**:

**User Engagement**:
- Daily/Monthly Active Users (DAU/MAU)
- Session duration
- Feature usage (resume uploads, interviews taken, debugging sessions)
- User retention (cohort analysis)

**Platform Performance**:
- API latency (p50, p95, p99)
- Error rates by endpoint
- Lambda cold starts
- SageMaker endpoint latency

**AI Model Performance**:
- Skill extraction accuracy (validated against manual labels)
- Emotion detection accuracy
- Interview evaluation correlation with human evaluators
- Model inference latency

**Business Metrics**:
- User employability score distribution
- Average score improvement over time
- Course completion rates
- Interview confidence improvement

**Implementation**:

**Real-Time Metrics** (CloudWatch):
- Lambda logs with structured JSON
- Custom metrics (API latency, error rates)
- Alarms for critical thresholds
- Dashboards for monitoring

**Batch Analytics** (Athena + S3):
- Export CloudWatch logs to S3 daily
- Query logs using Athena (SQL)
- Generate reports (weekly, monthly)
- Store aggregated metrics in DynamoDB

**Example CloudWatch Log**:
```json
{
  "timestamp": "2026-02-15T10:30:00Z",
  "userId": "uuid",
  "event": "skill_gap_analyzed",
  "matchPercentage": 75,
  "duration": 1.2,
  "success": true
}
```

**Dependencies**: CloudWatch, S3, Athena

**AWS Service Mapping**:
- CloudWatch Logs for log aggregation
- CloudWatch Metrics for custom metrics
- Athena for SQL queries on logs
- S3 for log storage


### 3.15 Monitoring & Logging

**Purpose**: System health monitoring, error tracking, debugging

**Technology**: CloudWatch, X-Ray, CloudTrail

**Logging Strategy**:

**Application Logs** (CloudWatch Logs):
- Lambda function logs (stdout/stderr)
- Structured JSON format
- Log levels: DEBUG, INFO, WARN, ERROR
- Retention: 30 days

**Access Logs** (API Gateway):
- Request/response logs
- Latency, status codes
- Client IP, user agent
- Retention: 90 days

**Audit Logs** (CloudTrail):
- AWS API calls
- IAM actions
- Resource changes
- Retention: 1 year

**Distributed Tracing** (X-Ray):
- End-to-end request tracing
- Service map visualization
- Latency analysis
- Error identification

**Monitoring Dashboards**:

**System Health Dashboard**:
- API Gateway request count, latency, errors
- Lambda invocations, duration, errors
- DynamoDB read/write capacity, throttles
- SageMaker endpoint invocations, latency

**AI Model Dashboard**:
- Model inference latency
- Model accuracy metrics
- Bias detection alerts
- Model version tracking

**User Experience Dashboard**:
- Page load times
- Feature usage
- Error rates by feature
- User satisfaction scores

**Alerting**:

**Critical Alerts** (PagerDuty):
- API error rate >5%
- Lambda function failures >10/min
- DynamoDB throttling
- SageMaker endpoint failures

**Warning Alerts** (Slack):
- API latency p95 >3s
- Lambda cold starts >20%
- Model accuracy drop >5%
- Bias detection flags

**Info Alerts** (Email):
- Daily summary reports
- Weekly analytics reports
- Monthly cost reports

**Dependencies**: CloudWatch, X-Ray, CloudTrail, SNS

**AWS Service Mapping**:
- CloudWatch for logs, metrics, alarms
- X-Ray for distributed tracing
- CloudTrail for audit logs
- SNS for alert notifications

---

## 4. AWS Cloud Architecture Design

### 4.1 Frontend Layer

**CloudFront**:
- Purpose: CDN for global content delivery
- Configuration:
  - Origin: S3 bucket (nexura-static)
  - Cache behavior: Cache static assets (JS, CSS, images) for 1 year
  - Compression: Gzip, Brotli enabled
  - SSL/TLS: TLS 1.3, custom domain (nexura.ai)
  - Geo-restriction: None (global access)
- Why: Reduces latency, improves page load times, reduces S3 costs

**S3**:
- Purpose: Static website hosting
- Configuration:
  - Bucket: nexura-static
  - Website hosting: Enabled
  - Versioning: Enabled
  - Encryption: SSE-S3
- Why: Cost-effective, scalable, integrates with CloudFront


### 4.2 Backend Layer

**API Gateway**:
- Purpose: REST API management
- Configuration:
  - Type: REST API (not HTTP API for advanced features)
  - Endpoints: ~20 resources (resumes, analysis, interviews, etc.)
  - Authorizer: Cognito User Pool
  - Throttling: 100 req/min per user, 10,000 req/sec burst
  - Caching: 5-minute TTL for GET requests
  - CORS: Enabled for frontend domain
- Why: Managed service, auto-scaling, built-in throttling, caching

**Lambda**:
- Purpose: Serverless compute for business logic
- Configuration:
  - Runtime: Node.js 20, Python 3.11
  - Memory: 512MB - 1GB (varies by function)
  - Timeout: 30s - 60s (varies by function)
  - Concurrency: Reserved 100, provisioned 10 for critical functions
  - Environment variables: Encrypted with KMS
- Functions:
  - Resume Parser (Python, 512MB, 60s)
  - Skill Gap Analyzer (Python, 1GB, 30s)
  - Interview Orchestrator (Node.js, 512MB, 30s)
  - Code Debugger (Node.js, 512MB, 30s)
  - Employability Calculator (Python, 512MB, 60s)
  - Notification Service (Node.js, 256MB, 15s)
- Why: Pay-per-use, auto-scaling, no server management, fast deployment

**Step Functions**:
- Purpose: Orchestrate complex workflows
- Configuration:
  - Type: Standard workflows
  - Workflows:
    - Mock Interview Flow (multi-stage Q&A)
    - Learning Path Generation (multi-step recommendation)
- Why: Visual workflow, error handling, retry logic, state management


### 4.3 AI/ML Layer

**SageMaker**:
- Purpose: Model training, hosting, inference
- Configuration:
  - Endpoints:
    - Skill Extraction Model (ml.t3.medium, 1-5 instances)
    - Audio Emotion Model (ml.t3.medium, 1-3 instances)
  - Auto-scaling: Target invocations 100/min per instance
  - Model artifacts: Stored in S3 (nexura-models)
  - Training jobs: Spot instances for cost savings
- Why: Managed ML infrastructure, auto-scaling, built-in monitoring

**Bedrock**:
- Purpose: Foundation model access (GPT-4, Claude)
- Configuration:
  - Models: Claude 3 Sonnet (primary), GPT-4 (fallback)
  - Use cases: Interview Q&A, code debugging, answer evaluation
  - Prompt templates: Stored in DynamoDB
  - Rate limiting: 100 req/min
- Why: No model management, latest models, pay-per-token

**Comprehend**:
- Purpose: NLP tasks (entity extraction, sentiment analysis)
- Configuration:
  - APIs: DetectEntities, DetectSentiment, DetectKeyPhrases
  - Use cases: Resume parsing, job description analysis
- Why: Pre-trained models, no training required, accurate

**Translate**:
- Purpose: Multilingual support
- Configuration:
  - Languages: English, Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati
  - Use cases: Code debugging explanations, UI translation
  - Caching: Translations cached in DynamoDB
- Why: High-quality translation, supports Indian languages

**Rekognition**:
- Purpose: Facial emotion detection
- Configuration:
  - API: DetectFaces with emotion analysis
  - Use cases: Mock interview emotion detection
- Why: Pre-trained model, accurate, no training required

**Transcribe**:
- Purpose: Speech-to-text
- Configuration:
  - Language: English (primary), Hindi (future)
  - Use cases: Mock interview audio transcription
- Why: Accurate transcription, supports multiple languages


### 4.4 Database Layer

**DynamoDB**:
- Purpose: Primary database for transactional data
- Configuration:
  - Capacity mode: On-demand (auto-scaling)
  - Tables: 10 tables (Users, Resumes, Skills, etc.)
  - Encryption: AWS-managed keys
  - Point-in-time recovery: Enabled
  - Global tables: Future (multi-region)
- Why: Serverless, auto-scaling, low latency, pay-per-request

**S3**:
- Purpose: Object storage for files
- Configuration:
  - Buckets: 4 buckets (resumes, videos, models, static)
  - Storage class: Standard (hot data), Glacier (cold data)
  - Lifecycle policies: Move to Glacier after 90 days
  - Versioning: Enabled for models and static
  - Encryption: SSE-KMS (sensitive data), SSE-S3 (others)
- Why: Unlimited storage, durable, cost-effective

**ElastiCache Redis** (Optional):
- Purpose: Session caching, API response caching
- Configuration:
  - Node type: cache.t3.micro (1 node)
  - Use cases: JWT token caching, skill taxonomy caching
  - TTL: 5 minutes for API responses, 1 hour for sessions
- Why: Reduces database load, improves response times

**RDS Aurora Serverless** (Future):
- Purpose: Relational analytics, complex queries
- Configuration:
  - Engine: PostgreSQL
  - Capacity: 2-16 ACUs (auto-scaling)
  - Use cases: Cross-table analytics, reporting
- Why: Serverless, auto-scaling, SQL support


### 4.5 Authentication Layer

**Cognito**:
- Purpose: User authentication and authorization
- Configuration:
  - User Pools:
    - Password policy: Min 8 chars, uppercase, lowercase, number, special char
    - MFA: Optional (TOTP, SMS)
    - Email verification: Required
    - OAuth: Google, LinkedIn
  - Identity Pools:
    - Federated identities for AWS resource access
    - Role-based access (student, admin)
  - Token expiration: Access token 1 hour, refresh token 30 days
- Why: Managed service, OAuth support, MFA, JWT tokens


### 4.6 Monitoring Layer

**CloudWatch**:
- Purpose: Logging, metrics, alarms
- Configuration:
  - Log groups: Per Lambda function, API Gateway
  - Metrics: Custom metrics for business KPIs
  - Alarms: Error rate, latency, throttling
  - Dashboards: System health, AI models, user experience
- Why: Integrated with AWS services, custom metrics, alarms

**X-Ray**:
- Purpose: Distributed tracing
- Configuration:
  - Enabled for: Lambda, API Gateway
  - Sampling: 10% of requests
  - Service map: Visualize request flow
- Why: End-to-end tracing, performance bottleneck identification

**CloudTrail**:
- Purpose: Audit logs
- Configuration:
  - Trail: All regions
  - Events: Management events, data events (S3, DynamoDB)
  - Storage: S3 bucket with encryption
- Why: Compliance, security auditing, forensics


### 4.7 Integration Layer

**EventBridge**:
- Purpose: Event-driven architecture, event routing
- Configuration:
  - Event buses: Default bus
  - Rules:
    - S3 object created → Resume Parser
    - Skill gap analyzed → Roadmap Generator
    - Interview completed → Employability Calculator
    - Daily schedule → Employability Calculator
  - Targets: Lambda, Step Functions, SNS, SQS
- Why: Decouples services, event-driven, scalable

**SNS**:
- Purpose: Pub/sub messaging, notifications
- Configuration:
  - Topics:
    - employability-score-updated
    - bias-alert
    - system-error
  - Subscriptions: Email, Lambda, SQS
- Why: Fan-out messaging, multiple subscribers, reliable

**SQS**:
- Purpose: Asynchronous job queues
- Configuration:
  - Queues:
    - video-processing-queue (FIFO)
    - notification-queue (Standard)
  - Dead-letter queue: Enabled for failed messages
  - Visibility timeout: 60s
- Why: Decouples services, retry logic, scalable

**Kinesis Video Streams**:
- Purpose: Real-time video streaming
- Configuration:
  - Streams: interview-video-stream
  - Retention: 24 hours
  - Consumers: Lambda (emotion detection)
- Why: Real-time processing, scalable, managed


### 4.8 Security Layer

**IAM**:
- Purpose: Access control for AWS resources
- Configuration:
  - Roles:
    - LambdaExecutionRole (DynamoDB, S3, SageMaker access)
    - SageMakerExecutionRole (S3, CloudWatch access)
    - CognitoAuthenticatedRole (limited S3 access)
  - Policies: Least privilege principle
- Why: Fine-grained access control, secure

**Secrets Manager**:
- Purpose: Store API keys, credentials
- Configuration:
  - Secrets:
    - Bedrock API keys
    - External API keys (job boards)
    - Database credentials (future RDS)
  - Rotation: Automatic rotation every 90 days
- Why: Secure storage, automatic rotation, auditing

**KMS**:
- Purpose: Encryption key management
- Configuration:
  - Keys:
    - DynamoDB encryption key
    - S3 encryption key (sensitive data)
    - Secrets Manager encryption key
  - Key rotation: Automatic annual rotation
- Why: Centralized key management, compliance

**WAF**:
- Purpose: Web application firewall
- Configuration:
  - Rules:
    - SQL injection protection
    - XSS protection
    - Rate limiting (1000 req/5min per IP)
    - Geo-blocking (optional)
  - Associated with: CloudFront, API Gateway
- Why: Protects against common attacks, DDoS mitigation

---

## 5. Architecture Diagram Narrative (Textual Diagram Description)

### 5.1 High-Level Architecture Flow

**Left to Right Flow**:

```
[User Browser]
    ↓ HTTPS
[CloudFront CDN] ← Caches static assets
    ↓
[S3 Static Hosting] ← React SPA
    ↓ API Calls (HTTPS)
[API Gateway] ← JWT validation via Cognito
    ↓
    ├─→ [Lambda: Resume Parser] → [S3: Resumes] → [EventBridge] → [Lambda: Skill Extractor] → [SageMaker: Skill Model]
    ├─→ [Lambda: Skill Gap Analyzer] → [DynamoDB: Skills, UserSkills] → [SageMaker: Skill Model]
    ├─→ [Lambda: Roadmap Generator] → [DynamoDB: LearningPaths, Resources]
    ├─→ [Lambda: Interview Orchestrator] → [Step Functions] → [Bedrock: GPT-4/Claude]
    ├─→ [Lambda: Code Debugger] → [Bedrock: GPT-4/Claude] → [AWS Translate]
    └─→ [Lambda: Employability Calculator] → [DynamoDB: EmployabilityScores] → [SNS: Notifications]

[Kinesis Video Streams] ← Real-time video from browser
    ↓
[Lambda: Stream Processor]
    ├─→ [Rekognition: Facial Emotion]
    └─→ [SageMaker: Audio Emotion Model]

[EventBridge] ← Event routing
    ├─→ Scheduled events (daily score calculation)
    ├─→ S3 events (resume uploaded)
    └─→ Custom events (skill gap analyzed, interview completed)

[CloudWatch] ← Logs, metrics, alarms
[X-Ray] ← Distributed tracing
[CloudTrail] ← Audit logs
```


### 5.2 Detailed Component Interaction

**Resume Upload Flow**:
```
User → Frontend → API Gateway → Lambda (Upload Handler) → S3 (pre-signed URL)
                                                              ↓
                                                         EventBridge
                                                              ↓
                                                    Lambda (Resume Parser)
                                                              ↓
                                                    SageMaker (Skill Extraction)
                                                              ↓
                                                    DynamoDB (Resumes, UserSkills)
```

**Skill Gap Analysis Flow**:
```
User → Frontend → API Gateway → Lambda (Skill Gap Analyzer)
                                        ↓
                                 Fetch user skills (DynamoDB)
                                        ↓
                                 Extract JD skills (SageMaker)
                                        ↓
                                 Calculate similarity (cosine)
                                        ↓
                                 Store results (DynamoDB)
                                        ↓
                                 Publish event (EventBridge)
                                        ↓
                                 Lambda (Roadmap Generator)
```

**Mock Interview Flow**:
```
User → Frontend → API Gateway → Lambda (Interview Orchestrator)
                                        ↓
                                 Step Functions (Interview Workflow)
                                        ↓
                                 Bedrock (Question Generation)
                                        ↓
                                 User answers (video/audio)
                                        ↓
                                 Kinesis Video Streams
                                        ↓
                                 Lambda (Stream Processor)
                                        ↓
                        ┌───────────────┴───────────────┐
                        ↓                               ↓
                 Rekognition (Facial)          SageMaker (Audio)
                        ↓                               ↓
                        └───────────────┬───────────────┘
                                        ↓
                                 DynamoDB (Emotions)
                                        ↓
                                 Bedrock (Answer Evaluation)
                                        ↓
                                 DynamoDB (Interview Results)
```

**Responsible AI Feedback Loop**:
```
AI Decision (any service)
        ↓
Lambda (Bias Monitor)
        ↓
Calculate fairness metrics
        ↓
DynamoDB (BiasAuditLogs)
        ↓
If flagged → SNS (Admin Alert)
        ↓
Human review → Mitigation action
```

### 5.3 Event-Driven Architecture Highlights

**Event Sources**:
- S3 object created (resume uploaded)
- DynamoDB stream (user skill updated)
- Scheduled events (daily score calculation)
- Custom events (skill gap analyzed, interview completed)

**Event Targets**:
- Lambda functions (processing)
- Step Functions (workflows)
- SNS topics (notifications)
- SQS queues (async processing)

**Event Flow Example**:
```
Resume uploaded (S3) → EventBridge → Lambda (Parser) → Skill extracted (custom event)
                                                              ↓
                                                         EventBridge
                                                              ↓
                                                    Lambda (Roadmap Generator)
```

---

## 6. AI/ML System Design

### 6.1 Skill Gap Model

**Architecture**: BERT-based skill embedding + cosine similarity

**Resume Embedding Strategy**:
1. Extract skills from resume (NER model)
2. For each skill:
   - Tokenize using BERT tokenizer
   - Generate embedding (768-dimensional vector)
   - Store in DynamoDB with skill name

**Job Description Embedding Strategy**:
1. Extract skills from JD (same NER model)
2. Generate embeddings (same process)
3. Cache embeddings for frequently accessed JDs

**Matching Logic**:
1. Compute cosine similarity between each user skill and each JD skill
2. Create similarity matrix (M x N)
3. For each JD skill:
   - Find max similarity with user skills
   - If max > threshold (0.7): skill matched
   - If max < threshold: skill missing
4. Calculate match percentage: (matched / total JD skills) × 100

**Fine-Tuning vs Foundation Model**:
- Base model: `bert-base-uncased` (pre-trained)
- Fine-tuning: Yes, on LinkedIn job postings + resume datasets
- Training data: 100K job postings, 50K resumes
- Fine-tuning task: NER for skill extraction
- Epochs: 3, Learning rate: 2e-5, Batch size: 16

**Inference Flow**:
1. User uploads resume → Extract text
2. Send text to SageMaker endpoint
3. SageMaker returns skills with confidence scores
4. Store skills in DynamoDB
5. Repeat for job description
6. Compute similarity in Lambda function


### 6.2 Roadmap Personalization Engine

**Approach**: Hybrid (rule-based + collaborative filtering)

**Rule-Based Component**:
1. Skill dependency graph:
   - Python → NumPy → Pandas → Scikit-learn → TensorFlow
   - HTML → CSS → JavaScript → React
2. Prioritization rules:
   - Foundational skills first (Python before TensorFlow)
   - High-demand skills prioritized (from job market data)
   - User's target role influences priority

**Collaborative Filtering Component**:
1. User-item matrix:
   - Users: All platform users
   - Items: Learning resources (courses)
   - Values: Completion status (0 or 1)
2. Similarity calculation:
   - Find users similar to current user (cosine similarity)
   - Recommend resources completed by similar users
3. Cold start problem:
   - New users: Use rule-based recommendations
   - New resources: Use content-based filtering (skill tags)

**User History Input**:
- Completed courses
- Time spent on each resource
- Skill acquisition rate
- Preferred learning style (video, text, hands-on)

**Adaptive Weighting Logic**:
1. Initial weights:
   - Relevance: 0.4
   - Rating: 0.3
   - Completion rate: 0.2
   - Recency: 0.1
2. Adjust based on user behavior:
   - If user prefers free resources: Increase cost weight
   - If user completes video courses faster: Increase video weight
   - If user abandons long courses: Decrease duration weight
3. Re-calculate recommendations weekly

**Output**:
- Ordered list of 10-20 resources
- Estimated completion time per resource
- Total roadmap duration
- Skill milestones (checkpoints)


### 6.3 Emotion Detection Model

**Architecture**: Multi-modal ensemble (visual + audio)

**Visual Pipeline**:

**Model**: Fine-tuned ResNet-50
- Base: ResNet-50 pre-trained on ImageNet
- Fine-tuned on: FER2013 + custom interview dataset
- Output: 6 emotions (neutral, happy, sad, angry, surprised, confused)

**Processing Steps**:
1. Extract frames from video (1 fps)
2. Preprocess:
   - Resize to 224x224
   - Normalize pixel values
   - Face detection (Rekognition)
   - Crop to face region
3. Pass through ResNet-50
4. Softmax layer for emotion probabilities
5. Temporal smoothing (moving average over 5 frames)

**Audio Pipeline**:

**Model**: Fine-tuned Wav2Vec 2.0
- Base: Wav2Vec 2.0 pre-trained on LibriSpeech
- Fine-tuned on: RAVDESS + custom interview dataset
- Output: 3 dimensions (confidence, nervousness, enthusiasm)

**Processing Steps**:
1. Extract audio from video
2. Preprocess:
   - Resample to 16kHz
   - Noise reduction
   - Normalize amplitude
3. Extract features:
   - MFCCs (Mel-frequency cepstral coefficients)
   - Pitch, energy, zero-crossing rate
4. Pass through Wav2Vec 2.0
5. Regression layer for emotion scores

**Fusion Layer**:

**Approach**: Weighted average
- Visual weight: 0.6 (more reliable for emotions)
- Audio weight: 0.4

**Fusion Logic**:
1. Align visual and audio timestamps
2. For each 5-second window:
   - Visual: Dominant emotion + confidence
   - Audio: Confidence, nervousness, enthusiasm scores
3. Combine:
   - Overall confidence = (visual_confidence × 0.6) + (audio_confidence × 0.4)
   - Nervousness = audio_nervousness
   - Enthusiasm = audio_enthusiasm
4. Generate insights:
   - If nervousness > 0.7: "High nervousness detected"
   - If confidence < 0.5: "Low confidence detected"

**Confidence Scoring**:
- High confidence: >0.7
- Medium confidence: 0.4-0.7
- Low confidence: <0.4

**Output**:
```json
{
  "overallConfidence": 0.68,
  "nervousness": 0.25,
  "enthusiasm": 0.72,
  "emotionTimeline": [
    {"timestamp": 5, "emotion": "CALM", "confidence": 0.85},
    {"timestamp": 10, "emotion": "HAPPY", "confidence": 0.72}
  ],
  "insights": ["Strong enthusiasm when discussing projects"],
  "recommendations": ["Practice technical explanations to build confidence"]
}
```


### 6.4 Code Debugging AI

**Approach**: Prompt engineering with foundation models (Bedrock)

**Prompt Engineering Strategy**:

**System Prompt**:
```
You are an expert programming tutor and debugging assistant.
Your goal is to help users understand and fix bugs in their code.

Guidelines:
1. Identify all bugs (syntax, logical, runtime)
2. Explain each bug clearly and concisely
3. Suggest fixes with explanations
4. Provide corrected code
5. Use simple language suitable for beginners
6. Be encouraging and supportive

Output format: JSON with bugs[], explanations[], fixedCode
```

**User Prompt Template**:
```
Language: {language}
Code:
```{language}
{code}
```

Task: Analyze the code and identify all bugs.
```

**Context Window Management**:
- Max code length: 500 lines (to fit in context window)
- If code > 500 lines: Ask user to provide specific function/section
- Context window: 8K tokens (Claude 3 Sonnet)
- Reserve 2K tokens for response

**Multilingual Output Generation**:

**Step 1**: Generate explanation in English (Bedrock)
**Step 2**: Translate to user's language (AWS Translate)
**Step 3**: Preserve code blocks and technical terms

**Translation Strategy**:
- Translate: Explanations, suggestions, comments
- Preserve: Code, function names, variable names, error messages
- Use placeholders: `{{CODE_BLOCK_1}}`, `{{ERROR_MSG_1}}`

**Example**:
```
English: "The function is missing a return statement"
Hindi: "फ़ंक्शन में return स्टेटमेंट गायब है"
Code: "def add(a, b):" (unchanged)
```

**Error Handling**:
- If Bedrock fails: Fallback to syntax-only validation
- If translation fails: Return English explanation
- If code too long: Return error with suggestion to shorten

**Caching Strategy**:
- Cache common bug patterns and explanations
- Cache translations for frequently used phrases
- TTL: 24 hours


---

## 7. Multilingual Architecture Design

### 7.1 Language Detection

**Strategy**: Automatic detection with user override

**Detection Methods**:
1. User profile language preference (stored in Cognito)
2. Browser language settings (Accept-Language header)
3. AWS Comprehend DetectDominantLanguage API (for user-generated content)

**Priority**:
1. User explicit selection (highest priority)
2. User profile preference
3. Browser language
4. Default to English

**Implementation**:
```javascript
// Frontend
const detectLanguage = () => {
  const userPreference = getUserProfile().language;
  const browserLang = navigator.language.split('-')[0];
  return userPreference || browserLang || 'en';
};
```

### 7.2 Translation Pipeline

**Static Content** (UI elements):
- Pre-translated JSON files (i18next)
- Stored in frontend codebase
- Loaded based on user language
- Example: `locales/hi/translation.json`

**Dynamic Content** (AI-generated):
- Real-time translation via AWS Translate
- Cached in DynamoDB for reuse
- Cache key: `{sourceText}_{targetLang}`
- TTL: 7 days

**User-Generated Content** (resumes, answers):
- Not translated (preserve original)
- Language detected for analytics

### 7.3 AWS Translate Integration

**Configuration**:
- Source language: English (AI outputs)
- Target languages: Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati
- Custom terminology: Technical terms (API, database, function)
- Batch translation: For large content (learning resources)

**API Usage**:
```python
import boto3

translate = boto3.client('translate')

response = translate.translate_text(
    Text='The function is missing a return statement',
    SourceLanguageCode='en',
    TargetLanguageCode='hi',
    TerminologyNames=['tech-terms']
)

translated_text = response['TranslatedText']
```

**Cost Optimization**:
- Cache translations in DynamoDB
- Batch translate learning resources (one-time)
- Use custom terminology to reduce errors

### 7.4 Multilingual Embedding Handling

**Challenge**: Skill names in different languages

**Solution**: Normalize to English before embedding
1. User enters skill in Hindi: "मशीन लर्निंग"
2. Translate to English: "Machine Learning"
3. Generate embedding for English term
4. Store both original and normalized in DynamoDB

**Skill Taxonomy**:
- Primary language: English
- Synonyms: Include Hindi, Tamil, etc.
- Example:
  ```json
  {
    "skillId": "uuid",
    "name": "Machine Learning",
    "synonyms": ["ML", "मशीन लर्निंग", "இயந்திர கற்றல்"]
  }
  ```

### 7.5 Regional Language Support

**Supported Languages**:
1. English (en)
2. Hindi (hi)
3. Tamil (ta)
4. Telugu (te)
5. Bengali (bn)
6. Marathi (mr)
7. Gujarati (gu)

**Language-Specific Considerations**:
- Right-to-left (RTL): Not required for Indian languages
- Font support: Use Noto Sans for all Indian languages
- Date/time formatting: Use Intl API for localization
- Number formatting: Indian numbering system (lakhs, crores)

**Example**:
```javascript
// Date formatting
const date = new Date();
const hindiDate = date.toLocaleDateString('hi-IN');
// Output: "१५/२/२०२६"

// Number formatting
const number = 1000000;
const indianFormat = number.toLocaleString('en-IN');
// Output: "10,00,000"
```

---

## 8. Employability Index Design

### 8.1 Feature Inputs

**Resume Quality Features**:
- Completeness: Number of filled sections / 7
- Formatting: ATS-friendly score (keyword density, structure)
- Length: Optimal length (1-2 pages)
- Contact info: Email, phone present (boolean)

**Skill Proficiency Features**:
- Number of skills: Count of unique skills
- Skill relevance: Average match % across target jobs
- Skill depth: Certifications count + projects count
- Skill recency: Skills acquired in last 6 months

**Interview Performance Features**:
- Answer quality: Average score across all interviews
- Communication: Clarity score (from transcript analysis)
- Confidence: Average confidence score (from emotion analysis)
- Consistency: Standard deviation of scores (lower is better)

**Learning Progress Features**:
- Completion rate: Completed courses / enrolled courses
- Acquisition rate: New skills / months on platform
- Consistency: Daily activity streak
- Time invested: Total hours spent learning

**Market Demand Features**:
- Job availability: Number of relevant job postings (from API)
- Salary potential: Average salary for user's skills
- Growth trend: Industry growth rate (from external data)
- Competition: Number of users with similar skills


### 8.2 Weighted Scoring System

**Component Weights** (based on industry research):
- Resume Quality: 20%
- Skill Proficiency: 30%
- Interview Performance: 25%
- Learning Progress: 15%
- Market Demand: 10%

**Normalization**:
All features normalized to 0-100 scale before weighting

**Normalization Functions**:

**Linear Normalization**:
```python
def normalize_linear(value, min_val, max_val):
    return ((value - min_val) / (max_val - min_val)) * 100
```

**Sigmoid Normalization** (for unbounded features):
```python
def normalize_sigmoid(value, midpoint=10):
    return 100 / (1 + math.exp(-(value - midpoint)))
```

**Example Calculation**:
```python
# User data
resume_completeness = 85
resume_formatting = 70
skill_count = 15
skill_relevance = 75
interview_avg = 80
learning_completion = 60

# Component scores
resume_score = (resume_completeness * 0.6) + (resume_formatting * 0.4)
# = (85 * 0.6) + (70 * 0.4) = 51 + 28 = 79

skill_score = normalize_sigmoid(skill_count) * 0.3 + skill_relevance * 0.7
# = 88 * 0.3 + 75 * 0.7 = 26.4 + 52.5 = 78.9

interview_score = interview_avg  # Already 0-100
# = 80

learning_score = learning_completion  # Already 0-100
# = 60

market_score = 70  # From external API

# Overall Employability Index
EI = (79 * 0.20) + (78.9 * 0.30) + (80 * 0.25) + (60 * 0.15) + (70 * 0.10)
   = 15.8 + 23.67 + 20 + 9 + 7
   = 75.47
   ≈ 75
```

### 8.3 Dynamic Recalibration

**Recalibration Triggers**:
1. User completes course (learning progress changes)
2. User takes mock interview (interview performance changes)
3. User updates resume (resume quality changes)
4. Daily scheduled recalculation (market demand changes)

**Recalibration Logic**:
1. Fetch latest data for all components
2. Recalculate component scores
3. Recalculate overall EI
4. Compare with previous score
5. If change > 5 points: Notify user
6. Store new score with timestamp

**Historical Tracking**:
- Store score daily in DynamoDB
- Partition key: userId
- Sort key: timestamp
- Enables trend analysis and visualization

### 8.4 Score Explainability

**Breakdown Display**:
```json
{
  "overallScore": 75,
  "components": {
    "resumeQuality": {"score": 79, "weight": 0.20, "contribution": 15.8},
    "skillProficiency": {"score": 78.9, "weight": 0.30, "contribution": 23.67},
    "interviewPerformance": {"score": 80, "weight": 0.25, "contribution": 20},
    "learningProgress": {"score": 60, "weight": 0.15, "contribution": 9},
    "marketDemand": {"score": 70, "weight": 0.10, "contribution": 7}
  },
  "insights": [
    "Your interview performance is strong (80/100)",
    "Focus on completing enrolled courses to improve learning progress (60/100)",
    "Your resume quality is good, but can be improved (79/100)"
  ],
  "recommendations": [
    "Complete 2 more courses to reach 80% completion rate",
    "Add 2 more projects to your resume",
    "Practice mock interviews to maintain high performance"
  ]
}
```

**Visualization**:
- Radar chart: 5 components
- Line chart: Score over time
- Bar chart: Component contributions

---

## 9. Responsible AI & Bias Monitoring Framework

### 9.1 Bias Detection Checkpoints

**Checkpoint 1: Data Collection**
- Remove PII during resume upload
- Anonymize user data for model training
- Ensure diverse training data (demographics, geographies)

**Checkpoint 2: Model Training**
- Monitor training data distribution
- Apply adversarial debiasing techniques
- Validate model on diverse test sets

**Checkpoint 3: Model Inference**
- Log all model predictions
- Monitor prediction distribution across demographics
- Flag anomalous patterns

**Checkpoint 4: Recommendation Generation**
- Ensure diversity in recommendations
- Avoid filter bubbles
- Monitor for keyword bias

**Checkpoint 5: User Feedback**
- Collect user feedback on fairness
- Allow users to report bias
- Incorporate feedback into model updates


### 9.2 Model Audit Logs

**Log Structure**:
```json
{
  "logId": "uuid",
  "timestamp": "2026-02-15T10:30:00Z",
  "eventType": "skill_gap_analyzed",
  "userId": "uuid",
  "decision": {
    "matchPercentage": 75,
    "missingSkills": ["Docker", "AWS"],
    "recommendations": ["Course A", "Course B"]
  },
  "fairnessMetrics": {
    "demographicParity": 0.08,
    "equalizedOdds": 0.12
  },
  "flagged": false
}
```

**Storage**:
- DynamoDB table: `BiasAuditLogs`
- Retention: 1 year
- Indexed by: userId, timestamp, eventType

**Aggregation**:
- Daily aggregation via Lambda (EventBridge scheduled)
- Calculate fairness metrics across all decisions
- Generate weekly reports

### 9.3 Fairness Metrics

**Demographic Parity**:
- Definition: P(positive outcome | group A) ≈ P(positive outcome | group B)
- Threshold: Difference < 0.1
- Example: P(high score | male) ≈ P(high score | female)

**Equalized Odds**:
- Definition: P(positive outcome | qualified, group A) ≈ P(positive outcome | qualified, group B)
- Threshold: Difference < 0.15
- Example: P(high score | qualified, urban) ≈ P(high score | qualified, rural)

**Disparate Impact**:
- Definition: (Positive rate for group A) / (Positive rate for group B)
- Threshold: 0.8 < ratio < 1.25
- Example: (High scores for women) / (High scores for men)

**Calculation**:
```python
def calculate_demographic_parity(group_a_positive, group_a_total, group_b_positive, group_b_total):
    rate_a = group_a_positive / group_a_total
    rate_b = group_b_positive / group_b_total
    return abs(rate_a - rate_b)

# Example
dp = calculate_demographic_parity(80, 100, 75, 100)
# = abs(0.8 - 0.75) = 0.05 (within threshold)
```

### 9.4 Explainability Strategy

**SHAP (SHapley Additive exPlanations)**:
- Calculate feature importance for each prediction
- Visualize top contributing features
- Provide to users on request

**Example**:
```json
{
  "prediction": "High employability score (85)",
  "topFeatures": [
    {"feature": "Interview performance", "contribution": +15},
    {"feature": "Skill proficiency", "contribution": +12},
    {"feature": "Resume quality", "contribution": +8}
  ]
}
```

**Natural Language Explanations**:
- Generate human-readable explanations
- Example: "Your high interview performance (+15 points) and strong skill proficiency (+12 points) contributed most to your score."

**Transparency**:
- Disclose when AI is used
- Explain model limitations
- Provide opt-out options (where applicable)

### 9.5 Human Review Fallback

**Trigger Conditions**:
- Bias flag raised (fairness metric violated)
- User reports bias
- High-stakes decision (future: job recommendations)

**Review Process**:
1. Flag decision in admin dashboard
2. Human reviewer examines decision and context
3. Reviewer approves, modifies, or rejects decision
4. Feedback incorporated into model retraining

**Admin Dashboard**:
- List of flagged decisions
- Fairness metrics visualization
- User feedback summary
- Action buttons (approve, modify, reject)

---

## 10. Data Flow Architecture

### 10.1 Batch vs Real-Time Processing

**Real-Time Processing**:
- Use cases: API requests, mock interviews, code debugging
- Technology: Lambda, API Gateway
- Latency: <2 seconds
- Examples:
  - User uploads resume → Parse immediately
  - User submits code → Debug immediately
  - User answers interview question → Evaluate immediately

**Batch Processing**:
- Use cases: Employability score calculation, bias audits, analytics
- Technology: Lambda (EventBridge scheduled), Step Functions
- Frequency: Daily, weekly
- Examples:
  - Calculate employability scores for all users (daily)
  - Generate bias audit reports (weekly)
  - Aggregate usage analytics (daily)

**Hybrid Processing**:
- Use cases: Learning path generation, emotion analysis
- Technology: Lambda + SQS
- Flow:
  - Real-time: Acknowledge request, queue job
  - Async: Process job, notify user when complete
- Examples:
  - Skill gap analyzed → Queue roadmap generation → Notify when ready
  - Interview completed → Queue emotion aggregation → Notify when ready


### 10.2 Data Retention Strategy

**Hot Data** (frequent access):
- Storage: DynamoDB, S3 Standard
- Retention: 90 days
- Examples: Recent resumes, interview sessions, scores

**Warm Data** (occasional access):
- Storage: S3 Intelligent-Tiering
- Retention: 90 days - 1 year
- Examples: Older resumes, historical scores

**Cold Data** (rare access):
- Storage: S3 Glacier
- Retention: 1+ years
- Examples: Archived resumes, old audit logs

**Data Deletion**:
- User-requested deletion: Immediate (GDPR compliance)
- Automatic deletion: After retention period
- Soft delete: Mark as deleted, purge after 30 days

**Lifecycle Policies**:
```json
{
  "Rules": [
    {
      "Id": "MoveToGlacier",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ]
    },
    {
      "Id": "DeleteOldVideos",
      "Status": "Enabled",
      "Expiration": {
        "Days": 30
      },
      "Filter": {
        "Prefix": "videos/"
      }
    }
  ]
}
```

### 10.3 Data Anonymization Pipeline

**Purpose**: Anonymize data for model training and analytics

**PII to Remove**:
- Name, email, phone, address
- Age, gender, ethnicity
- Photos, videos (faces blurred)

**Anonymization Process**:
1. Identify PII using NER (AWS Comprehend)
2. Replace with placeholders:
   - Name → `[NAME]`
   - Email → `[EMAIL]`
   - Phone → `[PHONE]`
3. Hash user IDs (SHA-256)
4. Store anonymized data in separate S3 bucket

**Example**:
```
Original: "John Doe, john@example.com, 555-1234"
Anonymized: "[NAME], [EMAIL], [PHONE]"
```

**Usage**:
- Model training: Use anonymized data
- Analytics: Use aggregated, anonymized data
- Reporting: Use anonymized data

### 10.4 Training Data Pipeline

**Data Sources**:
1. User-generated data (resumes, interview transcripts)
2. External datasets (LinkedIn, Kaggle)
3. Synthetic data (generated for edge cases)

**Pipeline Steps**:
1. **Collection**: Aggregate data from sources
2. **Anonymization**: Remove PII
3. **Labeling**: Manual labeling (for supervised learning)
4. **Validation**: Quality checks, remove duplicates
5. **Storage**: Store in S3 (nexura-training-data)
6. **Versioning**: Track dataset versions

**Labeling**:
- Tool: Amazon SageMaker Ground Truth
- Tasks: Skill extraction, emotion labeling
- Workers: Internal team + crowdsourcing

**Data Versioning**:
- Format: `dataset-v1.0.json`, `dataset-v1.1.json`
- Metadata: Creation date, size, source, changes
- Storage: S3 with versioning enabled

**Model Retraining**:
- Frequency: Monthly
- Trigger: New labeled data available (>10K samples)
- Process: SageMaker training job
- Validation: Test on holdout set, compare with previous model
- Deployment: If accuracy improves, deploy new model

---

## 11. Scalability Strategy

### 11.1 Horizontal Scaling

**Lambda**:
- Auto-scaling: Automatic (AWS-managed)
- Concurrency: 1000 concurrent executions (default)
- Reserved concurrency: 100 for critical functions
- Provisioned concurrency: 10 for low-latency functions

**API Gateway**:
- Auto-scaling: Automatic (AWS-managed)
- Throttling: 10,000 requests/second (burst)
- Regional endpoints: Future multi-region deployment

**DynamoDB**:
- Capacity mode: On-demand (auto-scaling)
- Partitioning: Automatic based on partition key
- Global tables: Future multi-region replication

**SageMaker**:
- Auto-scaling: Target invocations per instance
- Min instances: 1
- Max instances: 5
- Scale-up: Add instance when invocations > 100/min
- Scale-down: Remove instance when invocations < 50/min


### 11.2 Auto-Scaling Groups

**ECS Fargate** (if used for long-running inference):
- Service auto-scaling: Target CPU utilization 70%
- Min tasks: 1
- Max tasks: 10
- Scale-up: Add task when CPU > 70% for 2 minutes
- Scale-down: Remove task when CPU < 30% for 5 minutes

**ElastiCache Redis** (if used):
- Node type: cache.t3.micro (1 node initially)
- Auto-scaling: Not supported for Redis
- Manual scaling: Upgrade to larger node type if needed

### 11.3 Lambda Concurrency

**Concurrency Limits**:
- Account limit: 1000 concurrent executions
- Reserved concurrency: Allocate to critical functions
- Provisioned concurrency: Pre-warm instances for low latency

**Critical Functions** (reserved concurrency):
- Resume Parser: 50
- Skill Gap Analyzer: 50
- Interview Orchestrator: 100
- Code Debugger: 50
- Total reserved: 250

**Provisioned Concurrency** (low latency):
- Interview Orchestrator: 10 instances
- Code Debugger: 5 instances

**Throttling Handling**:
- Retry with exponential backoff
- Queue requests in SQS if throttled
- Alert if throttling persists

### 11.4 Database Partitioning

**DynamoDB Partitioning**:
- Partition key: userId (evenly distributed)
- Sort key: timestamp, resourceId (for range queries)
- Hot partitions: Monitor with CloudWatch
- Mitigation: Add randomness to partition key if needed

**Example**:
```
Partition key: userId#shard
Shard: userId % 10 (distribute across 10 shards)
```

**S3 Partitioning**:
- Prefix: `{userId}/` (automatic partitioning)
- Performance: 3,500 PUT/s, 5,500 GET/s per prefix

### 11.5 CDN Usage

**CloudFront**:
- Cache static assets: JS, CSS, images (1 year TTL)
- Cache API responses: GET requests (5 minutes TTL)
- Edge locations: Global (200+ locations)
- Benefits:
  - Reduced latency (serve from nearest edge)
  - Reduced origin load (cache hits)
  - DDoS protection (AWS Shield)

**Cache Invalidation**:
- On deployment: Invalidate all static assets
- On data update: Invalidate specific API responses
- Cost: First 1000 invalidations/month free

---

## 12. Security Architecture

### 12.1 IAM Policies

**Principle**: Least privilege (grant minimum permissions required)

**Lambda Execution Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/Nexura*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::nexura-*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sagemaker:InvokeEndpoint"
      ],
      "Resource": "arn:aws:sagemaker:*:*:endpoint/nexura-*"
    }
  ]
}
```

**SageMaker Execution Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::nexura-models/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    }
  ]
}
```

### 12.2 Data Encryption

**At Rest**:
- DynamoDB: AWS-managed encryption (default)
- S3: SSE-KMS for sensitive data (resumes), SSE-S3 for others
- EBS volumes: Encrypted (if using EC2/ECS)
- Secrets Manager: Encrypted with KMS

**In Transit**:
- API Gateway: TLS 1.3
- CloudFront: TLS 1.3
- Internal services: TLS 1.2+ (Lambda to DynamoDB, S3)

**Encryption Keys**:
- KMS: Customer-managed keys for sensitive data
- Key rotation: Automatic annual rotation
- Key policies: Restrict access to specific roles

### 12.3 JWT Authentication

**Token Structure**:
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-uuid",
    "email": "user@example.com",
    "role": "student",
    "language": "hi",
    "iat": 1234567890,
    "exp": 1234571490
  },
  "signature": "..."
}
```

**Token Lifecycle**:
- Access token: 1 hour expiration
- Refresh token: 30 days expiration
- Refresh flow: Exchange refresh token for new access token
- Revocation: Store revoked tokens in DynamoDB (blacklist)

**Validation**:
- API Gateway: Cognito authorizer validates JWT
- Lambda: Verify signature, expiration, claims
- Blacklist check: Query DynamoDB for revoked tokens


### 12.4 Role-Based Access Control

**Roles**:
1. **Student** (default):
   - Upload resume
   - Analyze skill gaps
   - Take mock interviews
   - Access learning paths
   - View own data

2. **Admin**:
   - All student permissions
   - View all users
   - Access analytics dashboard
   - Review bias alerts
   - Manage system settings

**Implementation**:
- Role stored in Cognito user attributes
- JWT includes role claim
- Lambda functions check role before executing actions
- API Gateway resource policies restrict endpoints by role

**Example**:
```javascript
// Lambda function
const checkRole = (event, requiredRole) => {
  const token = event.requestContext.authorizer.claims;
  if (token.role !== requiredRole) {
    throw new Error('Unauthorized');
  }
};

// Usage
checkRole(event, 'admin');
```

### 12.5 API Throttling

**Rate Limits**:
- Per user: 100 requests/minute
- Per IP: 1000 requests/minute
- Burst: 200 requests

**Implementation**:
- API Gateway usage plans
- Throttling settings per endpoint
- Custom throttling logic in Lambda (if needed)

**Response**:
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "retryAfter": 60
  }
}
```

**DDoS Protection**:
- AWS Shield Standard (automatic)
- WAF rate limiting rules
- CloudFront geo-blocking (if needed)

---

## 13. API Design Overview

### 13.1 REST Endpoint Examples

**Authentication**:
```
POST   /v1/auth/register
POST   /v1/auth/login
POST   /v1/auth/refresh
POST   /v1/auth/logout
POST   /v1/auth/forgot-password
POST   /v1/auth/reset-password
```

**Resume Management**:
```
POST   /v1/resumes/upload
GET    /v1/resumes/{resumeId}
PUT    /v1/resumes/{resumeId}
DELETE /v1/resumes/{resumeId}
GET    /v1/resumes/user/{userId}
```

**Skill Gap Analysis**:
```
POST   /v1/analysis/skill-gap
GET    /v1/analysis/{analysisId}
GET    /v1/analysis/user/{userId}
DELETE /v1/analysis/{analysisId}
```

**Learning Paths**:
```
GET    /v1/learning-paths/{userId}
POST   /v1/learning-paths/progress
GET    /v1/learning-paths/recommendations
PUT    /v1/learning-paths/{pathId}
```

**Mock Interviews**:
```
POST   /v1/interviews/start
POST   /v1/interviews/{sessionId}/submit-answer
POST   /v1/interviews/{sessionId}/complete
GET    /v1/interviews/{sessionId}
GET    /v1/interviews/user/{userId}
GET    /v1/interviews/{sessionId}/feedback
```

**Code Debugging**:
```
POST   /v1/debug/analyze
GET    /v1/debug/history/{userId}
```

**Employability Score**:
```
GET    /v1/employability/{userId}
GET    /v1/employability/{userId}/history
GET    /v1/employability/{userId}/insights
```

**Admin**:
```
GET    /v1/admin/users
GET    /v1/admin/analytics
GET    /v1/admin/bias-alerts
PUT    /v1/admin/settings
```

### 13.2 Request/Response Structure

**Request Example** (POST /v1/analysis/skill-gap):
```json
{
  "userId": "uuid",
  "jobDescription": "We are looking for a Python developer with experience in Django, REST APIs, and PostgreSQL...",
  "targetRole": "Backend Developer"
}
```

**Response Example** (Success):
```json
{
  "data": {
    "analysisId": "uuid",
    "matchPercentage": 75,
    "matchedSkills": ["Python", "REST APIs"],
    "missingSkills": [
      {
        "name": "Django",
        "importance": 0.9,
        "category": "framework"
      },
      {
        "name": "PostgreSQL",
        "importance": 0.85,
        "category": "database"
      }
    ],
    "proficiencyGaps": [
      {
        "name": "REST APIs",
        "userLevel": "beginner",
        "requiredLevel": "intermediate"
      }
    ]
  },
  "meta": {
    "timestamp": "2026-02-15T10:30:00Z",
    "version": "v1"
  }
}
```

**Response Example** (Error):
```json
{
  "error": {
    "code": "INVALID_JOB_DESCRIPTION",
    "message": "Unable to extract skills from job description",
    "details": "Job description is too short or malformed",
    "timestamp": "2026-02-15T10:30:00Z"
  }
}
```

### 13.3 Versioning Strategy

**URL Versioning**:
- Format: `/v1/`, `/v2/`
- Current version: v1
- Deprecation: v1 supported for 1 year after v2 release

**Version Migration**:
- Announce new version 3 months in advance
- Provide migration guide
- Support both versions during transition
- Deprecate old version after 1 year

**Breaking Changes**:
- Require new version (v2)
- Non-breaking changes: Same version (v1.1)

---

## 14. Database Design (High-Level Schema)

### 14.1 Users Table

**DynamoDB Table**: `Users`

**Schema**:
```
PK: userId (UUID)
Attributes:
  - email (String)
  - name (String)
  - language (String) // en, hi, ta, etc.
  - role (String) // student, admin
  - createdAt (String, ISO 8601)
  - lastLogin (String, ISO 8601)
  - profileComplete (Boolean)

GSI: email-index
  - PK: email
  - Projection: ALL
```


### 14.2 Resume Metadata Table

**DynamoDB Table**: `Resumes`

**Schema**:
```
PK: resumeId (UUID)
SK: userId (UUID)
Attributes:
  - s3Key (String) // S3 object key
  - fileName (String)
  - fileSize (Number) // bytes
  - parsedData (Map)
    - contact (Map): email, phone
    - education (List): degree, institution, year
    - experience (List): company, role, duration
    - skills (List): skill names
    - certifications (List): cert names
  - uploadedAt (String, ISO 8601)
  - version (Number)

GSI: userId-uploadedAt-index
  - PK: userId
  - SK: uploadedAt
  - Projection: ALL
```

### 14.3 Skill Gap Results Table

**DynamoDB Table**: `SkillGapAnalyses`

**Schema**:
```
PK: analysisId (UUID)
SK: userId (UUID)
Attributes:
  - jobId (String) // Reference to job description
  - jobTitle (String)
  - matchPercentage (Number)
  - matchedSkills (List): skill names
  - missingSkills (List)
    - name (String)
    - importance (Number)
    - category (String)
  - proficiencyGaps (List)
    - name (String)
    - userLevel (String)
    - requiredLevel (String)
  - createdAt (String, ISO 8601)

GSI: userId-createdAt-index
  - PK: userId
  - SK: createdAt
  - Projection: ALL
```

### 14.4 Interview Results Table

**DynamoDB Table**: `InterviewSessions`

**Schema**:
```
PK: sessionId (UUID)
SK: userId (UUID)
Attributes:
  - jobId (String)
  - jobTitle (String)
  - questions (List)
    - questionId (String)
    - text (String)
    - type (String) // technical, behavioral
  - answers (List)
    - questionId (String)
    - text (String)
    - audioS3Key (String)
    - videoS3Key (String)
    - evaluation (Map)
      - score (Number)
      - feedback (String)
      - strengths (List)
      - weaknesses (List)
  - emotions (List)
    - timestamp (Number)
    - emotion (String)
    - confidence (Number)
  - overallScore (Number)
  - createdAt (String, ISO 8601)
  - completedAt (String, ISO 8601)

GSI: userId-createdAt-index
  - PK: userId
  - SK: createdAt
  - Projection: ALL
```

### 14.5 Roadmap Table

**DynamoDB Table**: `LearningPaths`

**Schema**:
```
PK: pathId (UUID)
SK: userId (UUID)
Attributes:
  - targetJobId (String)
  - targetRole (String)
  - skills (List)
    - name (String)
    - priority (Number)
    - status (String) // not_started, in_progress, completed
  - resources (List)
    - resourceId (String)
    - title (String)
    - provider (String)
    - url (String)
    - duration (String)
    - cost (String)
    - language (String)
    - rating (Number)
    - completed (Boolean)
  - progress (Number) // 0-100
  - estimatedDuration (String)
  - createdAt (String, ISO 8601)
  - updatedAt (String, ISO 8601)

GSI: userId-updatedAt-index
  - PK: userId
  - SK: updatedAt
  - Projection: ALL
```

### 14.6 Audit Logs Table

**DynamoDB Table**: `BiasAuditLogs`

**Schema**:
```
PK: logId (UUID)
SK: timestamp (String, ISO 8601)
Attributes:
  - eventType (String) // skill_gap_analyzed, interview_evaluated, etc.
  - userId (String)
  - decision (Map) // Varies by event type
  - fairnessMetrics (Map)
    - demographicParity (Number)
    - equalizedOdds (Number)
    - disparateImpact (Number)
  - flagged (Boolean)
  - reviewStatus (String) // pending, approved, rejected
  - reviewedBy (String) // Admin userId
  - reviewedAt (String, ISO 8601)

GSI: flagged-timestamp-index
  - PK: flagged
  - SK: timestamp
  - Projection: ALL
```

---

## 15. Deployment & CI/CD Strategy

### 15.1 Infrastructure as Code

**Tool**: AWS CDK (TypeScript)

**Stack Structure**:
```
nexura-infrastructure/
├── lib/
│   ├── frontend-stack.ts       // S3, CloudFront
│   ├── api-stack.ts            // API Gateway, Lambda
│   ├── ai-stack.ts             // SageMaker, Bedrock
│   ├── data-stack.ts           // DynamoDB, S3 buckets
│   ├── auth-stack.ts           // Cognito
│   ├── monitoring-stack.ts     // CloudWatch, X-Ray
│   └── security-stack.ts       // IAM, KMS, Secrets Manager
├── bin/
│   └── nexura.ts               // App entry point
├── package.json
└── cdk.json
```

**Deployment Commands**:
```bash
# Install dependencies
npm install

# Synthesize CloudFormation template
cdk synth

# Deploy to dev
cdk deploy --all --context env=dev

# Deploy to prod
cdk deploy --all --context env=prod
```

**Environment Configuration**:
```typescript
// lib/config.ts
export const config = {
  dev: {
    region: 'ap-south-1',
    domainName: 'dev.nexura.ai',
    lambdaMemory: 512,
    sagemakerInstanceType: 'ml.t3.medium'
  },
  prod: {
    region: 'ap-south-1',
    domainName: 'nexura.ai',
    lambdaMemory: 1024,
    sagemakerInstanceType: 'ml.m5.large'
  }
};
```


### 15.2 CI/CD via GitHub Actions

**Pipeline Stages**:

**1. Build Stage**:
```yaml
name: Build
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm install
      - run: npm run build
      - run: npm run lint
      - run: npm test
```

**2. Test Stage**:
```yaml
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit
      - run: npm run test:integration
      - run: npm run test:coverage
      - uses: codecov/codecov-action@v3
```

**3. Deploy Stage** (dev):
```yaml
  deploy-dev:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    needs: [build, test]
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - run: npm run cdk:deploy -- --context env=dev
```

**4. Deploy Stage** (prod):
```yaml
  deploy-prod:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    needs: [build, test]
    environment: production
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
      - run: npm run cdk:deploy -- --context env=prod
```

**5. Smoke Tests**:
```yaml
  smoke-test:
    runs-on: ubuntu-latest
    needs: [deploy-prod]
    steps:
      - run: npm run test:smoke
```

### 15.3 Environment Separation

**Environments**:
1. **Development** (dev.nexura.ai)
   - Purpose: Active development, testing
   - Data: Synthetic data
   - Deployment: Automatic on push to `develop` branch

2. **Staging** (staging.nexura.ai)
   - Purpose: Pre-production testing, QA
   - Data: Anonymized production data
   - Deployment: Manual approval required

3. **Production** (nexura.ai)
   - Purpose: Live user traffic
   - Data: Real user data
   - Deployment: Manual approval + smoke tests

**Environment Variables**:
```bash
# Dev
API_URL=https://api-dev.nexura.ai
BEDROCK_MODEL=claude-3-haiku
SAGEMAKER_ENDPOINT=nexura-skill-extraction-dev

# Prod
API_URL=https://api.nexura.ai
BEDROCK_MODEL=claude-3-sonnet
SAGEMAKER_ENDPOINT=nexura-skill-extraction-prod
```

### 15.4 Model Deployment Lifecycle

**Training**:
1. Data scientist trains model locally or in SageMaker
2. Model artifacts saved to S3 (nexura-models/skill-extraction/v1.2/)
3. Model registered in SageMaker Model Registry

**Validation**:
1. Deploy model to dev endpoint
2. Run validation tests (accuracy, latency)
3. Compare with current production model
4. If better: Proceed to staging

**Staging Deployment**:
1. Deploy to staging endpoint
2. Run integration tests
3. Manual QA testing
4. Approval from data science team

**Production Deployment**:
1. Blue-green deployment:
   - Deploy new model to green endpoint
   - Route 10% traffic to green
   - Monitor metrics (accuracy, latency, errors)
   - If successful: Route 100% to green
   - If failed: Rollback to blue
2. Update Lambda environment variables to point to new endpoint
3. Monitor for 24 hours

**Rollback**:
1. Revert Lambda environment variables to previous endpoint
2. Investigate issues
3. Fix and redeploy

---

## 16. Future Evolution Architecture

### 16.1 Microservices Separation

**Current**: Monolithic Lambda functions

**Future**: Separate microservices
- Resume Service (ECS Fargate)
- Skill Gap Service (ECS Fargate)
- Interview Service (ECS Fargate)
- Learning Path Service (ECS Fargate)

**Benefits**:
- Independent scaling
- Technology flexibility (Python, Node.js, Go)
- Team autonomy
- Fault isolation

**Communication**:
- Synchronous: REST APIs (API Gateway)
- Asynchronous: EventBridge, SQS

### 16.2 Mobile App Scaling

**Current**: Web-only (React SPA)

**Future**: Native mobile apps
- iOS (Swift)
- Android (Kotlin)
- React Native (cross-platform)

**Backend Changes**:
- GraphQL API (Apollo Server) for efficient data fetching
- Push notifications (SNS, Firebase Cloud Messaging)
- Offline mode (local storage, sync on reconnect)

**Mobile-Specific Features**:
- Camera integration (resume scanning)
- Voice input (interview practice)
- Biometric authentication (Face ID, fingerprint)

### 16.3 Real-Time Analytics

**Current**: Batch analytics (daily)

**Future**: Real-time analytics
- Kinesis Data Streams for event ingestion
- Kinesis Data Analytics for real-time processing
- QuickSight for live dashboards

**Use Cases**:
- Real-time user engagement metrics
- Live interview performance tracking
- Instant bias detection alerts

### 16.4 Marketplace Integration

**Current**: Curated learning resources

**Future**: Marketplace for courses and mentors
- Course providers can list courses
- Mentors can offer 1-on-1 coaching
- Payment processing (Stripe, Razorpay)
- Rating and review system

**Architecture Changes**:
- New microservices: Marketplace Service, Payment Service
- New database tables: Courses, Mentors, Transactions
- Stripe/Razorpay integration

### 16.5 Enterprise Versioning

**Current**: B2C (individual users)

**Future**: B2B (enterprise customers)
- Multi-tenant architecture
- Organization management (teams, roles)
- SSO integration (SAML, OIDC)
- Custom branding
- Advanced analytics (team performance)

**Architecture Changes**:
- Tenant isolation (separate DynamoDB tables per tenant)
- Cognito User Pools per tenant
- Custom domain per tenant
- Usage-based billing

---

**Document Version**: 2.0  
**Last Updated**: 2026-02-15  
**Status**: Production-Ready Design  
**Next Steps**: Implementation, Testing, Deployment

