# C4 Architecture Diagrams — LocalScript API

## Level 1: System Context

```mermaid
graph LR
    Client["Client\n(browser, curl, IDE)"]
    API["LocalScript API\nFastAPI :8080"]
    LLM["LLM Backend\n(Ollama / llama.cpp)"]

    Client -->|"HTTP POST /generate\nSSE /generate/stream"| API
    API -->|"OpenAI-compatible API\nchat completions"| LLM
    LLM -->|"generated code\nstructured output"| API
    API -->|"SSE events:\nquestion, code, error"| Client

    classDef client fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#000
    classDef api fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef llm fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#000

    class Client client
    class API api
    class LLM llm
```

## Level 2: Containers

```mermaid
graph TB
    subgraph "LocalScript API Container"
        direction TB

        FastAPI["FastAPI Application\n\nRoutes:\n- POST /generate\n- POST /generate/stream\n- POST /session/id/answer\n- GET /health"]

        Engine["LocalScriptEngine\n\n- LangGraph shim\n- Lua validation\n- Session manager\n- Retry logic (max 3)"]

        FastAPI -->|"Depends"| Engine
    end

    subgraph "External Services"
        LLM["LLM Backend\n(Ollama / llama.cpp)"]
        Lua["Lua Runtime\n\n- luac -p (syntax)\n- luacheck (lint)\n- lua (run)"]
    end

    Engine -->|"OpenAI API"| LLM
    Engine -->|"subprocess"| Lua

    Client(("Client")) -->|"HTTP/SSE"| FastAPI

    classDef fastapi fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    classDef engine fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef external fill:#f5f5f5,stroke:#616161,stroke-width:2px,color:#000
    classDef client fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000

    class FastAPI fastapi
    class Engine engine
    class LLM,Lua external
    class Client client
```

## Level 3: Components (LocalScriptEngine)

```mermaid
graph TB
    subgraph "LocalScriptEngine"
        direction TB

        LLMClient["LangChainLLMClient\n\n- ChatOpenAI wrapper\n- structured output\n- tool calling\n- OpenAI-compatible"]

        Shim["EvalLangGraphShim\n\n- EvalTask to LangGraphState\n- ToolOrchestrator\n- classify, executor, guardrails\n- state to ExecutionResult"]

        SessionMgr["SessionManager\n\n- create/get/finish session\n- asyncio.Queue for answers\n- wait_for_answer (timeout 120s)\n- clarification tracking"]

        Validator["LuaCodeEvaluator\n\n- syntax: luac -p\n- lint: luacheck\n- run: lua + timeout\n- sequential validation"]

        LLMClient -->|"smart_client"| Shim
        SessionMgr -.->|"session"| Shim
        Shim -->|"generated code"| Validator

        style LLMClient fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#000
        style Shim fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
        style SessionMgr fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#000
        style Validator fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000
    end

    Retry["Retry Loop\n\n- generate via LangGraph\n- check for QUESTION\n- validate Lua\n- save code or retry\n- max 3 retries"]

    Shim -->|"code + result"| Retry
    Retry -->|"error feedback"| Shim
    Validator -->|"valid/invalid"| Retry

    style Retry fill:#fce4ec,stroke:#880e4f,stroke-width:2px,stroke-dasharray: 5 5,color:#000
```

## Level 4: Code — LangGraph Eval Pipeline

```mermaid
graph TB
    subgraph "LangGraph Eval Pipeline"
        direction TB

        Start([Start])
        Classify["classify_task_node\n\nDefines role sequence:\nmulti, executor,\nstrategist, reviewer"]

        Strategist["strategist_subgraph\n\n- Architecture analysis\n- Plan generation\n- FinalPlanResponse\n- tools: read_file, grep_search"]

        Executor["executor_subgraph\n\n- Tool-calling loop (max 10)\n- read_file, write_file, grep_search\n- FinalCodeResponse\n- clarification loop (max 3)"]

        Reviewer["reviewer_subgraph\n\n- Code review\n- FinalReviewResponse\n- verdict: APPROVED/WARN/BLOCKER"]

        Guardrails["guardrails_node\n\n- Architecture boundary check\n- blocker triggers error_handler"]

        ErrorHandler["error_handler\n\n- Error handling\n- retry_count less than max_retries?"]

        End([End])

        Start --> Classify
        Classify -->|"strategist"| Strategist
        Classify -->|"executor"| Executor
        Classify -->|"reviewer"| Reviewer

        Strategist -->|"plan ready"| Executor
        Executor -->|"code ready"| Reviewer
        Reviewer -->|"review done"| Guardrails

        Guardrails -->|"ok"| End
        Guardrails -->|"blocker"| ErrorHandler

        ErrorHandler -->|"retry"| Executor
        ErrorHandler -->|"fail"| End

        subgraph "Executor Subgraph - Clarification Loop"
            ExecNode["executor node\n\ngenerate code via tool-calling"]
            ExtractCode["extract_code_node\n\nExtracts code from response"]
            Clarification["clarification_node\n\nIf QUESTION: wait for answer\nmax_qa_rounds = 3"]

            ExecNode --> ExtractCode
            ExtractCode -->|"no question"| AfterRole
            ExtractCode -->|"QUESTION:"| Clarification
            Clarification -->|"answer received"| ExecNode
        end

        Executor -.->|"internal"| ExecNode

        style Start fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px,color:#000
        style End fill:#ffcdd2,stroke:#c62828,stroke-width:3px,color:#000
        style Classify fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
        style Strategist fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#000
        style Executor fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
        style Reviewer fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#000
        style Guardrails fill:#fce4ec,stroke:#880e4f,stroke-width:2px,color:#000
        style ErrorHandler fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
    end
```

## Level 4: Code — SSE Streaming Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant A as FastAPI
    participant S as SessionManager
    participant E as LocalScriptEngine
    participant G as LangGraphShim
    participant V as LuaCodeEvaluator

    C->>A: POST /generate/stream
    A->>S: create(session_id)
    S-->>A: Session created
    A-->>C: SSE: session_created

    A->>E: generate_code_stream()

    loop Retry (max 3)
        E->>G: run_task(task)

        alt LLM asks question (QUESTION:)
            G-->>E: result with QUESTION:
            E-->>C: SSE: question
            C->>A: POST /session/id/answer
            A->>S: answer_queue.put(answer)
            S-->>E: answer received (timeout 120s)
            E->>E: Create new task with feedback

        else LLM generates code
            G-->>E: result with code
            E->>V: evaluate(code)

            V-->>E: valid=True/False

            alt Validation failed
                E-->>C: SSE: error
                E->>E: Retry with error_feedback

            else Validation passed
                E->>E: save_code()
                E-->>C: SSE: validating
                E-->>C: SSE: code
            end
        end
    end

    S->>S: finish(session_id)
```

## Level 4: Code — Modules and Dependencies

```mermaid
graph LR
    subgraph "localscript_api"
        main["main.py\n\n- FastAPI app\n- lifespan\n- routes\n- Depends"]
        engine["engine.py\n\n- LocalScriptEngine\n- generate_code\n- generate_code_stream\n- _validate_lua"]
        sessions["sessions.py\n\n- Session dataclass\n- SessionManager\n- wait_for_answer"]
        sse["sse.py\n\n- format_sse_event\n- sse_error\n- sse_code_only"]
        schemas["schemas.py\n\n- GenerateRequest\n- GenerateResponse\n- SseEventType\n- SseEvent"]
        api["api.py\n\n- APIRouter placeholder"]
    end

    subgraph "eval"
        shim["langgraph_eval_shim.py\n\n- EvalLangGraphShim\n- run_task\n- ToolOrchestrator"]
        lua_eval["lua_evaluator.py\n\n- LuaCodeEvaluator\n- syntax/lint/run"]
        models["models.py\n\n- EvalTask\n- EvalResult\n- FinalCodeResponse"]
    end

    subgraph "swarm/langgraph"
        graph_mod["graph.py\n\n- LangGraphEvalPipeline\n- build_eval_graph"]
        nodes["nodes.py\n\n- executor_node\n- reviewer_node\n- guardrails_node"]
        state["state.py\n\n- LangGraphState\n- create_initial_state"]
    end

    main --> engine
    main --> sessions
    main --> sse
    main --> schemas

    engine --> shim
    engine --> lua_eval
    engine --> sessions

    shim --> graph_mod
    shim --> nodes
    shim --> state

    classDef api_mod fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    classDef eval_mod fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef swarm_mod fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#000

    class main,engine,sessions,sse,schemas,api api_mod
    class shim,lua_eval,models eval_mod
    class graph_mod,nodes,state swarm_mod
```
