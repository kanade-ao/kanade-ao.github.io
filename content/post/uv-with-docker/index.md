---
title: uvによるDockerfile最適化
description: uvによるPython開発で、Dockerイメージを最適化するために、コマンドの詳細を分析する
date: 2025-12-21 15:00:00+0900
tags:
    - python
    - uv
    - docker
categories:
    - python
    - docker
---

# Docker with uv

最近の開発で、Pythonのパッケージ管理ツールを **uv** に切り替えて、それに伴うDockerイメージの最適化方法を知りたいです。

人気のあるレポジトリ [FastAPIテンプレート](https://github.com/fastapi/full-stack-fastapi-template/blob/master/backend/Dockerfile)
にあるDockerfileを勉強材料にしました。

これからステップ・バイ・ステップで、Dockerfileの分析を行います。

```docker
FROM python:3.10
ENV PYTHONUNBUFFERED=1
```

`PYTHONUNBUFFERED`を空以外の値に設定すると、Python標準出力`stdout`・標準エラー`stderr`ストリームのバッファリングを行わず、
すぐにターミナルに書き出される。

これはなにかのエラーになるとき、デバッグしやすくなります。

```docker
WORKDIR /app/

# Install uv
# Ref: https://docs.astral.sh/uv/guides/integration/docker/#installing-uv
COPY --from=ghcr.io/astral-sh/uv:0.5.11 /uv /uvx /bin/

# Place executables in the environment at the front of the path
# Ref: https://docs.astral.sh/uv/guides/integration/docker/#using-the-environment
ENV PATH="/app/.venv/bin:$PATH"

# Compile bytecode
# Ref: https://docs.astral.sh/uv/guides/integration/docker/#compiling-bytecode
ENV UV_COMPILE_BYTECODE=1
```

`UV_COMPILE_BYTECODE`を設定することにより、パッケージをインストールする時に、`.py`をバイトコード`.pyc`ファイルにビルドされる。

起動時間を短縮することができそうです。


```docker
# uv Cache
# Ref: https://docs.astral.sh/uv/guides/integration/docker/#caching
ENV UV_LINK_MODE=copy
```

公式サイトにより、この設定はハードリンクが使えない場合に`uv`が警告しないようにするものです。

```docker
# Install dependencies
# Ref: https://docs.astral.sh/uv/guides/integration/docker/#intermediate-layers
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project
```

Dockerのキャッシュを利用するため、`uv sync`を実行するとき、パッケージのみインストールし、プロジェクトファイルを無視する。

```docker
ENV PYTHONPATH=/app

COPY ./scripts /app/scripts

COPY ./pyproject.toml ./uv.lock ./alembic.ini /app/

COPY ./app /app/app
COPY ./tests /app/tests

# Sync the project
# Ref: https://docs.astral.sh/uv/guides/integration/docker/#intermediate-layers
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync

CMD ["fastapi", "run", "--workers", "4", "app/main.py"]
```

すべてのパッケージがインストールされた後で、プロジェクト関連ファイルをコピーし、再度`uv sync`で同期する。

最後の部分をカスタマイズしたら、自分のプロジェクトに利用することができます。