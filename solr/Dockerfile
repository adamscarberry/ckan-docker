# Build latest gosu using a pinned Alpine variant with security updates
FROM golang:1.25.7-alpine AS gosu-builder

# Install necessary build tools
RUN apk add --no-cache git ca-certificates

# Clone the gosu repository and build the binary
RUN git clone https://github.com/tianon/gosu.git /src && \
    cd /src && \
    CGO_ENABLED=0 go build -o /gosu . && \
    chmod +x /gosu

# Runtime stage
FROM ckan/ckan-solr:2.10-solr9

USER root

# Copy the gosu binary from the build stage to the runtime image
COPY --from=gosu-builder /gosu /usr/sbin/gosu

# Ensure it's executable
RUN chmod +x /usr/sbin/gosu

USER solr