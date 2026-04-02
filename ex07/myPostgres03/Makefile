IMAGE_NAME=postgres:15.4
NETWORK_NAME=mynetwork03
SERVICE_NAME=postgrestest
VOLUME_NAME=myvolume03


build:
	docker image build -t $(IMAGE_NAME) .

run:
	docker run \
	--network $(NETWORK_NAME) \
	-e POSTGRES_PASSWORD=mysecretpassword \
	--mount type=volume,source=$(VOLUME_NAME),target=/var/lib/postgresql/data \
	--name $(SERVICE_NAME) \
	-d $(IMAGE_NAME)

stop:
	docker container stop $(IMAGE_NAME)
	
check-container:
	docker container ls