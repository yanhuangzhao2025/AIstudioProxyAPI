# 使用 Python 3.10 作为基础镜像
FROM python:3.10-slim-bookworm AS builder

# 设置环境变量
ARG DEBIAN_FRONTEND=noninteractive
ARG PROXY_ADDR
ENV HTTP_PROXY=${PROXY_ADDR}
ENV HTTPS_PROXY=${PROXY_ADDR}
ENV POETRY_HOME="/opt/poetry"
ENV POETRY_VERSION=1.8.3
ENV PATH="${POETRY_HOME}/bin:${PATH}"

# 设置代理并安装 curl
RUN if getent hosts host.docker.internal >/dev/null 2>&1; then \
    echo 'Acquire::http::Proxy "http://host.docker.internal:7890";' > /etc/apt/apt.conf.d/99proxy; \
    elif [ -n "$PROXY_ADDR" ]; then \
    printf 'Acquire::http::Proxy "%s";\nAcquire::https::Proxy "%s";\n' "$PROXY_ADDR" "$PROXY_ADDR" > /etc/apt/apt.conf.d/99proxy; \
    fi && \
    apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    rm -f /etc/apt/apt.conf.d/99proxy

# 安装 Poetry
RUN curl -sSL https://install.python-poetry.org | python3 - --version ${POETRY_VERSION}

# 设置工作目录
WORKDIR /app_builder
COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false --local && \
    poetry install --no-root --no-dev --no-interaction --no-ansi

# 创建最终的 Python 容器
FROM python:3.10-slim-bookworm

# 继承环境变量
ARG DEBIAN_FRONTEND=noninteractive
ARG PROXY_ADDR
ENV HTTP_PROXY=${PROXY_ADDR}
ENV HTTPS_PROXY=${PROXY_ADDR}

# 安装依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libdbus-1-3 libdrm2 libgbm1 libgtk-3-0 libnspr4 libnss3 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxdamage1 libxext6 libxfixes3 libxrandr2 libxrender1 libxtst6 ca-certificates fonts-liberation libasound2 libpangocairo-1.0-0 libpango-1.0-0 libu2f-udev \
    supervisor curl && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# 创建用户和工作目录
RUN groupadd -r appgroup && useradd -r -g appgroup -s /bin/bash -d /app appuser
WORKDIR /app

# 复制 Python 包
COPY --from=builder /usr/local/lib/python3.10/site-packages/ /usr/local/lib/python3.10/site-packages/
COPY --from=builder /usr/local/bin/ /usr/local/bin/
COPY --from=builder /opt/poetry/bin/poetry /usr/local/bin/poetry

# 复制应用代码
COPY . .

# 安装 Playwright
RUN camoufox fetch && python -m playwright install firefox

# 目录权限设置
RUN mkdir -p /app/logs /app/auth_profiles/active /app/auth_profiles/saved /app/certs /app/browser_utils/custom_scripts \
    /home/appuser/.cache/ms-playwright /home/appuser/.mozilla && \
    chown -R appuser:appgroup /app /home/appuser

# 复制 supervisord 配置
COPY supervisord.conf /etc/supervisor/conf.d/app.conf

# camoufox 缓存逻辑
RUN mkdir -p /var/cache/camoufox && \
    if [ -d /root/.cache/camoufox ]; then cp -a /root/.cache/camoufox/* /var/cache/camoufox/; fi && \
    mkdir -p /app/.cache && ln -s /var/cache/camoufox /app/.cache/camoufox

# 更新浏览器数据
RUN python update_browserforge_data.py

# 清理代理环境变量
ENV HTTP_PROXY=""
ENV HTTPS_PROXY=""

# 设置端口
EXPOSE 2048 3120

# 设置用户和环境变量
USER appuser
ENV HOME=/app
ENV PLAYWRIGHT_BROWSERS_PATH=/home/appuser/.cache/ms-playwright
ENV PYTHONUNBUFFERED=1
ENV PORT=8000
ENV DEFAULT_FASTAPI_PORT=2048
ENV DEFAULT_CAMOUFOX_PORT=9222
ENV STREAM_PORT=3120
ENV SERVER_LOG_LEVEL=INFO
ENV DEBUG_LOGS_ENABLED=false
ENV AUTO_CONFIRM_LOGIN=true
ENV SERVER_PORT=2048
ENV INTERNAL_CAMOUFOX_PROXY=""

# 启动 supervisord
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/app.conf"]