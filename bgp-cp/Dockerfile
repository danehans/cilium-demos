FROM ubuntu:20.04

# Install bird and other utilities
RUN apt-get update && apt-get install -y bird iproute2 curl && apt-get clean

# Ensure packet forwarding between interfaces is allowed
RUN echo 1 > /proc/sys/net/ipv4/ip_forward

# Configuration will be mounted, so no need to ADD/COPY
RUN mkdir -p /var/run/bird

CMD ["bird", "-c", "/config/bird.conf", "-s", "/var/run/bird/bird.ctl", "-d"]
