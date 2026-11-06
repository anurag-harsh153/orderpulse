So yes, your containers run only while Docker Desktop’s engine is running. Closing the window usually leaves it running in the tray, while quitting from the tray icon stops it and your database with it. After a restart, run docker compose -f docker/docker-compose.yml up -d again, or docker start orderpulse-postgres.

docker compose -f docker/docker-compose.yml up -d

docker ps
docker exec -it orderpulse-postgres psql -U orderpulse -d orderpulse -c "select version();"