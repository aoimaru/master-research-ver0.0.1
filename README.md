
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
        -> phase-2で注目
    ],
    [
        ["SC-APT-GET-UPDATE"]
    ],
    [
        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-YES"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-F-NO-INSTALL-RECOMMENDS"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:GNUPG"],
        ["SC-APT-GET-INSTALL", "SC-APT-GET-PACKAGES", "SC-APT-GET-PACKAGE:DIRMNGR"]
        -> phase-2で注目
    ],
    [
        ["SC-EXPORT", "SC-EXPORT-TARGET", "BASH-ASSIGN", "BASH-ASSIGN-LHS", "BASH-VARIABLE:GNUPGHOME"],
        ["SC-EXPORT", "SC-EXPORT-TARGET", "BASH-ASSIGN", "BASH-ASSIGN-RHS", "BASH-DOUBLE-QUOTED","BASH-DOLLAR-PARENS", "SC-MKTEMP", "SC-MKTEMP-F-DIRECTORY"]
    ],
    ******


    "phase-2":



```