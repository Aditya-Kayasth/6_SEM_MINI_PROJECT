# Implementation Plan: Axiom AI - Autonomous Data-to-Insight Agent MVP

## Overview

This implementation plan breaks down the Axiom AI MVP into 6 phases following the original design approach. The system is an autonomous agent-driven pipeline built with FastAPI, LangGraph, LangChain, Pandas, DuckDB, E2B, and Streamlit. Each task builds incrementally, with testing sub-tasks marked as optional (*) for faster MVP delivery. The implementation uses Python 3.11+ and operates within strict memory constraints (512MB RAM).

## Tasks

### Phase 1: Universal Ingestion Engine

- [ ] 1. Set up project structure and core dependencies
  - Create project directory structure: `src/`, `tests/`, `config/`
  - Create `requirements.txt` with pinned versions: fastapi, langchain, langgraph, pandas, duckdb, e2b-code-interpreter, streamlit, pydantic, psutil
  - Create `.env.example` with required environment variables: OPENAI_API_KEY, E2B_API_KEY, MAX_UPLOAD_SIZE_MB, CACHE_TTL_MINUTES, MAX_MEMORY_MB
  - Create `src/__init__.py` and basic package structure
  - _Requirements: 20.1, 20.2, 20.3, 20.9, 20.10_

- [ ] 2. Implement file ingestion with automatic format detection
  - [ ] 2.1 Create `src/ingestion/engine.py` with IngestionEngine class
    - Implement `ingest_file()` method accepting BinaryIO and filename
    - Add CSV encoding detection (UTF-8, Latin-1, Windows-1252)
    - Add CSV delimiter detection (comma, semicolon, tab, pipe)
    - Add Excel file reading for .xlsx and .xls formats
    - Implement file size validation (reject > 100MB)
    - Return tuple of (DataFrame, metadata dict)
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.9, 1.10, 1.11_

  - [ ]* 2.2 Write property test for ingestion data integrity
    - **Property 1: Ingestion Preserves Data Integrity**
    - **Validates: Requirements 1.9, 1.10, 1.11**
    - Test that cleaned shape never exceeds original shape
    - Test that all DataFrames have string column names

- [ ] 3. Implement header detection algorithm
  - [ ] 3.1 Create `detect_header_row()` method in IngestionEngine
    - Implement scoring heuristics: non-null ratio, string ratio, unique ratio, text length, non-numeric check
    - Examine first 10 rows as candidates
    - Return row index with highest score
    - Add confidence warning flag to metadata when score is low
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8_

  - [ ]* 3.2 Write unit tests for header detection
    - Test with files having headers at different positions
    - Test with files having multiple potential header rows
    - Test edge cases: all numeric rows, empty rows, single row files
    - _Requirements: 2.1, 2.6, 2.7_

- [ ] 4. Implement data boundary detection
  - [ ] 4.1 Create `infer_data_boundaries()` method in IngestionEngine
    - Detect and skip empty leading rows
    - Detect and exclude empty trailing rows and footer notes
    - Return tuple of (data_start_row, data_end_row)
    - _Requirements: 1.7, 1.8_

  - [ ]* 4.2 Write unit tests for boundary detection
    - Test files with leading empty rows
    - Test files with trailing footer notes
    - Test files with both leading and trailing noise
    - _Requirements: 1.7, 1.8_

- [ ] 5. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 2: Semantic Profiling Layer

- [ ] 6. Define semantic profile data models
  - [ ] 6.1 Create `src/profiling/models.py` with Pydantic models
    - Define ColumnProfile model with fields: name, raw_dtype, semantic_type, functional_role, nullable, sample_values, inferred_constraints
    - Define SemanticProfile model with fields: columns, row_count, primary_keys, metrics, dimensions, temporal_columns
    - Add validation rules: semantic_type enum, functional_role enum, sample_values max 5 items
    - Add cross-field validation: column name references must exist
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.10_

  - [ ]* 6.2 Write property test for semantic profile completeness
    - **Property 2: Semantic Profile Completeness**
    - **Validates: Requirements 3.1, 3.2, 3.3, 3.10**
    - Test that profile contains exactly one entry per column
    - Test that all semantic types are from valid set
    - Test that row count matches DataFrame length

- [ ] 7. Implement semantic profiler with LLM integration
  - [ ] 7.1 Create `src/profiling/profiler.py` with SemanticProfiler class
    - Implement `profile_dataframe()` method using LangChain LLM
    - Extract column names and 5-row sample from DataFrame
    - Construct LLM prompt with data sample and schema requirements
    - Use structured output mode to enforce SemanticProfile schema
    - Implement retry logic for invalid LLM responses
    - Cache profiles to avoid redundant LLM calls
    - _Requirements: 3.1, 3.11, 3.12, 3.13, 18.1, 18.2, 18.4, 18.5, 18.6, 18.7_

  - [ ]* 7.2 Write unit tests for semantic profiler
    - Test with mock LLM responses
    - Test schema validation and retry logic
    - Test caching behavior
    - _Requirements: 3.12, 3.13, 18.4_

- [ ] 8. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 3: Automated Data Preparation Engine

- [ ] 9. Implement type enforcement and cleaning
  - [ ] 9.1 Create `src/preparation/engine.py` with DataPreparationEngine class
    - Implement `prepare_data()` method accepting DataFrame and SemanticProfile
    - Implement `enforce_types()` to convert columns based on semantic_type
    - Implement `clean_numeric_artifacts()` to strip currency symbols, percentages, commas
    - Convert metric columns to numeric dtype
    - Convert datetime columns to datetime dtype
    - Convert category columns to categorical dtype
    - Convert boolean columns with text mapping
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.13_

  - [ ]* 9.2 Write property test for data preparation type safety
    - **Property 3: Data Preparation Type Safety**
    - **Validates: Requirements 4.1, 4.2, 4.3, 4.4, 4.12**
    - Test that metric columns have numeric dtypes after preparation
    - Test that datetime columns have datetime dtypes
    - Test that cleaned DataFrame has same or fewer rows

- [ ] 10. Implement missing value handling
  - [ ] 10.1 Create `handle_missing_values()` method in DataPreparationEngine
    - Impute measure columns with median
    - Impute dimension columns with mode
    - Drop rows with missing identifier values
    - _Requirements: 4.8, 4.9, 4.10_

  - [ ]* 10.2 Write unit tests for missing value handling
    - Test median imputation for metrics
    - Test mode imputation for categories
    - Test row dropping for identifiers
    - _Requirements: 4.8, 4.9, 4.10_

- [ ] 11. Implement deduplication and DuckDB integration
  - [ ] 11.1 Create `deduplicate()` method in DataPreparationEngine
    - Remove duplicate rows based on primary keys if identified
    - Keep first occurrence of duplicates
    - _Requirements: 4.11, 4.12_

  - [ ] 11.2 Create `execute_duckdb_query()` method for complex transformations
    - Accept DataFrame and SQL query string
    - Execute query using DuckDB in-memory
    - Return transformed DataFrame
    - _Requirements: 4.13_

  - [ ]* 11.3 Write unit tests for deduplication and DuckDB
    - Test duplicate removal with various primary key combinations
    - Test DuckDB query execution
    - _Requirements: 4.11_

- [ ] 12. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 4: Supervisor Agent & Honesty Protocol (HITL)

- [ ] 13. Define agent state and workflow models
  - [ ] 13.1 Create `src/agents/models.py` with agent data models
    - Define AgentState TypedDict with fields: upload_id, user_query, semantic_profile, dataframe, validation_result, generated_code, execution_result, error_count, human_feedback, messages
    - Define ValidationResult Pydantic model with fields: feasible, reasoning, required_columns, missing_columns
    - Define GeneratedCode Pydantic model with fields: code, explanation, required_libraries, expected_output_type
    - Define ExecutionResult Pydantic model with fields: success, output, chart_base64, error, traceback, execution_time_ms
    - _Requirements: 5.7, 5.8, 5.9, 6.3, 6.4, 6.5, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10_

  - [ ]* 13.2 Write unit tests for model validation
    - Test Pydantic model validation rules
    - Test field constraints and enums
    - _Requirements: 5.7, 5.9_

- [ ] 14. Implement query validation and honesty protocol
  - [ ] 14.1 Create `src/agents/supervisor.py` with SupervisorAgent class
    - Implement `validate_query_feasibility()` node function
    - Construct LLM prompt with query and semantic profile
    - Use structured output to get ValidationResult
    - Check if required columns are present
    - Check if column types support requested analysis
    - Check if query is semantically coherent
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.7, 5.8, 5.9_

  - [ ]* 14.2 Write property test for query validation consistency
    - **Property 4: Query Validation Consistency**
    - **Validates: Requirements 5.1, 5.2, 5.7, 5.9**
    - Test that feasible queries only require existing columns
    - Test that infeasible queries always include reasoning

- [ ] 15. Build LangGraph workflow with HITL interrupts
  - [ ] 15.1 Create `build_supervisor_graph()` method in SupervisorAgent
    - Create StateGraph with AgentState
    - Add nodes: validate_query, hitl_interrupt, generate_code, execute_code, handle_error
    - Set entry point to validate_query
    - Add conditional routing after validation: feasible → generate_code, infeasible → hitl_interrupt
    - Add edge from hitl_interrupt back to validate_query for feedback loop
    - Add conditional routing after execution: success → END, retry → handle_error, fail → END
    - Compile graph with MemorySaver checkpointer
    - Configure interrupt_before=["hitl_interrupt"]
    - _Requirements: 5.5, 5.6, 5.10, 5.11, 5.12_

  - [ ]* 15.2 Write integration tests for LangGraph workflow
    - Test routing logic for feasible queries
    - Test routing logic for infeasible queries
    - Test HITL interrupt and resume behavior
    - _Requirements: 5.5, 5.6, 5.10, 5.12_

- [ ] 16. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 5: Generative Execution & Visualization

- [ ] 17. Implement code generation agent
  - [ ] 17.1 Create `src/agents/coder.py` with CoderAgent class
    - Implement `generate_code()` node function
    - Build LLM prompt with query, semantic profile, and DataFrame sample
    - Include error context if this is a retry attempt
    - Use structured output to get GeneratedCode
    - Specify requirements: use 'df' DataFrame, generate visualization, save to 'output.png', use only allowed libraries
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.9, 6.10, 6.11, 17.1, 17.2, 17.5, 17.6, 17.7_

  - [ ]* 17.2 Write unit tests for code generation
    - Test with mock LLM responses
    - Test error context incorporation on retry
    - Test structured output enforcement
    - _Requirements: 6.10, 6.11_

- [ ] 18. Implement code safety validation
  - [ ] 18.1 Create `validate_code_safety()` function in CoderAgent
    - Check for dangerous patterns: os.system, subprocess, eval, exec, __import__, requests, urllib, socket
    - Allow open() only for 'output.png'
    - Return tuple of (is_safe, rejection_reason)
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9, 7.10, 7.11_

  - [ ]* 18.2 Write property test for code generation safety
    - **Property 5: Code Generation Safety**
    - **Validates: Requirements 6.1, 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9, 7.10**
    - Test that generated code is non-empty
    - Test that code passes safety validation
    - Test that code contains no dangerous operations

- [ ] 19. Implement E2B code executor
  - [ ] 19.1 Create `src/execution/executor.py` with E2BExecutor class
    - Implement `execute_code()` method accepting code, DataFrame, and timeout
    - Initialize E2B CodeInterpreter with API key
    - Serialize DataFrame to CSV and inject into sandbox context
    - Execute generated code in sandbox
    - Capture stdout, stderr, and execution time
    - Extract base64-encoded chart if generated
    - Handle execution errors and timeouts
    - Return ExecutionResult with success status
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10, 8.11, 17.3, 17.4_

  - [ ]* 19.2 Write property test for execution result completeness
    - **Property 6: Execution Result Completeness**
    - **Validates: Requirements 8.5, 8.6, 8.7, 8.8, 8.9, 8.10**
    - Test that successful executions have output or chart
    - Test that failed executions include error messages
    - Test that execution time is always non-negative

- [ ] 20. Implement self-healing retry logic
  - [ ] 20.1 Create `execute_code_node()` and `handle_execution_error()` in supervisor
    - Implement execute_code_node to call E2BExecutor
    - Increment error_count on failure
    - Implement route_after_execution to check success/retry/fail
    - Route to handle_error if error_count < 3
    - Route to END if error_count >= 3
    - Pass error traceback back to generate_code for retry
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7_

  - [ ]* 20.2 Write property test for retry loop termination
    - **Property 7: Retry Loop Termination**
    - **Validates: Requirements 9.1, 9.2, 9.4, 9.7**
    - Test that retry loop terminates after at most 3 failures
    - Test that error_count accurately reflects failures

- [ ] 21. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 6: Transparency & Frontend Rendering

- [ ] 22. Implement FastAPI backend with REST endpoints
  - [ ] 22.1 Create `src/api/main.py` with FastAPI application
    - Initialize FastAPI app with CORS middleware
    - Implement POST /upload endpoint accepting UploadFile
    - Orchestrate ingestion, profiling, and preparation pipeline
    - Generate unique upload_id and cache DataFrame with semantic_profile
    - Return UploadResponse with upload_id, filename, row_count, column_count, semantic_profile
    - _Requirements: 12.1, 12.2, 12.8, 12.9, 12.10, 12.11, 15.1, 15.2, 15.3, 15.4_

  - [ ] 22.2 Implement POST /query endpoint
    - Accept QueryRequest with upload_id and query
    - Retrieve cached DataFrame and profile
    - Initialize AgentState and invoke SupervisorAgent workflow
    - Return QueryResponse with result_id, status, visualization_url, summary
    - _Requirements: 12.3, 12.4, 12.5_

  - [ ] 22.3 Implement GET /result/{result_id} endpoint
    - Retrieve completed analysis result from cache
    - Return visualization and summary
    - _Requirements: 12.6_

  - [ ]* 22.4 Write integration tests for API endpoints
    - Test complete upload-to-result pipeline
    - Test error handling for invalid requests
    - Test cache expiration behavior
    - _Requirements: 12.1, 12.3, 12.6, 15.6, 15.7, 15.8_

- [ ] 23. Implement LangServe SSE streaming
  - [ ] 23.1 Add LangServe routes to FastAPI app
    - Import add_routes from langserve
    - Add supervisor graph to /agent endpoint
    - Configure SSE streaming for agent state updates
    - Emit events on node transitions with node name and message
    - Throttle events to max 1 per 500ms
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 10.8, 10.9, 12.7_

  - [ ]* 23.2 Write integration tests for SSE streaming
    - Test SSE connection and event reception
    - Test event throttling
    - _Requirements: 10.1, 10.5, 10.9_

- [ ] 24. Implement memory management and caching
  - [ ] 24.1 Create `src/utils/memory.py` with memory monitoring
    - Implement memory usage monitoring using psutil
    - Reject uploads when memory exceeds 400MB
    - Implement LRU cache with 30-minute TTL for datasets
    - Auto-purge expired cache entries
    - _Requirements: 11.1, 11.2, 11.3, 11.6, 11.7, 15.5, 15.6, 15.7, 15.8_

  - [ ] 24.2 Optimize memory usage in data processing
    - Use streaming reads for files > 50MB
    - Convert low-cardinality string columns to categorical
    - Implement memory allocation limits: 150MB ingestion, 200MB preparation, 50MB agent state
    - _Requirements: 11.4, 11.5, 11.8, 11.9, 11.10_

  - [ ]* 24.3 Write property test for memory constraint adherence
    - **Property 8: Memory Constraint Adherence**
    - **Validates: Requirements 1.11, 4.13, 11.2**
    - Test that operations stay within 512MB limit
    - Test that no disk writes occur except E2B sandbox

- [ ] 25. Implement rate limiting and security
  - [ ] 25.1 Add rate limiting middleware to FastAPI app
    - Implement 10 requests per minute per IP address limit
    - Return 429 Too Many Requests when exceeded
    - _Requirements: 13.1, 13.2_

  - [ ] 25.2 Add security measures
    - Enforce HTTPS in production
    - Validate file extensions before processing
    - Sanitize and escape user queries before LLM prompts
    - Never log DataFrame contents or user data
    - Log only metadata: upload_id, query, timestamps, error types
    - Return generic error messages to clients
    - Log detailed errors server-side
    - _Requirements: 13.3, 13.4, 13.5, 13.6, 13.7, 13.8, 13.9, 13.10_

  - [ ]* 25.3 Write unit tests for security measures
    - Test rate limiting enforcement
    - Test input sanitization
    - Test logging behavior
    - _Requirements: 13.1, 13.5, 13.6, 13.7, 13.8_

- [ ] 26. Implement error handling and recovery
  - [ ] 26.1 Create comprehensive error handlers in FastAPI app
    - Handle file upload errors with specific messages
    - Suggest reducing file size when limit exceeded
    - List supported formats when format is unsupported
    - Warn when header detection confidence is low
    - Implement exponential backoff for LLM API retries (2s, 4s, 8s)
    - Return 503 when LLM API unavailable after retries
    - Retry code generation with enhanced safety instructions on validation failure
    - Suggest simplifying query on execution timeout
    - Instruct re-upload when cache expires
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.5, 16.6, 16.7, 16.8, 16.9, 16.10_

  - [ ]* 26.2 Write unit tests for error handling
    - Test all error scenarios and messages
    - Test retry logic with exponential backoff
    - _Requirements: 16.1, 16.5, 16.6, 16.7_

- [ ] 27. Implement Streamlit frontend
  - [ ] 27.1 Create `src/frontend/app.py` with Streamlit application
    - Add title and file upload widget for CSV and Excel
    - Display filename, row count, column count after upload
    - Display semantic profile in expandable section
    - Show each column's name, semantic_type, functional_role
    - _Requirements: 14.1, 14.2, 14.3, 14.4_

  - [ ] 27.2 Add query interface and agent log streaming
    - Add text input for natural language queries
    - Add section for agent reasoning logs
    - Connect to LangServe SSE endpoint on query submission
    - Append agent state updates to reasoning log in real-time
    - _Requirements: 14.5, 14.6, 14.7_

  - [ ] 27.3 Add result visualization and HITL handling
    - Display generated visualization when execution succeeds
    - Display code explanation with results
    - Display rejection reason on HITL interrupt
    - Provide input for user feedback on HITL
    - Display error message and traceback when execution fails after retries
    - _Requirements: 14.8, 14.9, 14.10, 14.11, 14.12_

  - [ ]* 27.4 Write end-to-end tests for frontend
    - Test complete user workflow from upload to visualization
    - Test HITL interrupt handling
    - Test error display
    - _Requirements: 14.1, 14.5, 14.8, 14.10, 14.12_

- [ ] 28. Create configuration and deployment files
  - [ ] 28.1 Create environment configuration
    - Create `config/settings.py` to read environment variables
    - Support configurable: MAX_UPLOAD_SIZE_MB, CACHE_TTL_MINUTES, MAX_MEMORY_MB, RATE_LIMIT_PER_MINUTE, LOG_LEVEL
    - Validate required keys: OPENAI_API_KEY, E2B_API_KEY
    - _Requirements: 20.1, 20.2, 20.3, 20.4, 20.5, 20.6, 20.7, 20.8_

  - [ ] 28.2 Create deployment documentation
    - Create README.md with setup instructions
    - Document environment variable requirements
    - Document Python version requirement (3.11+)
    - Create docker-compose.yml for local development
    - _Requirements: 20.9_

- [ ] 29. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP delivery
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at phase boundaries
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- The implementation follows the 6-phase plan from the original user request
- All code uses Python 3.11+ with type hints
- Memory constraints (512MB RAM) are enforced throughout
- Security and safety are prioritized in code generation and execution
