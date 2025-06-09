---
title: Building Docker Containers in Jenkins with a Docker-in-Docker Agent
description: How I set up my dream environment for executing docker based jenkins pipelines.
date: '06/09/2025'
icon: https://www.gravatar.com/avatar/c61b120bdaedf42832ad4c9e391b3929?s=120&r=g&d=404
author: Kalitsune
tags:
  - jenkins
  - docker
published: true
---
## Introduction 

Running Docker builds within Jenkins is a common need, but it can be tricky to set up securely and efficiently. To solve this, I've created a custom Jenkins agent with Docker-in-Docker (DinD) that simplifies the entire process. This container provides a secure, isolated environment for your containerized builds, tests, and deployments.

### What is Docker-in-Docker (DinD)?

Docker-in-Docker, or DinD, is a method for running a Docker daemon inside a Docker container. This creates an environment that is completely isolated from the host's Docker daemon. For a Jenkins setup, this means that your builds have their own dedicated Docker environment, preventing any conflicts or security concerns with the underlying host.

### A Secure and Efficient Jenkins Agent

This Jenkins agent is designed with security and reliability in mind. Here are some of its key features:

* **Rootless Jenkins Agent:** The agent itself runs as a non-root `jenkins` user, which is a critical security best practice. This minimizes the potential impact of any vulnerabilities.
* **Docker Access for Jenkins:** The `jenkins` user is added to the `docker` group within the container, granting it the necessary permissions to interact with the Docker daemon without needing root privileges.
* **Graceful Startup:** An entrypoint script ensures that the Docker daemon is fully up and running before the Jenkins agent starts. This prevents race conditions where Jenkins might try to use Docker before it's ready.

### How to Use It

To get started, launch a jenkins agent running `kalithekitsune/jenkins-agent-docker:latest` (or your own image).

⚠️ When launching the container, please use the `--privileged` flag. This is a requirement for Docker-in-Docker, as it needs elevated permissions to manage container runtimes.

In my case, I have a docker cloud set up so I had to check this in the container settings of my docker agent template:
![checkbox in the Jenkins ui](/blog/JenkinsDockerAgent/checkbox.png)

Once your agent is successfully running you can try running docker based pipelines such as this one:

Example of a `Jenkinsfile` that uses this agent to run a simple Node.js script inside a container

```groovy
pipeline {
    agent {
        docker {
            image 'node:16-alpine'
        }
    }
    stages {
        stage('Test') {
            steps {
                sh 'node --eval "console.log(process.platform, process.env.CI)"'
            }
        }
    }
}
```

When you run this pipeline, Jenkins will pull the `node:16-alpine` image and execute the script inside it. The output will look like this:

```
+ node --eval 'console.log(process.platform, process.env.CI)'
linux true
```

Confirming that the container is indeed working!

### Under the Hood: The Dockerfile and Entrypoint

For those interested in how the image is built, here’s a look at the `Dockerfile`:

```dockerfile
# Use the official Jenkins agent image as the base
FROM jenkins/inbound-agent:latest

# Switch to the root user to install packages
USER root

# Install Docker
RUN curl -fsSL "https://get.docker.com/" | sh

# Add the 'jenkins' user to the 'docker' group to allow Docker access
RUN usermod -aG docker jenkins

# Create a volume for Docker's data directory
RUN mkdir -p /var/lib/docker
VOLUME /var/lib/docker

# Copy the entrypoint script into the container
COPY entrypoint.sh entrypoint.sh

# Set the entrypoint to our custom script
ENTRYPOINT ["sh", "entrypoint.sh"]
```

The magic happens in the `entrypoint.sh` script, which orchestrates the startup sequence.

```sh
#!/bin/sh

# Start the Docker daemon in the background
nohup /usr/bin/dockerd > /var/log/dockerd.log 2>&1 &

# Wait for the Docker daemon to become available
echo "Waiting for Docker daemon to start..."
attempts=0
max_attempts=30
while ! docker info > /dev/null 2>&1; do
  if [ ${attempts} -eq ${max_attempts} ]; then
    echo "Docker daemon failed to start after ${max_attempts} seconds." >&2
    cat /var/log/dockerd.log >&2
    exit 1
  fi
  attempts=$((attempts + 1))
  sleep 1
done
echo "Docker daemon started successfully."

# Switch to the 'jenkins' user and execute the standard jenkins-agent command
exec su jenkins -s /bin/sh -c '/usr/local/bin/jenkins-agent "$@"' -- jenkins-agent "$@"
```

The script first starts the Docker daemon, waits for it to be ready, and then switches to the `jenkins` user to launch the Jenkins agent.

### Conclusion

I hope this article could be of use to you,
this container is very useful to me so I hope you'll like it!
