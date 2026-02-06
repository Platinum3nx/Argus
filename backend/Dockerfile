FROM python:3.11-slim

# 1. Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    libgmp-dev \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# 2. Set ELAN_HOME to a global directory (Crucial for GitHub Actions)
ENV ELAN_HOME="/usr/local/elan"
ENV PATH="${ELAN_HOME}/bin:${PATH}"

# 3. Install elan and the specific Lean 4 version (must match lean-toolchain)
RUN curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh -s -- -y --default-toolchain leanprover/lean4:v4.14.0

# 4. Install Dafny 4.x
# Download from GitHub releases and extract to /opt/dafny
ARG DAFNY_VERSION=4.9.0
RUN mkdir -p /opt/dafny && \
    curl -L "https://github.com/dafny-lang/dafny/releases/download/v${DAFNY_VERSION}/dafny-${DAFNY_VERSION}-x64-ubuntu-20.04.zip" -o /tmp/dafny.zip && \
    unzip -q /tmp/dafny.zip -d /opt && \
    mv /opt/dafny-${DAFNY_VERSION}-x64-ubuntu-20.04/* /opt/dafny/ 2>/dev/null || mv /opt/dafny-*/* /opt/dafny/ 2>/dev/null || true && \
    chmod +x /opt/dafny/dafny && \
    rm -rf /tmp/dafny.zip /opt/dafny-*

# Add Dafny to PATH
ENV PATH="/opt/dafny:${PATH}"

# 5. Test Dafny installation
RUN echo 'Testing Dafny installation...' && dafny --version || echo 'Dafny installation check (may fail in some environments)'

# 6. Set working directory
WORKDIR /app

# 7. CACHE-BUST for Lean project - change this to force Mathlib rebuild
ARG LEAN_PROJECT_VERSION=2026-02-05-v13-minimal-imports
RUN echo "Lean project version: ${LEAN_PROJECT_VERSION}"

# 8. Copy the lean_project Lean project (with Mathlib dependency)
COPY lean_project ./lean_project

# 9. Download Mathlib cache (required for split_ifs tactic)
WORKDIR /app/lean_project
RUN lake update && lake exe cache get

# 10. Test that Lean works
RUN echo 'Testing Lean installation...' && lean --version && lake --version

# 11. Return to app directory
WORKDIR /app

# 12. Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 13. Cache-bust for backend code
ARG BUILD_VERSION=2026-02-06-v18-dafny-integration
RUN echo "Build version: ${BUILD_VERSION}"

# 14. Copy backend code
COPY . ./backend

# 15. Setup environment
EXPOSE 8000
ENV PYTHONPATH=/app

# 16. Entrypoint
RUN chmod +x /app/backend/entrypoint.sh

CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]