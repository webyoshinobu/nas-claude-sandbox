FROM eclipse-temurin:21-jdk

RUN apt-get update && \
    apt-get install -y maven && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /workspace

CMD ["bash"]
