#
# This Dockerfile packages the SolarView main app (i.e. solarview-fb).
#
# The image is created in two stages. The first stage downloads the binaries,
# stores the release date and selects the correct binaries for the architecture.
# The second image prepares runtime dependencies and defines the start command.
#
ARG RUNTIME_BASE_IMAGE=debian:stable-slim

# builder image, prepares the runtime image
FROM alpine AS builder
ARG TARGETARCH
ARG APP_NAME
RUN test -n "${APP_NAME}" || { echo >&2 "Missing build-arg: APP_NAME"; exit 1; }
ARG APP_ARCHIVE_URL="http://www.solarview.info/downloads/${APP_NAME}.zip"
ARG APP_ARCHIVE_FILE="/build/${APP_NAME}.zip"
ARG APP_BINARIES="${APP_NAME}"
ARG APP_HOME="/opt/${APP_NAME}"

WORKDIR "${APP_HOME}"
ADD "${APP_ARCHIVE_URL}" "${APP_ARCHIVE_FILE}"
# the 'unzip' package is used because busybox-unzip does not retain timestamps
RUN apk add unzip && unzip "${APP_ARCHIVE_FILE}"
RUN stat -c %y "${APP_ARCHIVE_FILE}" >.release_date
# select the binaries for the target architecture
RUN [ -z "${APP_BINARIES}" ] || { \
      target_arch="${TARGETARCH:-$(arch)}" && \
      case "${target_arch}" in \
        386|amd64) sv_arch=x86  ;; \
        arm*)      sv_arch=rpi  ;; \
        mips)      sv_arch=7390 ;; \
        mipsel)    sv_arch=71xx ;; \
        *)         echo >&2 "Unsupported architecture: ${target_arch}" && exit 1;; \
      esac && \
      backup_dir="Andere/71xx" && \
      mkdir -p "${backup_dir}" && \
      for binary in ${APP_BINARIES}; do \
        mv "${binary}" "${backup_dir}" && \
        chmod +x "Andere/${sv_arch}/${binary}" && \
        ln -s "Andere/${sv_arch}/${binary}" "${binary}" ; \
      done \
    }
# the built-in CSmtp binary requires an old libssl version that is not easy to
# install on current operating systems, thus we install a portable drop-in
# replacement for it
COPY CSmtp "${APP_HOME}/CSmtp.sh"
RUN ln -sf "${APP_HOME}/CSmtp.sh" CSmtp

# runtime image
FROM ${RUNTIME_BASE_IMAGE} AS runtime
ARG TARGETARCH
ARG APP_NAME
ENV APP_NAME="${APP_NAME}"
ENV APP_HOME="/opt/${APP_NAME}"
ENV APP_RUNTIME="/var/opt/${APP_NAME}"
COPY --from=builder "${APP_HOME}" "${APP_HOME}"
# curl is necessary for mail and ftp upload; the builtin version does not work on the
# 386 architecture, so we use an external version
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && \
    apt-get install -y curl && \
    if [ "${TARGETARCH:-$(arch)}" = 'amd64' ]; then apt-get install -y libc6-i386; fi && \
    apt-get clean && \
    ln -sf "$(which curl)" "${APP_HOME}/curl"
WORKDIR "${APP_RUNTIME}"
# Workaround for a bug on armhf: the symlinks in /etc/ssl/certs/ with names
# xxxxxxx.0 are missing. update-ca-certificates -f recreates them.
RUN update-ca-certificates -f
# When the container is started, the files from the release archive are copied to
# a runtime directory, which is mounted to the host. When a file on the host is
# newer, it is not overwritten. This technique allows to update the binaries and
# keep the user data without a special update mechanism.
CMD  [ "/bin/sh", "-c", "cp -u -a \"${APP_HOME}\"/* \"${APP_RUNTIME}/\" && exec \"./${APP_NAME}\" ${ARGS} -nofork" ]
VOLUME "${APP_RUNTIME}"
