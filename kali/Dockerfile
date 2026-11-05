FROM kalilinux/kali-rolling

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    nmap \
    libcap2-bin \
    iproute2 \
    iputils-ping \
    dnsutils \
    curl \
    tcpdump \
    netcat-openbsd \
    traceroute \
    whois \
    nikto \
    whatweb \
    gobuster \
    && setcap -r /usr/lib/nmap/nmap \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

CMD ["sleep", "infinity"]