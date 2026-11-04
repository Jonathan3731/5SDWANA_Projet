FROM python:3.12-alpine
WORKDIR /app/server/python
COPY server/python/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt gunicorn==26.2.0
COPY server/python/ .
COPY client/html /app/client/html
ENV STATIC_DIR=../../client/html
ENV HOME=/tmp
USER nobody
EXPOSE 4242
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:4242", "server:app"]
