FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /srv
COPY backend/pyproject.toml ./
COPY backend/requirements.lock ./
COPY backend/app ./app
RUN pip install --no-cache-dir --upgrade pip==26.2.2 && \
    pip install --no-cache-dir --require-hashes -r requirements.lock && \
    pip install --no-cache-dir --no-deps . && \
    groupadd --gid 10001 remedi && \
    useradd --uid 10001 --gid remedi --no-create-home remedi && \
    mkdir /data && chown remedi:remedi /data
USER 10001:10001
EXPOSE 8000
CMD ["uvicorn", "app.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000", "--workers", "1", "--no-access-log"]
