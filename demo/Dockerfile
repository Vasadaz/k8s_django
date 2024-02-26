FROM python:3.11.3-alpine

WORKDIR /app/

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN pip install --upgrade pip && \
    pip install --no-cache-dir Django==3.2.2 && \
    django-admin startproject demo .

EXPOSE 8000

CMD python manage.py runserver 0.0.0.0:8000

