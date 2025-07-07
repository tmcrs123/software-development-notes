Docker

Everything needs to be in the SAME network
Docker exposes sql via a port so if using via VS connection string is localhost, if using for container to container coms then connection string is sqlserver

docker-compose --env-file .env-docker up -d --build


crete azure auth thing (like cognmito)

- entra tenant and account
- external tenant
- change users permissions to admin
- add stupid user flow to allow users to create new accounts without being in MS world