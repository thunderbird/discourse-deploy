# syntax=docker/dockerfile:1.7
#
# Custom Discourse image for mzla-eks-workloads01.
#
# We don't use discourse_docker's `launcher bootstrap` because that
# pipeline wants to run db:migrate against a localhost Postgres at build
# time. Our data tier (RDS Postgres, ElastiCache Redis) is external, so
# we build the image with no DB at all and let a Kubernetes Job run
# migrations against RDS at deploy time.
#
# Base: discourse/base:release ships Ruby 3.4, libpq, ImageMagick,
# ffmpeg, and the rest of Discourse's system deps. It does NOT contain
# the Discourse source -- we clone it here.
#
# Built for linux/arm64 only (workloads01 default node groups are Graviton).

ARG DISCOURSE_VERSION=v3.4.4
FROM discourse/base:release

ARG DISCOURSE_VERSION

ENV RAILS_ENV=production \
    DISCOURSE_VERSION=${DISCOURSE_VERSION} \
    SKIP_DB_AND_REDIS_PLUGIN_BUILD=1

USER root
WORKDIR /var/www

# discourse/base:release ships /var/www/discourse pre-populated with
# placeholder dirs/symlinks for the launcher's expected layout. Wipe and
# replace with our cloned source at the pinned version. --depth 1 keeps
# the layer small.
RUN rm -rf /var/www/discourse \
 && git clone --depth 1 --branch ${DISCOURSE_VERSION} \
        https://github.com/discourse/discourse.git \
        /var/www/discourse \
 && chown -R discourse:www-data /var/www/discourse

WORKDIR /var/www/discourse

# Match discourse_docker's runtime layout for /shared so log + upload
# paths are predictable. /shared lives in the container at build time
# and is *not* a volume -- uploads go to S3 via DISCOURSE_USE_S3.
RUN mkdir -p /shared/log/rails \
             /shared/uploads \
             /shared/backups \
             /shared/tmp/backups \
             /shared/tmp/restores \
 && touch /shared/log/rails/production.log \
          /shared/log/rails/production_errors.log \
          /shared/log/rails/unicorn.stdout.log \
          /shared/log/rails/unicorn.stderr.log \
          /shared/log/rails/sidekiq.log \
 && chown -R discourse:www-data /shared \
 && ln -sf /shared/uploads public/uploads \
 && ln -sf /shared/backups public/backups \
 && for f in production production_errors unicorn.stdout unicorn.stderr sidekiq; do \
        ln -sf /shared/log/rails/$f.log log/$f.log; \
    done

USER discourse

# Install gems (production + assets, no test/development).
RUN bundle config set --local deployment 'true' \
 && bundle config set --local without 'development test' \
 && bundle install --jobs $(nproc) --retry 3

# Precompile assets.
#
# Two flags do the heavy lifting at build time:
#
# - SKIP_DB_AND_REDIS_PLUGIN_BUILD=1 (set in ENV above) -- skips plugin
#   code paths that would otherwise need a live DB/Redis.
# - DISCOURSE_SKIP_REDIS=1 -- makes GlobalSetting.skip_redis? return true.
#   secret_token initializer calls safe_secret_key_base, which falls
#   back to Redis to read SECRET_TOKEN if DISCOURSE_SECRET_KEY_BASE is
#   blank OR doesn't match VALID_SECRET_KEY (=/\A[0-9a-f]{128}\z/).
#   With skip_redis=1 it short-circuits to SecureRandom.hex(64) instead
#   -- no Redis connection attempted.
#   Source: app/models/global_setting.rb safe_secret_key_base in v3.4.4.
# - DB_HOST/USERNAME/PASSWORD inlined as harmless placeholders too, in
#   case any other initializer touches those configs (no connection is
#   attempted at this stage).
#
# Runtime overrides every value here from the discourse-config ConfigMap
# and discourse-{db,redis,app}-credentials Secrets.
RUN DISCOURSE_SKIP_REDIS=1 \
    DISCOURSE_DB_HOST=localhost \
    DISCOURSE_DB_USERNAME=discourse \
    DISCOURSE_DB_PASSWORD=placeholder \
    DISCOURSE_REDIS_HOST=localhost \
    bundle exec rake assets:precompile

USER root
# Strip apt + tmp cruft to keep the layer lean.
RUN rm -rf /var/cache/apt/archives/*.deb \
           /tmp/* \
           /var/tmp/* \
           /root/.cache

EXPOSE 3000

USER discourse
WORKDIR /var/www/discourse

# Default command: Puma (web). discourse-sidekiq Deployment overrides
# this with `bundle exec sidekiq ...` to run as the worker process.
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
