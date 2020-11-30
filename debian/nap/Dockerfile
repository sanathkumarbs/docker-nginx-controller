ARG STORE_UUID=False
FROM debian:stretch-slim as nginx-installer

LABEL maintainer="NGINX Controller Engineering"

# Download certificate (nginx-repo.crt) and key (nginx-repo.key) from the customer portal (https://cs.nginx.com)
# and copy to the build context
COPY nginx-repo.* /etc/ssl/nginx/
COPY nginx-plus-api.conf /etc/nginx/conf.d/
COPY entrypoint.sh /

ENV NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
# Install NGINX Plus
RUN set -ex \
  && apt-get update && apt-get upgrade -y \
  && apt-get install --no-install-recommends --no-install-suggests -y curl sudo procps apt-utils apt-transport-https ca-certificates gnupg1 distro-info-data libmpdec2 \
  lsb-release python python-minimal binutils net-tools \
  && \
  found=''; \
  for server in \
    ha.pool.sks-keyservers.net \
    hkp://keyserver.ubuntu.com:80 \
    hkp://p80.pool.sks-keyservers.net:80 \
    pgp.mit.edu \
  ; do \
    echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
    apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
  done; \
  test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
  echo "Acquire::https::plus-pkgs.nginx.com::Verify-Peer \"true\";" >> /etc/apt/apt.conf.d/90nginx \
  && echo "Acquire::https::plus-pkgs.nginx.com::Verify-Host \"true\";" >> /etc/apt/apt.conf.d/90nginx \
  && echo "Acquire::https::plus-pkgs.nginx.com::SslCert     \"/etc/ssl/nginx/nginx-repo.crt\";" >> /etc/apt/apt.conf.d/90nginx \
  && echo "Acquire::https::plus-pkgs.nginx.com::SslKey      \"/etc/ssl/nginx/nginx-repo.key\";" >> /etc/apt/apt.conf.d/90nginx \
  && printf "deb https://plus-pkgs.nginx.com/debian stretch nginx-plus\n" > /etc/apt/sources.list.d/nginx-plus.list \
  # NGINX Javascript module needed for APIM
  && apt-get update && apt-get install -y nginx-plus nginx-plus-module-njs

FROM nginx-installer as agent-installer
# Install Controller Agent
ARG CONTROLLER_URL
ARG API_KEY
ENV ENV_CONTROLLER_API_KEY=$API_KEY
ARG STORE_UUID
RUN curl -k -sS -L ${CONTROLLER_URL} > install.sh \
  && sed -i 's/^assume_yes=""/assume_yes="-y"/' install.sh \
  && sh ./install.sh -y

FROM agent-installer as nap-installer
# Install nginx-app-protect
RUN apt-get install -y app-protect \
  && sed -i "6 a load_module modules/ngx_http_app_protect_module.so;" /etc/nginx/nginx.conf


FROM nap-installer as cleaner
# cleanup sensitive nginx-plus data
RUN rm /etc/ssl/nginx/nginx-repo.* \
  && rm /etc/apt/sources.list.d/nginx-plus.list \
  && rm /etc/apt/apt.conf.d/90nginx \
  && apt-key del $NGINX_GPGKEY \
  && rm -rf /var/lib/apt/lists/*


FROM cleaner as runner
## e.g '1234567890'
ARG API_KEY
ENV ENV_CONTROLLER_API_KEY=$API_KEY

# e.g https://<fqdn>/install/controller-agent
ARG CONTROLLER_URL
ENV ENV_CONTROLLER_URL=$CONTROLLER_URL

# e.g True or False
ARG STORE_UUID
ENV ENV_CONTROLLER_STORE_UUID=$STORE_UUID

# e.g Instance location already defined in Controller
ARG LOCATION
ENV ENV_CONTROLLER_LOCATION=$LOCATION
# Forward request logs to Docker log collector
RUN ln -sf /dev/stdout /var/log/nginx-controller/agent.log \
  && ln -sf /dev/stderr /var/log/nginx/error.log

COPY custom_log_format.json /etc/nginx/

EXPOSE 80

STOPSIGNAL SIGTERM

ENTRYPOINT ["sh", "/entrypoint.sh"]