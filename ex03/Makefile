IMAGE_NAME = myweb01

build:
	docker image build -t $(IMAGE_NAME) .

rebuild:
	docker build --no-cache -t $(IMAGE_NAME) .

# -d : 백그라운드 실행, -p : 포트 포워딩
run:
	docker run -d -p 8000:8000 $(IMAGE_NAME)

stop:
	docker container stop $(IMAGE_NAME)
	
check-container:
	docker container ls