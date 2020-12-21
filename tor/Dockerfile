FROM alpine:3.12

RUN apk -U add tor torsocks

EXPOSE 9050 53/udp

COPY torrc /etc/tor/

USER tor
CMD ["tor", "-f", "/etc/tor/torrc"]
