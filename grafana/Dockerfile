FROM centos
RUN yum update -y
RUN yum install wget -y
RUN wget https://dl.grafana.com/oss/release/grafana-8.0.6-1.x86_64.rpm
RUN yum install grafana-8.0.6-1.x86_64.rpm -y
WORKDIR /usr/share/grafana
EXPOSE 3000
CMD [ "/usr/sbin/grafana-server", "cfg:default.paths.data=/var/lib/grafana"  , "--config=/etc/grafana/grafana.ini" ]

