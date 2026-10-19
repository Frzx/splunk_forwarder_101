FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

RUN mkdir -p /app/logs && chown appuser:appgroup /app/logs

RUN pip install --no-cache-dir doris-mcp-server==0.6.1

USER appuser

EXPOSE 3000

CMD ["doris-mcp-server", "--transport", "http", "--host", "0.0.0.0", "--port", "3000"]
