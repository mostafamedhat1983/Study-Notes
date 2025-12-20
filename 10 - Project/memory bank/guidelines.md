---
tags:
  - memory-bank
---
# Development Guidelines

## Code Quality Standards

### Python Code Formatting
- **Import Organization**: Standard library imports first, third-party imports second, local imports last with blank lines between groups
- **Function Documentation**: Comprehensive docstrings for all functions using triple quotes with clear parameter and return descriptions
- **Type Hints**: Extensive use of typing module (Optional, List, Dict) for function parameters and return values
- **Variable Naming**: Descriptive snake_case names (e.g., `conversation_history`, `ai_response`, `session_id`)
- **Constants**: ALL_CAPS for configuration constants (e.g., `API_BASE_URL`, `BEDROCK_MODEL_ID`, `AWS_REGION`)

### Error Handling Patterns
- **Comprehensive Exception Handling**: Multiple specific exception types caught with appropriate error messages
- **Logging Integration**: Structured logging with different levels (INFO, ERROR) for operational visibility
- **User-Friendly Messages**: Technical errors translated to user-friendly messages in API responses
- **Graceful Degradation**: Fallback behaviors when services are unavailable (e.g., API health checks)

### Configuration Management
- **Environment Variables**: All configuration via `os.getenv()` with sensible defaults
- **Centralized Config**: Configuration dictionaries (e.g., `DB_CONFIG`) for related settings
- **Security-First**: No hardcoded credentials or sensitive values in source code

## Architectural Patterns

### FastAPI Backend Structure
- **Lifespan Management**: `@asynccontextmanager` for proper startup/shutdown lifecycle
- **Dependency Injection**: Global connection pools initialized at startup, cleaned up at shutdown
- **Middleware Stack**: CORS, rate limiting, and error handling middleware properly configured
- **Pydantic Models**: Strong typing with validation rules (min_length, max_length, Field descriptions)
- **Resource Management**: Async context managers for database connections (`async with db_pool.acquire()`)

### Streamlit Frontend Patterns
- **Session State Management**: Comprehensive initialization function for all session variables
- **Component Separation**: Modular UI functions (render_sidebar, render_chat_interface, render_error_banner)
- **Custom CSS Integration**: Embedded styles for consistent UI appearance and branding
- **Error Recovery**: Progressive error handling with user guidance and retry mechanisms

### Database Integration
- **Connection Pooling**: aiomysql pool with SSL/TLS encryption and proper sizing (minsize=1, maxsize=10)
- **SQL Security**: Parameterized queries to prevent SQL injection attacks
- **Index Strategy**: Strategic database indexes on session_id and created_at for query performance
- **Transaction Management**: Autocommit enabled for simple operations, explicit transactions for complex workflows

### AWS Service Integration
- **Client Initialization**: Lazy loading of AWS clients with proper error handling
- **Service Authentication**: Pod Identity for credential-free AWS service access
- **API Error Handling**: Specific handling for AWS ClientError with user-friendly error translation
- **Request Optimization**: Conversation context management (last 5 exchanges) to optimize token usage

## Security Implementation

### Secrets Management
- **Zero Credential Exposure**: No credentials in code, environment variables loaded at runtime
- **SSL/TLS Everywhere**: Database connections, AWS API calls, and inter-service communication encrypted
- **Input Validation**: Pydantic models with length limits and type validation for all user inputs
- **Rate Limiting**: API endpoint protection with per-IP rate limits (5 requests/minute)

### Kubernetes Security
- **Network Policies**: Zero-trust networking (default deny, explicit allow)
- **Security Context**: Non-root containers with dropped capabilities
- **RBAC**: Service account permissions limited to required resources
- **Runtime Monitoring**: Falco for detecting suspicious container behavior

### Vulnerability Management
- **Trivy Scanning**: CI/CD pipeline scans (fails on CRITICAL, reports HIGH/MEDIUM/LOW)
- **Dependency Updates**: Regular updates for security patches
- **Runtime Monitoring**: Falco detects suspicious behavior and security events

### Error Information Disclosure
- **Sanitized Error Messages**: Technical details logged but not exposed to end users
- **Generic Error Responses**: Consistent error format without revealing internal system details
- **Audit Trail**: Comprehensive logging for security monitoring and debugging

## Development Practices

### Code Organization
- **Single Responsibility**: Each function has a clear, focused purpose
- **Separation of Concerns**: UI logic, business logic, and data access clearly separated
- **Reusable Components**: Modular functions that can be tested and maintained independently
- **Configuration Externalization**: All environment-specific values configurable via environment variables

### Testing Considerations
- **Health Check Endpoints**: Built-in endpoints for monitoring and automated testing
- **Error Simulation**: Comprehensive error handling allows for failure testing
- **State Management**: Session state can be easily reset for testing scenarios
- **Database Schema**: Proper indexing and constraints for data integrity testing

### Performance Optimization
- **Async Operations**: Non-blocking I/O for database and AWS service calls
- **Connection Reuse**: Database connection pooling to minimize connection overhead
- **Context Management**: Limited conversation history (5 exchanges) to control token costs
- **Resource Cleanup**: Proper cleanup of connections and resources in lifespan handlers

## API Design Patterns

### RESTful Conventions
- **HTTP Status Codes**: Appropriate status codes (200, 404, 500, 503) for different scenarios
- **Request/Response Models**: Strongly typed Pydantic models for API contracts
- **Error Response Format**: Consistent error structure with detail messages
- **Resource Naming**: Clear, descriptive endpoint paths (/chat, /health, /chat/session/{id})

### Documentation Standards
- **OpenAPI Integration**: FastAPI automatic documentation generation with descriptions
- **Model Documentation**: Field descriptions and validation rules in Pydantic models
- **Endpoint Documentation**: Clear docstrings explaining purpose, parameters, and responses

## Architecture Decisions & Trade-offs

### Pod Security Standards (Evaluated and Reverted)
**Decision**: Initially implemented Kubernetes Pod Security Standards (Baseline enforcement) but reverted after encountering compatibility issues with CI/CD pipeline.

**The Problem**:
- Pod Security Standards Baseline policy blocks privileged containers
- Jenkins pipeline uses Docker-in-Docker (DinD) which requires `privileged: true`
- DinD needs host Docker daemon access to build images
- Applying PSS to `default` namespace affected all pods (application + Jenkins agents)

**Solutions Evaluated**:
1. **Separate Jenkins namespace** (no Pod Security restrictions) - Complex, requires 6 file changes across 2 repos
2. **Kaniko** (rootless container builds) - Simpler but encountered pip/frontend build issues
3. **Relax Pod Security to privileged** - Loses security benefits

**Final Decision**: Reverted to commit before Pod Security Standards, kept Falco Redis fix
- Maintains working Docker-in-Docker build system
- Keeps all other security features (network policies, RBAC, Falco, Trivy)
- Trade-off: No Pod Security Standards enforcement for deployment simplicity

**Learning**: Pod Security Standards are excellent for production workloads but can conflict with CI/CD tools requiring privileged access. Consider separate namespaces or rootless build tools (Kaniko) for future implementations.

## Deployment Considerations

### Container Optimization
- **Multi-Stage Builds**: Efficient Docker images with minimal attack surface
- **Non-Root Execution**: Containers run as non-privileged users for security
- **Health Probes**: Kubernetes-compatible health check endpoints for orchestration
- **Graceful Shutdown**: Proper signal handling for zero-downtime deployments

### Observability
- **Structured Logging**: JSON-formatted logs with correlation IDs for distributed tracing
- **Metrics Exposure**: Health endpoints provide operational metrics for monitoring
- **Error Tracking**: Comprehensive error logging with context for debugging
- **Performance Monitoring**: Database query timing and AWS API call latency tracking