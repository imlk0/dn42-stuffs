FROM ubuntu/bind9:latest

COPY --chown=bind:bind ./named.conf /etc/bind/named.conf
COPY --chown=bind:bind ./zones /etc/bind/zones
COPY --chown=bind:bind ./dnssec_keys /etc/bind/dnssec_keys

# 在构建时对zone进行签名，同时还会基于DNSKEY产生DS记录
RUN cd /etc/bind/zones && \
    dnssec-signzone -K ../dnssec_keys -o imlk.dn42. ./imlk.dn42.zone && { echo "Generated DS record:"; cat ./dsset-imlk.dn42. ; } && \
    dnssec-signzone -K ../dnssec_keys -o 0/26.96.22.172.in-addr.arpa. ./0%2F26.96.22.172.in-addr.arpa.zone && { echo "Generated DS record:"; cat ./dsset-0%2F26.96.22.172.in-addr.arpa. ; } && \
    dnssec-signzone -K ../dnssec_keys -o e.0.a.8.a.a.2.d.2.4.d.f.ip6.arpa. ./e.0.a.8.a.a.2.d.2.4.d.f.ip6.arpa.zone && { echo "Generated DS record:"; cat ./dsset-e.0.a.8.a.a.2.d.2.4.d.f.ip6.arpa. ; }
    

CMD ["sh", "-c", "\
# 加去往dn42的路由
ip r add 172.20.0.0/14 via $DN42_GATEWAY_V4 && \
ip -6 r del default && ip -6 r add default via $DN42_GATEWAY_V6 && \
# 启动服务
docker-entrypoint.sh \
"]
