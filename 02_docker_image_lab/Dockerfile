FROM python:3.12-slim AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim

ARG LAB_LOGIN=unknown
ARG LAB_TOKEN=unknown

LABEL org.lab.login="$LAB_LOGIN" \
      org.lab.token="$LAB_TOKEN" \
      org.opencontainers.image.title="image-lab"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    ROCKET_SIZE=Small \
    PORT=8000 \
    LAB_LOGIN=$LAB_LOGIN \
    LAB_TOKEN=$LAB_TOKEN

WORKDIR /opt/app

COPY --from=builder /install /usr/local
COPY app/ ./app/

RUN useradd --create-home --shell /bin/bash appuser \
    && chown -R appuser:appuser /opt/app
USER appuser

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health').status==200 else 1)"

EXPOSE 8000

VOLUME ["/tmp"]
ENTRYPOINT ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "app.app:app"]