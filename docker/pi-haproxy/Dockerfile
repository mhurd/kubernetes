FROM arm32v7/haproxy:1.8.2
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
RUN apt-get update \
    && apt-get install -y iproute2 traceroute dnsutils net-tools less vim

