FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1

# Create non-root user (recommended)
RUN useradd -m appuser

WORKDIR /app

# Install rq-dashboard directly from PyPI
RUN pip install --no-cache-dir rq-dashboard gunicorn

USER appuser

EXPOSE 9181

# Run with gunicorn (production-safe)
CMD ["gunicorn", "-b", "0.0.0.0:9181", "rq_dashboard:app"]
