FROM debian:bookworm-slim

# explicitly set user/group IDs
RUN set -eux; \
	groupadd -r postgres --gid=999; \
# https://salsa.debian.org/postgresql/postgresql-common/blob/997d842ee744687d99a2b2d95c1083a2615c79e8/debian/postgresql-common.postinst#L32-35
	useradd -r -g postgres --uid=999 --home-dir=/var/lib/postgresql --shell=/bin/bash postgres; \
# also create the postgres user's home directory with appropriate permissions
# see https://github.com/docker-library/postgres/issues/274
	install --verbose --directory --owner postgres --group postgres --mode 1777 /var/lib/postgresql


# make the "en_US.UTF-8" locale so postgres will be utf-8 enabled by default
RUN set -eux; \
	if [ -f /etc/dpkg/dpkg.cfg.d/docker ]; then \
# if this file exists, we're likely in "debian:xxx-slim", and locales are thus being excluded so we need to remove that exclusion (since we need locales)
		grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
		sed -ri '/\/usr\/share\/locale/d' /etc/dpkg/dpkg.cfg.d/docker; \
		! grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
	fi; \
	apt-get update; apt-get install -y --no-install-recommends locales; rm -rf /var/lib/apt/lists/*; \
	echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen; \
	locale-gen; \
	locale -a | grep 'en_US.utf8'
ENV LANG=en_US.utf8


RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libnss-wrapper \
		xz-utils \
		zstd make bison flex g++ clang libreadline-dev zlib1g-dev perl \
	;

ADD replan-stmt /usr/src
WORKDIR "/usr/src/"
RUN ls
RUN ./configure CFLAGS="-O3" --prefix=/usr/pgbin --without-icu
RUN make
RUN make -C contrib
RUN make install
RUN make -C contrib install

RUN rm -rf /usr/src

# Envoronment Viarables ########################################################
ENV PGDATA=/usr/pgdata
ENV PGUSER=postgres
ENV PGDATABASE=postgres
ENV PGPORT=5432
ENV LD_LIBRARY_PATH=/usr/pgbin/lib
ENV PATH=/usr/pgbin/bin:/usr/local/bin/:$PATH
ENV PGHOST=localhost
ENV PG_CONFIG=/usr/pgbin/bin/pg_config/
# ##############################################################################

RUN mkdir $PGDATA
RUN chown postgres:postgres $PGDATA
USER postgres
RUN initdb

RUN echo "\n\
  compute_query_id = 'on' \n\
  checkpoint_timeout = 86399 \n\
  fsync = 'off' \n\
  from_collapse_limit = 20 \n\
  join_collapse_limit = 20 \n\
  max_parallel_workers_per_gather = 0 \n\
  min_parallel_table_scan_size = 0 \n\
  min_parallel_index_scan_size = 0 \n\
  work_mem='32MB' \n\
  replan_enable = 'on' \n\
  query_inadequate_execution_time = 1000 \n\
  statement_timeout = 3600000 \n\
  show_node_sign = 'on' \n\
  shared_preload_libraries = 'pg_prewarm' \n\
  pg_prewarm.autoprewarm = 'on' \n\
  pg_prewarm.autoprewarm_interval = 0 \n\
  " >> $PGDATA/postgresql.conf

#RUN pg_ctl -D $PGDATA start

#COPY docker-entrypoint.sh /usr/local/bin/
#ENTRYPOINT ["docker-entrypoint.sh"]

STOPSIGNAL SIGINT
EXPOSE 5432
CMD ["postgres"]
