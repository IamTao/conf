FROM ubuntu:16.04

COPY telegraf_1.8.3-1_amd64.deb install.deb
RUN dpkg -i install.deb \
 && rm install.deb

COPY telegraf.conf /etc/telegraf/telegraf.conf

EXPOSE 8125/udp 8092/udp 8094

CMD telegraf

# Required environment variables:
ENV INFLUXDB_HOST='' \
    INFLUXDB_PORT='8092' \
    INFLUXDB_DB='jobmonitor'
