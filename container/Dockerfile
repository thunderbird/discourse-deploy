# Image build for Discourse on EKS.
#
# Discourse upstream does not publish a runtime image; the supported way
# to build one is `./launcher bootstrap <app>` against an app.yml in
# discourse_docker. We do that inside this multi-stage build and copy
# the resulting filesystem out as our runtime image.
#
# This Dockerfile is meant to be built by the GitHub Actions workflow
# at .github/workflows/build.yml -- which runs ./launcher bootstrap
# in a privileged runner, then exports the produced container as a
# tarball that we COPY here. (Bootstrap requires Docker-in-Docker and
# can't run inside a normal multi-stage build.)
#
# If you're rebuilding locally:
#   git clone https://github.com/discourse/discourse_docker
#   cp containers/app.yml discourse_docker/containers/app.yml
#   cd discourse_docker && sudo ./launcher bootstrap app
#   sudo ./launcher cleanup
#   docker save local_discourse/app -o /tmp/discourse.tar
#   docker import /tmp/discourse.tar 668807881758.dkr.ecr.eu-central-1.amazonaws.com/discourse:<tag>
#   docker push 668807881758.dkr.ecr.eu-central-1.amazonaws.com/discourse:<tag>
#
# This Dockerfile is a placeholder that documents the expected layout
# for when we move the bootstrap into a single buildx step (requires
# rootful + privileged buildkit -- not yet wired up).

FROM scratch
LABEL org.opencontainers.image.source="https://github.com/thunderbird/discourse-deploy"
LABEL org.opencontainers.image.description="Discourse forum image, built via discourse_docker launcher with discourse-openid-connect and discourse-prometheus plugins."
LABEL org.opencontainers.image.licenses="GPL-2.0-or-later"
