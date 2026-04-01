IMAGE_NAME=mypostgres03
VOLUME_NAME=myvolume03


build:1
	docker image build -t $(IMAGE_NAME) .

run:
	docker run -e POSTGRES_PASSWORD=mysecretpassword \
	--mount type=volume,source=$(VOLUME_NAME),target=/var/lib/postgresql/data \
	--name $(IMAGE_NAME) \
	-d $(IMAGE_NAME)

stop:
	docker container stop $(IMAGE_NAME)
	
check-container:
	docker container ls