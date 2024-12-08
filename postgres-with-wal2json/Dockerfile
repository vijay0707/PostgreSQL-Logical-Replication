FROM postgres:15

# Install build dependencies
RUN apt-get update && apt-get install -y \
    postgresql-server-dev-15 \
    build-essential \
    git \
    && rm -rf /var/lib/apt/lists/*

# Clone and install wal2json
RUN git clone https://github.com/eulerto/wal2json.git /tmp/wal2json \
    && cd /tmp/wal2json \
    && make && make install \
    && rm -rf /tmp/wal2json

# Copy the custom postgresql.conf to configure PostgreSQL
COPY postgresql.conf /etc/postgresql/postgresql.conf.sample

# Ensure PostgreSQL loads the wal2json extension
RUN echo "shared_preload_libraries = 'wal2json'" >> /etc/postgresql/postgresql.conf.sample
