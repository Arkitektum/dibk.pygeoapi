# =================================================================
#
# DiBK image: our pygeoapi fork with the Starlette stack and our
# provider/formatter packages, built on a plain Python image.
#
# The build context is the repository root, not this directory:
#
#   docker build -f docker/dibk/Dockerfile -t dibk-pygeoapi:0.24.0 .
#
# Context excludes come from Dockerfile.dockerignore next to this
# file, which BuildKit uses in place of the root .dockerignore.
# Building with DOCKER_BUILDKIT=0 falls back to the root one and
# sends a much larger context.
#
# Why not FROM geopython/pygeoapi: that image installs some twenty
# Python packages with apt (GDAL, shapely, pyproj, pandas, scipy...),
# compiled against Ubuntu's NumPy 1.x, and exposes them to /venv via
# --system-site-packages. Any package that pulls in NumPy 2 breaks
# them. Here every Python package comes from PyPI into a plain venv,
# and the GDAL bindings are compiled against the system libgdal.
#
# =================================================================

ARG PYTHON_VERSION=3.12
ARG DEBIAN_RELEASE=trixie
ARG UV_VERSION=0.13.10


FROM ghcr.io/astral-sh/uv:${UV_VERSION} AS uv


# --- stage 1: build the venv -------------------------------------
#
# Compilers and -dev packages stay in this stage; only /venv is
# copied into the runtime image. Both stages use the same base image,
# so the venv's interpreter path and ABI match.

FROM python:${PYTHON_VERSION}-slim-${DEBIAN_RELEASE} AS build

COPY --from=uv /uv /usr/local/bin/uv

# libgdal-dev: headers and gdal-config for the GDAL bindings.
# libpq-dev: psycopg2, the driver behind pygeoapi's postgresql+psycopg2
# URLs. Neither is declared by our provider packages; the old base
# image supplied both through apt.
# git: the git+https packages below.
RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc g++ git libgdal-dev libpq-dev \
    && rm -rf /var/lib/apt/lists/*

ENV UV_NO_CACHE=1 \
    UV_COMPILE_BYTECODE=1 \
    UV_PYTHON_DOWNLOADS=never

# python3 -m venv rather than uv venv, so the venv has pip for images
# built on top of this one.
RUN python3 -m venv /venv

# The GDAL bindings must match libgdal exactly. They are built without
# build isolation, against the NumPy installed first: in an isolated
# build some GDAL versions find no NumPy and silently skip gdal_array,
# which gdal.UseExceptions() imports.
RUN uv pip install --python /venv/bin/python3 numpy setuptools wheel \
    && uv pip install --python /venv/bin/python3 --no-build-isolation-package GDAL \
       "GDAL[numpy]==$(gdal-config --version)"

# The five git packages are our own. gunicorn<24 matches upstream's
# Dockerfile; uvicorn supplies the gunicorn worker class.
ARG PYTHON_PACKAGES="\
    gunicorn<24 \
    psycopg2 \
    starlette \
    uvicorn \
    git+https://github.com/Arkitektum/dibk.pygeoapi.plugins@dibk_pygeoapi_plugins-v0.5.0#subdirectory=packages/dibk_pygeoapi_plugins \
    git+https://github.com/Arkitektum/dibk.pygeoapi.plugins@postgresql_provider-v0.5.3#subdirectory=packages/postgresql_provider \
    git+https://github.com/Arkitektum/dibk.pygeoapi.plugins@mvt_postgresql_provider-v0.5.0#subdirectory=packages/mvt_postgresql_provider \
    git+https://github.com/Arkitektum/dibk.pygeoapi.plugins@styles_provider-v0.5.0#subdirectory=packages/styles_provider \
    git+https://github.com/Arkitektum/dibk.pygeoapi.plugins@jsonfg_formatter-v0.5.0#subdirectory=packages/jsonfg_formatter \
    git+https://github.com/Arkitektum/dibk.pygeoapi.plugins@gml_npad_formatter-v0.5.1#subdirectory=packages/gml_npad_formatter"

WORKDIR /src

COPY . .

# pygeoapi from this checkout, with the dependencies in
# requirements.txt, plus the packages above in one resolution.
RUN uv pip install --python /venv/bin/python3 . ${PYTHON_PACKAGES}


# --- stage 2: the runtime image ----------------------------------

FROM python:${PYTHON_VERSION}-slim-${DEBIAN_RELEASE}

# Shared libraries behind the GDAL bindings and psycopg2. libgdal36
# is GDAL 3.10 on trixie; change it together with DEBIAN_RELEASE.
ARG DEB_PACKAGES="\
    libgdal36 \
    libpq5"

RUN apt-get update \
    && apt-get install -y --no-install-recommends ${DEB_PACKAGES} \
    && rm -rf /var/lib/apt/lists/*

COPY --from=build /venv /venv

# The entrypoint is upstream's, unchanged. It generates the OpenAPI
# document and starts gunicorn from /pygeoapi. No config ships with
# the image: upstream's default.config.yml is a demo that needs
# tests/data and SpatiaLite. Images built on this one supply their
# own, at /pygeoapi/local.config.yml or via PYGEOAPI_CONFIG.
COPY docker/entrypoint.sh /entrypoint.sh

# We run under Starlette. The entrypoint defaults to Flask with gevent
# workers, and gevent is not installed here.
ENV PATH=/venv/bin:${PATH} \
    WSGI_APP=pygeoapi.starlette_app:APP \
    WSGI_WORKER_CLASS=uvicorn.workers.UvicornH11Worker

WORKDIR /pygeoapi

# Fail the build rather than the container: our fork must be the
# pygeoapi installed, and the compiled packages must load against
# the runtime libraries (gdal_array also loads NumPy).
RUN python3 -c "import sys, pygeoapi; \
    sys.exit(0 if '+dibk' in pygeoapi.__version__ else \
    'unexpected pygeoapi version: ' + pygeoapi.__version__)" \
    && python3 -c "from osgeo import gdal, gdal_array, ogr, osr; import psycopg2, shapely, pyproj, starlette, uvicorn.workers"

ENTRYPOINT ["/entrypoint.sh"]
