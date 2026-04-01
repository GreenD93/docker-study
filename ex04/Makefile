IMAGE_NAME = mynginx01

build:
	docker image build -t $(IMAGE_NAME) .

rebuild:
	docker build --no-cache -t $(IMAGE_NAME) .

run:
	docker run -p 80:80 -d $(IMAGE_NAME)

stop:
	docker container stop $(IMAGE_NAME)
	
check-container:
	docker container ls