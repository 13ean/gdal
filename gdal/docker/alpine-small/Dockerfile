##
# osgeo/gdal:alpine-small

# This file is available at the option of the licensee under:
# Public domain
# or licensed under X/MIT (LICENSE.TXT) Copyright 2019 Even Rouault <even.rouault@spatialys.com>

FROM alpine:latest as builder

# Derived from osgeo/proj by Howard Butler <howard@hobu.co>
MAINTAINER Even Rouault <even.rouault@spatialys.com>

# Setup build env for PROJ
RUN apk add --no-cache wget curl unzip make libtool autoconf automake pkgconfig g++ sqlite sqlite-dev

# For GDAL
RUN apk add --no-cache \
    linux-headers \
    curl-dev \
    zlib-dev zstd-dev \
    libjpeg-turbo-dev libpng-dev openjpeg-dev libwebp-dev expat-dev postgresql-dev

ARG PROJ_VERSION=master

# Build PROJ
RUN wget -q https://github.com/OSGeo/proj.4/archive/${PROJ_VERSION}.zip -O proj.4-${PROJ_VERSION}.zip \
    && unzip -q proj.4-${PROJ_VERSION}.zip \
    && rm -f proj.4-${PROJ_VERSION}.zip \
    && cd proj.4-${PROJ_VERSION} \
    && ./autogen.sh \
    && CFLAGS='-DPROJ_RENAME_SYMBOLS -O2' CXXFLAGS='-DPROJ_RENAME_SYMBOLS -O2' \
        ./configure --prefix=/usr/local --disable-static --enable-lto \
    && make -j$(nproc) \
    && make install DESTDIR="/build" \
    && cd .. \
    && rm -rf proj.4-${PROJ_VERSION} \
    && mv /build/usr/local/lib/libproj.so.15.0.0 /build/usr/local/lib/libinternalproj.so.15.0.0 \
    && ln -s libinternalproj.so.15.0.0 /build/usr/local/lib/libinternalproj.so.15 \
    && ln -s libinternalproj.so.15.0.0 /build/usr/local/lib/libinternalproj.so \
    && rm /build/usr/local/lib/libproj.*  \
    && ln -s libinternalproj.so.15.0.0 /build/usr/local/lib/libproj.so.15 \
    && strip -s /build/usr/local/lib/libinternalproj.so.15.0.0 \
    && for i in /build/usr/local/bin/*; do strip -s $i 2>/dev/null || /bin/true; done

ARG GDAL_VERSION=master

# Build GDAL
RUN wget -q https://github.com/OSGeo/gdal/archive/${GDAL_VERSION}.zip -O gdal-${GDAL_VERSION}.zip \
    && unzip -q gdal-${GDAL_VERSION}.zip \
    && rm -f gdal-${GDAL_VERSION}.zip \
    && cd gdal-${GDAL_VERSION}/gdal \
    && ./configure --prefix=/usr --without-libtool \
    --with-hide-internal-symbols \
    --with-proj=/build/usr/local \
    --with-libtiff=internal --with-rename-internal-libtiff-symbols \
    --with-geotiff=internal --with-rename-internal-libgeotiff-symbols \
    --enable-lto \
    && make -j$(nproc) \
    && make install DESTDIR="/build" \
    && cd ../.. \
    && rm -rf gdal-${GDAL_VERSION} \
    && strip -s /build/usr/lib/libgdal.so.2.5.0 \
    && for i in /build/usr/bin/*; do strip -s $i 2>/dev/null || /bin/true; done \
    # Remove resource files of uncompiled drivers
    && (for i in \
            # BAG driver
            /build/usr/share/gdal/bag*.xml \
            # unused
            /build/usr/share/gdal/*.svg \
            # unused
            /build/usr/share/gdal/*.png \
            # GMLAS driver
            /build/usr/share/gdal/gmlas* \
            # netCDF driver
            /build/usr/share/gdal/netcdf_config.xsd \
       ;do rm $i; done)

RUN \
    curl -LOs http://download.osgeo.org/proj/proj-datumgrid-latest.zip \
    && unzip -q -j -u -o proj-datumgrid-latest.zip  -d /build/usr/local/share/proj \
    && rm -f *.zip

# Build final image
FROM alpine:latest as runner

RUN date

RUN apk add --no-cache \
        libstdc++ \
        sqlite-libs \
        libcurl \
        zlib zstd-libs\
        libjpeg-turbo libpng openjpeg libwebp expat libpq \
    # libturbojpeg.so is not used by GDAL. Only libjpeg.so*
    && rm -f /usr/lib/libturbojpeg.so* \
    # Only libwebp.so is used by GDAL
    && rm -f /usr/lib/libwebpmux.so* /usr/lib/libwebpdemux.so* /usr/lib/libwebpdecoder.so*

COPY --from=builder  /build/usr/local/bin/ /usr/local/bin/
COPY --from=builder  /build/usr/local/lib/ /usr/local/lib/
COPY --from=builder  /build/usr/local/include/ /usr/local/include/
COPY --from=builder  /build/usr/local/share/proj/ /usr/local/share/proj/

COPY --from=builder  /build/usr/bin/ /usr/bin/
COPY --from=builder  /build/usr/lib/ /usr/lib/
COPY --from=builder  /build/usr/include/ /usr/include/
COPY --from=builder  /build/usr/share/gdal/ /usr/share/gdal/
