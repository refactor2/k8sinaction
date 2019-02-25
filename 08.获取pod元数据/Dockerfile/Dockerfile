FROM alpine
ADD kubectl /kubectl
ADD kubectl-proxy.sh /kubectl-proxy.sh
RUN chmod +x /kubectl-proxy.sh && chmod +x /kubectl
ENTRYPOINT /kubectl-proxy.sh