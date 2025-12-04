# 🎄 Day 24: Production Deployment

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-23
- Understanding of agent safety

## 📝 Overview

Today we prepare our agents for the real world with **production deployment** patterns and best practices!

### Production Considerations

```
┌─────────────────────────────────────────────────────────────┐
│              PRODUCTION REQUIREMENTS                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ⚡ Performance: Fast response times                         │
│  📈 Scalability: Handle increased load                       │
│  🔒 Security: Protect data and systems                       │
│  📊 Observability: Monitor and debug                         │
│  🔄 Reliability: Handle failures gracefully                  │
│  💰 Cost: Optimize resource usage                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Agent as a Service (FastAPI)

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import Optional
import asyncio
import uuid

app = FastAPI(title="Agent API")

# Request/Response models
class AgentRequest(BaseModel):
    message: str
    session_id: Optional[str] = None
    stream: bool = False

class AgentResponse(BaseModel):
    response: str
    session_id: str
    tokens_used: int
    execution_time: float

# Task storage for async processing
tasks = {}

@app.post("/agent/chat", response_model=AgentResponse)
async def chat(request: AgentRequest):
    import time
    start_time = time.time()
    
    session_id = request.session_id or str(uuid.uuid4())
    
    try:
        # Get or create agent session
        agent = get_or_create_agent(session_id)
        
        # Run agent
        response = await agent.run_async(request.message)
        
        execution_time = time.time() - start_time
        
        return AgentResponse(
            response=response,
            session_id=session_id,
            tokens_used=agent.last_token_count,
            execution_time=execution_time
        )
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/agent/chat/stream")
async def chat_stream(request: AgentRequest):
    from fastapi.responses import StreamingResponse
    
    async def generate():
        agent = get_or_create_agent(request.session_id)
        async for chunk in agent.stream(request.message):
            yield f"data: {chunk}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )

# Background task for long-running operations
@app.post("/agent/task")
async def create_task(request: AgentRequest, background_tasks: BackgroundTasks):
    task_id = str(uuid.uuid4())
    tasks[task_id] = {"status": "pending", "result": None}
    
    background_tasks.add_task(run_agent_task, task_id, request.message)
    
    return {"task_id": task_id}

@app.get("/agent/task/{task_id}")
async def get_task_status(task_id: str):
    if task_id not in tasks:
        raise HTTPException(status_code=404)
    return tasks[task_id]

async def run_agent_task(task_id: str, message: str):
    tasks[task_id]["status"] = "running"
    try:
        agent = create_agent()
        result = await agent.run_async(message)
        tasks[task_id] = {"status": "completed", "result": result}
    except Exception as e:
        tasks[task_id] = {"status": "failed", "error": str(e)}
```

### Monitoring & Observability

```python
import logging
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from prometheus_client import Counter, Histogram, start_http_server

# Setup tracing
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Prometheus metrics
REQUEST_COUNT = Counter(
    'agent_requests_total',
    'Total agent requests',
    ['status', 'agent_type']
)

REQUEST_DURATION = Histogram(
    'agent_request_duration_seconds',
    'Request duration in seconds',
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0]
)

TOKEN_USAGE = Counter(
    'agent_tokens_total',
    'Total tokens used',
    ['type']  # input, output
)

ERROR_COUNT = Counter(
    'agent_errors_total',
    'Total agent errors',
    ['error_type']
)

class MonitoredAgent:
    def __init__(self, agent, logger=None):
        self.agent = agent
        self.logger = logger or logging.getLogger(__name__)
    
    async def run(self, message: str) -> str:
        with tracer.start_as_current_span("agent_run") as span:
            span.set_attribute("input_length", len(message))
            
            import time
            start_time = time.time()
            
            try:
                result = await self.agent.run_async(message)
                
                # Record metrics
                duration = time.time() - start_time
                REQUEST_DURATION.observe(duration)
                REQUEST_COUNT.labels(status="success", agent_type="default").inc()
                TOKEN_USAGE.labels(type="input").inc(self.agent.input_tokens)
                TOKEN_USAGE.labels(type="output").inc(self.agent.output_tokens)
                
                span.set_attribute("output_length", len(result))
                span.set_attribute("duration", duration)
                
                self.logger.info(
                    f"Agent completed",
                    extra={
                        "duration": duration,
                        "input_tokens": self.agent.input_tokens,
                        "output_tokens": self.agent.output_tokens
                    }
                )
                
                return result
            
            except Exception as e:
                REQUEST_COUNT.labels(status="error", agent_type="default").inc()
                ERROR_COUNT.labels(error_type=type(e).__name__).inc()
                
                span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
                span.record_exception(e)
                
                self.logger.error(f"Agent error: {e}", exc_info=True)
                raise

# Start metrics server
start_http_server(8000)
```

### Caching for Performance

```python
import hashlib
import json
from functools import lru_cache
import redis

class AgentCache:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.ttl = 3600  # 1 hour
    
    def _cache_key(self, message: str, context: dict = None) -> str:
        content = json.dumps({"message": message, "context": context}, sort_keys=True)
        return f"agent:{hashlib.md5(content.encode()).hexdigest()}"
    
    async def get_cached(self, message: str, context: dict = None) -> Optional[str]:
        key = self._cache_key(message, context)
        cached = self.redis.get(key)
        if cached:
            return cached.decode()
        return None
    
    async def set_cached(self, message: str, response: str, context: dict = None):
        key = self._cache_key(message, context)
        self.redis.setex(key, self.ttl, response)

class CachedAgent:
    def __init__(self, agent, cache: AgentCache):
        self.agent = agent
        self.cache = cache
    
    async def run(self, message: str, use_cache: bool = True) -> str:
        if use_cache:
            cached = await self.cache.get_cached(message)
            if cached:
                return cached
        
        result = await self.agent.run_async(message)
        
        if use_cache:
            await self.cache.set_cached(message, result)
        
        return result
```

### Kubernetes Deployment

```yaml
# agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: agent-service
  template:
    metadata:
      labels:
        app: agent-service
    spec:
      containers:
      - name: agent
        image: your-registry/agent-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: anthropic-api-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: agent-service
spec:
  selector:
    app: agent-service
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Health Checks

```python
from fastapi import FastAPI
import anthropic

app = FastAPI()

@app.get("/health")
async def health_check():
    """Liveness probe - is the service running?"""
    return {"status": "healthy"}

@app.get("/ready")
async def readiness_check():
    """Readiness probe - can the service handle requests?"""
    checks = {}
    
    # Check LLM connection
    try:
        client = anthropic.Anthropic()
        # Quick test call
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=10,
            messages=[{"role": "user", "content": "Hi"}]
        )
        checks["llm"] = "ok"
    except Exception as e:
        checks["llm"] = f"error: {str(e)}"
    
    # Check database
    try:
        # Your database check here
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {str(e)}"
    
    # Check cache
    try:
        # Your cache check here
        checks["cache"] = "ok"
    except Exception as e:
        checks["cache"] = f"error: {str(e)}"
    
    all_ok = all(v == "ok" for v in checks.values())
    
    if all_ok:
        return {"status": "ready", "checks": checks}
    else:
        from fastapi import Response
        return Response(
            content=json.dumps({"status": "not_ready", "checks": checks}),
            status_code=503,
            media_type="application/json"
        )
```

## 📖 Reading Materials

1. [FastAPI Best Practices](https://fastapi.tiangolo.com/deployment/)
2. [Kubernetes Documentation](https://kubernetes.io/docs/home/)
3. [OpenTelemetry for Python](https://opentelemetry.io/docs/instrumentation/python/)

## ✅ Activity: Deploy Your Agent

Create `day24-deployment/` with:

1. FastAPI agent server
2. Health check endpoints
3. Docker and Kubernetes configs
4. Monitoring setup

### Complete This Day

[![Complete Day 24](https://img.shields.io/badge/Complete--Day--24-28a745?logo=github)](../../issues/new?title=Complete+Day+24&labels=complete-day-24&body=✅+I%27ve+completed+Day+24%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-23 | Previous Days | Complete |
| 🎁 **Day 24** | Production Deployment | ✅ Current |
| 🎁 **Day 25** | 🎄 Final Project | 🔜 Tomorrow! |
