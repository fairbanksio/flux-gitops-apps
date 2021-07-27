FROM alpine:latest
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
USER appuser
ENTRYPOINT ["/entrypoint.sh"]
