# Implementation Plan

- [-] 1. Set up project structure and configuration system

  - Create directory structure following the design document layout
  - Implement configuration loading from YAML with environment variable overrides
  - Create plan.schema.json for JSON validation
  - Set up logging configuration with timezone support (default: Asia/Bangkok)
  - Add determinism.seed configuration for reproducible testing
  - _Requirements: 15.1, 15.2, 15.3, 25.1_
  - _Exit Gate M1: Config loads successfully, schema validation works_

- [ ] 2. Implement core data structures and state management
  - Create A2A-compatible AgentState class with all required fields
  - Implement LogEntry class with correlation IDs and redaction support
  - Create Action, Task, and other data models following A2A patterns
  - Add state serialization/deserialization for persistence
  - _Requirements: 1.1, 14.1, 14.2, 19.1, 19.2_

- [ ] 3. Build OpenAI Completions API integration
  - Create OpenAI client wrapper with retry logic and exponential backoff
  - Implement token usage tracking and cost calculation
  - Add error handling for OpenAI-specific errors (rate limits, quota, etc.)
  - Create completion request/response data structures
  - _Requirements: 13.2, 20.1, 20.2, 21.1, 21.2_

- [ ] 4. Implement tool registry with error isolation
  - Create base ToolRegistry class with standardized error handling
  - Implement mock tools (calculator, calendar, stock, analysis, plotter)
  - Add security validation for file operations and path traversal protection
  - Implement error taxonomy: TOOL_ERROR, NET_TIMEOUT, RATE_LIMIT, INVALID_ARGS, UNKNOWN_TOOL
  - Ensure executor never raises exceptions - only returns {error, code} and logs
  - _Requirements: 5.1, 5.2, 5.3, 13.1, 23.1, 23.2, 23.3_
  - _Exit Gate M1: Tool stubs work, error codes standardized_

- [ ] 5. Build planner module with strict JSON validation
  - Implement planner_H function with OpenAI Completions API integration
  - Add strict "JSON-only" validation: if output contains non-JSON noise → reprompt once (counts toward budget) → hard-fail with plan_schema_error
  - Create prompt templates for small and big models with seed support for determinism
  - Implement plan parsing with error recovery and schema validation
  - _Requirements: 8.1, 8.2, 8.3, 18.1, 18.2, 18.3_
  - _Exit Gate M2: Planner JSON validated, reprompt logic works_

- [ ] 6. Create executor module for tool execution
  - Implement executor_L function to process action plans sequentially
  - Add evidence and artifact collection following A2A patterns
  - Add max_concurrent_tool_calls config (default: 1 for V1, concurrent execution optional)
  - Add comprehensive error handling without exception propagation
  - _Requirements: 5.4, 5.5, 13.1, 13.4_
  - _Exit Gate M2: Executor sequential works, error handling complete_

- [ ] 7. Implement task verifiers for different task types
  - Create base verifier interface with metric calculation
  - Implement math task verifier with exact numeric comparison
  - Build calendar verifier with timezone normalization
  - Create stock task verifier checking data completeness and artifacts
  - _Requirements: 5.1, 5.2, 5.3, 25.2, 25.3_

- [ ] 8. Build ACT gate with adaptive halting logic
  - Implement ACT decision logic with metric improvement tracking
  - Add stall detection with configurable patience
  - Create budget checking for combined llm budget (small+big) and all resource limits
  - Implement escalation rules: if budget hit and escalate_when="fail" → stop with finish_reason="budget" (no escalate)
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 11.1, 11.2, 11.3_
  - _Exit Gate M3: ACT + escalation both modes work_

- [ ] 9. Create escalation system for tier management
  - Implement escalator module to switch between small and big models
  - Add context preservation during escalation
  - Implement retry logic for big model with failure context
  - Add escalation event logging
  - _Requirements: 1.4, 3.3, 4.3, 11.4, 11.5_

- [ ] 10. Implement execution modes (HRM and fallback)
  - Create hrm_small_then_escalate mode with full H/L loop
  - Implement fallback_only mode with single/mini loop
  - Add mode-specific budget constraints and behavior
  - Integrate all components into complete execution workflows
  - _Requirements: 3.1, 3.2, 24.1, 24.2_

- [ ] 11. Build time and timezone handling system
  - Implement now_provider() function for mockable time operations (tests can freeze time)
  - Add timezone configuration (default: Asia/Bangkok) and normalization
  - Create calendar verifier that MUST normalize to configured timezone
  - Add timestamp generation for all log entries with timezone
  - _Requirements: 25.1, 25.2, 25.3, 25.4_
  - _Exit Gate M4: Timezone-correct calendar normalization works_

- [ ] 12. Create comprehensive logging and correlation system
  - Implement structured logging with REQUIRED fields: run_id, task_id, step_id, parent_step_id, tier, step_type in every log line
  - Add PII redaction with configurable allowlist/denylist and unit test for redaction
  - Create log aggregation and JSONL output for CI
  - Implement log filtering and search capabilities
  - _Requirements: 14.1, 14.2, 14.3, 19.1, 19.2, 19.3_
  - _Exit Gate M4: Logs JSONL with correlation, redaction tested_

- [ ] 13. Implement idempotency and cancellation support
  - Add idempotency key handling with result caching
  - Create cancellation token system for graceful task termination
  - Implement resource cleanup on cancellation
  - Add retry behavior specification for idempotent operations
  - _Requirements: 22.1, 22.2_

- [ ] 14. Build security and validation layer
  - Enforce max_artifact_mb limits and sanitize filenames (ASCII/underscore only)
  - Block path traversal attacks and validate file paths
  - Add per-tool argument allowlists and input validation
  - Create security limits enforcement (file size, token limits)
  - _Requirements: 23.1, 23.2, 23.3, 23.4, 23.5_
  - _Exit Gate M4: Security hardening complete, path traversal blocked_

- [ ] 15. Create main run_case API interface
  - Implement public run_case function with A2A-compatible interface
  - Add parameter validation and default value handling
  - Create response formatting with all required metrics
  - Integrate idempotency and cancellation support
  - _Requirements: 9.1, 9.2, 17.1, 17.2_

- [ ] 16. Build evaluation and testing harness
  - Create test dataset loader for JSON format
  - Implement metrics calculation (success@1, escalation rate, latency)
  - Add CI quality gates: success@1 ≥0.95, escalation ≤0.15, p95 easy ≤2.0s
  - Run same test suite twice with fixed seed and assert identical traces for determinism
  - _Requirements: 16.1, 16.2, 16.3, 26.1, 26.2_
  - _Exit Gate M5: CI gates pass, determinism verified_

- [ ] 17. Implement comprehensive error handling
  - Add retry strategies for all external service calls
  - Create graceful degradation for tool failures
  - Implement error aggregation and reporting
  - Add error rate monitoring and alerting thresholds
  - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5_

- [ ] 18. Create unit tests for core components
  - Write tests for state management and data structures
  - Test OpenAI API integration with mocked responses
  - Create tool registry tests with error simulation
  - Add planner tests: malformed output → reprompt → schema pass
  - Add bonus test cases: calendar ambiguous format, stock 29 days fail, rate-limit 429 retry, idempotent cache
  - _Requirements: All components need unit test coverage >70%_
  - _Exit Gate M1: Unit tests >70% coverage_

- [ ] 19. Build integration tests for complete workflows
  - Test end-to-end execution for both modes
  - Create escalation scenario tests
  - Add timeout and cancellation integration tests
  - Test error recovery and retry mechanisms
  - _Requirements: Complete workflow validation_

- [ ] 20. Implement performance optimization and monitoring
  - Add performance profiling and bottleneck identification
  - Create memory usage monitoring and optimization
  - Cap max_prompt_tokens/max_completion_tokens per tier for cost control
  - Add performance regression detection in CI
  - _Requirements: 12.1, 12.2, 12.3_
  - _Exit Gate M5: Performance report emitted, cost controls active_

- [ ] 21. Create documentation and examples
  - Write API documentation with usage examples
  - Create configuration guide with all available options
  - Add troubleshooting guide for common issues
  - Create deployment and operational guides
  - _Requirements: User documentation and operational guides_

- [ ] 22. Final integration and system testing
  - Run complete test suite with all quality gates
  - Perform load testing and stress testing
  - Validate all CI gates and performance targets
  - Create final deployment package and release artifacts
  - _Requirements: 26.1, 26.2, 26.3, 26.4, 26.5_
## M
ilestone Exit Gates

**M1 Skeleton (Steps 1-4)**: Config loads; tool stubs work; OpenAI wrapper retries; unit tests >70%

**M2 Planning/Execution (Steps 5-7)**: Planner JSON validated; executor sequential; three verifiers pass on sample dataset

**M3 Control (Steps 8-10)**: ACT + escalation both modes; big retry capped to 1; success@1 ≥95% on test set

**M4 Platform (Steps 11-15)**: Timezone-correct calendar; logs JSONL with correlation; run_case API returns full counters/artifacts

**M5 Quality (Steps 16-20)**: CI gates on success@1 ≥0.95, escalation ≤0.15, p95 easy ≤2.0s; performance report emitted

**M6 Docs/Release (Steps 21-22)**: Usage/config/troubleshooting docs; final package + artifacts

## Dependency Order

**Parallel Start**: Steps 2, 3, 4 can run in parallel after Step 1
**Sequential Dependencies**: 
- Step 5 (planner) depends on 3 + 1
- Step 6 (executor) depends on 4 + 2  
- Step 7 (verifiers) can start after 2
- Steps 8-10 (ACT, escalate, modes) depend on 5-7
- Steps 11-12 (time/logging extras) extend 1/2
- Steps 13-15 (idempotency, security, API) after core loop (8-10)
- Steps 16-22 (eval, tests, perf, docs, release) last

## Bonus Test Cases

- Malformed planner output → reprompt → schema pass
- Calendar with ambiguous local format ("09/08/2025") → normalized to "Monday, September 8, 2025"  
- Stock tool returns 29 days → verifier fails; small escalates → big fixes
- Rate-limit 429 on small → backoff retry succeeds within budgets
- Idempotent run_case re-invocation with same key returns cached result