# Coduel

A competitive programming platform for real-time multiplayer coding battles with Docker-based secure code execution.

## Overview

Coduel is an online judge system enabling programmers to compete head-to-head by solving algorithmic problems. The platform determines winners based on accuracy, execution time, and memory usage with a 10% tolerance threshold for fair comparison.

## Features

**Multiplayer Competition**
- Real-time battles with configurable rounds (default: best of 3)
- Private room system with unique join codes
- Live code spectating for transparency
- Performance-based winner determination

**Language Support**
- C (C17 standard)
- C++ (C++20 standard)
- Python 3
- Java
- JavaScript (Node.js)

**Secure Execution**
- Docker container isolation per submission
- Configurable CPU and memory limits
- Network disabled in judge containers
- Read-only filesystem with temporary workspace

**Problem Management**
- Web-based CRUD operations
- Markdown-formatted problem statements
- Public samples and hidden test cases
- Difficulty levels: fast, easy, medium, hard
- Tag-based categorization

**Performance Metrics**
- Multiple runs per test for accuracy (default: 3 runs)
- Median time and memory calculation
- Detailed execution logs
- Test case visibility control

## Architecture

The system uses a microservices architecture with five main components:

**Web Server** (Express + Socket.IO, Port 5173)
- Static file serving for frontend
- WebSocket connections for real-time features
- Room state management
- Code synchronization between players
- Match result broadcasting

**API Server** (FastAPI, Port 8000)
- RESTful endpoints for problem operations
- Code submission validation and queueing
- Problem metadata management
- Result retrieval

**Worker Service** (Python + Docker SDK)
- Monitors Redis queues for jobs
- Compiles code in isolated containers
- Executes tests with resource monitoring
- Calculates performance metrics
- Stores results in Redis

**Judge Engine** (Docker Container)
- Ubuntu-based with compilers: GCC, G++, Python, OpenJDK, Node.js
- Executes compile_run.sh for build and test
- Collects metrics via run_with_metrics.py
- Network isolation and read-only filesystem

**Redis** (Port 6379)
- Job queues: queue:compile, queue:run
- Submission metadata: sub:{id}
- Source code: code:{id}
- Results: run_result:{id}
- Compilation logs: compile_log:{id}

## Workflow

1. User submits code via web interface
2. API validates and pushes to Redis compile queue with metadata
3. Worker picks job and creates temporary workspace
4. Worker spawns judge container to compile code
5. On success, job moves to run queue
6. Worker spawns judge container with test case volume mounts
7. Judge executes code multiple times per test case
8. Metrics collected and stored in Redis
9. Results returned via API endpoint or WebSocket

## Installation

**Prerequisites**
- Docker 20.10 or higher
- Docker Compose 2.0 or higher
- Available ports: 5173, 8000, 6379

**Quick Start**
```bash
git clone https://github.com/Saudadeeee/Coduel.git
cd Coduel/Coduel
docker-compose up --build -d
```

**Verify Installation**
```bash
docker ps
```
Expected containers: oj_web, oj_api, oj_worker, oj_redis, oj_judge_builder

**Access Application**
- Web interface: http://localhost:5173
- API documentation: http://localhost:8000/docs

## Configuration

Create .env.judge file for custom worker settings:
```bash
RUNS_PER_TEST=3
PERFORMANCE_TOLERANCE=0.10
CPU_LIMIT=2.0
MEM_LIMIT=1g
COMPILE_TIMEOUT=60
RUN_TIMEOUT=60
```

Defaults if not specified:
- CPU_LIMIT: Half of system cores
- MEM_LIMIT: Half of system RAM
- RUNS_PER_TEST: 3
- PERFORMANCE_TOLERANCE: 0.10 (10%)

## Usage

**Training Mode**
1. Navigate to main menu and select Training Mode
2. Choose a problem from the list
3. Write solution in code editor
4. Select programming language
5. Submit and view results

**Multiplayer Mode**

Host:
1. Click Host Room from main menu
2. Configure settings: rounds, difficulty, spectator mode, language
3. Share generated room code
4. Wait for opponent and ready confirmation
5. Start match

Player:
1. Click Join Room from main menu
2. Enter host-provided room code
3. Mark ready
4. Solve problems and submit solutions

**Code Spectating**
When enabled by host, click eye icon in workspace to view opponent code in real-time.

## API Reference

**Problem Endpoints**

List problems:
```
GET /problems
Response: {"problems": [...]}
```

Get problem:
```
GET /problem/{problem_id}
Response: {meta, statement, tests, samples}
```

Create problem:
```
POST /problem-add
Body: {
  "title": string,
  "difficulty": "easy"|"medium"|"hard"|"fast",
  "description": string,
  "sample_input": string,
  "sample_output": string,
  "tests": [{"input": string, "output": string, "visibility": "public"|"hidden"}],
  "time_limit_ms": number,
  "memory_limit_kb": number,
  "tags": string[]
}
```

Update problem:
```
POST /problem-edit
Body: {"problem_id": string, ...fields}
```

Delete problem:
```
POST /problem-edit
Body: {"problem_id": string, "delete": true}
```

**Submission Endpoints**

Submit code:
```
POST /problem/submit
Body: {
  "problem_id": string,
  "language": "c"|"cpp"|"py"|"java"|"js",
  "code": string,
  "std": string (optional)
}
Response: {"submission_id": string}
```

Check status:
```
GET /problem/submission/{submission_id}
Response: {
  "meta": {"status": string, "problem_id": string, "language": string},
  "compile_log": string,
  "run_result": {
    "verdict": string,
    "tests_passed": number,
    "tests_total": number,
    "performance": {
      "accuracy": number,
      "median_elapsed_seconds": number,
      "median_memory_kb": number
    },
    "test_details": [...]
  }
}
```

Compare submissions:
```
POST /api/compare-submissions
Body: {"submissionA": string, "submissionB": string}
Response: {
  "submissionA": {...},
  "submissionB": {...},
  "comparison": {
    "winner": "A"|"B"|"TIE",
    "reason": "accuracy"|"time"|"memory"|"all_metrics_equal_within_tolerance",
    "details": {...}
  }
}
```

## Winner Determination Algorithm

The system uses a 3-tier comparison with tolerance:

1. Accuracy: Higher tests_passed/tests_total wins immediately
2. Execution Time: Compare median times with 10% tolerance
3. Memory Usage: Compare median memory with 10% tolerance

If all metrics within tolerance, result is TIE.

Example:
```
Player A: 100% accuracy, 0.005s, 2048KB
Player B: 100% accuracy, 0.006s, 2048KB
Time difference: 20% (exceeds 10% tolerance)
Result: Player A wins on time
```

## Project Structure

```
Coduel/
├── api/                    FastAPI server and validation logic
├── judge/                  Docker image with compilers and execution scripts
├── problems/               Problem repository with metadata and test cases
├── web/                    Express server, Socket.IO, and HTML frontend
├── worker/                 Background job processor for compilation and testing
├── worker_tmp/             Temporary directory for code execution
├── docker-compose.yml      Service orchestration
└── .env.judge              Worker configuration
```

## Troubleshooting

**Services won't start**
```bash
docker-compose down -v
docker-compose up --build --force-recreate
```

**Port conflicts**
Edit docker-compose.yml ports section or stop conflicting processes.

**Worker not processing**
```bash
docker logs oj_worker -f
docker exec oj_redis redis-cli LLEN queue:compile
```

**Code not syncing**
Check browser DevTools Network tab for WebSocket connection.
Check web server logs: `docker logs oj_web -f`

**Execution timeout**
Increase RUN_TIMEOUT in .env.judge and restart worker.

## Testing

**Test API**
```bash
curl http://localhost:8000/problems
```

**Test submission**
```bash
curl -X POST http://localhost:8000/problem-submit \
  -H "Content-Type: application/json" \
  -d '{"problem_id":"001-sum-two","language":"cpp","code":"..."}'
```

**Test multiplayer**
Open http://localhost:5173 in two browser windows (one incognito), create room in first, join in second.

## Development

**Code organization**
- api/app.py: FastAPI routes and Pydantic models (700 lines)
- worker/worker.py: Job processing and Docker execution (526 lines)
- web/server.js: Express routes and Socket.IO handlers (758 lines)
- web/public/workspace.html: Frontend code editor (2477 lines)
- judge/compile_run.sh: Compilation and test execution script

**Key technologies**
- FastAPI: API framework with automatic validation
- Socket.IO: Real-time bidirectional communication
- Redis: Message queue and result cache
- Docker SDK: Container orchestration
- CodeMirror: Code editor with syntax highlighting

**Adding new language**
1. Update SOURCE_FILE_MAP in worker/worker.py
2. Add compilation case in judge/compile_run.sh
3. Update ALLOWED_LANG in api/app.py
4. Test compilation and execution

## Contributing

1. Fork repository
2. Create feature branch: `git checkout -b feature-name`
3. Make changes with clear commit messages
4. Test thoroughly
5. Open pull request with description

Follow ESLint/Prettier for JavaScript, Black for Python.

## License

MIT License - Copyright (c) 2025 Saudadeeee

Permission is granted to use, copy, modify, merge, publish, distribute, sublicense, and sell copies of this software. See LICENSE file for full terms.

## Acknowledgments

Inspired by CodinGame Clash of Code. Built with Docker, FastAPI, Socket.IO, Express.js, Redis, and CodeMirror.

## Support

- GitHub Issues: https://github.com/Saudadeeee/Coduel/issues
- Documentation: Project README and code comments
- Logs: `docker-compose logs -f` for debugging
