FROM python:3.11-slim

# 1. Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    libgmp-dev \
    unzip \
    wget \
    apt-transport-https \
    && rm -rf /var/lib/apt/lists/*

# 2. Set ELAN_HOME to a global directory (Crucial for GitHub Actions)
ENV ELAN_HOME="/usr/local/elan"
ENV PATH="${ELAN_HOME}/bin:${PATH}"

# 3. Install elan and the specific Lean 4 version (must match lean-toolchain)
RUN curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh -s -- -y --default-toolchain leanprover/lean4:v4.14.0

# 4. Install .NET 6 Runtime (required by Dafny 4.x)
RUN wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && \
    dpkg -i packages-microsoft-prod.deb && \
    rm packages-microsoft-prod.deb && \
    apt-get update && \
    apt-get install -y dotnet-runtime-6.0 && \
    rm -rf /var/lib/apt/lists/*

# 5. Install Dafny 4.9.0
ARG DAFNY_BUILD_VERSION=v4-with-dotnet6
RUN echo "Dafny build: ${DAFNY_BUILD_VERSION}"

ARG DAFNY_VERSION=4.9.0
RUN curl -L "https://github.com/dafny-lang/dafny/releases/download/v${DAFNY_VERSION}/dafny-${DAFNY_VERSION}-x64-ubuntu-20.04.zip" -o /tmp/dafny.zip && \
    unzip -q /tmp/dafny.zip -d /opt && \
    rm /tmp/dafny.zip && \
    ls -la /opt && \
    # The zip extracts to a folder like 'dafny' or 'dafny-4.9.0-...'
    mv /opt/dafny* /opt/dafny 2>/dev/null || true && \
    ls -la /opt/dafny && \
    chmod +x /opt/dafny/dafny

ENV PATH="/opt/dafny:${PATH}"

# 6. Test Dafny installation
RUN echo 'Testing Dafny...' && \
    dafny --version && \
    echo '✅ Dafny is working!'

# 7. Set working directory
WORKDIR /app

# 8. CACHE-BUST for Lean project
ARG LEAN_PROJECT_VERSION=2026-02-05-v13-minimal-imports
RUN echo "Lean project version: ${LEAN_PROJECT_VERSION}"

# 9. Copy the lean_project
COPY lean_project ./lean_project

# 10. Download Mathlib cache
WORKDIR /app/lean_project
RUN lake update && lake exe cache get

# 11. Test that Lean works
RUN echo 'Testing Lean installation...' && lean --version && lake --version

# 12. Return to app directory
WORKDIR /app

# 13. Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 14. Cache-bust for backend code
ARG BUILD_VERSION=2026-02-06-v21-dotnet-dafny
RUN echo "Build version: ${BUILD_VERSION}"

# 15. Copy backend code
COPY . ./backend

# 16. Setup environment
EXPOSE 8000
ENV PYTHONPATH=/app

# 17. Entrypoint
RUN chmod +x /app/backend/entrypoint.sh

CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]