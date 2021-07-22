FROM centos
RUN yum update -y
RUN yum install wget -y
RUN wget https://github.com/prometheus/prometheus/releases/download/v2.28.1/prometheus-2.28.1.linux-amd64.tar.gz
RUN yum install tar -y
RUN tar -xzf prometheus-2.28.1.linux-amd64.tar.gz
WORKDIR /prometheus-2.28.1.linux-amd64
COPY prometheus.yml .
EXPOSE 9090
CMD ./prometheus
