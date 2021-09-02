ARG ENABLE_PROXY=false
ARG APISIX_VERSION=2.9
ARG ETCD_VERSION=v3.5.0

# Build Apache APISIX
FROM centos:8.3.2011
# FROM openresty/openresty:1.19.3.1-alpine-fat AS production-stage

ARG APISIX_VERSION
LABEL apisix_version="${APISIX_VERSION}"

ARG ENABLE_PROXY

RUN set -ex \
    && yum -y install yum-utils\
	&& yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo \
	&& yum install -y pcre openresty which tzdata wget\
	&& yum install -y https://github.com/apache/apisix/releases/download/${APISIX_VERSION}/apisix-${APISIX_VERSION}-0.el7.x86_64.rpm \
	&& yum clean all \
	&& sed -i 's/PASS_MAX_DAYS\t99999/PASS_MAX_DAYS\t60/g' /etc/login.defs


# we are moving to centos, thus alpine related stuff not needed
# RUN set -x \
#     && (test "${ENABLE_PROXY}" != "true" || /bin/sed -i 's,http://dl-cdn.alpinelinux.org,https://mirrors.aliyun.com,g' /etc/apk/repositories) \
#     && apk add --no-cache --virtual .builddeps \
#     automake \
#     autoconf \
#     libtool \
#     pkgconfig \
#     cmake \
#     git \
#     pcre \
#     pcre-dev \
#     && mkdir ~/.luarocks \
#     && luarocks config variables.OPENSSL_LIBDIR /usr/local/openresty/openssl/lib \
#     && luarocks config variables.OPENSSL_INCDIR /usr/local/openresty/openssl/include \
#     && luarocks install https://github.com/apache/apisix/raw/master/rockspec/apisix-${APISIX_VERSION}-0.rockspec --tree=/usr/local/apisix/deps \
#     && cp -v /usr/local/apisix/deps/lib/luarocks/rocks-5.1/apisix/${APISIX_VERSION}-0/bin/apisix /usr/bin/ \
#     && (if [ "$APISIX_VERSION" = "master" ] || [ "$APISIX_VERSION" \> "2.2" ]; then echo 'use shell ';else bin='#! /usr/local/openresty/luajit/bin/luajit\npackage.path = "/usr/local/apisix/?.lua;" .. package.path'; sed -i "1s@.*@$bin@" /usr/bin/apisix ; fi;) \
#     && mv /usr/local/apisix/deps/share/lua/5.1/apisix /usr/local/apisix \
#     && apk del .builddeps build-base make unzip

# Build etcd
FROM centos:8.3.2011 AS etcd-stage

ARG ETCD_VERSION
LABEL etcd_version="${ETCD_VERSION}"

WORKDIR /tmp
RUN set -eux \
    && yum -y install yum-utils \
    && yum install -y wget unzip \
    && yum clean all 

# https://github.com/etcd-io/etcd/releases/download/v3.4.14/etcd-v3.4.14-linux-amd64.tar.gz

RUN wget -q https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz \
    && tar -xvzf etcd-${ETCD_VERSION}-linux-amd64.tar.gz --no-same-owner \
    && ln -s etcd-${ETCD_VERSION}-linux-amd64 etcd

# Finally combine all the resources into one image
FROM centos:8.3.2011 AS last-stage


#https://github.com/apache/apisix/releases/download/2.9/apisix-2.9-0.el7.x86_64.rpm
ARG ENABLE_PROXY
ARG APISIX_VERSION
# add runtime for Apache APISIX
RUN set -eux \
    && yum -y install yum-utils \
    && yum install -y bash libstdc++ curl \
	&& yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo \
	&& yum install -y pcre openresty which tzdata wget \
	&& yum install -y https://github.com/apache/apisix/releases/download/${APISIX_VERSION}/apisix-${APISIX_VERSION}-0.el7.x86_64.rpm \
    && yum clean all



WORKDIR /usr/local/apisix

# COPY --from=production-stage /usr/local/openresty/ /usr/local/openresty/
# COPY --from=production-stage /usr/local/apisix/ /usr/local/apisix/
# COPY --from=production-stage /usr/bin/apisix /usr/bin/apisix

COPY --from=etcd-stage /tmp/etcd/etcd /usr/bin/etcd
COPY --from=etcd-stage /tmp/etcd/etcdctl /usr/bin/etcdctl

ENV PATH=$PATH:/usr/local/openresty/luajit/bin:/usr/local/openresty/nginx/sbin:/usr/local/openresty/bin

EXPOSE 9080 9443 2379 2380

CMD ["sh", "-c", "(nohup etcd >/tmp/etcd.log 2>&1 &) && sleep 10 && /usr/bin/apisix init && /usr/bin/apisix init_etcd && /usr/local/openresty/bin/openresty -p /usr/local/apisix -g 'daemon off;'"]

STOPSIGNAL SIGQUIT
