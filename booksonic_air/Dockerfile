#============================#
#  ALEXBELGIUM'S DOCKERFILE  #
#============================#
#           _.------.
#       _.-`    ('>.-`"""-.
# '.--'`       _'`   _ .--.)
#    -'         '-.-';`   `
#    ' -      _.'  ``'--.
#        '---`    .-'""`
#               /`
#=== Home Assistant Addon ===#

#################
# 1 Build Image #
#################

ARG BUILD_FROM
ARG BUILD_VERSION
FROM $BUILD_FROM

##################
# 2 Modify Image #
##################

# Set S6 wait time
ENV S6_CMD_WAIT_FOR_SERVICES=1 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_SERVICES_GRACETIME=0

ENV BOOKSONIC_AIR_SETTINGS="/data"

# Global LSIO modifications
ARG CONFIGLOCATION="/data"
# hadolint ignore=SC2015, SC2013
RUN \
    # Avoid custom-init.d duplications
    for file in $(grep -sril 'Potential tampering with custom' /etc/cont-init.d /etc/services.d /etc/s6-overlay/s6-rc.d); do rm -f "$file"; done \
    \
    # Create new config folder if needed
    && for file in $(grep -srl "PUID:-911" /etc/cont-init.d /etc/s6-overlay/s6-rc.d); do sed -i "1a mkdir -p $CONFIGLOCATION" "$file"; done \
    \
    # Allow UID and GID setting
    && for file in $(grep -srl "PUID:-911" /etc/cont-init.d /etc/s6-overlay/s6-rc.d); do sed -i 's/bash/bashio/g' "$file" && sed -i '1a PUID="$(if [ -n $(bashio::config 'PUID') ]; then echo "0"; else bashio::config 'PUID'; fi)"' "$file" && sed -i '1a PGID="$(if [ -n $(bashio::config 'PGID') ]; then echo "0"; else bashio::config 'PGID'; fi)"' "$file"; done \
    \
    # Correct config location
    && for file in $(grep -Esril "/config[ '\"/]|/config\$" /etc/logrotate.d /defaults /etc/cont-init.d /etc/services.d /etc/s6-overlay/s6-rc.d); do sed -i "s=/config=$CONFIGLOCATION=g" "$file"; done \
    \
    # Avoid chmod /config
    && for files in /etc/services.d/*/* /etc/cont-init.d/* /etc/s6-overlay/s6-rc.d/*/*;do if [ -f $files ] && [[ ! -z $(awk '/chown.*abc:abc.*\\/,/.*\/config( |$)/{print FILENAME}' "$files") ]] ; then sed -i "s|/config$|/data|g" "$files"; fi ;done

##################
# 3 Install apps #
##################

# Add rootfs
COPY rootfs/ /

# Corrects permissions for s6 v3
RUN if [ -d /etc/cont-init.d ]; then chmod -R 755 /etc/cont-init.d; fi && \
    if [ -d /etc/services.d ]; then chmod -R 755 /etc/services.d; fi && \
    if [ -f /entrypoint.sh ]; then chmod 755 /entrypoint.sh; fi

# Modules
ARG MODULES="00-banner.sh 92-local_mounts.sh 92-smb_mounts.sh"

# Automatic modules download
RUN if ! command -v bash >/dev/null 2>/dev/null; then (apt-get update && apt-get install -yqq --no-install-recommends bash || apk add --no-cache bash) >/dev/null; fi \
    && if ! command -v curl >/dev/null 2>/dev/null; then (apt-get update && apt-get install -yqq --no-install-recommends curl || apk add --no-cache curl) >/dev/null; fi \
    && apt-get update && apt-get install -yqq --no-install-recommends ca-certificates || apk add --no-cache ca-certificates >/dev/null || true \
    && mkdir -p /etc/cont-init.d \
    && for scripts in $MODULES; do echo "$scripts" && curl -f -L -s -S "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/$scripts" -o /etc/cont-init.d/"$scripts" && [ "$(sed -n '/\/bin/p;q' /etc/cont-init.d/"$scripts")" != "" ] || (echo "script failed to install $scripts" && exit 1); done \
    && chmod -R 755 /etc/cont-init.d || printf '%s\n' "${MODULES}" >/MODULESFILE

# Manual apps
ENV PACKAGES="jq \
    curl \
    cifs-utils \
    keyutils"

# Automatic apps & bashio
# hadolint ignore=SC2015
RUN if ! command -v bash >/dev/null 2>/dev/null; then (apt-get update && apt-get install -yqq --no-install-recommends bash || apk add --no-cache bash) >/dev/null; fi \
    && if ! command -v curl >/dev/null 2>/dev/null; then (apt-get update && apt-get install -yqq --no-install-recommends curl || apk add --no-cache curl) >/dev/null; fi \
    && curl -f -L -s -S "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/automatic_packages.sh" --output /automatic_packages.sh \
    && chmod 777 /automatic_packages.sh \
    && eval /./automatic_packages.sh "${PACKAGES:-}" \
    && rm /automatic_packages.sh || (printf '%s\n' "${PACKAGES:-}" > /ENVFILE; \
    if [ -f /etc/s6-overlay/s6-rc.d/init-migrations/run ]; then chmod +x /etc/cont-init.d/00-aaa_dockerfile_backup.sh && sed -i "1a /./etc/cont-init.d/00-aaa_dockerfile_backup.sh" /etc/s6-overlay/s6-rc.d/init-migrations/run;fi)

################
# 4 Entrypoint #
################

#RUN chmod 777 /entrypoint.sh
#WORKDIR /
#ENTRYPOINT [ "/usr/bin/env" ]
#CMD [ "/entrypoint.sh" ]
#SHELL ["/bin/bash", "-o", "pipefail", "-c"]

############
# 5 Labels #
############

ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_DESCRIPTION
ARG BUILD_NAME
ARG BUILD_REF
ARG BUILD_REPOSITORY
ARG BUILD_VERSION
LABEL \
    io.hass.name="${BUILD_NAME}" \
    io.hass.description="${BUILD_DESCRIPTION}" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="alexbelgium (https://github.com/alexbelgium)" \
    org.opencontainers.image.title="${BUILD_NAME}" \
    org.opencontainers.image.description="${BUILD_DESCRIPTION}" \
    org.opencontainers.image.vendor="Home Assistant Add-ons" \
    org.opencontainers.image.authors="alexbelgium (https://github.com/alexbelgium)" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://github.com/alexbelgium" \
    org.opencontainers.image.source="https://github.com/${BUILD_REPOSITORY}" \
    org.opencontainers.image.documentation="https://github.com/${BUILD_REPOSITORY}/blob/main/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}
