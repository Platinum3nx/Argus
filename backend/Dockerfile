FROM python:3.11-slim

# 1. Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    libgmp-dev \
    unzip \
    wget \
    && rm -rf /var/lib/apt/lists/*

# 2. Set ELAN_HOME to a global directory (Crucial for GitHub Actions)
ENV ELAN_HOME="/usr/local/elan"
ENV PATH="${ELAN_HOME}/bin:${PATH}"

# 3. Install elan and the specific Lean 4 version (must match lean-toolchain)
RUN curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh -s -- -y --default-toolchain leanprover/lean4:v4.14.0

# 4. Install .NET 6 Runtime using official install script (auto-detects OS)
ENV DOTNET_ROOT="/opt/dotnet"
ENV PATH="${DOTNET_ROOT}:${PATH}"
RUN curl -sSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh && \
    chmod +x /tmp/dotnet-install.sh && \
    /tmp/dotnet-install.sh --channel 6.0 --runtime dotnet --install-dir /opt/dotnet && \
    rm /tmp/dotnet-install.sh && \
    dotnet --info

# 5. Install Dafny 4.9.0
ARG DAFNY_BUILD_VERSION=v5-dotnet-script
RUN echo "Dafny build: ${DAFNY_BUILD_VERSION}"

ARG DAFNY_VERSION=4.9.0
RUN curl -L "https://github.com/dafny-lang/dafny/releases/download/v${DAFNY_VERSION}/dafny-${DAFNY_VERSION}-x64-ubuntu-20.04.zip" -o /tmp/dafny.zip && \
    unzip -q /tmp/dafny.zip -d /opt && \
    rm /tmp/dafny.zip && \
    ls -la /opt && \
    mv /opt/dafny* /opt/dafny 2>/dev/null || true && \
    ls -la /opt/dafny && \
    chmod +x /opt/dafny/dafny || chmod +x /opt/dafny/Dafny || true

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
ARG BUILD_VERSION=2026-02-06-v22-dotnet-install-script
RUN echo "Build version: ${BUILD_VERSION}"

# 15. Copy backend code
COPY . ./backend

# 16. Setup environment
EXPOSE 8000
ENV PYTHONPATH=/app

# 17. Entrypoint
RUN chmod +x /app/backend/entrypoint.sh

CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]