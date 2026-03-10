# Requirements Document: Axiom AI - Autonomous Data-to-Insight Agent MVP

## Introduction

Axiom AI is an autonomous agent-driven system that transforms unstructured data files (CSV and Excel) into validated visual insights through intelligent data understanding, automated cleaning, and code generation. The system addresses the challenge of analyzing messy, human-formatted data by combining semantic understanding with transparent, executable code generation. Built for deployment on free-tier cloud infrastructure (512MB RAM constraint), the system emphasizes honesty through query validation, transparency through real-time agent state streaming, and safety through isolated code execution.

## Glossary

- **System**: The complete Axiom AI application including backend, frontend, and agent components
- **Ingestion_Engine**: Component responsible for parsing uploaded files and detecting data structure
- **Semantic_Profiler**: Component that uses LLM to understand column meanings and functional roles
- **Data_Prep_Engine**: Component that cleans and transforms data based on semantic understanding
- **Supervisor_Agent**: LangGraph agent that orchestrates workflow and validates query feasibility
- **Coder_Agent**: LangGraph agent that generates executable Python code for analysis
- **E2B_Executor**: Component that executes generated code in isolated cloud sandbox
- **Backend**: FastAPI server exposing REST endpoints and SSE streams
- **Frontend**: Streamlit web interface for user interaction
- **Semantic_Profile**: Structured metadata describing column types, roles, and constraints
- **HITL**: Human-in-the-loop validation checkpoint for infeasible queries
- **Agent_State**: Shared state object passed between LangGraph workflow nodes
- **Execution_Result**: Structured output from code execution containing success status and outputs

## Requirements

### Requirement 1: File Upload and Ingestion

**User Story:** As a data analyst, I want to upload messy CSV and Excel files with inconsistent formatting, so that I can analyze data without manual cleaning.

#### Acceptance Criteria

1. WHEN a user uploads a CSV file, THE Ingestion_Engine SHALL accept files with extensions .csv
2. WHEN a user uploads an Excel file, THE Ingestion_Engine SHALL accept files with extensions .xlsx and .xls
3. WHEN a file exceeds 100MB, THE System SHALL reject the upload and return error message "File size exceeds 100MB limit"
4. WHEN a file has an unsupported extension, THE System SHALL reject the upload and return error message "Unsupported file format"
5. WHEN a CSV file is uploaded, THE Ingestion_Engine SHALL automatically detect the file encoding from UTF-8, Latin-1, and Windows-1252
6. WHEN a CSV file is uploaded, THE Ingestion_Engine SHALL automatically detect the delimiter from comma, semicolon, tab, and pipe
7. WHEN a file contains empty leading rows, THE Ingestion_Engine SHALL skip them and identify the header row position
8. WHEN a file contains empty trailing rows or footer notes, THE Ingestion_Engine SHALL exclude them from the data boundaries
9. WHEN ingestion completes, THE System SHALL return a DataFrame with at least 1 row and 1 column
10. WHEN ingestion completes, THE System SHALL return metadata containing header_row, data_start_row, data_end_row, original_shape, and cleaned_shape
11. THE Ingestion_Engine SHALL perform all processing in memory without writing to disk

### Requirement 2: Header Detection

**User Story:** As a data analyst, I want the system to automatically identify column headers in messy files, so that I don't need to manually specify header locations.

#### Acceptance Criteria

1. WHEN a file contains multiple potential header rows, THE Ingestion_Engine SHALL score each candidate row based on heuristics
2. WHEN scoring header candidates, THE Ingestion_Engine SHALL assign higher scores to rows with complete non-null values
3. WHEN scoring header candidates, THE Ingestion_Engine SHALL assign higher scores to rows with string values rather than numeric values
4. WHEN scoring header candidates, THE Ingestion_Engine SHALL assign higher scores to rows with unique values
5. WHEN scoring header candidates, THE Ingestion_Engine SHALL assign higher scores to rows with text length between 5 and 30 characters
6. WHEN header detection completes, THE Ingestion_Engine SHALL return the row index with the highest score as the header row
7. WHEN header detection confidence is low, THE System SHALL include a warning flag in the metadata
8. THE Ingestion_Engine SHALL examine at most the first 10 rows as potential header candidates

### Requirement 3: Semantic Profiling

**User Story:** As a data analyst, I want the system to understand what my columns mean beyond just data types, so that it can perform intelligent analysis and cleaning.

#### Acceptance Criteria

1. WHEN a DataFrame is ingested, THE Semantic_Profiler SHALL generate a profile for every column
2. WHEN profiling a column, THE Semantic_Profiler SHALL assign a semantic_type from the set: id, metric, category, datetime, text, boolean
3. WHEN profiling a column, THE Semantic_Profiler SHALL assign a functional_role from the set: identifier, measure, dimension, timestamp, description
4. WHEN profiling a column, THE Semantic_Profiler SHALL extract up to 5 sample values
5. WHEN profiling a column, THE Semantic_Profiler SHALL determine if the column is nullable
6. WHEN profiling completes, THE Semantic_Profiler SHALL identify primary key columns
7. WHEN profiling completes, THE Semantic_Profiler SHALL identify metric columns containing numeric measures
8. WHEN profiling completes, THE Semantic_Profiler SHALL identify dimension columns containing categorical values
9. WHEN profiling completes, THE Semantic_Profiler SHALL identify temporal columns containing datetime values
10. WHEN profiling completes, THE Semantic_Profile SHALL contain a row_count matching the DataFrame length
11. THE Semantic_Profiler SHALL send only column names and 5-row samples to the LLM, not the full dataset
12. THE Semantic_Profiler SHALL enforce the SemanticProfile Pydantic schema on LLM responses
13. WHEN an LLM response violates the schema, THE Semantic_Profiler SHALL reject it and retry

### Requirement 4: Data Preparation and Cleaning

**User Story:** As a data analyst, I want the system to automatically clean and transform my data based on semantic understanding, so that analysis operates on high-quality data.

#### Acceptance Criteria

1. WHEN a column has semantic_type "metric", THE Data_Prep_Engine SHALL convert it to numeric dtype
2. WHEN a column has semantic_type "datetime", THE Data_Prep_Engine SHALL convert it to datetime dtype
3. WHEN a column has semantic_type "category", THE Data_Prep_Engine SHALL convert it to categorical dtype
4. WHEN a column has semantic_type "boolean", THE Data_Prep_Engine SHALL map text values to boolean
5. WHEN a numeric column contains currency symbols, THE Data_Prep_Engine SHALL strip them before conversion
6. WHEN a numeric column contains percentage signs, THE Data_Prep_Engine SHALL strip them before conversion
7. WHEN a numeric column contains commas, THE Data_Prep_Engine SHALL strip them before conversion
8. WHEN a column with functional_role "measure" has missing values, THE Data_Prep_Engine SHALL impute with median
9. WHEN a column with functional_role "dimension" has missing values, THE Data_Prep_Engine SHALL impute with mode
10. WHEN a column with functional_role "identifier" has missing values, THE Data_Prep_Engine SHALL drop those rows
11. WHEN primary keys are identified, THE Data_Prep_Engine SHALL remove duplicate rows based on those keys
12. WHEN data preparation completes, THE Data_Prep_Engine SHALL return a DataFrame with the same or fewer rows than the input
13. THE Data_Prep_Engine SHALL perform all operations in memory without disk I/O

### Requirement 5: Query Validation and Honesty Protocol

**User Story:** As a data analyst, I want the system to honestly tell me when it cannot answer my question, so that I don't waste time on infeasible queries.

#### Acceptance Criteria

1. WHEN a user submits a query, THE Supervisor_Agent SHALL validate if the query is answerable with available data
2. WHEN validating a query, THE Supervisor_Agent SHALL check if all required columns are present in the semantic profile
3. WHEN validating a query, THE Supervisor_Agent SHALL check if column types support the requested analysis
4. WHEN validating a query, THE Supervisor_Agent SHALL check if the query is semantically coherent
5. WHEN a query is feasible, THE Supervisor_Agent SHALL route to the Coder_Agent
6. WHEN a query is infeasible, THE Supervisor_Agent SHALL route to HITL interrupt
7. WHEN a query is infeasible, THE validation result SHALL include a reasoning explanation
8. WHEN a query is infeasible, THE validation result SHALL list missing columns
9. WHEN a query is feasible, THE validation result SHALL list required columns
10. WHEN HITL interrupt occurs, THE System SHALL pause execution and request human feedback
11. WHEN HITL interrupt occurs, THE Frontend SHALL display the rejection reason to the user
12. WHEN a user provides feedback after HITL, THE Supervisor_Agent SHALL re-validate the modified query

### Requirement 6: Code Generation

**User Story:** As a data analyst, I want the system to generate executable Python code for my analysis, so that I can see exactly what operations are performed.

#### Acceptance Criteria

1. WHEN a query is validated as feasible, THE Coder_Agent SHALL generate Python code to answer the query
2. WHEN generating code, THE Coder_Agent SHALL use the semantic profile to understand available columns
3. WHEN generating code, THE Coder_Agent SHALL include a natural language explanation of the approach
4. WHEN generating code, THE Coder_Agent SHALL list required libraries
5. WHEN generating code, THE Coder_Agent SHALL specify the expected output type from: chart, table, metric, text
6. WHEN generating code, THE Coder_Agent SHALL assume a DataFrame named "df" is available
7. WHEN generating code, THE Coder_Agent SHALL use only allowed libraries: pandas, numpy, matplotlib, plotly, seaborn
8. WHEN generating code, THE Coder_Agent SHALL save visualizations to "output.png"
9. WHEN generating code, THE Coder_Agent SHALL include print statements for summary statistics or insights
10. WHEN previous execution failed, THE Coder_Agent SHALL incorporate the error traceback into the generation prompt
11. THE Coder_Agent SHALL enforce structured output using the GeneratedCode Pydantic schema

### Requirement 7: Code Safety Validation

**User Story:** As a system administrator, I want to ensure generated code cannot perform dangerous operations, so that the system remains secure.

#### Acceptance Criteria

1. WHEN code is generated, THE System SHALL validate it for safety before execution
2. WHEN code contains "os.system", THE System SHALL reject it with error "System command execution not allowed"
3. WHEN code contains "subprocess", THE System SHALL reject it with error "Subprocess execution not allowed"
4. WHEN code contains "eval(", THE System SHALL reject it with error "eval() function not allowed"
5. WHEN code contains "exec(", THE System SHALL reject it with error "exec() function not allowed"
6. WHEN code contains "__import__", THE System SHALL reject it with error "Dynamic imports not allowed"
7. WHEN code contains "requests.", THE System SHALL reject it with error "Network requests not allowed"
8. WHEN code contains "urllib", THE System SHALL reject it with error "Network requests not allowed"
9. WHEN code contains "socket", THE System SHALL reject it with error "Socket operations not allowed"
10. WHEN code contains "open(" without "output.png", THE System SHALL reject it with error "File operations not allowed except for chart saving"
11. WHEN code passes all safety checks, THE System SHALL proceed to execution

### Requirement 8: Code Execution in Isolated Sandbox

**User Story:** As a data analyst, I want my analysis code to execute safely in an isolated environment, so that I can trust the results without security concerns.

#### Acceptance Criteria

1. WHEN code passes safety validation, THE E2B_Executor SHALL execute it in an isolated cloud sandbox
2. WHEN executing code, THE E2B_Executor SHALL inject the DataFrame into the execution context
3. WHEN executing code, THE E2B_Executor SHALL enforce a 30-second timeout
4. WHEN execution exceeds the timeout, THE E2B_Executor SHALL terminate it and return error "Execution timeout after 30 seconds"
5. WHEN execution completes successfully, THE E2B_Executor SHALL capture stdout output
6. WHEN execution generates a visualization, THE E2B_Executor SHALL extract it as base64-encoded image
7. WHEN execution fails with an exception, THE E2B_Executor SHALL capture the full Python traceback
8. WHEN execution completes, THE E2B_Executor SHALL record the execution time in milliseconds
9. WHEN execution succeeds, THE Execution_Result SHALL contain at least one of: output or chart_base64
10. WHEN execution fails, THE Execution_Result SHALL contain a non-empty error message
11. THE E2B_Executor SHALL ensure the sandbox has no outbound network connectivity

### Requirement 9: Self-Healing Retry Logic

**User Story:** As a data analyst, I want the system to automatically fix code errors when possible, so that I get results without manual intervention.

#### Acceptance Criteria

1. WHEN code execution fails, THE Supervisor_Agent SHALL increment the error_count
2. WHEN error_count is less than 3, THE Supervisor_Agent SHALL route back to the Coder_Agent for retry
3. WHEN retrying code generation, THE Coder_Agent SHALL include the previous error traceback in the prompt
4. WHEN error_count reaches 3, THE Supervisor_Agent SHALL terminate the workflow and return failure
5. WHEN execution succeeds, THE Supervisor_Agent SHALL route to completion
6. WHEN all retries are exhausted, THE System SHALL return the last error traceback to the user
7. THE System SHALL ensure the retry loop terminates after at most 3 failed attempts

### Requirement 10: Agent State Streaming

**User Story:** As a data analyst, I want to see real-time updates of what the agent is thinking and doing, so that I understand the analysis process.

#### Acceptance Criteria

1. WHEN a query is submitted, THE Backend SHALL expose a LangServe SSE endpoint for agent state streaming
2. WHEN the Supervisor_Agent transitions between nodes, THE System SHALL emit a state update event
3. WHEN a state update occurs, THE event SHALL include the current node name
4. WHEN a state update occurs, THE event SHALL include a human-readable message describing the action
5. WHEN the Frontend connects to the SSE endpoint, THE System SHALL stream all state updates in real-time
6. WHEN validation occurs, THE System SHALL stream the validation reasoning
7. WHEN code generation occurs, THE System SHALL stream the code explanation
8. WHEN execution occurs, THE System SHALL stream the execution status
9. THE System SHALL throttle SSE events to at most 1 event per 500 milliseconds to reduce bandwidth

### Requirement 11: Memory Constraint Adherence

**User Story:** As a system administrator, I want the system to operate within 512MB RAM, so that it can be deployed on free-tier cloud infrastructure.

#### Acceptance Criteria

1. THE System SHALL monitor memory usage using psutil
2. WHEN memory usage exceeds 400MB, THE System SHALL reject new file uploads
3. WHEN memory usage exceeds 400MB, THE System SHALL return error "Dataset too large for processing"
4. WHEN processing a file, THE Ingestion_Engine SHALL use streaming reads for files larger than 50MB
5. WHEN a string column has less than 50% unique values, THE Data_Prep_Engine SHALL convert it to categorical dtype
6. THE System SHALL implement an LRU cache with 30-minute TTL for uploaded datasets
7. WHEN cache TTL expires, THE System SHALL automatically purge the dataset from memory
8. THE System SHALL allocate at most 150MB for file ingestion
9. THE System SHALL allocate at most 200MB for data preparation
10. THE System SHALL allocate at most 50MB for agent state management

### Requirement 12: API Endpoints

**User Story:** As a frontend developer, I want well-defined REST endpoints, so that I can integrate the analysis capabilities into user interfaces.

#### Acceptance Criteria

1. THE Backend SHALL expose a POST /upload endpoint for file uploads
2. WHEN /upload receives a file, THE Backend SHALL return upload_id, filename, row_count, column_count, and semantic_profile
3. THE Backend SHALL expose a POST /query endpoint for analysis requests
4. WHEN /query receives upload_id and query, THE Backend SHALL invoke the Supervisor_Agent workflow
5. WHEN /query completes, THE Backend SHALL return result_id, status, visualization_url, and summary
6. THE Backend SHALL expose a GET /result/{result_id} endpoint for retrieving completed results
7. THE Backend SHALL expose a GET /agent endpoint for LangServe SSE streaming
8. THE Backend SHALL validate all request payloads using Pydantic models
9. WHEN a request payload is invalid, THE Backend SHALL return 400 Bad Request with validation errors
10. THE Backend SHALL enable CORS for specified frontend origins
11. THE Backend SHALL compress responses using gzip

### Requirement 13: Rate Limiting and Security

**User Story:** As a system administrator, I want to prevent abuse and ensure secure operation, so that the system remains available and protected.

#### Acceptance Criteria

1. THE Backend SHALL enforce rate limiting of 10 requests per minute per IP address
2. WHEN rate limit is exceeded, THE Backend SHALL return 429 Too Many Requests
3. THE Backend SHALL enforce HTTPS for all API communication in production
4. THE Backend SHALL validate file extensions before processing
5. THE Backend SHALL sanitize user queries before including them in LLM prompts
6. THE Backend SHALL escape special characters in user input
7. THE Backend SHALL never log DataFrame contents or user data
8. THE Backend SHALL log only metadata: upload_id, query, timestamps, error types
9. THE Backend SHALL return generic error messages to clients without exposing stack traces
10. THE Backend SHALL log detailed errors server-side for debugging

### Requirement 14: Frontend User Interface

**User Story:** As a data analyst, I want an intuitive web interface to upload files and ask questions, so that I can interact with the system without technical knowledge.

#### Acceptance Criteria

1. THE Frontend SHALL provide a file upload widget accepting CSV and Excel files
2. WHEN a file is uploaded, THE Frontend SHALL display the filename, row count, and column count
3. WHEN a file is uploaded, THE Frontend SHALL display the semantic profile in an expandable section
4. WHEN displaying the semantic profile, THE Frontend SHALL show each column's name, semantic_type, and functional_role
5. THE Frontend SHALL provide a text input for natural language queries
6. WHEN a query is submitted, THE Frontend SHALL display a section for agent reasoning logs
7. WHEN agent state updates are received via SSE, THE Frontend SHALL append them to the reasoning log
8. WHEN execution completes successfully, THE Frontend SHALL display the generated visualization
9. WHEN execution completes successfully, THE Frontend SHALL display the code explanation
10. WHEN HITL interrupt occurs, THE Frontend SHALL display the rejection reason
11. WHEN HITL interrupt occurs, THE Frontend SHALL provide an input for user feedback
12. WHEN execution fails after all retries, THE Frontend SHALL display the error message and last traceback

### Requirement 15: Caching and Session Management

**User Story:** As a data analyst, I want my uploaded data to remain available for multiple queries, so that I don't need to re-upload for each question.

#### Acceptance Criteria

1. WHEN a file is uploaded, THE Backend SHALL generate a unique upload_id
2. WHEN data preparation completes, THE Backend SHALL cache the DataFrame and semantic_profile in memory
3. WHEN caching data, THE Backend SHALL associate it with the upload_id
4. WHEN caching data, THE Backend SHALL record the current timestamp
5. WHEN a query references an upload_id, THE Backend SHALL retrieve the cached data
6. WHEN an upload_id is not found in cache, THE Backend SHALL return 404 Not Found with message "Upload session expired"
7. THE Backend SHALL implement a 30-minute TTL for cached uploads
8. WHEN cache TTL expires, THE Backend SHALL automatically remove the entry
9. THE Backend SHALL implement LRU eviction when memory usage approaches limits

### Requirement 16: Error Handling and Recovery

**User Story:** As a data analyst, I want clear error messages and recovery guidance, so that I can resolve issues and successfully complete my analysis.

#### Acceptance Criteria

1. WHEN a file upload fails, THE System SHALL return a specific error message indicating the cause
2. WHEN file size exceeds limits, THE error message SHALL suggest reducing file size or sampling data
3. WHEN file format is unsupported, THE error message SHALL list supported formats
4. WHEN header detection confidence is low, THE System SHALL warn the user to verify column names
5. WHEN LLM API quota is exceeded, THE System SHALL retry with exponential backoff (2s, 4s, 8s delays)
6. WHEN LLM API is unavailable after retries, THE System SHALL return 503 Service Unavailable
7. WHEN generated code fails safety validation, THE System SHALL retry generation with enhanced safety instructions
8. WHEN execution times out, THE error message SHALL suggest simplifying the query
9. WHEN memory limit is approached, THE System SHALL reject new uploads with a descriptive message
10. WHEN cache expires, THE error message SHALL instruct the user to re-upload the file

### Requirement 17: Visualization Generation

**User Story:** As a data analyst, I want the system to generate appropriate visualizations for my queries, so that I can understand insights visually.

#### Acceptance Criteria

1. WHEN a query requests a chart, THE Coder_Agent SHALL generate code using matplotlib or plotly
2. WHEN generating a chart, THE code SHALL save it to "output.png"
3. WHEN execution completes, THE E2B_Executor SHALL extract the chart as a base64-encoded PNG
4. WHEN the Frontend receives a chart, THE Frontend SHALL decode and display it
5. WHEN a query requests a table, THE Coder_Agent SHALL generate code to format a DataFrame
6. WHEN a query requests metrics, THE Coder_Agent SHALL generate code to calculate and print statistics
7. THE generated visualizations SHALL include appropriate titles, axis labels, and legends

### Requirement 18: LLM Integration

**User Story:** As a system administrator, I want flexible LLM provider integration, so that I can use different AI services based on cost and availability.

#### Acceptance Criteria

1. THE System SHALL support OpenAI as the primary LLM provider
2. THE System SHALL accept LLM API keys via environment variables
3. WHEN an LLM API call fails, THE System SHALL log the error with timestamp
4. WHEN an LLM returns invalid JSON, THE System SHALL retry the request
5. THE System SHALL use structured output mode to enforce Pydantic schemas
6. THE System SHALL include system prompts emphasizing safety and constraints
7. THE System SHALL limit LLM context to necessary information only (no full datasets)

### Requirement 19: Testing and Validation

**User Story:** As a developer, I want comprehensive test coverage, so that I can confidently deploy and maintain the system.

#### Acceptance Criteria

1. THE System SHALL include unit tests for all core components
2. THE System SHALL achieve at least 80% line coverage for core logic
3. THE System SHALL achieve 100% coverage for safety validation functions
4. THE System SHALL include property-based tests using Hypothesis
5. THE System SHALL include integration tests for the complete upload-to-result pipeline
6. THE System SHALL include tests for all error handling paths
7. THE System SHALL include tests for LangGraph workflow routing
8. THE System SHALL include tests for E2B execution with mock responses

### Requirement 20: Deployment and Configuration

**User Story:** As a system administrator, I want simple deployment with environment-based configuration, so that I can run the system in different environments.

#### Acceptance Criteria

1. THE System SHALL read configuration from environment variables
2. THE System SHALL require OPENAI_API_KEY or equivalent LLM provider key
3. THE System SHALL require E2B_API_KEY for code execution
4. THE System SHALL support configurable MAX_UPLOAD_SIZE_MB
5. THE System SHALL support configurable CACHE_TTL_MINUTES
6. THE System SHALL support configurable MAX_MEMORY_MB
7. THE System SHALL support configurable RATE_LIMIT_PER_MINUTE
8. THE System SHALL support configurable LOG_LEVEL
9. THE System SHALL run on Python 3.11 or higher
10. THE System SHALL provide a requirements.txt with pinned dependency versions
