FROM python:3.13-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

ENV YOUR_NAME="Syrac"

EXPOSE 5500

ENTRYPOINT ["python", "app.py"]