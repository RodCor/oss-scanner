FROM rust:slim-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_BUILD_JOBS=2 \
    RUSTFLAGS="-C force-frame-pointers=yes" \
    KIMETSU_USER_BRAIN=0

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential ca-certificates clang cmake curl git libssl-dev \
      nodejs npm pkg-config python3 python3-venv \
    && rm -rf /var/lib/apt/lists/*

# The scanner supplies the public Kimetsu checkout as the build context.
COPY . /src
WORKDIR /src

# Keep sources, Cargo's dependency cache, debug symbols, and test executables
# available for offline investigation. Include TLS and both host integrations.
RUN cargo fetch --locked \
    && cargo build --workspace --locked --features pi,openclaw,tls \
    && cargo test --workspace --locked --features pi,openclaw,tls --no-run

RUN npm --prefix npm/kimetsu-sdk ci \
    && npm --prefix npm/kimetsu-sdk run build

# Test failures remain visible but do not prevent investigation of the image.
RUN timeout 600 cargo test --workspace --locked --offline \
      --features pi,openclaw,tls -- --test-threads=1 \
    || echo "WARNING: Rust tests failed or exceeded 600 seconds; inspect and rerun individual tests"
RUN npm --prefix npm/kimetsu-sdk test \
    || echo "WARNING: TypeScript SDK tests reported failures"

ENV PATH="/src/target/debug:${PATH}" \
    CARGO_NET_OFFLINE=true
