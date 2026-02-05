FROM python:3.11-slim

# 1. Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    libgmp-dev \
    && rm -rf /var/lib/apt/lists/*

# 2. Set ELAN_HOME to a global directory (Crucial for GitHub Actions)
ENV ELAN_HOME="/usr/local/elan"
ENV PATH="${ELAN_HOME}/bin:${PATH}"

# 3. Install elan and the specific Lean 4 version (must match lean-toolchain)
RUN curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh -s -- -y --default-toolchain leanprover/lean4:v4.14.0

# 4. Set working directory
WORKDIR /app

# 5. CACHE-BUST for Lean project - change this to force Mathlib rebuild
ARG LEAN_PROJECT_VERSION=2026-02-05-v13-minimal-imports
RUN echo "Lean project version: ${LEAN_PROJECT_VERSION}"

# 6. Copy the lean_project Lean project (with Mathlib dependency)
COPY lean_project ./lean_project

# 7. Download Mathlib cache (required for split_ifs tactic)
WORKDIR /app/lean_project
RUN lake update && lake exe cache get

# 8. Test that Lean works
RUN echo 'Testing Lean installation...' && lean --version && lake --version

# 9. Return to app directory
WORKDIR /app

# 10. Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 11. Cache-bust for backend code
ARG BUILD_VERSION=2026-02-05-v14-omega-fix
RUN echo "Build version: ${BUILD_VERSION}"

# 12. Copy backend code
COPY . ./backend

# 13. Setup environment
EXPOSE 8000
ENV PYTHONPATH=/app

# 14. Entrypoint
RUN chmod +x /app/backend/entrypoint.sh

CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]