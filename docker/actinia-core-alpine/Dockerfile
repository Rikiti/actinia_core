FROM mundialis/grass-py3-pdal:stable-alpine as grass
FROM mundialis/esa-snap:7.0 as snap
FROM mundialis/actinia-core:alpine-build-pkgs as build

LABEL authors="Carmen Tawalika,Anika Bettge,Markus Neteler,Sören Gebbert"
LABEL maintainer="tawalika@mundialis.de,bettge@mundialis.de,neteler@mundialis.de,soerengebbert@gmail.com"

COPY . /src/actinia_core

WORKDIR /src/actinia_core
RUN git checkout .
COPY requirements-alpine.txt requirements-alpine.txt
# scipy installs but wheel building fails. Helping packages were
# RUN add openblas openblas-dev lapack lapack-dev gfortran
RUN sed -i 's+scikit-learn+#scikit-learn+g' requirements-alpine.txt
RUN python3 setup.py sdist bdist_wheel -d /build
# RUN pip3 wheel -r requirements-alpine.txt -w /build
# RUN python3 -m pep517.build --out-dir /build . && \


FROM mundialis/actinia-core:alpine-runtime-pkgs as actinia

ENV LC_ALL "en_US.UTF-8"
ENV GDAL_CACHEMAX=2000
ENV GRASS_COMPRESSOR=ZSTD
ENV GRASS_SKIP_MAPSET_OWNER_CHECK 1

USER root

# ESA SNAP SETUP
ENV LD_LIBRARY_PATH ".:$LD_LIBRARY_PATH"
COPY --from=snap /root/.snap /root/.snap
COPY --from=snap /usr/local/snap /usr/local/snap
RUN (cd /root/.snap/snap-python/snappy && python3 setup.py install)
RUN /usr/bin/python3 -c 'from snappy import ProductIO'
RUN /usr/bin/python3 /root/.snap/about.py

# GRASS GIS SETUP
COPY --from=grass /usr/local/bin/grass /usr/local/bin/grass
COPY --from=grass /usr/local/grass* /usr/local/grass/
RUN pip3 install --upgrade pip six grass-session
RUN ln -s /usr/local/grass /usr/local/grass7
RUN ln -s /usr/local/grass `grass --config path`
RUN grass --tmp-location EPSG:4326 --exec g.version -rge && \
    pdal --version && \
    python3 --version

# GRASS GIS addons BUILD
RUN git clone https://github.com/mundialis/openeo-addons.git /src/openeo-addons
COPY docker/grass_addons_list.csv /src/grass_addons_list.csv
# TODO: remove patch when python 3.8.2 is here
RUN apk add curl
RUN curl -L https://github.com/mmacata/alpine-python381-patch/releases/download/0.0.0/python3-3.8.1-r1.apk > /src/python3-3.8.1-r1.apk
RUN apk add --allow-untrusted /src/python3-3.8.1-r1.apk

RUN while IFS=, read -r ADDON SERVER; do unset URL; test -z $SERVER || URL="url=$SERVER"; grass --tmp-location EPSG:4326 --exec g.extension -s extension=$ADDON $URL;done < /src/grass_addons_list.csv

# actinia-core and plugins BUILD
# Currently, ADD always breaks cache, as "download_count" increases every time
# ADD https://api.github.com/repos/mundialis/actinia_statistic_plugin/releases/latest /scratch/actinia_statistic_plugin_latest_release.json
WORKDIR /build
# Get statistics plugin (22648768 = 0.0.1)
RUN curl https://api.github.com/repos/mundialis/actinia_statistic_plugin/releases/22648768 > resp.json && \
    name=`cat resp.json | jq '.assets[0].name' | tr -d '"'` && \
    cat resp.json | jq '.assets[0].browser_download_url' | xargs curl -L --output /build/$name && rm resp.json
# Get satellite plugin (22686533 = 0.0.1)
RUN curl https://api.github.com/repos/mundialis/actinia_satellite_plugin/releases/22686533 > resp.json && \
    name=`cat resp.json | jq '.assets[0].name' | tr -d '"'` && \
    cat resp.json | jq '.assets[0].browser_download_url' | xargs curl -L --output /build/$name && rm resp.json
# Get actinia-gdi plugin (23445806 = 0.1.5)
RUN curl https://api.github.com/repos/mundialis/actinia-gdi/releases/23445806 > resp.json && \
    name=`cat resp.json | jq '.assets[0].name' | tr -d '"'` && \
    cat resp.json | jq '.assets[0].browser_download_url' | xargs curl -L --output /build/$name && rm resp.json
# Install actinia-core and plugins
COPY --from=build /build/*.whl /build/
RUN apk add curl && curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && python3 get-pip.py pip==20.0.2 && rm get-pip.py
RUN pip3 install /build/*
# See in build stage why we need this here
RUN pip3 install scikit-learn

# Duplicate install actinia_core requirements. They are already wheels in
# /build folder, keep to check match of required packages
COPY requirements-alpine.txt /src/requirements-alpine.txt
RUN pip3 install -r /src/requirements-alpine.txt

# Copy actinia config file and start scripts + set needed envs
COPY docker/actinia-core-alpine/actinia.cfg /etc/default/actinia
COPY docker/actinia-core-alpine/start.sh /src/start.sh
COPY docker/actinia-core-alpine/start-dev.sh /src/start-dev.sh
ENV ACTINIA_CUSTOM_TEST_CFG /etc/default/actinia
ENV GISBASE ""

# Create the database directories
RUN mkdir -p /actinia_core/grassdb && \
    mkdir -p /actinia_core/resources && \
    mkdir -p /actinia_core/workspace/tmp && \
    mkdir -p /actinia_core/workspace/temp_db && \
    mkdir -p /actinia_core/workspace/actinia && \
    mkdir -p /actinia_core/workspace/download_cache && \
    mkdir -p /actinia_core/userdata && \
    ln -s /actinia_core /root/actinia

VOLUME /grassdb
WORKDIR /src/actinia_core

ENTRYPOINT ["/bin/sh"]
CMD ["/src/start.sh"]

EXPOSE 8088
EXPOSE 9191
