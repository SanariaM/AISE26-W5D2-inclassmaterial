# W5D2 — Advanced Software Architecture (DDD + Hexagonal)

**Cohort:** AISE 2026 · **Week/Day:** W5D2 · **Duration:** 3 hours (6:30–9:30pm ET) · **Format:** Zoom

> This single README consolidates everything you need to run the session and submit deliverables: slides outline, two breakout handouts, live‑coding code, quickstart, assignment, and rubric.

---

## Quickstart (one copy you can paste)

### Mac/Linux (bash/zsh)

```bash
# 0) Clone and enter
# git clone https://github.com/<org>/aise26-w5d2-architecture.git
cd aise26-w5d2-architecture

# 1) Create & activate venv
python3 -m venv .venv
source .venv/bin/activate

# 2) Install deps
pip install -r requirements.txt

# 3) Run tests
pytest -q

# 4) Run the API (FastAPI + Uvicorn)
uvicorn app:app --reload
# → http://127.0.0.1:8000/docs
```

### Windows (PowerShell)

```powershell
# 0) Clone and enter
# git clone https://github.com/<org>/aise26-w5d2-architecture.git
cd aise26-w5d2-architecture

# 1) Create & activate venv
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2) Install deps
pip install -r requirements.txt

# 3) Run tests
pytest -q

# 4) Run the API
uvicorn app:app --reload
# → http://127.0.0.1:8000/docs
```

**Requirements**

```
# requirements.txt
fastapi
uvicorn
pytest
pydantic
```

---

## Learning Objectives

* Explain why architecture decisions matter as systems scale.
* Define DDD concepts: **Entities, Value Objects, Aggregates, Repositories, Domain Services**.
* Describe **Hexagonal (Ports & Adapters)** and how it decouples business logic from frameworks/infrastructure.
* Refactor a simple app into **domain / application / infrastructure** layers.
* Connect architecture choices to CI/CD for faster, safer pipelines.

**Secondary**

* Spot coupling/smells (framework leakage, “God” modules).
* Place logic in the right layer.
* Explain how architecture supports automated tests and deploys.

---

## Career Relevance (1 slide)

* Clean architecture → easier onboarding, faster delivery.
* Clear seams enable safer refactors and A/Bs.
* Interview‑ready language: *domain model, adapters, ports, repositories, test seams*.

---

## Before Class (assign in LMS)

* Read: short primer on **DDD** and **Hexagonal Architecture**.
* Watch: 1–2 recent talks on hexagonal/DDD.
* Complete the **5‑question Pre‑Class Quiz** in Canvas.

---

## Repository Structure (starter)

```text
.
├── app.py                          # FastAPI adapter (HTTP)
├── requirements.txt
├── src/
│   ├── domain/                     # Pure business logic
│   │   └── order.py
│   ├── application/                # Use cases + ports
│   │   ├── order_service.py
│   │   └── ports.py
│   └── infrastructure/             # Adapters (DB, HTTP, etc.)
│       └── inmemory_repo.py
└── tests/
    └── test_order_service.py
```

---

## Live‑Coding Code (copy into files)

### `src/domain/order.py`

```python
from dataclasses import dataclass, field
from typing import List

@dataclass(frozen=True)
class OrderItem:
    name: str
    quantity: int
    price: float

    def line_total(self) -> float:
        return self.quantity * self.price

@dataclass
class Order:
    id: str
    items: List[OrderItem] = field(default_factory=list)
    status: str = "OPEN"  # OPEN/CANCELED/COMPLETED

    def total(self) -> float:
        return sum(item.line_total() for item in self.items)

    def cancel(self) -> None:
        if self.status != "CANCELED":
            self.status = "CANCELED"
```

### `src/application/ports.py`

```python
from typing import Protocol, Optional
from src.domain.order import Order

class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def get_by_id(self, id: str) -> Optional[Order]: ...
    def delete(self, id: str) -> None: ...
```

### `src/application/order_service.py`

```python
from typing import List
from src.domain.order import Order, OrderItem
from src.application.ports import OrderRepository

class OrderService:
    def __init__(self, repository: OrderRepository) -> None:
        self.repository = repository

    def create_order(self, items: List[OrderItem], order_id: str) -> Order:
        order = Order(id=order_id, items=items)
        self.repository.save(order)
        return order

    def cancel_order(self, order_id: str) -> bool:
        order = self.repository.get_by_id(order_id)
        if not order:
            return False
        order.cancel()
        self.repository.save(order)
        return True
```

### `src/infrastructure/inmemory_repo.py`

```python
from typing import Optional, Dict
from src.domain.order import Order

class InMemoryOrderRepository:
    def __init__(self) -> None:
        self._db: Dict[str, Order] = {}

    def save(self, order: Order) -> None:
        self._db[order.id] = order

    def get_by_id(self, id: str) -> Optional[Order]:
        return self._db.get(id)

    def delete(self, id: str) -> None:
        if id in self._db:
            del self._db[id]
```

### `tests/test_order_service.py`

```python
import uuid
from src.application.order_service import OrderService
from src.infrastructure.inmemory_repo import InMemoryOrderRepository
from src.domain.order import OrderItem

# test order creation and totals
def test_create_order_and_total():
    repo = InMemoryOrderRepository()
    service = OrderService(repo)
    order_id = str(uuid.uuid4())

    order = service.create_order(
        [OrderItem(name="book", quantity=2, price=12.5),
         OrderItem(name="pen", quantity=3, price=1.2)],
        order_id=order_id
    )

    assert order.id == order_id
    assert round(order.total(), 2) == round(2*12.5 + 3*1.2, 2)
    assert repo.get_by_id(order_id) is not None

# test cancel behavior
def test_cancel_order():
    repo = InMemoryOrderRepository()
    service = OrderService(repo)
    order_id = str(uuid.uuid4())

    service.create_order([OrderItem(name="book", quantity=1, price=10.0)], order_id)
    ok = service.cancel_order(order_id)

    assert ok is True
    assert repo.get_by_id(order_id).status == "CANCELED"

# cancel non-existent order
def test_cancel_missing_order():
    repo = InMemoryOrderRepository()
    service = OrderService(repo)

    ok = service.cancel_order(str(uuid.uuid4()))
    assert ok is False
```

### `app.py` (FastAPI adapter)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List
import uuid

from src.application.order_service import OrderService
from src.infrastructure.inmemory_repo import InMemoryOrderRepository
from src.domain.order import OrderItem

class ItemPayload(BaseModel):
    name: str = Field(..., min_length=1)
    quantity: int = Field(..., ge=1)
    price: float = Field(..., ge=0.0)

class CreateOrderPayload(BaseModel):
    items: List[ItemPayload]

repo = InMemoryOrderRepository()
service = OrderService(repo)
app = FastAPI(title="W5D2 DDD + Hexagonal Demo")

@app.post("/orders")
def create_order(payload: CreateOrderPayload):
    items = [OrderItem(name=i.name, quantity=i.quantity, price=i.price) for i in payload.items]
    order_id = str(uuid.uuid4())
    order = service.create_order(items, order_id)
    return {"order_id": order.id, "total": order.total()}

@app.get("/orders/{order_id}")
def get_order(order_id: str):
    order = repo.get_by_id(order_id)
    if not order:
        raise HTTPException(status_code=404, detail="Not found")
    return {
        "order_id": order.id,
        "items": [i.__dict__ for i in order.items],
        "total": order.total(),
        "status": order.status,
    }

@app.delete("/orders/{order_id}")
def cancel_order(order_id: str):
    if not service.cancel_order(order_id):
        raise HTTPException(status_code=404, detail="Not found")
    return {"ok": True}
```

---

## Slides Outline (ready to drop into your deck)

1. **Why Think at Scale (Engage, 6:30–6:45)**

* Pain points: pipeline slowdown, brittle tests, onboarding drag.
* Visual: *Flat monolith vs. layered*.
* Prompt: “What breaks first when code gets messy?”

2. **DDD Concepts (6:45–7:15)**

* Entity, Value Object, Aggregate/Aggregate Root, Repository, Domain Service.
* Mini‑examples (e‑commerce: Money, Address, Order/OrderItem).
* Diagram: Order → OrderItem, Repository boundary.

3. **Breakout #1 briefing (7:15–7:35)**

* Folder refactor kata; goals; output expectations.

4. **Live‑Coding (7:40–8:05)**

* Create domain model, ports, service, in‑memory repo; show tests passing.

5. **Breakout #2 briefing (8:05–8:25)**

* Add `cancel_order` + test; decide where logic lives.

6. **Hexagonal / Ports & Adapters (8:30–8:50)**

* Core knows only ports; adapters implement (HTTP/DB/etc.).
* Add FastAPI adapter; prove swapability.

7. **Architecture ↔ CI/CD (8:50–9:05)**

* Testing pyramid; run domain tests first; adapters later.

8. **Assignment & Wrap‑Up (9:05–9:15)**

* Refactor previous work; rubric; survey reminder.

---

## Breakout Session Handouts (print/share these)

### Handout 1 — Folder Refactor Kata (20 min)

**Goal**: Restructure one module into three layers.

**Steps**

1. Create directories: `src/domain`, `src/application`, `src/infrastructure`.
2. Move one existing business rule/model into **domain**.
3. Define a **Repository port** with `save`, `get_by_id` in **application**.
4. Keep framework/db code in **infrastructure**.

**Discussion Prompts**

* If you swapped FastAPI→Flask tomorrow, what files change?
* Why is this code in domain vs infrastructure?

**Acceptance Criteria**

* Build still runs; at least one unit test covers domain logic.

---

### Handout 2 — Add Use Case + Domain Test (20 min)

**Goal**: Add `cancel_order(order_id)` and test it without HTTP.

**Steps**

1. Add method in `OrderService` (application) calling domain behavior.
2. Update repo implementation as needed.
3. Write tests in `tests/test_order_service.py` (no FastAPI).

**Acceptance Criteria**

* Test for happy path (cancels) and missing order (False/404 via adapter).

**Stretch**

* Add `complete()` transition with guard; test invalid transitions.

---

## CI/CD Note (optional YAML)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt
      - run: pytest -q  # domain/application tests first by default
```

---

## After‑Class Assignment — Refactor for Clarity (≈45 min)

**Do**: Pick a prior assignment (W4D4/W5D1). Reorganize into `src/domain`, `src/application`, `src/infrastructure`. Move core logic to domain; keep frameworks in infrastructure; coordinate via application services/ports.

**Deliverables**

1. Updated GitHub repo (new structure) with **passing tests**.
2. CI workflow (green checks) — domain tests run fast.
3. README section (250–500 words) reflecting on decisions & CI benefits.

**Rubric (10 pts +2 bonus)**

* Folder separation & clarity **(3)**
* CI/CD ordering (domain tests first) **(2)**
* Working code/tests **(2)**
* Reflection quality (250–500 words) **(3)**
* **Bonus (+2)**: interface‑based repository + two implementations; simple architecture diagram.

---

## Quizzes (drop into LMS)

### Pre‑Class Quiz (5 Q)

* DDD core concept? (Aggregates)
* Goal of Hexagonal? (Decouple business logic from external systems)
* What is an Entity? (Domain object with identity)
* Why layering helps CI/CD? (Faster, isolated tests)
* Which layer must not know frameworks? (Domain)

### After‑Class Quiz (5 Q)

* Which layer should *not* depend on frameworks? (Domain)
* Purpose of a Repository? (Abstraction to retrieve/save aggregates)
* In Hexagonal, what’s an Adapter? (Implementation connecting external system to a port)
* How does layering speed CI? (Run fast unit tests first; isolate adapters)
* Main benefit of separating layers? (Maintainability/testability; reduced coupling)

---

## Teaching Script Cues (use as speaker notes)

* **Open**: “You built CI yesterday. Today we keep it fast as the codebase grows.”
* **Explain DDD** with one e‑commerce example and a whiteboard sketch.
* **Callouts** while coding: “Notice the domain has zero FastAPI imports.”
* **Debriefs** after each breakout: prompt trade‑offs and where code belongs.
* **Close**: Tie clean seams to safer deploys and confident refactors.

---

## Instructor Checklist

* [ ] Test install on fresh machine (Mac/Win/Linux)
* [ ] Verify code blocks compile in clean venv
* [ ] Prep breakout room assignments (3–4 per room)
* [ ] Keep a parking‑lot doc for deep dives
* [ ] Timer ready for 20‑min katas

---

### Credits

Aligned to AISE house style for Python Foundations & Architecture. (Replace org/repo names, add logos as needed.)
