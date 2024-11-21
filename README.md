# Directus Backend

Here I've put all the necessary resources to deploy the Directus backend on AWS using Docker and Docker-Compose - see docker-compose.yml.

-----------
## Note: There's the Dockerfile approach and the Docker-Compose approach.

Key Differences between Dockerfile and Docker-Compose

The Dockerfile is used to define the environment and build an image for an individual service, while docker-compose.yml ties all these services together into a cohesive application. So the
micro-service approach that's highly recommended to build highly scalable apps is better implemented via the Dockerfile approach because it allows one app for one container. Best Practices
Use Dockerfile to define how an image is built for portability and reuse. Use docker-compose.yml for development, testing, and local orchestration.

For production, consider tools like Kubernetes or Docker Swarm for advanced orchestration needs.

For now, we are going to use the docker-compose.yml approach.
------------

# Before to start

Rename .env.example to .env

# Using docker-compose.yml for Deployment

Advantages:
- Simplicity: docker-compose.yml defines all services in a single file.
- Reproducibility: We can easily replicate the setup on AWS or other environments.
- Portability: Named volumes allow for persistent storage without worrying about underlying directories.

To ensure data integrity when we migrate from local setup to AWS, we have made backups of the Directus backend data - the database for the collections and everything and also for the files.

## Database backup - locally

We connect to the db container to make a dump of the current database data as follows:
sh-5.1# mysqldump -u root -proot_password directus_db > directus_db_backup.sql

Now we have copied that dump from our db container to our local directory in the physical machine as follows:

1- We run the following to check that the db container is running and under what name
docker ps

2- Suppose we are already in the directory where we want to copy the db dump hen we run the following command
docker cp directus-local-db-1:/directus_db_backup.sql .

## Files Backup

1- We run the following command to identify the volume used to store files in Directus
docker volume ls

2- Then we run the following command to backup our files
docker run --rm -v directus-local_directus_uploads:/data -v $(pwd):/backup alpine tar czf /backup/directus_uploads_backup.tar.gz -C /data .

directus_db_backup.sql and directus_uploads_backup.tar.gz are the files we are going to restore on AWS after a fresh deployment of a Directus instance using our docker-compose.yml file.

## Deployment on AWS - just an example of what's possible

1- Prepare an AWS EC2 Instance
Choose an appropriate instance type (e.g., t2.micro for testing or t2.medium for small-scale production).

Install Docker and Docker Compose on the instance.
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker

2- Transfer docker-compose.yml to AWS: Use scp or another method to upload the file 
scp docker-compose.yml ubuntu@<AWS_PUBLIC_IP>:/home/ubuntu/

3- Deploy the Services/containers -> Run docker-compose to start the services
docker-compose up -d

4- Restore Backups
Database: Transfer the directus_db_backup.sql to the AWS instance and restore it using the earlier command.
Uploads: Transfer the directus_uploads_backup.tar.gz file to AWS and restore it as shown

5- Test the Deployment
Access Directus via http://<AWS_PUBLIC_IP>:8055 to ensure everything is working.

6- Improvements to Consider
- Environment File: Use an .env file to manage environment variables, making your docker-compose.yml cleaner and more secure.

Example .env:
DIRECTUS_DATABASE_HOST=db
DIRECTUS_DATABASE_PORT=3306
DIRECTUS_DATABASE_NAME=directus_db
DIRECTUS_DATABASE_USER=directus
DIRECTUS_DATABASE_PASSWORD=
ADMIN_EMAIL=
ADMIN_PASSWORD=
DIRECTUS_APP_URL=http://localhost:8055
PUBLIC_URL=http://localhost:8055
CORS_ENABLED=true
CORS_ORIGIN=http://localhost:3000
SECRET=


NB: Use this command to generate a new secret for prod environment
openssl rand -base64 32


Then the updated docker-compose.yml
environment:
  - DIRECTUS_DATABASE_HOST=${DIRECTUS_DATABASE_HOST}
  - DIRECTUS_DATABASE_PORT=${DIRECTUS_DATABASE_PORT}
  ...


- Persistent Storage on AWS
Use an Elastic Block Store (EBS) volume for your directus_uploads and db_data instead of local Docker volumes. This ensures data durability even if the instance stops.

- Backup Automation
Schedule database and file backups using cron jobs or AWS services like AWS Backup.
