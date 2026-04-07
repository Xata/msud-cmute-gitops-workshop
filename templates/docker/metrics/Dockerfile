FROM eclipse-temurin:21-jre

RUN apt-get update && apt-get install -y curl

RUN useradd -m -d /server minecraft
WORKDIR /server
USER minecraft

ARG PAPER_VERSION=1.21.11
ARG PAPER_BUILD=69

RUN curl -o paper.jar -H "User-Agent: msud-workshop-student (https://github.com/msu-denver)" \
    https://api.papermc.io/v2/projects/paper/versions/${PAPER_VERSION}/builds/${PAPER_BUILD}/downloads/paper-${PAPER_VERSION}-${PAPER_BUILD}.jar

# Exposes on: localhost:9940/metrics
RUN mkdir -p plugins && \
    curl -L -o plugins/prometheus-exporter.jar \
    https://github.com/sladkoff/minecraft-prometheus-exporter/releases/download/v3.1.0/prometheus-exporter-3.1.0.jar

RUN echo "eula=true" > eula.txt

EXPOSE 25565 9940
CMD ["java", "-Xms1G", "-Xmx2G", "-jar", "paper.jar", "--nogui"]