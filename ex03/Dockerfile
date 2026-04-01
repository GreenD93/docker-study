FROM python:3.11.6

WORKDIR /usr/src/app

COPY . .

RUN python -m pip install --upgrade pip # 이미지 빌드시 실행되는 명령어
RUN pip install -r requirements.txt

WORKDIR ./myapp 

CMD python manage.py runserver 0.0.0.0:8000 # 컨테이너 실행시 실행되는 명령어
EXPOSE 8000