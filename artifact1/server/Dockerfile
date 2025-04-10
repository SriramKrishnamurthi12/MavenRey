# 1st stage, build the app
FROM maven:3.6-jdk-11 as build

WORKDIR /helidon

# Create a first layer to cache the "Maven World" in the local repository.
# Incremental docker builds will always resume after that, unless you update
# the pom
ADD pom.xml .
ADD client/pom.xml client/pom.xml
ADD server/pom.xml server/pom.xml
ADD spec/api.yaml spec/api.yaml
RUN mvn -pl server package --am -DskipTests -DskipOpenApiGenerate

# Do the Maven build!
# Incremental docker builds will resume here when you change sources
ADD client/src client/src
ADD server/src server/src
RUN mvn -pl server package --am -DskipTests
RUN echo "Build done!"

# 2nd stage, build the runtime image
FROM openjdk:11-jre-slim
RUN groupadd -g 999 helidon && \
    useradd -r -u 999 -g helidon helidon
USER helidon
WORKDIR /home/helidon

# Copy the binary built in the 1st stage
COPY --from=build /helidon/server/target/artifact1-server.jar ./
COPY --from=build /helidon/server/target/libs ./libs

CMD ["java", "-jar", "artifact1-server.jar"]

EXPOSE 8080
