FROM python:3.6.2
LABEL maintainer twtrubiks
ENV PYTHONUNBUFFERED 1
RUN mkdir /docker_api2
WORKDIR /docker_api2
COPY . /docker_api2
RUN pip install -r requirements.txt
# RUN pip install  -i  https://pypi.python.org/simple/  -r requirements.txt