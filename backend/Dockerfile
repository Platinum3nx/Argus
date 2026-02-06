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

# 4. Install Dafny 4.9.0
# Cache buster to force rebuild of Dafny installation
ARG DAFNY_BUILD_VERSION=v2-robust-find
RUN echo "Dafny build version: ${DAFNY_BUILD_VERSION}"

# Robust install: Download, find binary, and symlink it
ARG DAFNY_VERSION=4.9.0
RUN mkdir -p /opt/dafny && \
    echo "Downloading Dafny ${DAFNY_VERSION}..." && \
    curl -L "https://github.com/dafny-lang/dafny/releases/download/v${DAFNY_VERSION}/dafny-${DAFNY_VERSION}-x64-ubuntu-20.04.zip" -o /tmp/dafny.zip && \
    echo "Extracting..." && \
    unzip -q /tmp/dafny.zip -d /opt/dafny_extract && \
    echo "Contents of /opt/dafny_extract:" && \
    ls -la /opt/dafny_extract && \
    echo "Finding dafny binary..." && \
    find /opt/dafny_extract -name "dafny" -type f | head -5 && \
    # Copy the dafny binary (it's usually a script, not necessarily executable yet)
    find /opt/dafny_extract -name "dafny" -type f -exec cp {} /usr/local/bin/dafny \; -quit && \
    chmod +x /usr/local/bin/dafny && \
    # Also find and copy z3
    find /opt/dafny_extract -name "z3" -type f -exec cp {} /usr/local/bin/z3 \; -quit || true && \
    chmod +x /usr/local/bin/z3 2>/dev/null || true && \
    # Clean up
    rm -rf /tmp/dafny.zip /opt/dafny_extract && \
    echo "Dafny installed to /usr/local/bin/dafny"

# Add to PATH and verify
ENV PATH="/usr/local/bin:${PATH}"

# 5. Test Dafny installation (MUST succeed)
RUN echo 'Testing Dafny installation...' && \
    which dafny && \
    dafny --version && \
    echo '✅ Dafny is working!'

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
ARG BUILD_VERSION=2026-02-06-v20-dafny-robust-install
RUN echo "Build version: ${BUILD_VERSION}"

# 14. Copy backend code
COPY . ./backend

# 15. Setup environment
EXPOSE 8000
ENV PYTHONPATH=/app

# 16. Entrypoint
RUN chmod +x /app/backend/entrypoint.sh

CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]