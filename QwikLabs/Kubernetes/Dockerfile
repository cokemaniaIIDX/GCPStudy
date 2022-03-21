FROM nginx:latest
LABEL maintainer="cokemaniaIIDX cokemaniaIIDX@gmail.com"

RUN echo "building nginx server..."
RUN yum update -y && yum clean all
ADD index.html /usr/share/