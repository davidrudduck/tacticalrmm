# Tactical RMM - Development Guidelines

> Remote Monitoring & Management platform backend (Python/Django + Go)

## 1. Core Principles

### Development Philosophy
- **TDD (Test-Driven Development)**: Write tests first, then implementation
  1. RED: Write a failing test
  2. GREEN: Write minimal code to pass
  3. REFACTOR: Clean up while keeping tests green
- **YAGNI**: Don't build features until they're needed
- **KISS**: Choose the simplest solution that works

### Non-Negotiable Standards
- **File size limit**: 500 lines maximum per file (strict for new files)
- **Type safety**: All Python functions must have type hints
- **Documentation**: Create `INDEX.md` in each new folder describing contents
- **Docs organization**: Keep documentation in `docs/` directory
- **Mobile-first**: Consider mobile UI/UX in all design decisions

## 2. Git Workflow

### Branching Strategy
- **`develop`**: Primary development branch - all PRs merge here
- **`release`**: Stable release branch - updated for major version releases
- Feature branches: `feature/description` or `dr/ticket-id-description`
- Bug fix branches: `fix/description`

### Pull Request Process
1. Create feature branch from `develop`
2. Implement changes with TDD
3. Run tests and linters locally
4. Create PR targeting `develop` on `davidrudduck/tacticalrmm`
5. Address review feedback
6. Squash merge when approved

### Commit Messages
- Use conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`
- Keep first line under 72 characters
- Reference issue numbers when applicable

## 3. Tech Stack

### Python Backend
| Component | Technology | Version |
|-----------|------------|---------|
| Framework | Django | 4.2 |
| API | Django REST Framework | 3.15 |
| Database | PostgreSQL | 13 |
| DB Driver | psycopg | 3.3 |
| Task Queue | Celery + Redis | 5.6 |
| Messaging | NATS | 2.12 |
| WebSockets | Django Channels | 4.3 |
| Auth | Knox + django-allauth | - |

### Code Quality Tools
| Tool | Purpose |
|------|---------|
| Black | Code formatting |
| isort | Import sorting |
| MyPy | Type checking |
| pytest | Testing |
| model_bakery | Test fixtures |

### Go Service (natsapi)
| Component | Technology | Version |
|-----------|------------|---------|
| Language | Go | 1.23 |
| Database | sqlx + lib/pq | - |
| Messaging | nats.go | 1.45 |
| Logging | logrus | 1.9 |
| Encoding | msgpack (ugorji/go/codec) | - |

## 4. Architecture

### Python/Django Structure
```text
api/tacticalrmm/
├── {app}/                  # Feature modules
│   ├── models.py           # Django ORM models
│   ├── views.py            # DRF API views
│   ├── serializers.py      # Request/response serialization
│   ├── permissions.py      # RBAC permission classes
│   ├── urls.py             # URL routing
│   ├── tasks.py            # Celery async tasks
│   ├── consumers.py        # WebSocket consumers (if needed)
│   └── tests.py            # Unit tests
├── tacticalrmm/            # Core Django config
│   ├── settings.py         # Django settings
│   ├── urls.py             # Root URL config
│   ├── helpers.py          # Shared utilities
│   ├── constants.py        # App-wide constants
│   ├── test.py             # TacticalTestCase base class
│   └── permissions.py      # Shared permission utilities
├── apiv3/                  # Agent-facing API
└── apiv4/                  # Internal API
```

### Go Service Structure
```text
natsapi/
├── main.go        # Entry point, CLI flags
├── svc.go         # NATS message handlers
├── types.go       # Type definitions
├── utils.go       # Utility functions
└── bin/           # Compiled binaries
```

### Data Flow
```text
Agent (Win/Linux/Mac) → NATS → Go Service → PostgreSQL
                                    ↓
Django API ← Redis/Celery ← WebSocket → Browser
```

## 5. Code Style

### Python Naming Conventions
```python
# Classes: PascalCase
class GetAddClients(APIView):
    pass

class ClientSerializer(ModelSerializer):
    pass

# Functions/methods: snake_case
def get_clients(request):
    pass

def filter_by_role(self, user):
    pass

# Variables: snake_case
client_count = 5
is_authenticated = True

# Constants: UPPER_SNAKE_CASE
TRMM_VERSION = "1.3.2"
DEFAULT_TIMEOUT = 30

# Model fields: snake_case
class Agent(models.Model):
    agent_id = models.CharField(max_length=200)
    last_seen = models.DateTimeField(null=True)
    operating_system = models.CharField(max_length=255)
```

### Go Naming Conventions
```go
// Exported (public): PascalCase
func Svc(logger *logrus.Logger, cfg string) { }
type CheckInNats struct { }

// Private: camelCase
var agentID string
func getConfig() { }

// Constants: PascalCase
const MaxRetries = 3
```

### Import Order (Python)
```python
# 1. Standard library
import os
from datetime import datetime

# 2. Third-party
from django.conf import settings
from rest_framework.views import APIView

# 3. Local application
from agents.models import Agent
from tacticalrmm.helpers import notify_error
```

## 6. Logging

### Python Logging (Best Practice)
```python
import logging

logger = logging.getLogger(__name__)

# Log with context
logger.info(
    "Client created",
    extra={
        "client_id": client.id,
        "user": request.user.username,
        "operation": "create",
    }
)

# Error logging with exception info
try:
    process_data()
except Exception as e:
    logger.error(
        "Failed to process data",
        extra={"agent_id": agent.id},
        exc_info=True
    )
```

### Go Logging
```go
import "github.com/sirupsen/logrus"

// Structured logging with fields
logger.WithFields(logrus.Fields{
    "agent_id": p.Agentid,
    "version":  p.Version,
    "topic":    msg.Reply,
}).Debugln("Processing agent hello")

// Error logging
if err != nil {
    logger.WithError(err).Errorln("Database update failed")
}
```

### Log Levels
| Level | Use Case |
|-------|----------|
| DEBUG | Development details, message contents |
| INFO | Normal operations, successful actions |
| WARNING | Unexpected but handled situations |
| ERROR | Failures requiring attention |
| FATAL | Unrecoverable errors (Go only) |

## 7. Testing

### TDD Workflow
1. **Write a failing test** that describes the expected behavior
2. **Run the test** to confirm it fails (RED)
3. **Write minimal code** to make the test pass (GREEN)
4. **Refactor** while keeping tests green
5. **Repeat** for each new feature or bug fix

### Python Test Structure
```python
from model_bakery import baker
from tacticalrmm.test import TacticalTestCase

class TestClientViews(TacticalTestCase):
    def setUp(self):
        self.authenticate()
        self.setup_coresettings()

    def test_get_clients_returns_list(self):
        # Arrange
        baker.make("clients.Client", _quantity=5)

        # Act
        response = self.client.get("/clients/", format="json")

        # Assert
        self.assertEqual(response.status_code, 200)
        self.assertEqual(len(response.data), 5)

    def test_get_clients_requires_authentication(self):
        self.check_not_authenticated("get", "/clients/")

    def test_add_client_with_valid_data(self):
        payload = {
            "client": {"name": "New Client"},
            "site": {"name": "Default Site"},
            "custom_fields": [],
        }
        response = self.client.post("/clients/", payload, format="json")
        self.assertEqual(response.status_code, 200)

    def test_add_client_rejects_invalid_name(self):
        payload = {
            "client": {"name": "Invalid|Name"},  # Pipe not allowed
            "site": {"name": "Site"},
            "custom_fields": [],
        }
        response = self.client.post("/clients/", payload, format="json")
        self.assertEqual(response.status_code, 400)
```

### Test Commands
```bash
# Run all tests
cd api/tacticalrmm && pytest

# Run specific test file
pytest clients/tests.py

# Run specific test
pytest -k test_get_clients

# Run with coverage
pytest --cov

# Run with verbose output
pytest -vv
```

### Test Fixtures with model_bakery
```python
# Create single object
client = baker.make("clients.Client")

# Create multiple objects
agents = baker.make("agents.Agent", _quantity=10)

# Create with specific values
agent = baker.make(
    "agents.Agent",
    hostname="test-pc",
    operating_system="Windows 10",
)

# Create with relationships
site = baker.make("clients.Site", client=client)
```

## 8. API Contracts

### REST Conventions
| Method | Action | Success Code |
|--------|--------|--------------|
| GET | List/Retrieve | 200 |
| POST | Create | 200 or 201 |
| PUT | Update | 200 |
| DELETE | Delete | 200 or 204 |

### Authentication
- Token-based via Knox (`Authorization: Token <token>`)
- Token TTL: 5 hours with auto-refresh
- Agent tokens use DRF Token auth

### View Pattern
```python
from django.shortcuts import get_object_or_404
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

from tacticalrmm.helpers import notify_error

from .models import Client
from .permissions import ClientsPerms
from .serializers import ClientSerializer

class GetUpdateDeleteClient(APIView):
    permission_classes = [IsAuthenticated, ClientsPerms]

    def get(self, request, pk):
        client = get_object_or_404(Client, pk=pk)
        return Response(ClientSerializer(client).data)

    def put(self, request, pk):
        client = get_object_or_404(Client, pk=pk)
        serializer = ClientSerializer(client, data=request.data, partial=True)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response("Client updated")

    def delete(self, request, pk):
        client = get_object_or_404(Client, pk=pk)
        client.delete()
        return Response("Client deleted")
```

### Error Responses
```python
from tacticalrmm.helpers import notify_error

# Returns Response with 400 status
return notify_error("Client name cannot contain | character")
```

## 9. Common Patterns

### Pattern 1: Permission Class
```python
from rest_framework import permissions
from tacticalrmm.permissions import _has_perm, _has_perm_on_client

class ClientsPerms(permissions.BasePermission):
    def has_permission(self, request, view) -> bool:
        if request.method == "GET":
            return _has_perm(request, "can_list_clients")
        elif request.method == "POST":
            return _has_perm(request, "can_manage_clients")
        return False
```

### Pattern 2: Celery Task
```python
from celery import shared_task

@shared_task
def send_agent_update(agent_id: str) -> str:
    from agents.models import Agent

    agent = Agent.objects.get(agent_id=agent_id)
    # ... perform async work
    return f"Updated agent {agent_id}"
```

### Pattern 3: Go NATS Handler
```go
nc.Subscribe("*", func(msg *nats.Msg) {
    var mh codec.MsgpackHandle
    mh.RawToString = true
    dec := codec.NewDecoderBytes(msg.Data, &mh)

    switch msg.Reply {
    case "agent-hello":
        go func() {
            var p trmm.CheckInNats
            if err := dec.Decode(&p); err == nil {
                now := time.Now().UTC()
                logger.Debugln("Hello", p.Agentid)

                stmt := `UPDATE agents_agent
                         SET last_seen=$1, version=$2
                         WHERE agent_id=$3`
                _, err = db.Exec(stmt, now, p.Version, p.Agentid)
                if err != nil {
                    logger.Errorln(err)
                }
            }
        }()
    }
})
```

### Pattern 4: Model with Enums
```python
from django.db import models
from tacticalrmm.constants import AgentMonType

class Agent(models.Model):
    class MonitoringType(models.TextChoices):
        SERVER = "server", "Server"
        WORKSTATION = "workstation", "Workstation"

    monitoring_type = models.CharField(
        max_length=30,
        choices=MonitoringType.choices,
        default=MonitoringType.SERVER,
    )
```

## 10. Development Commands

### Python Backend
```bash
# Navigate to API directory
cd api/tacticalrmm

# Activate virtual environment
source ../env/bin/activate

# Run development server
python manage.py runserver

# Database migrations
python manage.py makemigrations
python manage.py migrate

# Run tests
pytest
pytest -k test_name          # Specific test
pytest --cov                  # With coverage

# Code formatting
black .
isort .

# Type checking
mypy --config-file=../../mypy.ini .
```

### Go Service
```bash
# Build
go build -o natsapi/bin/natsapi ./natsapi

# Run tests
go test ./...

# Format code
gofmt -w .
goimports -w .
```

### Docker Development
```bash
# Start all services
cd docker && docker-compose up -d

# View logs
docker-compose logs -f tactical

# Rebuild after changes
docker-compose build tactical
```

## 11. AI Coding Assistant Instructions

1. **Run formatters before committing**: `black . && isort .` for Python, `gofmt -w .` for Go
2. **Follow TDD**: Write tests first, then implement the feature
3. **Respect file limits**: Keep files under 500 lines; split if approaching limit
4. **Use type hints**: All Python functions must have type annotations
5. **Extend TacticalTestCase**: Use it as base class for all tests
6. **Use baker.make()**: Create test fixtures with model_bakery, not manual instantiation
7. **Follow permission patterns**: Always use `permission_classes` on API views
8. **Use notify_error()**: For 400 error responses, use the helper function
9. **Check mypy**: Ensure type checking passes before committing
10. **Create INDEX.md**: Add documentation index files in new directories
11. **Consult existing code**: Look at similar files for patterns before implementing
12. **Keep it simple**: Choose the simplest solution that meets requirements
