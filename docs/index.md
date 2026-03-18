---
hide:
  - toc
---
# TDD - Test Driven Development

## Tools for TDD

- **pytest**: Our primary testing framework.
- **pytest-django**: Django integration for pytest.
- **pytest-mocker**: Mocking functions in tests.
- **pytest-cov**: Generating coverage reports.
- **factory-boy & faker**: Generating realistic test data.

## Example Test Case

We use `factory-boy` to generate consistent test data and `pytest-django` for seamless integration.

```python
import pytest
from apps.games.models import Game
from tests.factories import GameFactory

@pytest.mark.django_db
def test_game_creation():
    """Verify that a game can be created successfully within a tenant context."""
    game = GameFactory(title="Cyberpunk 2077")
    assert game.title == "Cyberpunk 2077"
    assert Game.objects.count() == 1
```

## Executing Test Cases

To run the full test suite efficiently:

```bash
# Run all tests
uv run pytest

# Run a specific module
uv run pytest tests/games/

# Run tests matching a keyword
uv run pytest -k "payment"
```

## Coverage & Quality

We use `pytest-cov` to track our progress towards full test coverage.

### Terminal Summary
```bash
uv run pytest --cov=.
```

### Detailed HTML Report
```bash
uv run pytest --cov=. --cov-report=html
```

## Report 

<iframe src="assets/htmlcov/index.html" width="100%" height="5100px"></iframe>
