# Your forecast dashboard as a container, for Google Cloud Run (built and deployed by deploy-dashboard.yml).
# Cloud Run sends web traffic to the port in $PORT (8080). The app only READS your bucket.
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    PORT=8080

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD streamlit run app.py --server.port=$PORT --server.address=0.0.0.0 --server.headless=true \
    --browser.gatherUsageStats=false
