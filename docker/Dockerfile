FROM bitwalker/alpine-elixir-phoenix:1.12

RUN git config --global https.proxy ${HTTPS_PROXY}
RUN git config --global http.proxy ${HTTPS_PROXY}
RUN echo "proxy = ${HTTPS_PROXY}" | tee $HOME/.curl >/dev/null 2>&1

ENV HTTPS_PROXY ${HTTPS_PROXY}

RUN apk --no-cache --update add alpine-sdk gmp-dev automake libtool inotify-tools autoconf python3 file

ENV GLIBC_REPO=https://github.com/sgerrand/alpine-pkg-glibc
ENV GLIBC_VERSION=2.30-r0

RUN set -ex && \
    apk --update add libstdc++ curl ca-certificates && \
    for pkg in glibc-${GLIBC_VERSION} glibc-bin-${GLIBC_VERSION}; \
        do curl -sSL ${GLIBC_REPO}/releases/download/${GLIBC_VERSION}/${pkg}.apk -o /tmp/${pkg}.apk; done && \
    apk add --allow-untrusted /tmp/*.apk && \
    rm -v /tmp/*.apk && \
    /usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib

# Get Rust
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

ENV PATH="$HOME/.cargo/bin:${PATH}"
ENV RUSTFLAGS="-C target-feature=-crt-static"

EXPOSE 4000

ENV PORT=4000 \
    MIX_ENV="prod" \
    SECRET_KEY_BASE="RMgI4C1HSkxsEjdhtGMfwAHfyT6CKWXOgzCboJflfSm4jeAlic52io05KB6mqzc5"

# Cache elixir deps
ADD mix.exs mix.lock ./
ADD apps/block_scout_web/mix.exs ./apps/block_scout_web/
ADD apps/explorer/mix.exs ./apps/explorer/
ADD apps/ethereum_jsonrpc/mix.exs ./apps/ethereum_jsonrpc/
ADD apps/indexer/mix.exs ./apps/indexer/



RUN HTTPS_PROXY="" HEX_MIRROR="https://hexpm.upyun.com" HEX_CDN="https://hexpm.upyun.com" HEX_HTTP_CONCURRENCY=3 HEX_HTTP_TIMEOUT=2400 mix do deps.get
RUN HTTPS_PROXY="" HEX_MIRROR="https://hexpm.upyun.com" HEX_CDN="https://hexpm.upyun.com" mix do local.rebar --force
RUN mix do deps.compile

ADD . .

ARG COIN
RUN if [ "$COIN" != "" ]; then \
        sed -i s/"POA"/"${COIN}"/g apps/block_scout_web/priv/gettext/en/LC_MESSAGES/default.po; \
        sed -i "/msgid \"Ether\"/{n;s/msgstr \"\"/msgstr \"${COIN}\"/g}" apps/block_scout_web/priv/gettext/default.pot; \
        sed -i "/msgid \"Ether\"/{n;s/msgstr \"\"/msgstr \"${COIN}\"/g}" apps/block_scout_web/priv/gettext/en/LC_MESSAGES/default.po; \
    fi

# Run forderground build and phoenix digest
RUN mix compile

RUN npm install npm@latest --registry=https://registry.npmmirror.com

# Add blockscout npm deps
RUN cd apps/block_scout_web/assets/ && \
    npm install --registry=https://registry.npmmirror.com && \
    npm run deploy && \
    cd -

RUN cd apps/explorer/ && \
    npm install && \
    apk update && apk del --force-broken-world alpine-sdk gmp-dev automake libtool inotify-tools autoconf python3

RUN mix phx.digest

RUN rm -rf ~/.gitconfig
RUN rm -rf ~/.curl

ENV HTTPS_PROXY ""
RUN echo "h $HTTP_PROXY"