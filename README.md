```bash


RUN set -ex \
	...
	&& wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz" \
	&& echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum -c - \
	&& mkdir -p /usr/src/ruby \
	&& tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1 \
	...


RUN set -ex \
	&& buildDeps=' \
		bison \
		dpkg-dev \
		libgdbm-dev \
		ruby \
	' \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends $buildDeps \
	&& rm -rf /var/lib/apt/lists/* \
	\
	&& wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz" \
	&& echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum -c - \
	&& mkdir -p /usr/src/ruby \
	&& tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1 \
	&& rm ruby.tar.xz \
	&& cd /usr/src/ruby \
	
	
[" command_1 ", " command_2 ", " command_3 "] --> ["RUN_N"]
[" command_2 ", " command_3 ", " command_4 "] --> ["RUN_N"]


["command_N-2", "command_N-1", " command_N "] --> ["RUN_N"]







[" cmd_vec_1 ", " cmd_vec_2 ", " cmd_vec_3 "] --> ["RUN_vec_N"]
[" cmd_vec_2 ", " cmd_vec_3 ", " cmd_vec_4 "] --> ["RUN_vec_N"]


["cmd_vec_N-2", "cmd_vec_N-1", " cmd_vec_N "] --> ["RUN_vec_N"]



[" cmd_vec_1 ", " cmd_vec_2 ", " cmd_vec_3 "] --> ["RUN_vec_1"]
[" cmd_vec_2 ", " cmd_vec_3 ", " cmd_vec_4 "] --> ["RUN_vec_1"]



[" cmd_vec_1 ", " cmd_vec_2 ", " cmd_vec_3 "] --> ["RUN_vec_N"]
[" cmd_vec_2 ", " cmd_vec_3 ", " cmd_vec_4 "] --> ["RUN_vec_N"]


["cmd_vec_N-2", "cmd_vec_N-1", " cmd_vec_N "] --> ["RUN_vec_N"]
[" cmd_vec_1 ", " cmd_vec_2 ", " cmd_vec_3 "] --> ["RUN_vec_N+1"]
[" cmd_vec_2 ", " cmd_vec_3 ", " cmd_vec_4 "] --> ["RUN_vec_N+1"]



```


## 基本的なアイディア
### Dockerfileを記述する際のコード補完機能は極めて弱い(ex. Dockerfile内で記載されている単語のみしか提案されない)
### 似ている記述の提案 -> grepに勝るものの提案
### 詰まるところDockerfileで重要なのは, コマンドの順序では？
例えば, 

```bash
```
使われている単語が一緒でも, コマンドの順序が変わると意味が変わってしまう.
grepなどでも, 似たような記述を検出可能できるが, 順番などは考慮できず適切でないパターンを提案してしまう?

そこで, コマンドの順序を考慮して学習させた, 学習モデル(分散表現)を用いて, 記述のパターンを検出, 提案する手法を提案する

今回は, Dockerfile内のRUN命令のみを研究対象として, RUN命令単位で提案を試みる

## トレーニングデータ
### データの単位

```bash
    "case": [
        ["SC-APT-GET-UPDATE"],
        -> このブロックでベクトル化

        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-YES"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-NO-INSTALL-RECOMMENDS"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:GNUPG"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:DIRMNGR"],
        -> このブロックでベクトル化

        ["SC-RM", "SC-RM-F-RECURSIVE"],
        ["SC-RM", "SC-RM-F-FORCE"],
        ["SC-RM", "SC-RM-PATHS", "SC-RM-PATH", "BASH-CONCAT", "BASH-LITERAL", "ABS-MAYBE-PATH"],
        ["SC-RM", "SC-RM-PATHS", "SC-RM-PATH", "BASH-CONCAT", "BASH-LITERAL", "ABS-APT-LISTS"],
        ["SC-RM", "SC-RM-PATHS", "SC-RM-PATH", "BASH-CONCAT", "BASH-LITERAL", "ABS-PATH-VAR"],
        ["SC-RM", "SC-RM-PATHS", "SC-RM-PATH", "BASH-CONCAT", "BASH-LITERAL", "ABS-PATH-ABSOLUTE"],
        ["SC-RM", "SC-RM-PATHS", "SC-RM-PATH", "BASH-CONCAT", "BASH-GLOB", "ABS-GLOB-STAR"]
        -> このブロックでベクトル化
    ]


    "original": 

    RUN apt-get update && 
        apt-get install -y \
            gnupg \
            dirmngr \
        --no-install-recommends && 
        rm -r /var/lib/apt/lists/*

```

### 学習データの生成方法
```bash
    "original":

    ******
    RUN set -eux; \
        \
        savedAptMark="$(apt-mark showmanual)"; \
        apt-get update; \
        \
        apt-get install -y --no-install-recommends gnupg dirmngr; \
        \
        export GNUPGHOME="$(mktemp -d)"; \
        for key in $GPG_KEYS; do \
            gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
        done; \
        \
        apt-get install -y --no-install-recommends wget ca-certificates; \
        \
        success=; \
        for url in $TOMCAT_TGZ_URLS; do \
            if wget -O tomcat.tar.gz "$url"; then \
                success=1; \
                break; \
            fi; \
        done; \

    ******

    "phase-1":

    [
        ["SC-SET", "SC-SET-F-E"],
        ["SC-SET", "SC-SET-F-U"],
        ["SC-SET", "SC-SET-F-X"]
    ],
    [
        ["BASH-ASSIGN", "BASH-ASSIGN-LHS", "BASH-VARIABLE:savedAptMark"],
        ["BASH-ASSIGN", "BASH-ASSIGN-RHS", "BASH-DOUBLE-QUOTED", "BASH-DOLLAR-PARENS"]
    ],
    [
        ["SC-APT-GET-UPDATE"]
    ],
    [
        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-YES"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-NO-INSTALL-RECOMMENDS"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:GNUPG"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:DIRMNGR"]
    ],
    [
        ["SC-EXPORT", "SC-EXPORT-TARGET", "BASH-ASSIGN", "BASH-ASSIGN-LHS", "BASH-VARIABLE:GNUPGHOME"],
        ["SC-EXPORT", "SC-EXPORT-TARGET", "BASH-ASSIGN", "BASH-ASSIGN-RHS", "BASH-DOUBLE-QUOTED","BASH-DOLLAR-PARENS", "SC-MKTEMP", "SC-MKTEMP-F-DIRECTORY"]
    ],
    ******

    "phase-1":

    [
        ["SC-SET", "SC-SET-F-E"],
        ["SC-SET", "SC-SET-F-U"],
        ["SC-SET", "SC-SET-F-X"]
    ],
    [
        ["BASH-ASSIGN", "BASH-ASSIGN-LHS", "BASH-VARIABLE:savedAptMark"],
        ["BASH-ASSIGN", "BASH-ASSIGN-RHS", "BASH-DOUBLE-QUOTED", "BASH-DOLLAR-PARENS"]
        -> phase-2で注目 ※1
    ],
    [
        ["SC-APT-GET-UPDATE"]
    ],
    [
        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-YES"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-NO-INSTALL-RECOMMENDS"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:GNUPG"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:DIRMNGR"]
        -> phase-2で注目 ※2
    ],
    [
        ["SC-EXPORT", "SC-EXPORT-TARGET", "BASH-ASSIGN", "BASH-ASSIGN-LHS", "BASH-VARIABLE:GNUPGHOME"],
        ["SC-EXPORT", "SC-EXPORT-TARGET", "BASH-ASSIGN", "BASH-ASSIGN-RHS", "BASH-DOUBLE-QUOTED","BASH-DOLLAR-PARENS", "SC-MKTEMP", "SC-MKTEMP-F-DIRECTORY"]
    ],
    ******


    "phase-2":

    "※1": 

    "0c1e517ccfa17cd28a2a1e54b6a017b6d7b94f0d-13-1": {
        "document": [
            ["BASH-ASSIGN", "BASH-ASSIGN-LHS", "BASH-VARIABLE:savedAptMark"],
            ["BASH-ASSIGN", "BASH-ASSIGN-RHS", "BASH-DOUBLE-QUOTED", "BASH-DOLLAR-PARENS"]
        ],
        "training_data": [
            "SC-SET",
            "SC-SET-F-E",
            "SC-SET",
            "SC-SET-F-U",
            "SC-SET",
            "SC-SET-F-X",
            "BASH-ASSIGN",
            "BASH-ASSIGN-LHS",
            "BASH-VARIABLE:savedAptMark",
            "BASH-ASSIGN",
            "BASH-ASSIGN-RHS",
            "BASH-DOUBLE-QUOTED",
            "BASH-DOLLAR-PARENS"
        ]
    },


    "※2": 

    "0c1e517ccfa17cd28a2a1e54b6a017b6d7b94f0d-13-3": {
        "document": [
            ["SC-APT-GET-INSTALL", "SC-APT-GET-F-YES"],
            ["SC-APT-GET-INSTALL", "SC-APT-GET-F-NO-INSTALL-RECOMMENDS"],
            ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:GNUPG"],
            ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:DIRMNGR"]
        ],
        "training_data": [
            "SC-SET",
            "SC-SET-F-E",
            "SC-SET",
            "SC-SET-F-U",
            "SC-SET",
            "SC-SET-F-X",
            "BASH-ASSIGN",
            "BASH-ASSIGN-LHS",
            "BASH-VARIABLE:savedAptMark",
            "BASH-ASSIGN",
            "BASH-ASSIGN-RHS",
            "BASH-DOUBLE-QUOTED",
            "BASH-DOLLAR-PARENS",
            "SC-APT-GET-UPDATE",
            "SC-APT-GET-INSTALL",
            "SC-APT-GET-F-YES",
            "SC-APT-GET-INSTALL",
            "SC-APT-GET-F-NO-INSTALL-RECOMMENDS",
            "SC-APT-GET-INSTALL",
            "SC-APT-GET-PACKAGES",
            "SC-APT-GET-PACKAGE:GNUPG",
            "SC-APT-GET-INSTALL",
            "SC-APT-GET-PACKAGES",
            "SC-APT-GET-PACKAGE:DIRMNGR"
        ]
    },

    ******

    $ "※2"少し詳しく

    "0c1e517ccfa17cd28a2a1e54b6a017b6d7b94f0d-13-3": {
        "document": [
            ["SC-APT-GET-INSTALL", "SC-APT-GET-F-YES"],
            ["SC-APT-GET-INSTALL", "SC-APT-GET-F-NO-INSTALL-RECOMMENDS"],
            ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:GNUPG"],
            ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:DIRMNGR"]
        ],
        "training_data": [
            ******

            "SC-SET",
            "SC-SET-F-E",
            "SC-SET",
            "SC-SET-F-U",
            "SC-SET",
            "SC-SET-F-X",

            ******

            "BASH-ASSIGN",
            "BASH-ASSIGN-LHS",
            "BASH-VARIABLE:savedAptMark",
            "BASH-ASSIGN",
            "BASH-ASSIGN-RHS",
            "BASH-DOUBLE-QUOTED",
            "BASH-DOLLAR-PARENS",

            ******

            "SC-APT-GET-UPDATE",

            ******

            "SC-APT-GET-INSTALL",
            "SC-APT-GET-F-YES",
            "SC-APT-GET-INSTALL",
            "SC-APT-GET-F-NO-INSTALL-RECOMMENDS",
            "SC-APT-GET-INSTALL",
            "SC-APT-GET-PACKAGES",
            "SC-APT-GET-PACKAGE:GNUPG",
            "SC-APT-GET-INSTALL",
            "SC-APT-GET-PACKAGES",
            "SC-APT-GET-PACKAGE:DIRMNGR"

            ******
        ]
    },



```


### Dockerfile サンプル-v1
```bash
# some of ruby's build scripts are written in ruby
#   we purge system ruby later to make sure our final image uses what we just built
RUN set -ex \
	\
	&& buildDeps=' \
		bison \
		dpkg-dev \
		libgdbm-dev \
		ruby \
	' \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends $buildDeps \
	&& rm -rf /var/lib/apt/lists/* \
	\
	&& wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz" \
	&& echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum -c - \
	\
	&& mkdir -p /usr/src/ruby \
	&& tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1 \
	&& rm ruby.tar.xz \
	\
	&& cd /usr/src/ruby \
	\

```

## Dockerfile サンプル-v2
```bash
RUN set -x \
	\
	&& savedAptMark="$(apt-mark showmanual)" \
	&& apt-get update && apt-get install -y --no-install-recommends \
		ca-certificates \
		gcc \
		libc6-dev \
		liblua5.3-dev \
		libpcre2-dev \
		libssl-dev \
		make \
		wget \
		zlib1g-dev \
	&& rm -rf /var/lib/apt/lists/* \
	\
	&& wget -O haproxy.tar.gz "$HAPROXY_URL" \
	&& echo "$HAPROXY_SHA256 *haproxy.tar.gz" | sha256sum -c \
	&& mkdir -p /usr/src/haproxy \
	&& tar -xzf haproxy.tar.gz -C /usr/src/haproxy --strip-components=1 \
	&& rm haproxy.tar.gz \
	\
	&& makeOpts=' \
		TARGET=linux2628 \
		USE_LUA=1 LUA_INC=/usr/include/lua5.3 \
		USE_GETADDRINFO=1 \
		USE_OPENSSL=1 \
		USE_PCRE2=1 USE_PCRE2_JIT=1 \
		USE_ZLIB=1 \
	' \
	&& make -C /usr/src/haproxy -j "$(nproc)" all $makeOpts \
	&& make -C /usr/src/haproxy install-bin $makeOpts \
	\
	&& mkdir -p /usr/local/etc/haproxy \
	&& cp -R /usr/src/haproxy/examples/errorfiles /usr/local/etc/haproxy/errors \
	&& rm -rf /usr/src/haproxy \
	\
	&& apt-mark auto '.*' > /dev/null \
	&& { [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; } \
	&& find /usr/local -type f -executable -exec ldd '{}' ';' \
		| awk '/=>/ { print $(NF-1) }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	&& apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false
```
