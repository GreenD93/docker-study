IMAGE_NAME = mynginx03
SERVICE_NAME=nginxtest
NETWORK_NAME=mynetwork03

build:
	docker image build -t $(IMAGE_NAME) .

rebuild:
	docker build --no-cache -t $(IMAGE_NAME) .

run:
	docker run -d \
	--name $(SERVICE_NAME) \
	--network $(NETWORK_NAME) \
	-p 80:80 -d $(IMAGE_NAME)

stop:
	docker container stop $(IMAGE_NAME)
	
check-container:
	docker container ls