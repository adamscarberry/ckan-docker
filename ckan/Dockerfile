# Base image src: https://github.com/ckan/ckan-docker-base/blob/main/ckan-2.11/Dockerfile
FROM ckan/ckan-base:2.11 AS builder

# Install any extensions needed by your CKAN instance
# See Dockerfile.dev for more details and examples

# Copy custom initialization scripts
COPY --chown=ckan-sys:ckan-sys docker-entrypoint.d/* /docker-entrypoint.d/

# Apply any patches needed to CKAN core or any of the built extensions (not the
# runtime mounted ones)
COPY --chown=ckan-sys:ckan-sys patches ${APP_DIR}/patches

USER ckan

RUN for d in $APP_DIR/patches/*; do \
        if [ -d $d ]; then \
            for f in `ls $d/*.patch | sort -g`; do \
                cd $SRC_DIR/`basename "$d"` && echo "$0: Applying patch $f to $SRC_DIR/`basename $d`"; patch -p1 < "$f" ; \
            done ; \
        fi ; \
    done

# -------------------------------
# Stage 2: Wolfi runtime
# -------------------------------
# FROM ghcr.io/usace/usace-wm-python:3.11 AS runtime
FROM cwbi-python:3.10 AS runtime

USER root

# Install Python + system dependencies required by CKAN
RUN apk update && apk add --no-cache \
        bash \
        # git \
        shadow \
        postgresql-client \
        # postgresql-dev \
        libxml2 \
        libxslt \
        libffi \
        zlib \
        libmagic-dev \
        # for python build
        # build-base \
        # openssl-dev \
        wget

# Set working directories
ARG APP_DIR=/srv/app
ARG SRC_DIR=/srv/app/src
WORKDIR $APP_DIR

# ---- Temporary build deps ----
RUN apk add --no-cache --virtual .build-deps \
    build-base \
    postgresql-dev \
    openssl-dev \
    git && \
    # ----------------
    # Upgrade pip toolchain
    # ----------------
    pip install --no-cache-dir --upgrade pip setuptools wheel && \
    # ----------------
    # Install CKAN + deps
    # ----------------
    pip install --no-cache-dir \
    "ckan[requirements] @ git+https://github.com/ckan/ckan.git@ckan-2.11.4" \
    uwsgi \
    "setuptools>=70.0.0" && \
    # ----------------
    # Extensions
    # ----------------
    pip install --no-cache-dir \
    git+https://github.com/okfn/ckanext-envvars.git@v0.0.6 && \
    # ----------------
    # Security patches
    # ----------------
    pip install --no-cache-dir --upgrade \
    urllib3>=2.6.3 \
    Werkzeug==3.1.5 && \
    # ----------------
    # Remove build deps
    # ----------------
    apk del .build-deps



# # Create the 'ckan-sys' group and users 'ckan-sys' and 'ckan' with matching UID/GID
RUN groupadd -g 502 ckan-sys && \
    useradd -rm -d /srv/app -s /bin/bash -g ckan-sys -u 502 ckan-sys && \
    useradd -rm -d /srv/app -s /bin/bash -g ckan-sys -u 503 ckan && \
    # Create directories (to avoid chown errors)
    mkdir -p /srv/app/src /srv/app/docker-entrypoint.d /srv/app/patches

# # Copy CKAN application from builder stage
COPY --from=builder /docker-entrypoint.d $APP_DIR/docker-entrypoint.d
COPY --from=builder /srv/app $APP_DIR

USER ckan

# Internals, you probably don't need to change these
ENV TZ=UTC
ENV APP_DIR=/srv/app
ENV SRC_DIR=${APP_DIR}/src
ENV CKAN_INI=${APP_DIR}/ckan.ini
ENV PIP_SRC=${SRC_DIR}
ENV CKAN_STORAGE_PATH=/var/lib/ckan
ENV GIT_URL=https://github.com/ckan/ckan.git

# Set entrypoint
CMD ["/srv/app/start_ckan.sh"]
