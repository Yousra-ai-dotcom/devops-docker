FROM python:3.12-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends openssh-client && \
    rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir ansible-core docker

RUN ansible-galaxy collection install community.docker

WORKDIR /workspace

CMD ["ansible-playbook", "--version"]