# Requirements Document

## Introduction

This feature implements an HRM-style decision-making agent that uses a small LLM as the primary planner/executor with H/L loops and ACT (adaptive halting) for efficient task execution. The system escalates to a big LLM only when needed, optimizing for both performance and cost efficiency. The agent supports two operational modes: full HRM loop with escalation and fallback-only mode.

## Requirements

### Requirement 1

**User Story:** As a developer, I want an agent that can solve tasks using a small LLM first, so that I can reduce computational costs while maintaining high success rates.

#### Acceptance Criteria

1. WHEN a task is submitted THEN the system SHALL initialize with the small model (gpt-5-nano) as the primary tier
2. WHEN the small model processes a task THEN the system SHALL use H/L loop with ACT for adaptive halting
3. WHEN the small model succeeds within budget THEN the system SHALL complete without escalating to the big model
4. WHEN the small model fails or exceeds budget THEN the system SHALL escalate to the big model (gpt-5)

### Requirement 2

**User Story:** As a system administrator, I want configurable budget controls, so that I can manage resource consumption and prevent runaway processes.

#### Acceptance Criteria

1. WHEN configuring the system THEN the system SHALL support max_llm_calls_small limit of 6 calls
2. WHEN configuring the system THEN the system SHALL support max_llm_calls_big limit of 2 calls  
3. WHEN configuring the system THEN the system SHALL support max_tool_calls limit of 6 calls
4. WHEN configuring the system THEN the system SHALL support max_seconds timeout of 60 seconds
5. WHEN any budget limit is exceeded THEN the system SHALL either escalate or stop based on escalate_when configuration

### Requirement 3

**User Story:** As a developer, I want the agent to support two operational modes, so that I can choose the appropriate strategy based on task complexity.

#### Acceptance Criteria

1. WHEN mode is "hrm_small_then_escalate" THEN the system SHALL run full H/L loop on small model before escalating
2. WHEN mode is "fallback_only" THEN the system SHALL run single/mini loop on small model before escalating
3. WHEN escalation occurs THEN the system SHALL allow the big model up to 2 attempts (main + 1 retry)
4. WHEN big model fails after retry THEN the system SHALL set finish_reason to "big_fail" and stop

### Requirement 4

**User Story:** As a developer, I want comprehensive error handling and retry mechanisms, so that temporary failures don't cause complete task failure.

#### Acceptance Criteria

1. WHEN a tool call fails THEN the system SHALL return error dict and continue execution without throwing exceptions
2. WHEN network timeouts occur THEN the system SHALL log the error and attempt retry if within budget
3. WHEN rate limiting is encountered THEN the system SHALL implement exponential backoff retry strategy
4. WHEN unknown tools are requested THEN the system SHALL log warning and skip the action
5. WHEN the big model is allowed retry THEN the system SHALL provide failure context from previous attempt

### Requirement 5

**User Story:** As a developer, I want the agent to work with various tool types, so that it can handle diverse task categories like math, calendar, and data analysis.

#### Acceptance Criteria

1. WHEN math tasks are submitted THEN the system SHALL use calculator tool and verify exact numeric answers
2. WHEN calendar queries are submitted THEN the system SHALL use calendar tool and normalize date formats
3. WHEN stock analysis is requested THEN the system SHALL use data_fetch_stock, numeric_analysis, and plotter tools
4. WHEN tools are executed THEN the system SHALL append results to evidence and artifacts
5. WHEN tool registry is accessed THEN all tools SHALL return dict format and never raise exceptions

### Requirement 6

**User Story:** As a developer, I want ACT (adaptive halting) functionality, so that the system can dynamically decide when to continue, stop, or escalate.

#### Acceptance Criteria

1. WHEN verifier passes THEN ACT gate SHALL return "stop" and set verified to true
2. WHEN progress stalls beyond no_progress_patience THEN ACT gate SHALL decide to escalate or stop
3. WHEN act_steps exceeds max_steps THEN ACT gate SHALL trigger budget-based decision
4. WHEN best_metric improves by less than min_improvement THEN the system SHALL increment stalled_rounds
5. WHEN escalate_when is "fail_or_budget" and budget exceeded THEN ACT gate SHALL return "escalate"

### Requirement 7

**User Story:** As a developer, I want comprehensive logging and state tracking, so that I can debug issues and analyze performance.

#### Acceptance Criteria

1. WHEN any operation occurs THEN the system SHALL log the event with timestamp and details
2. WHEN LLM calls are made THEN the system SHALL increment appropriate counters (llm_calls_small/big)
3. WHEN tool calls are executed THEN the system SHALL increment tool_calls counter
4. WHEN state changes occur THEN the system SHALL update tier, verified, finish_reason, and answer fields
5. WHEN evaluation runs THEN the system SHALL output success@1 rate and average resource usage metrics

### Requirement 8

**User Story:** As a developer, I want the planner to output structured JSON plans, so that the executor can reliably parse and execute actions.

#### Acceptance Criteria

1. WHEN planner is called THEN it SHALL output ONLY JSON format with 1-3 actions
2. WHEN JSON is parsed THEN the system SHALL validate action length ≤ 3
3. WHEN JSON is parsed THEN the system SHALL validate all tool names exist in TOOL_REGISTRY
4. WHEN big model replans THEN it SHALL receive previous plan and failure context
5. WHEN planning fails THEN the system SHALL log error and handle gracefully

### Requirement 9

**User Story:** As a platform integrator, I want explicit API contracts, so that services can integrate reliably.

#### Acceptance Criteria

1. WHEN the agent is called THEN it SHALL expose run_case(input: string, expected_tools?: string[], gold_answer?: string|null) entrypoint
2. WHEN run_case completes THEN it SHALL return {answer?, verified, finish_reason, logs, artifacts, counters}
3. WHEN inputs are provided THEN they SHALL be UTF-8 text with max size configurable via max_input_chars (default: 8,192)
4. WHEN outputs are generated THEN they SHALL be JSON format with binary artifacts referenced by paths or URIs
5. WHEN calendar tools are used THEN timezone SHALL default to system tz unless explicitly provided

### Requirement 10

**User Story:** As an executor, I need a strict plan schema, so that I can avoid parsing ambiguity.

#### Acceptance Criteria

1. WHEN planner outputs JSON THEN it SHALL conform to strict schema with tool and args fields
2. WHEN schema validation fails THEN the system SHALL log plan_schema_error and reprompt once
3. WHEN tools not in TOOL_REGISTRY are requested THEN the system SHALL reject with unknown_tool warning
4. WHEN plan has 1-3 items THEN the system SHALL accept and process the plan
5. WHEN plan validation passes THEN the executor SHALL process each action sequentially

### Requirement 11

**User Story:** As an SRE, I need deterministic halting and escalation behavior, so that system behavior is predictable.

#### Acceptance Criteria

1. WHEN ACT computes metrics THEN improved SHALL be (metric - best_metric) >= min_improvement
2. WHEN passed is true THEN the system SHALL stop with finish_reason="success"
3. WHEN budget limits exceeded OR stalled_rounds > patience THEN the system SHALL escalate or stop based on escalate_when
4. WHEN escalation occurs THEN tier SHALL switch to big and evidence SHALL be preserved
5. WHEN escalation happens THEN logs SHALL append {event:"escalate_to_big"}

### Requirement 12

**User Story:** As a product owner, I want measurable targets for latency and cost, so that I can track system performance.

#### Acceptance Criteria

1. WHEN running "easy" single-tool tasks THEN p95 end-to-end latency SHALL be ≤ 2.0s in fallback_only mode
2. WHEN processing test set THEN average big-model escalation rate SHALL be ≤ 15%
3. WHEN calculating costs THEN mean token cost per case SHALL be within configured limits
4. WHEN running provided test set THEN success@1 SHALL be ≥ 95%
5. WHEN performance targets are not met THEN the system SHALL log performance warnings

### Requirement 13

**User Story:** As a developer, I want robust failure semantics, so that temporary failures don't cause complete system failure.

#### Acceptance Criteria

1. WHEN tool exceptions occur THEN they SHALL be caught and returned as {error:"...", code?} in evidence
2. WHEN LLM provider errors occur THEN they SHALL be retried with exponential backoff (base=0.5s, max_retries=2)
3. WHEN big model gets retry THEN failure context SHALL be included in the prompt
4. WHEN network timeouts happen THEN the system SHALL implement appropriate retry strategies
5. WHEN retries are exhausted THEN the system SHALL fail gracefully with proper error reporting

### Requirement 14

**User Story:** As QA, I want full traceability, so that I can debug issues and analyze system behavior.

#### Acceptance Criteria

1. WHEN any step executes THEN it SHALL append to logs with timestamp, tier, step_type, and relevant details
2. WHEN runner processes cases THEN it SHALL emit JSONL trace per case for offline analysis
3. WHEN operations complete THEN counters SHALL be reported in final payload
4. WHEN debugging is needed THEN logs SHALL contain sufficient detail for root cause analysis
5. WHEN audit is required THEN all system actions SHALL be traceable through logs

### Requirement 15

**User Story:** As an administrator, I need centralized, file-based config, so that I can manage system settings efficiently.

#### Acceptance Criteria

1. WHEN system starts THEN it SHALL load config.yaml with models, budget, act, verifier_thresholds, flags
2. WHEN environment variables are set THEN numeric limits SHALL be overridable (e.g., BUDGET_MAX_SECONDS)
3. WHEN invalid config keys exist THEN startup SHALL fail with clear error message
4. WHEN configuration changes THEN system SHALL validate all required fields are present
5. WHEN config is loaded THEN system SHALL use provided values or documented defaults

### Requirement 16

**User Story:** As a tester, I need a repeatable harness, so that I can validate system behavior consistently.

#### Acceptance Criteria

1. WHEN runner is called THEN it SHALL accept dataset JSON array with {id, input, expected_tools?, gold_answer?}
2. WHEN each case runs THEN harness SHALL collect outputs and compute success@1, avg_llm_small, avg_llm_big metrics
3. WHEN CI runs THEN harness SHALL fail if success@1 < target or escalation rate > target
4. WHEN tests complete THEN system SHALL provide comprehensive performance metrics
5. WHEN test cases are added THEN harness SHALL handle them without code changes

### Requirement 17

**User Story:** As a developer, I want clear enumerations and defaults, so that configuration is unambiguous.

#### Acceptance Criteria

1. WHEN finish_reason is set THEN it SHALL be one of: "success" | "budget" | "big_fail"
2. WHEN mode is specified THEN it SHALL be one of: "hrm_small_then_escalate" | "fallback_only"
3. WHEN escalate_when is configured THEN it SHALL be one of: "fail" | "fail_or_budget" with default "fail_or_budget"
4. WHEN config keys are omitted THEN documented defaults SHALL apply
5. WHEN system starts THEN all defaults SHALL be listed in config.yaml.example

### Requirement 18

**User Story:** As a parser, I need strict JSON schema validation, so that plan execution is reliable.

#### Acceptance Criteria

1. WHEN planner outputs JSON THEN it SHALL conform to strict schema with required tool and args fields
2. WHEN schema validation fails THEN system SHALL log plan_schema_error and reprompt once
3. WHEN tool values are provided THEN they SHALL exist in TOOL_REGISTRY
4. WHEN reprompt fails THEN system SHALL count toward llm_calls_* and handle gracefully
5. WHEN plan is valid THEN executor SHALL process actions in sequence

### Requirement 19

**User Story:** As a security officer, I want structured logging with PII redaction, so that sensitive data is protected.

#### Acceptance Criteria

1. WHEN steps execute THEN they SHALL append JSON log objects with ts, tier, step_type, and relevant fields
2. WHEN logs contain sensitive data THEN PII/secret fields SHALL be redacted per allowlist
3. WHEN redaction is applied THEN redaction policy SHALL be unit-tested
4. WHEN logs are generated THEN they SHALL include token_usage, tool, args, output_summary, metric, decision
5. WHEN debugging is needed THEN logs SHALL provide sufficient detail while maintaining security

### Requirement 20

**User Story:** As a cost analyst, I want detailed token and cost accounting, so that I can track resource usage.

#### Acceptance Criteria

1. WHEN operations complete THEN counters SHALL include llm_calls_small, llm_calls_big, tool_calls, act_steps
2. WHEN token usage occurs THEN it SHALL be tracked by tier with input, output, and cost fields
3. WHEN final payload is generated THEN it SHALL include aggregated token/cost per tier and total
4. WHEN cost tracking is enabled THEN all LLM calls SHALL record accurate usage metrics
5. WHEN reporting is needed THEN cost data SHALL be available in structured format

### Requirement 21

**User Story:** As a reliability engineer, I want specified retry and backoff behavior, so that transient failures are handled consistently.

#### Acceptance Criteria

1. WHEN LLM/provider/network errors occur THEN system SHALL use exponential backoff: base=0.5s, multiplier 2.0, jitter ±20%
2. WHEN retries are attempted THEN max_retries SHALL be 2 while respecting budgets
3. WHEN tool-level retries are enabled THEN they SHALL not exceed max_tool_calls
4. WHEN backoff is applied THEN it SHALL include jitter to prevent thundering herd
5. WHEN retry budget is exhausted THEN system SHALL fail gracefully

### Requirement 22

**User Story:** As a tester, I want deterministic and reproducible behavior, so that I can debug issues reliably.

#### Acceptance Criteria

1. WHEN seed config is provided THEN planner prompts and sampling SHALL be deterministic where supported
2. WHEN runs execute THEN run_id SHALL be included in outputs/logs for correlation
3. WHEN tool execution order matters THEN it SHALL be deterministic given same inputs
4. WHEN debugging issues THEN reproduction SHALL be possible with same seed and inputs
5. WHEN cross-run analysis is needed THEN run_id SHALL enable correlation

### Requirement 23

**User Story:** As a security engineer, I want safety guardrails, so that the system operates within safe boundaries.

#### Acceptance Criteria

1. WHEN planner requests tools THEN only allowlisted tools SHALL be permitted
2. WHEN denylisted tools are requested THEN they SHALL be rejected with unknown_tool error
3. WHEN artifacts are generated THEN max size SHALL be configurable (max_artifact_mb, default 5 MB)
4. WHEN inputs are provided THEN they SHALL be validated for length (max_input_chars, default 8192)
5. WHEN validation fails THEN system SHALL reject with clear error message

### Requirement 24

**User Story:** As a configuration manager, I want mode-specific budget controls, so that different modes operate with appropriate constraints.

#### Acceptance Criteria

1. WHEN "fallback_only" mode is used THEN act.max_steps SHALL be 0 or 1 (configurable)
2. WHEN "hrm_small_then_escalate" mode is used THEN act.max_steps default SHALL be 3 unless overridden
3. WHEN mode-specific budgets are set THEN they SHALL be documented and enforced
4. WHEN budget validation occurs THEN mode-specific limits SHALL be checked
5. WHEN modes switch THEN appropriate budget constraints SHALL apply

### Requirement 25

**User Story:** As a calendar tool user, I want consistent time handling, so that date operations are reliable.

#### Acceptance Criteria

1. WHEN timezone is not specified THEN system SHALL default to system TZ
2. WHEN calendar outputs are generated THEN they SHALL be normalized to "Monday, September 8, 2025" format
3. WHEN date parsing occurs THEN it SHALL be locale-agnostic
4. WHEN ambiguous dates are encountered THEN they SHALL be rejected with clear error
5. WHEN timezone is provided THEN it SHALL override system default

### Requirement 26

**User Story:** As a CI/CD engineer, I want automated quality gates, so that regressions are caught early.

#### Acceptance Criteria

1. WHEN CI runs THEN it SHALL fail if success@1 < 0.95 OR escalation_rate > 0.15 OR p95_latency_easy > 2.0s
2. WHEN CI completes THEN artifacts SHALL include metrics.json and trace.jsonl per case
3. WHEN quality gates fail THEN CI SHALL provide clear failure reasons
4. WHEN metrics are collected THEN they SHALL be comparable across runs
5. WHEN performance degrades THEN CI SHALL block deployment until fixed