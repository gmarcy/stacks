# Step 1: Build the user's application
FROM {{.stack.baseimage}} as compile

USER root

RUN dnf --disableplugin=subscription-manager install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm \
  && dnf --disableplugin=subscription-manager install -y unzip curl ca-certificates wget xmlstarlet \
  && dnf --disableplugin=subscription-manager update -y; dnf --disableplugin=subscription-manager clean all

RUN  groupadd java_group \
  && useradd --gid java_group --shell /bin/bash --create-home java_user \
  && mkdir -p /mvn/repository \
  && chown -R java_user:java_group /mvn \
  && mkdir -p /config \
  && chown -R java_user:java_group /config \
# make a well known place for shared library jars seperate from the rest of the defaultServer contents (to help with caching)
  && mkdir /configlibdir \
  && chown -R java_user:java_group /configlibdir \
  &&  mkdir /shared \
  && chown -R java_user:java_group /shared \
  && mkdir -p /project \
  && mkdir -p /usr/share/maven \
  && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn \
  && chown -R java_user:java_group /project /opt/java/openjdk /usr/share/maven

USER java_user

#
# Install maven
#
# This section based upon:
#   https://github.com/kabanero-io/collections/blob/master/docker/Dockerfile-maven
#

ARG MAVEN_VERSION=3.6.3
ARG USER_HOME_DIR="/root"
ARG SHA=c35a1803a6e70a126e80b2b3ae33eed961f83ed74d18fcd16909b2d44d7dada3203f1ffe726c17ef8dcca2dcaa9fca676987befeadc9b9f759967a8cb77181c0
ARG BASE_URL=https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/

RUN mkdir -p /usr/share/maven/ref \
  && curl -fsSL -o /tmp/apache-maven.tar.gz ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
  && echo "${SHA}  /tmp/apache-maven.tar.gz" | sha512sum -c - \
  && tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 \
  && rm -f /tmp/apache-maven.tar.gz

ENV MAVEN_HOME /usr/share/maven
ENV MAVEN_CONFIG "$USER_HOME_DIR/.m2"

# Copy and build the dev.appsody:java-openliberty parent pom
COPY --chown=java_user:java_group ./pom.xml /project/pom.xml
RUN cd /project && mvn -B install dependency:go-offline -DskipTests

# Prime image
#   a) Prime .m2/repository with common artifacts 
#   b) Create target/liberty/wlp/usr/servers/defaultServer dir
COPY --chown=java_user:java_group ./preload-m2-pom.xml /project/user-app/preload-m2-pom.xml
RUN cd /project/user-app && \ 
    mvn -B -f /project/user-app/preload-m2-pom.xml liberty:install-server dependency:go-offline && \
    rm /project/user-app/preload-m2-pom.xml

# Copy and run a simple version check
COPY --chown=java_user:java_group ./util /project/util
RUN     /project/util/check_version build

# Copy the validate.sh script and application pom.xml
COPY --chown=java_user:java_group ./validate.sh /project/user-app/validate.sh
# -- This is the first app-specific piece --
COPY --chown=java_user:java_group ./user-app/pom.xml /project/user-app/pom.xml
# Validate 
RUN cd /project/user-app && ./validate.sh build

# Copy the rest of the application source
COPY --chown=java_user:java_group ./user-app /project/user-app

# Build (and run unit tests) 
#  also liberty:create copies config from src->target
#  also remove quick-start-security.xml since it's convenient for local dev mode but should not be in the production image.
RUN cd /project/user-app && \
    echo "QUICK START SECURITY IS NOT SECURE FOR PRODUCTION ENVIRONMENTS. IT IS BEING REMOVED" \
    rm -f src/main/liberty/config/configDropins/defaults/quick-start-security.xml && \
    mvn -Pappsody-build -B liberty:create package

# process any resources or shared libraries - if they are present in the dependencies block for this project (there may be none potentially)
# test to see if each is present and move to a well known location for later processing in the next stage
# 
RUN cd /project/user-app/target/liberty/wlp/usr/servers && \
    if [ -d ./defaultServer/lib ]; then mv ./defaultServer/lib /configlibdir; fi && \
    if [ ! -d /configlibdir/lib ]; then mkdir /configlibdir/lib; fi && \
    mv -f defaultServer/* /config/ && \
    if [ -d ../shared ]; then mv ../shared/* /shared/; fi

# Step 2: Package Open Liberty image
FROM openliberty/open-liberty:{{.stack.libertyversion}}-kernel-java8-openj9-ubi

#2a) copy any resources 
COPY --chown=1001:0 --from=compile /shared /opt/ol/wlp/usr/shared/

# 2b) next copy shared library
#      but can't assume config/lib exists - copy from previous stage to a tmp holding place and test
COPY --chown=1001:0 --from=compile /configlibdir/ /config

# 2c) Server config, bootstrap.properties, and everything else
COPY --chown=1001:0 --from=compile /config/ /config/


# 2d) Changes to the application binary
COPY --chown=1001:0 --from=compile /project/user-app/target/*.[ew]ar /config/apps
RUN configure.sh && \
    chmod 664 /opt/ol/wlp/usr/servers/defaultServer/configDropins/defaults/keystore.xml
