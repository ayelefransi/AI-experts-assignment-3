# AI Software Engineer Assignment

This repository contains a simple Python application with a bug. The goal is to fix the bug and ensure the tests pass.

## Running Tests

### Local

1. Install dependencies:
    ```bash
    pip install -r requirements.txt
    ```

2. Run tests:
    ```bash
    pytest -v
    ```
    
    > **Note:** If you encounter an `ImportError` related to `pytest-asyncio` (due to global environment conflicts), run:
    > ```bash
    > pytest -v -p no:asyncio
    > ```

### Docker

1. Build the Docker image:
    ```bash
    docker build -t assignment-app .
    ```

2. Run tests in Docker:
    ```bash
    docker run assignment-app
    ```
