# This is slightly adjusted Dockerfile from
# https://github.com/docker-library/postgres

# set ALPINE_VERSION= [ edge 3.21 3.20 3.19 3.18 ]
ARG ALPINE_VERSION=3.21
FROM alpine:${ALPINE_VERSION}

ARG ALPINE_VERSION

# Set PG_MAJOR = [ 17 16 ]
ARG PG_MAJOR=17
ENV PG_MAJOR=${PG_MAJOR}

# set compiler: [ clang gcc ]
ARG BUILD_CC_COMPILER=clang
ENV BUILD_CC_COMPILER=${BUILD_CC_COMPILER}

# Enable debug mode and preserve the build environments for debugging.
# In this case, each image size exceeds 1GB
ARG DEBUG_MODE=false
ENV DEBUG_MODE=${DEBUG_MODE}

# Define build dependencies for LLVM [ llvm-dev clang ]
# These include the specific versions of 'llvm-dev' and 'clang' suitable for the current version of PostgreSQL.
# They are useful for building downstream extensions using the same LLVM, like PostGIS alpine https://github.com/postgis/docker-postgis
# Note: Some older PostgreSQL version does not support LLVM 16. Therefore, for Alpine >=3.18, please use "llvm15-dev clang15".
# Reference: https://github.com/docker-library/postgres/pull/1077
ARG DOCKER_PG_LLVM_DEPS="llvm-dev clang"
ENV DOCKER_PG_LLVM_DEPS=${DOCKER_PG_LLVM_DEPS}

# 70 is the standard uid/gid for "postgres" in Alpine
# https://git.alpinelinux.org/aports/tree/main/postgresql/postgresql.pre-install?h=3.12-stable
RUN set -eux; \
	addgroup -g 70 -S postgres; \
	adduser -u 70 -S -D -G postgres -H -h /var/lib/postgresql -s /bin/sh postgres; \
	mkdir -p /var/lib/postgresql; \
	chown -R postgres:postgres /var/lib/postgresql

# su-exec (gosu-compatible) is installed further down

# make the "en_US.UTF-8" locale so postgres will be utf-8 enabled by default
# alpine doesn't require explicit locale-file generation
ENV LANG=en_US.utf8

RUN mkdir -p /usr/src/postgresql/contrib/orioledb

COPY . /usr/src/postgresql/contrib/orioledb

RUN mkdir /docker-entrypoint-initdb.d /docker-default-initdb.d

RUN mkdir -p /var/run/postgresql && chown -R postgres:postgres /var/run/postgresql && chmod 2777 /var/run/postgresql

RUN set -eux; \
	\
	PGTAG=$(grep "^$PG_MAJOR: " /usr/src/postgresql/contrib/orioledb/.pgtags | cut -d' ' -f2-) ; \
	ORIOLEDB_VERSION=$(grep "^#define ORIOLEDB_VERSION" /usr/src/postgresql/contrib/orioledb/include/orioledb.h | cut -d'"' -f2) ; \
	ORIOLEDB_BUILDTIME=$(date -Iseconds) ; \
	ALPINE_VERSION=$(cat /etc/os-release | grep VERSION_ID | cut -d = -f 2 | cut -d . -f 1,2 | cut -d _ -f 1) ; \
	\
	# To get support for all locales: IF >=Alpine3.16 THEN install icu-data-full
	# https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.16.0#ICU_data_split
	# https://github.com/docker-library/postgres/issues/327#issuecomment-1201582069
	case "$ALPINE_VERSION" in 3.13 | 3.14 | 3.15 )  EXTRA_ICU_PACKAGES='' ;; \
		3.16 | 3.17 | 3.18 | 3.19 | 3.20 | 3.21* )  EXTRA_ICU_PACKAGES=icu-data-full ;; \
		*) : ;; \
	esac ; \
	\
	echo "PG_MAJOR=$PG_MAJOR" ; \
	echo "PGTAG=$PGTAG" ; \
	echo "BUILD_CC_COMPILER=$BUILD_CC_COMPILER" ; \
	echo "ORIOLEDB_VERSION=$ORIOLEDB_VERSION" ; \
	echo "ORIOLEDB_BUILDTIME=$ORIOLEDB_BUILDTIME" ; \
	echo "ALPINE_VERSION=$ALPINE_VERSION" ; \
	echo "EXTRA_ICU_PACKAGES=$EXTRA_ICU_PACKAGES" ; \
	echo "DOCKER_PG_LLVM_DEPS=$DOCKER_PG_LLVM_DEPS" ; \
	echo "DEBUG_MODE=$DEBUG_MODE" ; \
	\
	# check if the custom llvm version is set, and if so, set the LLVM_CONFIG and CLANG variables
	CUSTOM_LLVM_VERSION=$(echo "$DOCKER_PG_LLVM_DEPS" | sed -n 's/.*llvm\([0-9]*\).*/\1/p') ; \
	if [ ! -z "${CUSTOM_LLVM_VERSION}" ];  then \
		echo "CUSTOM_LLVM_VERSION=$CUSTOM_LLVM_VERSION" ; \
		export LLVM_CONFIG="/usr/lib/llvm${CUSTOM_LLVM_VERSION}/bin/llvm-config" ; \
		export CLANG=clang-${CUSTOM_LLVM_VERSION} ; \
		if [[ "${BUILD_CC_COMPILER}" == "clang" ]]; then \
			export BUILD_CC_COMPILER=clang-${CUSTOM_LLVM_VERSION} ; \
			echo "fix: BUILD_CC_COMPILER=clang-${CUSTOM_LLVM_VERSION}" ; \
		fi ; \
	fi ; \
	\
	apk add --no-cache --virtual .build-deps \
		${DOCKER_PG_LLVM_DEPS} \
		bison \
		coreutils \
		curl \
		dpkg-dev dpkg \
		flex \
		g++ \
		gcc \
		krb5-dev \
		libc-dev \
		libedit-dev \
		libxml2-dev \
		libxslt-dev \
		linux-headers \
# needed for s3 support
		curl-dev \
		make \
		openldap-dev \
		openssl-dev \
# configure: error: prove not found
		perl-utils \
# configure: error: Perl module IPC::Run is required to run TAP tests
		perl-ipc-run \
		perl-dev \
		python3 \
		python3-dev \
		tcl-dev \
		util-linux-dev \
		zlib-dev \
		zstd-dev \
# https://www.postgresql.org/docs/10/static/release-10.html#id-1.11.6.9.5.13
		icu-dev \
# https://www.postgresql.org/docs/14/release-14.html#id-1.11.6.5.5.3.7
		lz4-dev \
	; \
	\
	apk add --no-cache \
		bash \
		su-exec \
		# tzdata is optional, but only adds around 1Mb to image size and is recommended by Django documentation:
		# https://docs.djangoproject.com/en/1.10/ref/databases/#optimizing-postgresql-s-configuration
		tzdata \
		# install extra icu packages ( >=Alpine3.16 )
		$EXTRA_ICU_PACKAGES \
	; \
	\
	curl -o postgresql.tar.gz \
			--header "Accept: application/vnd.github.v3.raw" \
			--remote-name \
			--location https://github.com/orioledb/postgres/tarball/$PGTAG; \
	mkdir -p /usr/src/postgresql; \
	tar \
		--extract \
		--file postgresql.tar.gz \
		--directory /usr/src/postgresql \
		--strip-components 1 \
	; \
	rm postgresql.tar.gz; \
	\
	cd /usr/src/postgresql; \
	\
	POSTGRESQL_VERSION=$(grep "PACKAGE_VERSION=" ./configure | cut -d"'" -f2) ; \
	echo "POSTGRESQL_VERSION=$POSTGRESQL_VERSION" ; \
	\
# update "DEFAULT_PGSOCKET_DIR" to "/var/run/postgresql" (matching Debian)
# see https://anonscm.debian.org/git/pkg-postgresql/postgresql.git/tree/debian/patches/51-default-sockets-in-var.patch?id=8b539fcb3e093a521c095e70bdfa76887217b89f
	awk '$1 == "#define" && $2 == "DEFAULT_PGSOCKET_DIR" && $3 == "\"/tmp\"" { $3 = "\"/var/run/postgresql\""; print; next } { print }' src/include/pg_config_manual.h > src/include/pg_config_manual.h.new; \
	grep '/var/run/postgresql' src/include/pg_config_manual.h.new; \
	mv src/include/pg_config_manual.h.new src/include/pg_config_manual.h; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
# explicitly update autoconf config.guess and config.sub so they support more arches/libcs
	cp /usr/src/postgresql/contrib/orioledb/docker/config.guess config/config.guess; \
	cp /usr/src/postgresql/contrib/orioledb/docker/config.sub config/config.sub; \
# configure options taken from:
# https://anonscm.debian.org/cgit/pkg-postgresql/postgresql.git/tree/debian/rules?h=9.5
	( CC=${BUILD_CC_COMPILER} ./configure \
		--build="$gnuArch" \
# "/usr/src/postgresql/src/backend/access/common/tupconvert.c:105: undefined reference to `libintl_gettext'"
#		--enable-nls \
		--enable-integer-datetimes \
		--enable-thread-safety \
		--enable-tap-tests \
# skip debugging info -- we want tiny size instead
#		--enable-debug \
		--disable-rpath \
		--with-uuid=e2fs \
		--with-gnu-ld \
		--with-pgport=5432 \
		--with-system-tzdata=/usr/share/zoneinfo \
		--prefix=/usr/local \
		--with-includes=/usr/local/include \
		--with-libraries=/usr/local/lib \
		--with-gssapi \
		--with-ldap \
		--with-tcl \
		--with-perl \
		--with-python \
#		--with-pam \
		--with-openssl \
		--with-libxml \
		--with-libxslt \
		--with-icu \
		--with-llvm \
		--with-lz4 \
		--with-zstd \
		# The "testgres" package expects the PostgreSQL version as the last word.
		# Therefore, the extra ${POSTGRESQL_VERSION} is added as a workaround.
		--with-extra-version=" ${ORIOLEDB_VERSION} PGTAG=${PGTAG} alpine:${ALPINE_VERSION}+${BUILD_CC_COMPILER} build:${ORIOLEDB_BUILDTIME} ${POSTGRESQL_VERSION}" \
	|| cat config.log ); \
	echo "ORIOLEDB_PATCHSET_VERSION = `echo $PGTAG | cut -d'_' -f2`" >> src/Makefile.global;

RUN set -eux; \
	cd /usr/src/postgresql; \
	\
	# install postgresql
	make -j "$(nproc)"; \
	make install;

# REMOVE_WHEN_CACHING BEGIN
RUN set -eux; \
	cd /usr/src/postgresql; \
	\
	make -C contrib -j "$(nproc)"; \
	make -C contrib install;

RUN set -eux; \
	# install orioledb extension
	cd /usr/src/postgresql/contrib/orioledb; \
	make USE_PGXS=1 IS_DEV=1 -j "$(nproc)"; \
	make USE_PGXS=1 IS_DEV=1 install;

RUN set -eux; \
	# Verify PostgreSQL installation
	cd / ; \
	postgres --version ; \
	initdb --version

RUN set -eux; \
	# Clean up only if not in debug mode
	if [ "$DEBUG_MODE" != "true" ]; then \
		runDeps="$( \
			scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
				| tr ',' '\n' \
				| sort -u \
				| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
				# Remove plperl, plpython and pltcl dependencies by default to save image size
				# To use the pl extensions, those have to be installed in a derived image
				| grep -v -e perl -e python -e tcl \
		)"; \
		apk add --no-cache --virtual .postgresql-rundeps \
			$runDeps \
		; \
		apk del --no-network .build-deps; \
		rm -rf \
			/usr/src/postgresql \
			/usr/local/share/doc \
			/usr/local/share/man \
			/tmp/* \
		; \
	fi ;

ENV PGDATA=/var/lib/postgresql/data
# this 777 will be replaced by 700 at runtime (allows semi-arbitrary "--user" values)
RUN mkdir -p "$PGDATA" && chown -R postgres:postgres "$PGDATA" && chmod 777 "$PGDATA"
VOLUME /var/lib/postgresql/data

RUN mkdir -p /etc/postgresql && chown -R postgres:postgres /etc/postgresql && chmod 700 /etc/postgresql
COPY --chown=postgres:postgres docker/init/postgresql.docker.conf /etc/postgresql/postgresql.conf
ENV PG_CONF=/etc/postgresql/postgresql.conf

ENV POSTGRES_INITDB_ARGS="--locale-provider=icu --icu-locale=en"
# REMOVE_WHEN_CACHING END

COPY docker/init/docker-entrypoint.sh /usr/local/bin/
# REMOVE_WHEN_CACHING BEGIN
COPY docker/init/default-orioledb.sh /docker-default-initdb.d/
# REMOVE_WHEN_CACHING END
ENTRYPOINT ["docker-entrypoint.sh"]

# We set the default STOPSIGNAL to SIGINT, which corresponds to what PostgreSQL
# calls "Fast Shutdown mode" wherein new connections are disallowed and any
# in-progress transactions are aborted, allowing PostgreSQL to stop cleanly and
# flush tables to disk, which is the best compromise available to avoid data
# corruption.
#
# Users who know their applications do not keep open long-lived idle connections
# may way to use a value of SIGTERM instead, which corresponds to "Smart
# Shutdown mode" in which any existing sessions are allowed to finish and the
# server stops when all sessions are terminated.
#
# See https://www.postgresql.org/docs/12/server-shutdown.html for more details
# about available PostgreSQL server shutdown signals.
#
# See also https://www.postgresql.org/docs/12/server-start.html for further
# justification of this as the default value, namely that the example (and
# shipped) systemd service files use the "Fast Shutdown mode" for service
# termination.
#
STOPSIGNAL SIGINT
#
# An additional setting that is recommended for all users regardless of this
# value is the runtime "--stop-timeout" (or your orchestrator/runtime's
# equivalent) for controlling how long to wait between sending the defined
# STOPSIGNAL and sending SIGKILL (which is likely to cause data corruption).
#
# The default in most runtimes (such as Docker) is 10 seconds, and the
# documentation at https://www.postgresql.org/docs/12/server-start.html notes
# that even 90 seconds may not be long enough in many instances.

EXPOSE 5432
CMD ["postgres", "-D", "/etc/postgresql"]
